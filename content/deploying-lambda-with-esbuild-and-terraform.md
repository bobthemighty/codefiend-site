+++
title = "Deploying Lambda with ESBuild and Terraform"
slug = "deploying-lambda-with-esbuild-and-terraform"
date = "2022-03-10"
category = "serverless"

[extra]
author = "Bob Gregory"

[taxonomies]
tags = ["serverless", "typescript", "terraform"]
+++

Generally when deploying Lambda functions, I use the [Serverless Framework](). This is a wrapper around AWS Cloudformation that makes it easier for engineers to author and deploy code. It's much easier to get started when using an abstraction like this, but sometimes we need to drop down and work at a lower-level. 

I recently needed to deploy a lambda function with a large number of triggers, and the cloudformation template generated by the serverless framework was too large to deploy. Instead I ended up configuring everything in plain Terraform. There are a few moving pieces to understand, flexibility. In this post I want to show you how to configure and deploy a simple lambda function, confguring everything ourselves in Terraform so we can see how it all fits together.

## What are we going to create?

We're going to build a simple lambda function that wakes up every day at 9am and checks stock prices for a small list of companies, then stashes the result in Dynamo. We'll configure Amazon EventBridge to raise an event on a schedule - like a cron job - and we'll use the event to trigger our Lambda function.

![Picture goes here, yo]()