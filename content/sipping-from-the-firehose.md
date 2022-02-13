+++
title = "Sipping from the firehose"
slug = "parsing-kinesis-firehose-json-in-python"
date = "2022-02-05"
category = "kata"

[extra]
author = "Bob Gregory"

[taxonomies]
tags = ["tdd", "basics"]
+++

Most solutions to reading Firehosed json data rely on pre-processing with a lambda, or struggle to deal with large files. A better solution is to use a streaming parser. Luckily, I have just the thing.

<!-- more -->

As part of building out our home-brewed stats system, we need to write a couple of lambdas to read the stored metrics. These metrics are written as JSON blobs to S3 by Kinesis Firehose. The individual hits are recorded as json objects, but Firehose has an interesting quirk - it writes files full of json objects concatenated together, with no delimiter.

For example, one hit in our schema looks like this

```json
{
  "u": "https://codefiend.co.uk/parsing-kinesis-firehose-json-in-python",
  "r": "www.google.com",
  "t": 1644670501,
  "v": 1900
}
```

but when we receive multiple hits in a short space of time, they're written to the same file, in a continuous stream like this:

```
{ "u": "https://codefiend.co.uk/parsing-kinesis-firehose-json-in-python", "r": "www.google.com","t": 1644670501, "v": 1900}{"u":"https://codefiend.co.uk/","r":"t.co","t":1644670500,v:500}{"u":"https://codefiend.co.uk/tackling-the-delivery-service-kata","r":"t.co","t":1644670517,v:950}
```

This breaks the python json parser, which is only built to handle a single object at a time.

## Why the other solutions suck

The three common solutions to the problem are 

1. Pre-process the data with a lambda 
2. Post-process the data with `str.split()`
3. Use [Dynamic Partioning](https://docs.aws.amazon.com/firehose/latest/dev/dynamic-partitioning.html#dynamic-partitioning-new-line-delimiter) to process the data in-flight and add a newline delimiter

Let's take them one at a time.

### Pre-process in a lambda

This is the most common solution I see. The idea is that we can attach a lambda function to the kinesis firehose so that we process the data before writing it to S3. We can use this to insert a newline character after each record so that our python can parse each line separately.

This strikes me as deeply inelegant. It means calling a lambda function every time we write a record, which can get expensive, and that lambda function is just appending one character to a base64 encoded blob.

```python
import base64

output = []

def lambda_handler(event, context):
    
    for record in event['records']:
    
        # decode the base64 encoded record
        payload = base64.b64decode(record['data']).decode('utf-8')

        # append one character
        row_w_newline = payload + "\n"

        # and encode it again
        row_w_newline = base64.b64encode(row_w_newline.encode('utf-8'))
        
        # then return the new record to kinesis for writing
        output_record = {
            'recordId': record['recordId'],
            'result': 'Ok',
            'data': row_w_newline
        }
        output.append(output_record)

    return {'records': output}
```

It _works_ but it's ungainly, and not cost effective at volume.

### Use Dynamic Partitioning

Amazon have added the ability to control the key used when writing data to S3. As part of that change, they also added an option to append a newline to each record. I suspect this is actually just a lambda under the hood, so it's really option 1, but Amazon builds it for you.

There's a small cost associated with dynamic partitioning, but this remains a _sensible_ option. If we were going to be sensible, though, we wouldn't bother building our own analytics system in the first place.

### Post-process when reading the data

Alternatively, we could split the data when we read it back. Given the string `{"a:"1, "b":2}{"a":3, "b":4}` we can run a regex or `str.split()` to give us two separate strings, and then parse those individually.

This method requires that we load the whole file into memory so that we can split it. For small files that might be okay, but some of the files in our S3 bucket might be prohibitively large. 

What we'd _like_ is to be able to stream large files, pulling out json objects as we encounter them. 

## Streaming JSON for fun and profit

There are a few streaming JSON parsers out there already. Before writing this post I did a few spikes, with [NAYA](https://github.com/danielyule/naya) and [Yajl-py](https://github.com/pykler/yajl-py). Both seemed like reasonable options, but then I stumbled across the `raw_decode` method of the `JSONDecoder` in the python standard library.

The `raw_decode` method does almost exactly what we want - it reads a string and extracts a JSON object from it, ignoring any data after the object closes. It also returns the index of the remaining data, so we can call the method in a loop.

Reading the [source code for JSONDecoder]()https://github.com/python/cpython/blob/f4c03484da59049eb62a9bf7777b963e2267d187/Lib/json/decoder.py#L343, we find that internally, it's using a `scanner`function. This is the magic we need. The scanner reads a string, and pulls a json object from the beginning. It then returns that object and the index of the string where the object finished. If there's no object found, it raises StopIteration

```python
from json import scanner, dumps
from json.decoder import JSONDecoder

import pytest

def test_scanner():
    decoder = JSONDecoder()
    
    # A scanner uses a Json Decoder to produce objects
    scan = scanner.make_scanner(decoder)

    datum = {"A": 1, "B": 2}
    data = dumps(datum) * 2

    # When we scan, we get back the first json object in the string
    result, idx = scan(data, 0)
    assert result == datum

    # We can scan a second time from the previous index
    result, idx = scan(data, idx)
    assert result == datum

    # And when we reach the end of the string, we get StopIteration
    with pytest.raises(StopIteration):
        scan(data, idx)
```

I've written a little package using this technique that can stream large files straight out of S3, and read the concatenated JSON objects.

```python
import firehose_sipper

# Read a single file out of S3

for entry in firehose_sipper.sip(bucket=some_bucket, key=some_key):
    print(entry)
    
# or go nuts and read all objects under a prefix
for entry in firehose_sipper.sip(bucket=some_bucket, prefix=some_prefix):
    print(entry)
```

Now I can leave the Firehose to its default configuration, and read the results with a one-liner.
