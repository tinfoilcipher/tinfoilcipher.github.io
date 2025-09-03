---
title: "Hybrid Windows Server DNS with AWS Route 53"
layout: post
date: 2020-10-06
categories: 
  - "aws"
  - "devops"
  - "windows"
tags: 
  - "aws"
  - "cloud"
  - "devops"
  - "dns"
  - "integration"
  - "networking"
  - "route53"
  - "windows"
---

Recently I had an requirement that I couldn't find documented outside of the abstract; migrating a single private DNS zone to AWS' hosted DNS service; [**Route 53**](https://aws.amazon.com/route53/) and conditionally forwarding queries for that zone from an existing Windows DNS infrastructure.

This isn't something I expected to be broken down blow by blow in the AWS documentation but there are plenty of Windows DNS infrastructures out there in the wild and it was something I had expected someone to have blogged about by now or at least see it hacked together in a few StackOverflow posts.

Sadly I couldn't find much, however with a little bit of understanding of the theory the process works really well so let's look at how to get set up.

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01.png" class="scaled-img-75">

## First of all, what's the problem?

In our infrastructure, we have several private DNS zones managed within a Windows DNS infrastructure. If we want to migrate one of these zones to Route 53 for any reason we're going to encounter an issue that our clients, who are receiving their IP configurations via DHCP will still be resolving all their DNS queries via our Windows DNS infrastructure.

We can't just go pointing to Route 53 as it won't know about any of our other zones, so we need to instruct our existing infrastructure to forward queries to our migrated zone.

## What Are We Working With?

In this example we'll be working with the following zones:

- madcaplaughs.co.uk
- madcaplaughs.dev
- madcaplaughs.testing
- madcaplaughs.staging

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/win1.png)

We will be migrating **madcaplaughs.staging** to Route 53, whilst the rest will remain in the Windows infrastructure.

In AWS, we already have the existing components in place:

- A VPC named **mcvpc** in _region_ **eu-west-2** with the address space **10.0.0.0/16**.
- Two subnets named **mcsubnet01** and **mcsubnet02** split over two _Availability Zones_ within the **eu-west-2** _region_ with the address spaces **10.0.1.0/24** and **10.0.2.0/24**.
- A security group has been created named **mc-staging-dns** which allows ingress and egress for TCP port 53 (DNS Queries) from our on-premise networks only.
- A site-to-site VPN is active between the on-premise infrastructure and AWS.

## Configuring a Private Zone in Route 53

The task here is quicker achieved via Terraform, however we're trying to look at the steps involved so let's use the GUI:

In the AWS Console we can create a new _Hosted Zone_ if we browse to **Route 53** \> **Hosted Zones** > **Create Hosted Zone**:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02.png">
  <figcaption>Enter the FQDN of the zone you want to route and an optional Description</figcaption>
</figure>

Below, select **Private Hosted Zone** and we'll select our Region and VPC:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03.png)

Finally, we can optionally set any _Tags_ and create the _Hosted Zone_:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/04.png)

After a few seconds you can see that the _Hosted Zone_ has created and that **Nameserver** and **Start of Authority** records have been created to use the _Route 53_ infrastructure. These cannot be removed or modified:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/05-1024x353.png)

Your records can now be either manually created or imported in an **[RFC 1035 standard Zone File](https://tools.ietf.org/html/rfc1035)**, in Windows the _Zone Files_ are located by default at **C:\\Windows\\System32\\dns**.

## Configuring a Route 53 Resolver

So our zone is now stood up in _Route 53_ but we currently have no means to route any queries to our new _Hosted Zone_. This functionality is provided by the _Route 53 Resolver_ service. For our needs we need to create an _Resolver Inbound Endpoint_ which will accept queries from our on-premise network only (controlled using our already existing security group).

We can create a new _Inbound Resolver_ by browsing to **Route 53** \> **Resolver** \> **Inbound Endpoint** > **Create Inbound Endpoint**:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/07.png">
  <figcaption>Provide an appropriate Name tag, and assign to the VPC. A Security Group is mandatory so we'll use an existing group which allows ingress/egress on TCP port 53 (DNS Queries) to/from our on-premise networks.</figcaption>
</figure>

An _Inbound Endpoint_ must have **at least** two IP addresses assigned to it to accept DNS queries, these IPs should be provided from subnets which are routable from our on-premise network. Our two subnets also **need to be in two different availability zones**:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/08.png">
  <figcaption>Since we're creating the infrastructure manually, let's use a static IP...</figcaption>
</figure>

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/09.png">
  <figcaption>...and again for the second subnet.</figcaption>
</figure>

Finally, we can optionally add tags then **Submit** the creation:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/10.png)

After a short while, the **Inbound Resolver** will be created, if we enter it's configuration we can see that it is active and has the two IP addresses we provided:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/13.png)

## Windows Server - Redirecting Queries

In order to complete the migration we first need to be sure that all records have been migrated to our _Route 53 Hosted Zone_, now in our Windows DNS console we can configure a new _Conditional Forwarder_:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/win2.png">
  <figcaption>The simplest menu in history</figcaption>
</figure>

The **DNS Domain** field should be completed with the FQDN of the domain we wish to forward to _Route 53_ and the **IP addresses of the master servers** list should be completed with the IP addresses used for our _Resolver Inbound Endpoint_.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/win3.png)

Click OK to create the _Conditional Forwarder_:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/win4.png)

Finally, delete the zone from the on-premise DNS server, as you can see above it has been deleted and all queries for this zone will now to _Route 53_.

As a final measure, the DNS cache should be cleared using **ipconfig**:

```powershell
ipconfig /flushdns
```

This process of configuring _Conditional_ _Forwarders_, deleting the local zone and clearing the DNS cache should be repeated on all local DNS servers, once that's done the zone will be fully migrated.
