---
title: "Terraform - Looking Up Previous Configuration Data With Remote States"
layout: post
date: 2020-07-10
categories: 
  - "automation"
  - "aws"
  - "devops"
tags: 
  - "automation"
  - "aws"
  - "cloud"
  - "devops"
  - "json"
  - "s3"
  - "terraform"
---

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-4.png)

## State Files, A quick refresher

State files are used by Terraform's in order to maintain it's "source of truth" on the current state of the configuration that it's responsible for. This serves three main functions:

- Ensuring that any resources that you try to add, remove or change are consistent as they will be checked against what is currently held in the _State File_ before any action is taken
- Ensuring that any manual changes to the environment can be removed automatically by comparing it to the _State File_, [removing configuration drift and enforcing immutability](/immutable-infrastructure-the-what-and-why/)
- Ensuring that an entire environment can be fully re-provisioned in a consistent state with minimal human interaction

_State Files_ are stored as JSON files and generated on a **terraform apply** operation. The can be recorded in an incredibly wide number of Terraform _storage backends_. By default if no _backend_ is defined in your HCL code then terraform will **locally save your _state file_ in your working directory**.

As you don't really want you save your states in source control (due to them saving credentials in plain text), we want to save them in a centrally available and secure location. In the example below I'm using an AWS S3 storage backend configured at the top of my **main.tf** file:

```terraform
#--main.tf
terraform {
    backend "s3" {
        bucket  = "tinfoilbucket" #--Name of the S3 bucket you're savhing to
        key     = "tinfoilnetwork.tfstate" #--Name of your state file
        region  = "eu-west-1" #--Optional region
    }
}
```

In order for the above to work, we will need an appropriately permissioned AWS provider which has access to the S3 bucket we're using.

## Remote States - Looking Up Data

In a lot of videos and forum posts, the function of _Remote States_ is often misunderstood as simply declaring a _remote storage backend_, a _Remote State_ however allows us to load the state file from a previous configuration, use it to lookup values that we know to be held within and then use them as variables in another configuration. This allows either yourself or other teams to consume information about your configuration programmatically and without complex templating.

The easiest way to achieve this is with use of the **Output** command in your source configuration, the below diagram looks at this process in a high level:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-2.png)

A couple of things to understand finally about **Remote States**:

- The **remote_state** _Data Source_ is a read only source, using this is totally safe and presents no risk to overwriting data on the state
- **remote_state** does not affect your **storage backend**, this should still be defined, however each configuration should, obviously, have a unique name
- If your _State File_ is going to be used by a large team and run the risk of being used to read data while it's also being written to, a [DynamoDB](https://aws.amazon.com/dynamodb/) locking table should also be employed to enable _State Locking_, see [here](https://www.terraform.io/docs/backends/types/s3.html) for details on this configuration.

## Remote States - A Practical Example

Lets say you have a team responsible for provisioning your networks, but you need to consume network in a digestible format for use in Terraform, we'll assume that the team creating the networks are also creating with Terraform and giving your access to their state files, our Network state file is going to be the example shown earlier, meaning our state file is going to be named **tinfoilnetwork.tfstate** and located in a bucket named **tinfoilbucket**.

So we have an idea of the data that we're trying to look up, let's have a look at a sample terraform file that our Network team might use and add a couple of _Outputs_ to make life easier:

```terraform
#--main.tf

# Provision Network Resources
resource "aws_vpc" "tinfoil" {
    cidr_block       = "10.0.0.0/16"
    instance_tenancy = "dedicated"
}

resource "aws_internet_gateway" "tinfoil" {
    vpc_id = aws_vpc.tinfoil.id
}

resource "aws_route_table" "tinfoil" {
    vpc_id = aws_vpc.tinfoil.id
    route {
        cidr_block = "0.0.0.0/0"
        gateway_id = aws_internet_gateway.tinfoil.id
    }
}

resource "aws_main_route_table_association" "tinfoil" {
    vpc_id         = aws_vpc.tinfoil.id
    route_table_id = aws_route_table.tinfoil.id
}

resource "aws_subnet" "tinfoil" {
    vpc_id      = aws_vpc.tinfoil.id
    cidr_block  = "10.0.1.0/24"
}

# Output Values
output "tinfoil_vpc_id" {
    value = aws_vpc.tinfoil.id
}

output "tinfoil_subnet_id" {
    value = aws_subnet.tinfoil.id
}
```

At the bottom of the file we see that we're creating 2 output values, these will make our life a lot easier in _Remote State_ lookup.

Let's assume we're a team outside of the Network team, but we need to know the ID's of the VPC and Subnet once they've been created so we can use them when we provision new resources, this is a common scenario which can be easily looked up using the **terraform_remote_state** _data source_, let's take a look below at and example of consuming that data in a new configuration where we want to create an EC2 instance and associated security group:

```terraform
data "terraform_remote_state" "tinfoilnetwork" {
    backend = "s3"
    config = {
        bucket = "tinfoilbucket"
        key     = "tinfoilnetwork.tfstate"
        region  = "eu-west-1"
    }
}

data "aws_ami" "tinfoil" {
    most_recent = true
    filter {
        name   = "name"
        values = ["ubuntu//assets/{{ page.path | split: '/' | last | split: '.' | first }}/hvm-ssd/ubuntu-xenial-16.04-amd64-server-*"]
    }
    filter {
        name   = "virtualization-type"
        values = ["hvm"]
    }
    owners = ["099720109477"] # Canonical
}

resource "aws_security_group" "tinfoil" {
    name        = "tinfoilgroup-ssh"
    description = "Allow SSH inbound traffic"
    vpc_id      = data.terraform_remote_state.tinfoilnetwork.outputs.tinfoil_vpc_id
    ingress {
        description = "SSH to Host from Home"
        from_port   = 22
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = var.egress_cidr_blocks
    }
}

resource "aws_key_pair" "tinfoil" {
    key_name   = "tinfoil-key"
    public_key = var.public_key
}

resource "aws_instance" "tinfoil" {
    ami                     = data.aws_ami.tinfoil.id
    instance_type           = "t2.micro"
    subnet_id               = data.terraform_remote_state.tinfoilnetwork.outputs.tinfoil_subnet_id
    count                   =  2
    key_name                = aws_key_pair.tinfoil.key_name
}
```

A couple of key values to be aware of here are:

- **Lines 1-8**: The **tinfoilnetwork.tfstate** _State File_ is being called using the **terraform_remote_state** _Data Source_ whilst this exposes **all root level modules**, it is easier to work with Outputs.
- **Line 26**: The VPC ID is being looked up from the _remote state_ and used as an input variable
- **Line 44**: The Subnet ID is being looked up from the _remote state_ and used as an input variable

Using the flexibility afforded to us by _Remote States_ allows teams to work much more flexibly and critically allows for much more fluid automation and CI/CD pipelines to be executed on a much wider scale.
