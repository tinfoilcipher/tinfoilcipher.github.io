---
title: "Terraform – AWS S3 Native State Locking"
layout: post
date: 2025-01-30
categories: 
  - "aws"
  - "devops"
  - "integrations"
tags: 
  - "devops"
  - "integration"
  - "s3"
  - "terraform"
---

A long while ago I wrote about [how to configure centralised State Locking for Terraform using Dynamo DB](/terraform-centralised-state-locking-with-dynamodb/). This configuration has become battle tested and fairly low cost solution for anyone using Terraform in AWS and scales well with pretty advanced configurations but it isn't without it's drawbacks. _DynamoDB_ can be a bit confusing to navigate if you're unfamiliar with it and releasing state locks can be a similarly scary experience so it isn't uncommon to find a Terraform implementation where _state locking_ has just been neglected (which you really shouldn't do).

Since the release of Terraform Version 1.10 experimental support for _state locking_ natively inside _S3_ has been quietly added, with the threat that _DynamoDB_ support is going to be dropped at some point in the future. So let's have a quick look at how it works.

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-4.png" class="scaled-img-75">

## How Does It Work?

Roughly the same as state locking with _DynamoDB_. When you perform a _plan_ or _apply_ operation a lock file is created within the same path as your _State File_ with the same name as your _State File_, suffixed with **.tflock**. While this file is present no other operations can be made that make any changes to the state, exactly the same as when using DynamoDB.

## Configuring Terraform

Configuring Terraform is straight forward and means only a couple of changes to the **terraform** _block_ as below:

```terraform
terraform {
    required_version = "~> 1.10" #--Minimum version 1.10
    required_providers {
      aws = {
        source  = "hashicorp/aws"
        version = ">= 5.84.0"
      }
    }
  
    backend "s3" {
      bucket       = "test-states-bucket-tfc" #--S3 Bucket Name
      key          = "example-state.tfstate" #--State file name
      region       = "eu-west-2"
      encrypt      = true
      use_lockfile = true #--Use native lock file
    }
  }
  
```

Now when performing a _plan_ or _apply_ operation, a lock file will appear within S3:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/image-8.png)

In the event that you discover a lock in place and you need to determine who holds the lock, you will need to open the **.tflock** and investigate it's contents. The contents are a JSON record of whoever initiated the lock in the same format that you are used to seeing in **DynamoDB**:

```json
{
  "ID": "6b156785-aaaf-3c11-fc54-d0df6154de44",
  "Operation": "OperationTypeApply",
  "Info": "",
  "Who": "welsh@test-box",
  "Version": "1.10.5",
  "Created": "2025-01-30T13:14:29.102922Z",
  "Path": "test-states-bucket-tfc/example-state.tfstate"
}

```

## Pesky IAM Permissions

The _Account_ or _Role_ interacting with _S3_ will need a minimum set of permissions to read and write objects in order to apply the lock. The Terraform documentation does not cover this for _S3 State Locking_ the same as it does for _DynamoDB_ (presumably because _S3_ permissions can get pretty complicated depending on your configuration) but since it's a bridge you will probably have to cross here the baseline permissions you will need (assuming you are working inside a single AWS account using regular **SSE** (Server Side Encryption) or no encryption):

```json
{
    "Statement": [
        {
            "Sid": "S3StateLocks"
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject"
            ],
            "Effect": "Allow",
            "Resource": "arn:aws:s3:::test-states-bucket-tfc/*"
        }
    ],
    "Version": "2012-10-17"
}
```

Simple!
