---
title: "Emulating AWS Services Using Localstack"
layout: post
date: 2022-02-17
categories: 
  - "aws"
  - "devops"
  - "integrations"
tags: 
  - "api"
  - "aws"
  - "cloud"
  - "devops"
  - "docker"
  - "integration"
  - "localstack"
---

One of the challenges that seems to crop up pretty frequently is reliably simulating a cloud platform or application without having to tediously configure a sandbox environment for every little change. Even when a sandbox **is** present, the cost of operating them can quickly run out of control and can still need several people to implement even a small change.

**[Localstack](https://localstack.cloud/)** is an emulator for an ever growing number of the core AWS services that runs in a single container and lets us mock API calls as if we were connected to the real thing. It's very simple to set up and provides a very powerful solution to a common problem.

![](images/01.png)

## A Lightweight Solution for Tests

What you get is an emulated version of **a large selection of core AWS services** sitting inside a single container on your machine (or inside your CI/CD pipeline).

Your client (be that a tool or your own code) can then make calls to AWS as normal which are redirected to Localstack and the appropriate status codes returned as if they were really being sent from AWS.

I find this solution particularly attractive as it removes a lot of hurdles that can get thrown up in testing (such as having to reconfigure networking, access rights etc.) only to discover that the test was doomed to fail anyway. A solution like this allows us to perform faster tests, at no financial cost, before anything hits an active cloud environment.

## Basic Installation

Installation is very straight forward as per the project's [**README**](https://github.com/localstack/localstack) and requires only **Python** (v3.6-3.9), **pip** and **Docker** to be installed. I'm going to assume a proper installation of the pre-requites already. We can install Localstack with:

```bash
#--Install localstack. DO NOT USE SUDO OR RUN AS ROOT
pip install localstack --user
```

Following installation, we can start Localstack in a detached container (and you can see the time of night I wrote this):

```bash
localstack start -d
#     __                     _______ __             __
#    / /   ____  _________ _/ / ___// /_____ ______/ /__
#   / /   / __ \/ ___/ __ `/ /\__ \/ __/ __ `/ ___/ //_/
#  / /___/ /_/ / /__/ /_/ / /___/ / /_/ /_/ / /__/ ,<
# /_____/\____/\___/\__,_/_//____/\__/\__,_/\___/_/|_|
#
# 💻 LocalStack CLI 0.14.0
#
# [21:08:12] starting LocalStack in Docker mode 🐳      localstack.py:115
#            preparing environment                      bootstrap.py:710
#            configuring container                      bootstrap.py:718
#            starting container                         bootstrap.py:724
# [21:08:14] detaching                  
```

Localstack runs on TCP port 4566 (which is where we're going to direct our API calls), if you want to validate this you can see the full verbose startup process by running **localstack logs**. We can also verify exactly which AWS services are offered (and there's a lot) by running **localstack status services**.

## Does It Work?

I'm not a developer, if **you** are then there's plenty of examples of how to integrate with your chosen SDK in the **[Localstack Docs](https://docs.localstack.cloud/integrations/sdks/https://docs.localstack.cloud/integrations/sdks/)**.

For the purposes of our demonstration we're going to try some simple tests with the AWS CLI, specifically with S3 and DynamoDB, "creating" a _Bucket_ and _Table_ with some content. Note that we can simply use the normal AWS CLI syntax and explicitly point to Localstack as our _endpoint-url_:

```bash
aws s3 mb s3://tinfoilcipher-test-bucket --endpoint-url=http://localhost:4566
# make_bucket: s3://tinfoilcipher-test-bucket

aws s3 cp test-file.txt s3://tinfoilcipher-test-bucket --endpoint-url=http://localhost:4566
# upload: ./test-file.txt to s3://tinfoilcipher-test-bucket/test-file.txt

aws dynamodb create-table --table-name TinfoilExample --attribute-definitions AttributeName=SomethingPointless,AttributeType=S --key-schema AttributeName=SomethingPointless,KeyType=HASH --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5 --endpoint-url=http://localhost:4566
#{
#    "TableDescription": {
#        "AttributeDefinitions": [
#            {
#                "AttributeName": "SomethingPointless",
#                "AttributeType": "S"
#            }
#        ],
#        "TableName": "TinfoilExample",
#        "KeySchema": [
#            {
#                "AttributeName": "SomethingPointless",
#                "KeyType": "HASH"
#            }
#        ],
#        "TableStatus": "ACTIVE",
#        "CreationDateTime": 1645049422.389,
#        "ProvisionedThroughput": {
#            "ReadCapacityUnits": 5,
#            "WriteCapacityUnits": 5
#        },
#        "TableSizeBytes": 0,
#        "ItemCount": 0,
#        "TableArn": "arn:aws:dynamodb:eu-west-1:000000000000:table/TinfoilExample",
#        "TableId": "86c13d13-4445-4352-9bc6-13442dd8d111"
#    }
#}
```

If we then try to query these services, we get just as good feedback:

```bash
aws s3 ls s3://tinfoilcipher-test-bucket --recursive --endpoint-url=http://localhost:4566
# 2022-02-16 21:10:03          1 test-file.txt

aws dynamodb list-tables --endpoint-url=http://localhost:4566
#{
#    "TableNames": [
#        "TinfoilExample"
#    ]
#}
```

As we can see, we are interacting with Localstack just fine as if we were really interacting with AWS.

## Orchestrated Deployment with Docker Compose

This is all good and well for strictly local use, but it will run in to limitations pretty fast if we want to do anything in a CI/CD pipeline or centralise services (and I'm a fan of making all things declarative wherever possible), so let's take a look at how to get the service running with Docker Compose as an example. Below is the **docker-compose.yml** to run Localstack in orchestration:

```yaml
version: "2.1"
services:
  localstack:
    image: localstack/localstack
    container_name: localstack
    ports:
      - "4566:4566" #--TCP Port to expose service
    environment:
      - SERVICES=s3,dynamodb #--Explicit list of services to enable. See https://docs.localstack.cloud/aws/feature-coverage/ for supported services
      - DEFAULT_REGION=eu-west-1 #--Region to use when mocking
      - DOCKER_HOST=unix:///var/run/docker.sock #--Linux hosts, if you're using Windows see https://docs.docker.com/desktop/faqs/#how-do-i-connect-to-the-remote-docker-engine-api
      - DATA_DIR=/tmp/localstack #--OPTIONAL. Configure Persistance
```

With this we can start Compose with:

```bash
docker-compose up
```

After pulling the relevant layers for the **localstack/localstack** image, the service will start:

```bash
# localstack    | Starting edge router (https port 4566)...
# localstack    | Ready.
# localstack    | [2022-02-16 21:15:56 +0000] [21] [INFO] Running on https://0.0.0.0:4566 (CTRL + C to quit)
# localstack    | 2022-02-16T21:15:56.695:INFO:hypercorn.error: Running on https://0.0.0.0:4566 (CTRL + C to quit)
```

We can now interact with Localstack exactly as before.

## Conclusion

We're really only scratching the surface here, we haven't waded in to any advanced scenarios, persistence, record and replay, running in Kubernetes or integrations with other tooling (all material for future posts). The documentation is very rich and the scenarios are practically endless if you're an AWS user.

In the next post on this topic I'll be taking a quick look at integrating Localstack with Terraform as a means of better validating modules and configurations without having to incur costs by running them repeatedly on live systems.
