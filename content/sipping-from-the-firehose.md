+++
title = "Sipping from the firehose"
slug = "parsing-kinesis-firehose-json-in-python"
date = "2022-02-13"
category = "oss"

[extra]
author = "Bob Gregory"

[taxonomies]
tags = ["python", "aws", "data"]
+++

Kinesis Firehose writes concatenated JSON objects to S3. Most python solutions to reading the data rely on pre-processing with a lambda, or struggle to deal with large files. A better solution is to use a streaming parser. Luckily, I have just the thing.

<!-- more -->

As part of building out our [home-brewed stats system](@/serverless-web-analytics/index.md), we need to write a couple of lambdas to read the stored metrics. These metrics are written as JSON blobs to S3 by Kinesis Firehose. The individual hits are recorded as json objects, but Firehose has an interesting quirk - it writes files full of json objects concatenated together, with no delimiter.

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

```json
{ "u": "https://codefiend.co.uk/parsing-kinesis-firehose-json-in-python", "r": "www.google.com","t": 1644670501, "v": 1900}{"u":"https://codefiend.co.uk/","r":"t.co","t":1644670500,v:500}{"u":"https://codefiend.co.uk/tackling-the-delivery-service-kata","r":"t.co","t":1644670517,v:950}
```

This breaks the python json parser, which is only built to handle a single object at a time. I've seen some solutions around the internet that either use a lambda to add delimiters to the data[^1] before it's written to S3, or read the whole file into memory, and split it apart[^2] before processing it. What we'd _like_ is to be able to stream large files, pulling out json objects as we encounter them. I found it surprising that there wasn't much information on how to deal with Kinesis' default processing mode in Python, which is probably the most common data-enginering language in AWS.

## Streaming JSON for fun and profit

There are a few streaming JSON parsers out there already. Before writing this post I did a few spikes, with [NAYA](https://github.com/danielyule/naya)[^3] and [Yajl-py](https://github.com/pykler/yajl-py)[^4]. Both seemed like reasonable options, but then I stumbled across the `raw_decode` method of the `JSONDecoder` in the python standard library.

The `raw_decode` method does almost exactly what we want - it reads a string and extracts a JSON object from it, ignoring any data after the object closes. It also returns the index of the remaining data, so we can call the method a second time and read the next object.

Reading the [source code for JSONDecoder](https://github.com/python/cpython/blob/f4c03484da59049eb62a9bf7777b963e2267d187/Lib/json/decoder.py#L343), we find that internally, it's using a `scanner` function. This is the magic we need. The scanner reads a string, and pulls a json object from the beginning. It then returns that object and the index of the string where the object finished. If there's no object found, it raises StopIteration

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

I've written a little package called [firehose-sipper](https://github.com/bobthemighty/firehose-sipper) using this technique that can stream large files straight out of S3, and read the concatenated JSON objects.

```python
import firehose_sipper

# Read a single file out of S3

for entry in firehose_sipper.sip(bucket=some_bucket, key=some_key):
    print(entry)
    
# or go nuts and read all objects under a prefix
for entry in firehose_sipper.sip(bucket=some_bucket, prefix=some_prefix):
    print(entry)
```

Now I can leave the Firehose to its default configuration, skip the extra lambdas, and read the results with a one-liner.

[^1]: For example [Append Newline to Amazon Kinesis Firehose JSON Formatted Records with Python and AWS Lambda](https://medium.com/analytics-vidhya/append-newline-to-amazon-kinesis-firehose-json-formatted-records-with-python-f58498d0177a)

[^2]: Eg. the Stackoverflow question "[Reading the data written to s3 by Amazon Kinesis Firehose stream](https://stackoverflow.com/questions/34468319/reading-the-data-written-to-s3-by-amazon-kinesis-firehose-stream)"

[^3]: [naya.py](https://gist.github.com/bobthemighty/9f4fd8fbb2435b8f6b8cf191dabdf37a#file-naya-py)

[^4]: [yajl.py](https://gist.github.com/bobthemighty/9f4fd8fbb2435b8f6b8cf191dabdf37a#file-yajl-py)
