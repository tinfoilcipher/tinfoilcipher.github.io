---
title: "Terraform Tricks - Working With AWS Subnets"
layout: post
date: 2025-01-21
categories: 
  - "aws"
  - "devops"
tags: 
  - "aws"
  - "cloud"
  - "devops"
  - "networking"
  - "terraform"
---

If you're using Terraform in AWS you'll very quickly find yourself needing to work with **[AWS Subnets](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html)**. This can be a surprisingly fussy and in a lot of Terraform configs you tend to see the same solution being employed; inputting a list of _Subnet IDs_ and CIDRs as a variables. Whilst there isn't exactly anything wrong with this and it **does** work, it can be a bit clunky, messy and sometimes just impractical if you are working with a complicated network. This article is going to look at a couple of simple methods for looking up subnet information and manipulating it.

The code used in these examples can be found in Github **[here](https://github.com/tinfoilcipher/blogexamples/tree/main/aws-subnet-data-sources)**.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-1.png)

## Data Sources and Tags

The foundation of working with subnet information is going to be using **[Terraform Data Sources](https://developer.hashicorp.com/terraform/language/data-sources)_,** **[_AWS Tags_](https://docs.aws.amazon.com/whitepapers/latest/tagging-best-practices/what-are-tags.html)** and some basic filtering. The philosophy here is that your subnets already exists as resources in the environment that you're working with and to that end you shouldn't have to waste time looking up their IDs or CIDRs manually just to paste them in to your code when the whole point of Terraform is to do this stuff dynamically.

Terraform provides us some built in _Data Sources_ which we can use to look up VPC and subnet data which we can then **[filter](https://registry.terraform.io/providers/fortinetdev/fortios/latest/docs/guides/fgt_filter)** using _Tags_, to this end it is essential that your subnets are suitably tagged. The main use case for tagging subnets tends to be identifying them as either [**public or private**](https://docs.aws.amazon.com/whitepapers/latest/tagging-best-practices/what-are-tags.html) (though really you can identify them as whatever you happen to be using them for). As an aside, if you are using the very useful official [**VPC module**](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest), there are input variables to add tags to your public and private subnets automatically.

The below example will look up data about our VPC and subnets:

```terraform
#--THIS DATA SOURCE WILL LOOK UP ALL VPC DATA, BASED ON VPC NAME
data "aws_vpc" "vpc" {
  filter {
    name   = "vpc-id"
    values = ["my-vpc"] #--ENTER THE NAME OF YOUR VPC HERE
  }
}

#--THIS DATA SOURCE WILL LOOK UP SUBNET IDS FOR SUBNETS TAGGED AS tier: public
data "aws_subnets" "public_subnets" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.vpc.id]
  }
  tags = {
    "tier" = "public"
  }
}

#--THIS DATA SOURCE WILL LOOK UP SUBNET IDS FOR SUBNETS TAGGED AS tier: private
data "aws_subnets" "private_subnets" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.vpc.id]
  }
  tags = {
    "tier" = "private"
  }
}
```

## Using Subnet IDs

So now that our data is loaded as a data source, how can it be used by a _Resource_? For scenarios where we just need subnet IDs this is pretty straight forward, taking the example of creating a new loadbalancer below:

```terraform
#--CREATE A PUBLIC FACING APPLICATION LOADBALANCER
resource "aws_lb" "alb" {
  name               = "alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = var.your_security_group_id
  subnets            = data.aws_subnets.public_subnets.ids
}

#--CREATE AN INTERNAL APPLICATION LOADBALANCER
resource "aws_lb" "alb" {
  name               = "alb"
  internal           = true
  load_balancer_type = "application"
  security_groups    = var.your_security_group_id
  subnets            = data.aws_subnets.private_subnets.ids
}

```

In scenarios where we need to iterate over the IDs rather than pass a complete list, we can do this most easily with a simple **count** operation, in the example below we will apply this to creating a new EC2 instance in each _private subnet_:

```terraform
#--CREATE AN EC2 INSTANCE IN EACH OF YOUR PRIVATE SUBNETS
#--NAMED INSTANCE-1, INSTANCE-2 ETC.
resource "aws_instance" "instance" {
    count               = length(data.aws_subnets.private_subnets.ids)
    ami                 = var.your_ami_id
    instance_type       = "t2.micro"
    key_name            = var.your_key_name
    subnet_id           = data.aws_subnets.private_subnets.ids[count.index]
    security_groups     = [var.your_security_group_id]
    tags                = {
        Name = "instance-${count.index}"
    }
}
```

## Working with Subnet CIDRs

So working with IDs is straight forward enough, but what about the CIDRs that correspond with our subnets? They are a bit trickier and involve further manipulation via another _Data Source_. This is because the **aws\_subnets** _Data Source_ doesn't actually contain the CIDR as a return value, it ONLY returns subnet IDs. We need to pass it's return values in to the confusingly named **aws\_subnet** _Data Source_ and then iterate over **THAT** to get any more useful values (such as the CIDR):

```terraform
#--THIS DATA SOURCE WILL LOOK UP ALL VPC DATA, BASED ON VPC NAME
data "aws_vpc" "vpc" {
  filter {
    name   = "vpc-id"
    values = ["my-vpc"] #--ENTER THE NAME OF YOUR VPC HERE
  }
}

#--THIS DATA SOURCE WILL LOOK UP SUBNET IDS FOR SUBNETS TAGGED AS tier: private
data "aws_subnets" "private_subnets" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.vpc.id]
  }
  tags = {
    "tier" = "private"
  }
}

#--THIS DATA SOURCE WILL LOOK UP INDIVIDUAL SUBNET DATA, BASED ON INDIVIDUAL SUBNET ID.
#--USE OF THE COUNT ARGUMENT WILL LOOK UP DATA FOR EACH OF THE SUBNET IDS TAGGED AS tier: private
data "aws_subnet" "private_subnets" {
  count  = length(data.aws_subnets.private_subnets.ids)
  vpc_id = data.aws_vpc.vpc.id
  id     = data.aws_subnets.private_subnets.ids[count.index]
}
```

Working with the data above, we can now pass our CIDRs directly to a _Resource_. In the example below we can see this employed in the creation of a new _Security Group_ and _Ingress Rule_:

```terraform
resource "aws_security_group" "sg" {
  vpc_id = data.aws_vpc.vpc.id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "example-sg"
  }
}

#--THE cidr_block ATTRIBUTE FOR EACH SUBNET IS PASSED TO THE cidr_ipv4 INPUT
#--THIS CREATES AN INGRESS RULE IN THE SECURITY GROUP FOR EACH OF OUR SUBNET CIDRS
resource "aws_vpc_security_group_ingress_rule" "dns" {
  count             = length(data.aws_subnet.private_subnets)
  security_group_id = aws_security_group.sg.id
  cidr_ipv4         = data.aws_subnet.private_subnets[count.index].cidr_block
  from_port         = 53
  ip_protocol       = "udp"
  to_port           = 53
}
```

The [aws_subnet](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/subnet) _Data Source_ has a few other return values that can be handy, but this is by far the most useful that I've come across and the most convenient way of looking up and manipulating subnet data. This is all technically documented in the Terraform docs but it seems to be quite poorly understood so hopefully someone finds this useful!
