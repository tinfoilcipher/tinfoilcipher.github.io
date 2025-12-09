---
title: "Configuring ECS Fargate and ECR with Private Subnets"
layout: post
date: 2025-01-29
categories: 
  - "aws"
  - "containers"
  - "devops"
tags: 
  - "containers"
  - "devops"
  - "ecs"
  - "networking"
  - "security"
---

Recently I found myself working with **[AWS ECS](https://aws.amazon.com/ecs/)** (Elastic Container Service) to host a simple application and using using **[AWS Fargate](https://aws.amazon.com/fargate/)** as the underlying compute. This pairing bills itself as a simple means of deploying containers without the hassle of standing up and configuring your own servers.

That's an attractive offer but I quickly ran in to some issues when trying to deploy to **[private subnets](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html#subnet-basics)**. Like so many other cloud systems this one prefers to be public by default which often isn't suitable when working in secure, governed environments and certainly wasn't an option for me.

In this post I'm going to look at how to deploy an ECS/Fargate service to truly **private** subnets and what extra infrastructure is needed to allow proper communication between services.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-3.png)

**TL;DR**. If you aren't really interested in reading this and you just want your problem to go away...well that's a shame, but I have written a Terraform module and **[stuck it in the module registry here](https://registry.terraform.io/modules/tinfoilcipher/ecr-private-subnet-endpoints/aws)** which will fix your problems. But beware of blindly applying configs written by strangers!

## Private Subnets Mean...What Exactly?

First of all let's clear up some terminology. Generally when we talk about "_Private Subnets_" in AWS, we're usually talking about something that isn't directly exposed to the internet but can still **ACCESS** the internet (so really, a subnet that doesn't have a route to an **[Internet Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html)** but **does** have a route to _[**NAT Gateway**](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)_, I.E. you can get traffic out but it can't get in).

This is the common wisdom, but that doesn't feel very private and if you've landed here you probably don't really have a problem at all because this configuration will work with _ECS Fargate_ and _ECR_ just fine out of the box. What we're talking about here is truly "private" subnets, which are isolated and have no access to the internet.

## Private ECR Repositories Mean...What Exactly?

_ECR Repositories_ come in two flavours, _Public_ and _Private_, but _Private_ here doesn't relate to IP networking the same as our subnets, rather it refers to authentication and the policies that can be attached to your _Repository_. Important to understand is that a **Private** **ECR Repository** **still has a _Public_ IP address**, we just have to authenticate before we can push and pull any images from it.

## Unhelpful Error Messages

So, with the preamble out of the way...

Chances are that if you've found your way here you are staring this error message in our ECS event stream:

<blockquote>
  ResourceInitializationError: unable to pull secrets or registry auth: The task cannot pull registry auth from Amazon ECR: There is a connection issue between the task and Amazon ECR. Check your task network configuration. RequestError: send request failed caused by: Post "https://api.ecr.region.amazonaws.com/": dial tcp x.x.X.x:443: i/o timeout
</blockquote>

This error message is useful in these sense that it tells us what is roughly going wrong, we either have no route from between ECS and ECR...or we do have a route, but we don't have any permission to read once we establish a connection...but it's unhelpful in that doesn't really tell you why this is the case or what to do about it, and these scenarios can be pretty difficult to diagnose in the opaque world of public cloud.

Understanding how traffic moves around inside a VPC is useful here. Since our _ECS Cluster_ is isolated within one or more private subnets, with no routes to go anywhere beyond it's own address space and our _ECR Repository_ is only accessible via a public IP address the problem is pretty obvious; our subnet is really a little bit too private. We will need to introduce some method to get access to _ECR_ that isn't via the public internet.

The below quick diagram should help to break down exactly what we're working with and where we're going wrong

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-3.png">
  <figcaption>Our _ECS Cluster_ is private...but a bit too private. We have effectively put it in a bubble where it can't even reach our ECR Repository!</figcaption>
</figure>

## Missing Endpoints

It's here that **[VPC Endpoints](https://docs.aws.amazon.com/whitepapers/latest/aws-privatelink/what-are-vpc-endpoints.html)** come in to play. The AWS documentation will give you **[an in depth breakdown of how endpoints and their underlying technology (AWS PrivateLink) work](https://docs.aws.amazon.com/vpc/latest/userguide/endpoint-services-overview.html)** so there isn't much value in me regurgitating it, but in a nutshell there are two types:

1. **Interface Endpoints**. These are a collection of network interfaces which catch DNS queries destined for some AWS service and redirect your traffic appropriately, keeping it inside your VPC. The broad effect of using an _Interface Endpoint_ is to apply a private IP to an otherwise public service (such as ECR).

2. **Gateway Endpoints**. Are used specifically for _S3_ or _DynamoDB_ and manipulate a specific _AWS Route Table_ by adding a _Prefix List_ (this is just a fancy name for a well known list of IP addresses that can be summarised as a single prefix). The effect of using this method is to allow calls to _S3_ or _DynamoDB_ from within a private subnet with no means of reaching the internet.

So to get those endpoints in place:

First we'll need to create an appropriate **Security Group** to manage access to our endpoints. In the AWS console browse to **VPC** > **Security Groups** > **Create security group**. Apply the rules shown in the table below (the **ingress** rules must be applied for each of your _Private Subnet_ _CIDRs_):

| Type      | Protocol | Port Range | Reason.     |
|-----------|----------|------------|-------------|
| HTTP      | TCP      | 80         | Image Pulls |
| HTTPS     | TCP      | 443        | Image Pulls |
| DNS Query | UDP      | 53         | DNS Queries |

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/image-7-1024x689.png">
  <figcaption>Substitute your subnet CIDRs as appropriate!</figcaption>
</figure>

To create our _Interface Endpoints_; browse to **VPC** > **Endpoints** > **Create endpoint**:

Select the following configuration options:

- **Name Tag**: Name the _Endpoint_ something suitable

- **Endpoint type**: AWS Services

- **Endpoint Service Name**: com.amazonaws.**<YOUR\_REGION>**.ecr.dkr

- **Endpoint Service Type**: Interface

- **VPC**: Your _VPC_ as appropriate

- **DNS name**: Enable DNS resolution

- **DNS record IP type**: IPv4

- **Subnets**: Your private _Subnets_ as appropriate

- **Subnet IP address type**: IPv4

- **Security groups**: The _Security Group_ created in the previous step

Click **Create Endpoint**.

If you are using ECS Fargate Version 1.4 (and you almost certainly are), repeat this process again and create a second endpoint for endpoint **com.amazonaws.<YOUR\_REGION>.ecr._api**.

This should be the _Interface Endpoints_ complete, we need to create a final _Gateway Endpoint_ for S3, repeat the creation process again with the below configurations:

- **Name Tag**: Name the _Endpoint_ something suitable

- **Endpoint type**: AWS Services

- **Endpoint Service Name**: com.amazonaws.**<YOUR_REGION>**.s3

- **Endpoint Service Type**: Gateway

- **VPC**: Your _VPC_ as appropriate

- **Route Tables**: Your _Private Route Table(s)_ as appropriate

With this in place our _Endpoints_ are ready for use:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/10-1024x223.png">
  <figcaption>Configured endpoints</figcaption>
</figure>

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/11.png">
  <figcaption>Injected _Prefix List_ corresponding to S3 Gateway Endpoint</figcaption>
</figure>

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/05-1.png)

Our ECS services can now run just fine and pull images from ECR without being exposed to the internet.

## Wait...What Has S3 Got to Do Anything?

Good question. This is actually detailed in a tiny paragraph of the configuration documentation [**here**](https://docs.aws.amazon.com/AmazonECR/latest/userguide/vpc-endpoints.html#ecr-setting-up-s3-gateway) but you would be forgiven for not understanding it at all. When ECS pulls an image from ECR it actually caches the image in a hidden S3 bucket that you have zero visibility of (in fact you have no visibility of this process at all), so if you are working in private subnets you will need to ensure that you also have a _VPC Gateway Endpoint_ for _S3_ otherwise your images will never find their way down to your cluster!

## Can't I Just Use A NAT Gateway? A Final Thought On Hidden Costs

The short answer is yes, you can technically. It might seem to make your life easier in the short term when things just seem to start working and you don't have to do so much configuration but this isn't very forward thinking (plus it might not even be an option you have if you are in a highly governed environment).

Every request that goes through a NAT Gateway and out in to the internet costs money, not a lot of money but it can all add up pretty quickly and you don't want to be on the wrong side of that.

A few minutes glancing around Stackoverflow etc. will show you a lot of horror stories where people's bills suddenly went through the roof in the middle of the night when their application started crashing and ECS started pulling images from ECR over and over again without any rate limits, such a scenario will lead to a huge amount of traffic flying through a NAT Gateway. Without proper network considerations this could easily be you and that wouldn't be very nice for anyone!
