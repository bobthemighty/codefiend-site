+++
title = "How I use ChatGPT"
slug = "how-i-use-chatgpt"
date = "2024-03-28"
category = "practices"

[extra]
author = "Bob Gregory"

[taxonomies]
tags = ["practices", "ai"]
+++

<link href="/chad.css" rel="stylesheet" />
I've had a few conversations recently about ChatGPT and how useful it is (or isn't) in software engineering. I wanted to talk about how I personally use ChatGPT as an aid to productivity.

<!-- more -->

I want to build a piece of software to use in our build pipeline. CarbonRe use a monorepo approach, and currently, we have to build and deploy every part of the stack together. Instead, every time we build an artifact - a docker image or a lambda function - I want to write a record of that artifact into a database. When I deploy our system, using Terraform and the Serverless Framework, I want to read the artifact versions from that database. 

This will allow me to only build the pieces that have actually changed, but still be able to give a complete list of version numbers to Terraform.

I've wanted to do this for a few months, but it's just hard enough to be a nuisance. Last week I fired up ChatGPT, or Chad, as we affectionately call him at work.

## Chats with Chad

### Setting the context

<dl class="chad">
    <dt>You</dt>
    <dd>I want a python command line tool that can store versions of built artifacts from a ci process in Dynamo. I should be able to store multiple artifacts for each build, with multiple invocations of the tool, fetch the latest version of all artifacts, or fetch the artifacts for an earlier build.</dd>
    <dt>Chad</dt>
    <dd>
    <p>
    Creating a Python command-line tool for managing CI artifacts with DynamoDB involves several steps, including setting up AWS credentials, creating a DynamoDB table, and implementing the command-line interface (CLI). Here's a high-level overview and a basic implementation to get you started.
    </p>
    <h4>Step 1: AWS Setup</h4>

* AWS Account: Ensure you have an AWS account and AWS CLI installed.
* AWS Credentials: Configure your AWS credentials using aws configure to set your access key, secret key, and default region.
* DynamoDB Table: Create a DynamoDB table. For simplicity, let's name it CIArtifacts with a primary key build_id (string) and a sort key artifact_name (string). You can create this table via the AWS Console or AWS CLI.
<h4>Step 2: Python Environment Setup</h4>

* Python Environment: Ensure you have Python installed. It's recommended to use a virtual environment.
* Install Boto3: Boto3 is the AWS SDK for Python. Install it via pip:

`pip install boto3`

<h4>Step 3: Implementing the CLI Tool</h4>

You can use the argparse module for creating the CLI. The tool will support three operations: storing artifacts, fetching the latest version of all artifacts, and fetching artifacts for a specific build.

Here's a basic implementation outline:

``` python
import argparse
import boto3
from boto3.dynamodb.conditions import Key

# Initialize a DynamoDB resource
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('CIArtifacts')

def store_artifact(build_id, artifact_name, artifact_location):
    """
    Store an artifact in the DynamoDB table.
    """
    table.put_item(
        Item={
            'build_id': build_id,
            'artifact_name': artifact_name,
            'artifact_location': artifact_location
        }
    )

def fetch_latest_artifacts():
    """
    Fetch the latest version of all artifacts.
    """
    # This requires a scan operation and some logic to determine the latest artifacts.
    # Implementation depends on your data structure and needs.

def fetch_artifacts_for_build(build_id):
    """
    Fetch artifacts for a specific build.
    """
    response = table.query(
        KeyConditionExpression=Key('build_id').eq(build_id)
    )
    return response['Items']

def main():
    parser = argparse.ArgumentParser(description="CI Artifact Manager")
    subparsers = parser.add_subparsers(dest='command')

    # Store artifact command
    store_parser = subparsers.add_parser('store')
    store_parser.add_argument('--build-id', required=True)
    store_parser.add_argument('--artifact-name', required=True)
    store_parser.add_argument('--artifact-location', required=True)

    # Fetch latest artifacts command
    fetch_latest_parser = subparsers.add_parser('fetch-latest')

    # Fetch artifacts for a build command
    fetch_build_parser = subparsers.add_parser('fetch-build')
    fetch_build_parser.add_argument('--build-id', required=True)

    args = parser.parse_args()

    if args.command == 'store':
        store_artifact(args.build_id, args.artifact_name, args.artifact_location)
    elif args.command == 'fetch-latest':
        fetch_latest_artifacts()
    elif args.command == 'fetch-build':
        artifacts = fetch_artifacts_for_build(args.build_id)
        print(artifacts)

if __name__ == '__main__':
    main()

```

<dd>
</dl>


### Thinking about the design we want

Okay, it's a start. We have the basics for talking to Dynamo, and the rough shape of a program. It annoys me that the thing assumes I'm an idiot at every step, but hey-ho. Chad noped out of building the "fetch_latest" function, because it depends on the exact data structure, which is fair if somewhat unambitious.

My idea is to build a transaction log.

<div class="notepaper">

| PK       | SK         | Name     | Identifier                         |
| -------- | ---------- | -------- |------------------------------------|
| Artifact | 1-backend  | Backend  | my-docker-repo.com/backend:1       |
| Artifact | 1-frontend | Frontend | s3://some-bucket/1/frontend.tar.gz |
| Artifact | 2-worker   | Worker   | s3://some-bucket/2/worker.pex      |
| Artifact | 2-backend  | Backend  | my-docker-repo.com/backend:2       |

</div>

In this table, we have the results of 2 builds, build 1, and build 2. In the first build, we produced artifacts for a backend docker image, and a front-end tarball, containing javascript and things. In the second build, we produced a pex file that we can deploy to Lambda, plus a new version of the backend. 

When I `fetch_latest` I want the following items:

```
Backend: my-docker-repo.com/backend:2
Frontend: s3://some-bucket/1/frontend.tar.gz
Worker: s3://some-bucket/2/worker.pex
```

This structure means that I can ask for all the artifacts up to a particular point in time. Let's add a few more builds.

<div class="notepaper">

| PK       | SK         | Name     | Identifier                         |
| -------- | ---------- | -------- |------------------------------------|
| Artifact | 1-backend  | Backend  | my-docker-repo.com/backend:1       |
| Artifact | 1-frontend | Frontend | s3://some-bucket/1/frontend.tar.gz |
| Artifact | 2-worker   | Worker   | s3://some-bucket/2/worker.pex      |
| Artifact | 2-backend  | Backend  | my-docker-repo.com/backend:2       |
| Artifact | 3-frontend | Frontend | s3://some-bucket/3/frontend.tar.gz |
| Artifact | 4-worker   | Worker   | s3://some-bucket/4/worker.pex      |
| Artifact | 5-backend  | Backend  | my-docker-repo.com/backend:5       |

</div>

Let's imagine that we deployed after build 5, so that:

* the frontend is at v3
* the worker at v4
* and the backend at v5

when we realise that there's a bug with the backend. I want to be able to say "fetch me everything up to build 4" so that we end up with:

* the frontend at v3
* the worker at v4
* and the backend at v2

To figure out what version everything is at, I can use a simple loop, something like this:

```python

def get_latest_version(items, up_to=None):
    # Create a dict of artifacts
    artifacts = dict()
    
    # For each item in the transaction log
    for item in items:

        # if the item's build number is more than the "up_to", bail.
        if up_to and item.build_number > up_to:
            break

        # otherwise, update this artifact version
        artifacts[item.name] = item.identifier
    return artifacts
```

This is sort of like event-sourcing, in that the state of the system is stored as a list of events that we can apply. This schema would be bad if I expected to have thousands of artifacts, because it's all in a single partition, and involves loading the full history, but I have a dozen or so, and I plan to snapshot the transactions every few days to keep the number of entries low.

### Rage against the machine

I have a few attempts with Chad, to get it to understand the concept. I should probably just have been more explicit about what I intended, instead of giving cryptic hints.

<dl>
<dt>You</dt>
<dd>I think I can use a single partition key, but use the build number as a sort key.</dd>
<dt>Chad</dt>

<dd>
Using a single partition key for all items and the build number as a sort key is a viable strategy if you structure your data correctly and your access patterns align with this design. This approach is beneficial when you want to query all artifacts across builds efficiently or fetch artifacts within a specific build.

Given this approach, let's assume the partition key is a constant (e.g., *"CI_Artifacts"*), and the sort key is a combination of the build number and artifact name (e.g., "123-ArtifactName"). This design allows you to use DynamoDB's query capabilities effectively.
</dd>
<dt>You</dt>
<dd>You don't need to scan, the build number gives an order over the artifacts.</dd>
<dt>You</dt>
<dd>You don't need to sort descending, you can maintain a dict of artifacts, and update the version for every record in the response. You can tombstone old records periodically.</dd>
<dt>You</dt>
<dd>Given that the items are ordered by build number, you don't need the if statement.</dd>
<dt>
</dl>


Ad nauseam.


### Making a big change, quickly

One of the things I _really_ appreciate about Chad is that I can quickly make a change that would take a long time by hand, if it's not conceptually hard. For example, we use Click to build all of our cli tooling, so I'd like to be consistent.

<dl>
<dt>You</dt>
<dd>Can you rewrite to use `click`. I dislike argparse.</dd>
<dt>Chad</dt>
<dd><Certainly! Using `click` instead of argparse can simplify the creation of command-line interfaces (CLIs) and make the code more readable. To rewrite the Python script to use click, you'll first need to ensure click is installed in your environment. If it's not already installed, you can do so by running pip install click.

Here's how you can rewrite the script using click:

```python
import click
import boto3
from boto3.dynamodb.conditions import Key

# Initialize a DynamoDB resource
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('CIArtifacts')

@click.group()
def cli():
    """CI Artifact Manager"""
    pass

@cli.command()
@click.option('--build-number', required=True, help='The build number of the artifact')
@click.option('--artifact-name', required=True, help='The name of the artifact')
@click.option('--artifact-location', required=True, help='The location where the artifact is stored')
def store(build_number, artifact_name, artifact_location):
    """
    Store an artifact in the DynamoDB table.
    """
    build_artifact = f"{build_number}-{artifact_name}"
    table.put_item(
        Item={
            'artifact_type': 'CI_Artifacts',
            'build_artifact': build_artifact,
            'artifact_location': artifact_location,
        }
    )
    click.echo(f'Artifact {artifact_name} for build {build_number} stored successfully.')

@cli.command(name='fetch-latest')
def fetch_latest():
...

if __name__ == '__main__':
    cli()
```

</dd>
</dl>

### Making tricky changes

Finally, although this is going to be a small table, and this script will only run a few times per day, I'd like to reduce the network calls to the minimum. My idea is that if we're running Github Actions, I can cache the Dynamo response for the duration of a single deployment, so that I can call "fetch_artifact" as many times as I like without further requests to Dynamo. Again, this is conceptually trivial, but would take me a little while to think about and to write.

<dl class="chad">
<dt>You</dt>
<dd>
I'd like to be able to request a single artifact, eg. 'artifacts fetch-latest backend where "backend" is the name of an artifact. I'd also like, if the variable "GH_RUN_ID" is set in the environment, to cache the results of the query on disk in "/tmp/artifacts-{GH_RUN_ID}" to avoid rebuilding the result set.
</dd>
<dt>Chad</dt>
<dd>
To extend the functionality of your CLI tool to allow requesting a single artifact by name and to cache the query results on disk when the GH_RUN_ID environment variable is set, we'll make the following modifications:

1. Add Support for Requesting a Single Artifact: Modify the fetch-latest command to accept an optional argument for the artifact name
2. Implement Caching: Before performing a query, check if the *GH_RUN_ID* environment variable is set and if the cache file exists. If so, load the results from the cache. After a query, cache the results if *GH_RUN_ID* is set.

```python
import os
import json
import click
import boto3
from boto3.dynamodb.conditions import Key

# Initialize a DynamoDB resource
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('CIArtifacts')

@click.group()
def cli():
    """CI Artifact Manager CLI"""
    pass

def cache_results(func):
    def wrapper(*args, **kwargs):
        gh_run_id = os.getenv('GH_RUN_ID')
        cache_file = f"/tmp/artifacts-{gh_run_id}" if gh_run_id else None

        if kwargs.get('build'):
            cache_file_suffix = f"-{kwargs['build']}"
        else:
            cache_file_suffix = ""
        cache_file = f"{cache_file}{cache_file_suffix}" if cache_file else None

        if cache_file and os.path.exists(cache_file):
            with open(cache_file, 'r') as f:
                cached_data = json.load(f)
                click.echo(json.dumps(cached_data, indent=4))
                return

        result = func(*args, **kwargs)

        if gh_run_id:
            with open(cache_file, 'w') as f:
                json.dump(result, f)

    return wrapper

@cli.command()
@click.option('--build-number', required=True, help='The build number associated with the artifact.')
@click.option('--name', required=True, help='The name of the artifact.')
@click.option('--artifact-id', required=True, help='A unique identifier for the artifact.')
def store(build_number, name, artifact_id):
    sort_key = f"{build_number}-{name}"
    table.put_item(
        Item={
            'artifact_type': 'CI_Artifacts',
            'build_artifact': sort_key,
            'artifact_id': artifact_id,
        }
    )
    click.echo(f'Artifact "{name}" with ID {artifact_id} for build {build_number} stored successfully.')

@cli.command(name='fetch-latest')
@click.argument('artifact_name', required=False)
@click.option('--build', required=False, help='Read up to this build number and disregard later builds.')
@cache_results
def fetch_latest(artifact_name=None, build=None):
    latest_artifacts = {}
    key_condition_expression = Key('artifact_type').eq('CI_Artifacts')
    if build:
        key_condition_expression &= Key('build_artifact').lte(f"{build}-z")

    response = table.query(
        KeyConditionExpression=key_condition_expression,
        ScanIndexForward=True
    )

    for item in response['Items']:
        _, name = item['build_artifact'].split('-', 1)
        latest_artifacts[name] = item

    if artifact_name:
        specific_artifact = latest_artifacts.get(artifact_name, None)
        return {artifact_name: specific_artifact} if specific_artifact else {artifact_name: f"No entry found for '{artifact_name}' up to build {build}."}

    return latest_artifacts

if __name__ == '__main__':
    cli()
```
</dd>

</dl>



And with that, we have a prototype. It stores artifacts as a transaction log, caches the result during a build, and lets me play around with it from the CLI.

## What's good?

This is a successful spike. It proves the rough design can work and gives me a starting point for the code. In total, I spent about an hour talking to Chad, trying things out and running them locally. I didn't have to Google for the Dynamo sdk docs, which is a bonus, because I hate them. I didn't have to copy-paste a bunch of Click related things that I use all the time but can't remember. If I'd written this by hand it probably would have taken me about a day to figure out all the components, because I am human and have a smol memory.

That means *I probably wouldn't have done this at all*: taking a whole day to build this is off-putting, but generating it in an hour is fine. That lowers the barrier for the sort of tools that I can build for myself, which has a compounding productivity impact.

## What's bad?

The code is _not good_. After I'd finished this, and the second script which uploads all the artifacts from our build process, I then *completely rewrote this code*. Chad ended up with a bunch of functions that had 6 arguments, the layering was wrong which made things more complicated than they needed to be, and so on. Chad is not a good software designer, but it _is_ an excellent generator of mediocre code.

Here's a snippet of the current working version

```python
class ArtifactRepository:
    def __init__(self, dynamo_table, verbose=False):
        ...

    def _fetch(self):
        latest_artifacts = {}
        key_condition_expression = Key("artifact_type").eq("CI_Artifacts")

        response = self.table.query(
            KeyConditionExpression=key_condition_expression, ScanIndexForward=True
        )

        for item in response["Items"]:
            _, name = item["build_artifact"].split("-", 1)
            latest_artifacts[name] = item["artifact_id"]
        return latest_artifacts

    def latest(self, build: str | None = None):
        cache_file = f"/tmp/artifacts-{self.run_id}"

        if build:
            cache_file_suffix = f"-{build}"
        else:
            cache_file_suffix = ""
        cache_file = f"{cache_file}{cache_file_suffix}" if cache_file else None

        if cache_file and os.path.exists(cache_file):
            with open(cache_file, "r") as f:
                cached_data = json.load(f)
                return cached_data

        result = self._fetch()

        if cache_file:
            with open(cache_file, "w") as f:
                json.dump(result, f)
        return result


    def store(self, build_number, name, artifact_id):
        ...
```

You can clearly see the ancestry here, there's lines of code that were generated directly by Chad, but I've moved things around to introduce a class, I've dropped the decorator for caching, and so on. I've also broken the "get artifacts up to build x functionality" because I'm dumb.

That, in part is because there are no tests. Chad could probably have generated me some tests, but I don't know if I would trust them. That's not a _huge_ problem for me, because this piece of code has a low [modifiability requirement](@/mustapo.md#Modifiability). Once it's working, and I've manually tested it, it'll probably just run in the same way until we replace it altogether, but it does mean that I can absent-mindedly remove a whole feature and not notice.

## What's tricky

I went into this with a very clear idea of the design that I wanted to achieve. I know quite a bit about Dynamo already, I've built event-sourced systems, I have a good grasp of the build system I'm working with and so on. Chad's first attempt was incomplete, and its subsequent attempts were bad. It took me some time to get to the design that I wanted. Chad does _not_ replace human expertise, but it _can_ replace laborious typing of prototype code.

If I _hadn't_ already:

* Chosen an appropriate technology, Dynamo
* Decided on a technical approach, a transaction log
* Figured out how I wanted to use this, via a Terraform external datasource

and so on, I don't know whether I would have gotten to a useful result.
