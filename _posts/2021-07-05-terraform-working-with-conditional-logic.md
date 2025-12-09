---
title: "Terraform - Working With Conditional Logic"
layout: post
date: 2021-07-05
categories: 
  - "automation"
  - "devops"
tags: 
  - "automation"
  - "cloud"
  - "devops"
  - "terraform"
---

Recently I've been having some fun with writing a fairly complex Terraform module which of course has to make use of Conditional Logic a fair bit. The Terraform documentation covers both _[**Conditionals**](https://www.terraform.io/docs/language/expressions/conditionals.html)_, **[Functions](https://www.terraform.io/docs/language/functions/index.html)** and **[Operators](https://www.terraform.io/docs/language/expressions/operators.html)** very well, but practical examples are a little lacking. In this short post I'm going to look at how _Conditionals_ work and a few helpful examples of using a few _Operators_ and _Functions_ to extend our functionality.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-7.png)

## Conditional Expressions

The Terraform documentation defined the syntax for a conditional expression is as follows:

```bash
#--Standard Syntax
condition ? true_val : false_val
```

I found that pretty unhelpful so let's break it down with a very common, practical example.

Our "condition" is going to be the _count_ _meta-argument_ being set to **true** within the **aws\_s3\_bucket** _Resource_. We're going to satisfy this by passing a _Variable_ called **provision** with "type" _bool_ as the value to be evaluated.

When a _boolean_ is evaluated by Terraform is returned as either 0 (false) or 1 (true). As we're using this to satisfy the _count_ argument this means that if we set our variable to **true** the resource will be created and if set to **false** nothing will happen. Our configuration will look like:

```terraform
variable "provision" {
    type        = bool
    description = "Optionally Create Bucket"
}

resource "aws_s3_bucket" "bucket" {
    count   = var.provision ? 1 : 0
    bucket  = "tinfoilbucket"
    acl     = "private"
}
```

If we execute an _apply_ with this configuration we can enter a true/false at run time:

```bash
terraform apply
# var.provision
#   Optionally Create Bucket
# 
   Enter a value: false
# 
# Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

terraform apply
# var.provision
#   Optionally Create Bucket
# 
   Enter a value: true
#
#  aws_s3_bucket.bucket[0]: Creating...
#  aws_s3_bucket.bucket[0]: Creation complete after 6s [id=tinfoilbucket]
# 
# Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

## Conditionally Creating Multiple Resources

A valuable option that isn't necessarily obvious out of the gate is the need to create **multiple** resources conditionally. The answer seems obvious usually, we define the number of resources to create using the _count_ _meta-argument_ however we're already using this to evaluate a true/false _condition_.

We can still use this if we simply pass a value higher than 1 to our **true_val** to be evaluated as a _conditional_ when using _count_, passing this count to be evaluated will result in this number of resources being created. The most dynamic way to do this is by measuring the length of an existing multi-element variable using the _length_ function.


In the below example we'll be adding an additional variable named **bucket_names**, this is a list containing two names. This example in brief will provision two _S3 Buckets_ when the _variable_ **var.provision** is set to **true**.

```terraform
variable "provision" {
    type        = bool
    description = "Optionally Create Multiple Buckets"
}

variable "bucket_names" {
    type        = list(string) #--Define variable as a list of strings
    description = ["tinfoilbucket1", "tinfoilbucket2"] #--Define the two items in the list
}

resource "aws_s3_bucket" "bucket" {
    count   = var.provision ? length(var.bucket_names) : 0 #--Evaluate conditional. If var.provision is true, provision the amount of buckets defined in var.bucket_names
    bucket  = var.bucket_names[count.index] #--Create buckets as named in var.bucket_names, iterating over list
    acl     = "private"
}
```

If we apply, we can see the logic in action and that our iteration works correctly:

```bash
terraform apply
# var.provision
#   Optionally Create Multiple Buckets
# 
   Enter a value: false
# 
# Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

terraform apply
# var.provision
#   Optionally Create Bucket
# 
   Enter a value: true
#
#  aws_s3_bucket.bucket[0]: Creating...
#  aws_s3_bucket.bucket[1]: Creating...
#  aws_s3_bucket.bucket[0]: Creation complete after 6s [id=tinfoilbucket1]
#  aws_s3_bucket.bucket[1]: Creation complete after 7s [id=tinfoilbucket2]
# 
# Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

## Further Options

This is good, but it's limited on it's own. Let's bring some _Operators_ in to the mix and see what other options we have with some more common examples.

In the below example we'll use **&&** and _Operator_ to ensure that an _S3 Bucket_ is deployed only when **BOTH** the **deploy\_environment** and **deploy\_storage** are true:

```terraform
variable "deploy_environment" {
    type        = bool
    description = "Optionally Create Keys"
}

variable "deploy_storage" {
    type        = bool
    description = "Optionally Create Keys"
}

resource "aws_s3_bucket" "bucket" {
    count   = var.deploy_environment && var.deploy_storage ? 1 : 0
    bucket  = "tinfoilbucket"
    acl     = "private"
}
```

In the below example we'll use both the **&&** and **!** _Operator_ to ensure that an S3 Bucket is deployed when the **deploy\_storage** variable is true but **NOT** when the deploy\_environment is true:

```terraform
variable "deploy_environment" {
    type        = bool
    description = "Optionally Create Keys"
}

variable "deploy_storage" {
    type        = bool
    description = "Optionally Create Keys"
}

resource "aws_s3_bucket" "bucket" {
    count   = !var.deploy_environment && var.deploy_storage ? 1 : 0
    bucket  = "tinfoilbucket"
    acl     = "private"
}
```

In the below example we'll use both the **| |** _Operator_ to ensure that an S3 Bucket is deployed when **EITHER** of the **deploy\_environment** and **deploy\_storage** are true:

```terraform
variable "deploy_environment" {
    type        = bool
    description = "Optionally Create Keys"
}

variable "deploy_storage" {
    type        = bool
    description = "Optionally Create Keys"
}

resource "aws_s3_bucket" "bucket" {
    count   = var.deploy_environment || var.deploy_storage ? 1 : 0
    bucket  = "tinfoilbucket"
    acl     = "private"
}
```

Other _Operators_ exist and innumerable combinations can be made from them.

## Taking A Look At The Coalesce Function

This functionality is great but I've found an increasing use for both the **coalesce** _Function_ and I don't often see it coming up in conversation.

A design I like to embrace is a common naming convention throughout a given environment. Within Terraform I typically like to create this name using some string manipulations in _Locals_ with the name based on some _Random_ generation, however I also like to give the operator the option to provide their own names via an _Input Variable_.

The _coalesce Function_ allows us to do both so let's take a look how that works in a config:

```terraform
variable "custom_bucket_name" { #--Input Variable for Custom Bucket name. Null by default
    type        = string
    description = "Custom Bucket Name"
    default     = null
}

resource "random_string" "bucket_id" { #--Generate random 5 letter ID for dynamic bucket name
    length    = 5
    min_lower = 5
    number    = false
    special   = false
}

locals {
    bucket_dynamic_name = "tinfoilbucket-${random_string.bucket_id.id}" #--Generate Dynamic Bucket Name
    bucket_name         = coalesce(var.custom_bucket_name, local.bucket_dynamic_name) #--Coalasece arguments
}

resource "aws_s3_bucket" "bucket" {
    bucket  = local.bucket_name #--Create Bucket with name from the results of coalesce
    acl     = "private"
}
```

If we apply this without defining a value for **custom\_bucket\_name** we should see a randomly generated bucket name:

```bash
terraform apply
# random_string.bucket_id: Creating...
# random_string.bucket_id: Creation complete after 0s [id=jdhuk]
# aws_s3_bucket.bucket: Creating...
# aws_s3_bucket.bucket: Creation complete after 3s [id=tinfoilbucket-jdhuk]
# 
# Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

If we set an ID for the bucket to **tinfoilbucket-abcde** we should see it take precedence:

```bash
terraform apply
# random_string.bucket_id: Creating...
# random_string.bucket_id: Creation complete after 0s [id=jdhuk]
# aws_s3_bucket.bucket: Creating...
# aws_s3_bucket.bucket: Creation complete after 3s [id=tinfoilbucket-abcde]]
# 
# Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

So what's happening here?

The behaviour of __coalesce__ is to iterate over a given number of arguments and land on the first one that **isn't** _null_. By setting the custom _Variable_ to have a default value of _null_ and making it the first option for __coalesce__ to use, we can ensure that if a custom name has provided for our bucket, it will be used and if not then a randomly generated name will be used. Very useful.

## Further Reading

There's a lot more power to unlock and this article really only touches on the tiniest part of the surface, for further reading I strongly recommend reading the [Terraform tutorial on Terraform Functions](https://learn.hashicorp.com/tutorials/terraform/functions?in=terraform/configuration-language).
