---
title: "Terraform and Elastic Kubernetes Service - Configuring Scheduled EC2 Autoscaling"
layout: post
date: 2020-10-21
categories: 
  - "automation"
  - "aws"
  - "containers"
  - "devops"
tags: 
  - "automation"
  - "cloud"
  - "devops"
  - "ec2"
  - "eks"
  - "integration"
  - "kubernetes"
  - "terraform"
---

Recently I've been looking at how to configure EC2 autoscaling schedules for EKS implementations, specifically delivering these schedule configurations via Terraform. This sounds like it should be rather simple on the surface but after getting the initial configuration to work an issue of **[idempotency](https://en.wikipedia.org/wiki/Idempotence)** presents itself. In this post I want to look at the issues presented and how to overcome them.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-2.png)

## Autoscaling Groups and Schedules

When an managed EKS Node Group is created via Terraform using the **[aws\_eks\_node\_group](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/eks_cluster)** resource, an autoscaling group is automatically created which contains the EC2 Instances which make up the Node Group in question, the ID of this Node Group is somewhat burried inside the state.

In the below configuration snippet, we are creating a node group named **tinfoilnodegroup1**:

```terraform
resource "aws_eks_node_group" "tinfoilnodegroup1" {
    cluster_name    = "tinfoilcluster1"
    node_group_name = "tinfoilnodegroup1"
    ...
}
```

Our Autoscaling group ID can be returned as the below return value:

```bash
aws_eks_node_group.tinfoilnodegroup1.resources.0.autoscaling_groups.0.name
```

## Configuring an Autoscaling Schedule

Now that we know the ID of the Autoscaling group, we can set up a schedule without any issue, the below configuration shows that the syntax is pretty painless:

```terraform
#--Scale Down Node Group To Zero
resource "aws_autoscaling_schedule" "tinfoilnodegroup1_scaledown" {
    scheduled_action_name   = "tinfoilnodegroup1_scaledown"
    min_size                = 0
    max_size                = 0
    desired_capacity        = 0
    start_time              = "2020-10-21T22:00:00Z"
    end_time                = "2030-10-21T22:00:00Z"
    recurrence              = "00 22 * * *"
    autoscaling_group_name = aws_eks_node_group.tinfoilnodegroup1.resources.0.autoscaling_groups.0.name
}

#--Scale Node Group Back Up
resource "aws_autoscaling_schedule" "tinfoilnodegroup1_scaleup" {
    scheduled_action_name   = "tinfoilnodegroup1_scaleup"
    min_size                = 2
    max_size                = 10
    desired_capacity        = 5
    start_time              = "2020-10-22T08:00:00Z"
    end_time                = "2030-10-21T08:00:00Z"
    recurrence              = "00 08 * * *"
    autoscaling_group_name = aws_eks_node_group.tinfoilnodegroup1.resources.0.autoscaling_groups.0.name
}
```

Critical here is the **recurrence** setting which uses the [cron](https://en.wikipedia.org/wiki/Cron) scheduling syntax (if you're no good at working those out just do yourself a favour and use a favourite site of mine; [crontab.guru](https://crontab.guru/) which will visualise the cron schedules for you).

There's a much bigger problem with this configuration however and that's the previously mentioned problem of [idempotency](https://en.wikipedia.org/wiki/Idempotence). We're using a static timestamp for the **start_time** value, so whilst this will work for our initial configuration it is ill-suited to continuous delivery (which is really the aim of true Infrastructure as Code).

The second we try and run this again our timestamp will probably have passed, so we'll get a nice warning telling us that our start time is in the past, which is something of a weakness to our configuration.

## Making the Configuration Idempotent

We looked in a previous post around **[the use and manipulation of timestamps using Terraform's built in timestamp functions](/terraform-tricks-working-with-timestamps/)** and these can help us to overcome this particular problem by ensuring that when we run this configuration incrementally that the start date is always ahead of our current time:

```terraform
#--Determine Scaling Times
locals {
    now        = timestamp()
    today      = formatdate("YYYY-MM-DD", local.now) #--Format date to YYY-MM-DD
    scale_down = "${local.today}T22:00:00Z" #--Suffix current day with specific scale down time
    scale_up   = timeadd(local.scale_down, "10h") #--Add 10 hours to reach 0800
}

#--Scale Down Node Group To Zero
resource "aws_autoscaling_schedule" "tinfoilnodegroup1_scaledown" {
    scheduled_action_name   = "tinfoilnodegroup1_scaledown"
    min_size                = 0
    max_size                = 0
    desired_capacity        = 0
    start_time              = local.scale_down #--Timestamp configured in locals
    end_time                = "2030-10-21T22:00:00Z"
    recurrence              = "00 22 * * *"
    autoscaling_group_name = aws_eks_node_group.tinfoilnodegroup1.resources.0.autoscaling_groups.0.name
}

#--Scale Node Group Back Up
resource "aws_autoscaling_schedule" "tinfoilnodegroup1_scaleup" {
    scheduled_action_name   = "tinfoilnodegroup1_scaleup"
    min_size                = 2
    max_size                = 10
    desired_capacity        = 5
    start_time              = local.scale_up #--Timestamp configured in locals
    end_time                = "2030-10-21T08:00:00Z"
    recurrence              = "00 08 * * *"
    autoscaling_group_name = aws_eks_node_group.tinfoilnodegroup1.resources.0.autoscaling_groups.0.name
}
```

By using some manipulation of the timestamps, we can ensure that our times for scaling up and down will always be ahead of our execution.
