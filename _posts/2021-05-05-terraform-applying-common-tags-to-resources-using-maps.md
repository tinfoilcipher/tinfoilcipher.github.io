---
title: "Terraform - Managing Common Resource Tags"
layout: post
date: 2021-05-05
categories: 
  - "automation"
  - "aws"
  - "azure"
  - "devops"
  - "gcp"
tags: 
  - "automation"
  - "aws"
  - "azure"
  - "cloud"
  - "devops"
  - "gcp"
  - "terraform"
---

**EDIT**: A few days after publishing this article, Hashicorp's official AWS _provider_ was updated to support _default tags_ **directly from the provider** (which is very simple and saves all of the work detailed in this article). This only works with AWS so if you're working in another cloud or for some reason you can't upgrade your provider...keep reading on, if you're only working in AWS and you CAN upgrade, stop reading and take a look at the Hashicorp blog post [here](https://www.hashicorp.com/blog/default-tags-in-the-terraform-aws-provider) which provides some very cool new functionality and is a lot easier to work with!

Given the scale and growth we encounter when working with cloud resources it's essential to work with **[_tags_](https://en.wikipedia.org/wiki/Tag_\(metadata\))** to properly manage the ownership, billing and other arbitrary data for related to our environments and resources (even the naming if you use AWS). Terraform already provides us with an excellent templating syntax in the form of **[HCL](https://www.terraform.io/docs/language/syntax/configuration.html)**, in this post we'll look at how to use pre-constructed _maps_ of tags to assign the same metadata to all resources in a given environment.

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-5.png" class="scaled-img-50">

## A Note on Clouds and Input

Most Terraform _resources_ already allow for the input of Tags, our aim here is to remove the manual input of the same tags for provisioning each different instance of every different _resource_ and replace that with a single standard set.

We'll be working with AWS in these examples, though the same logic also applies to GCP and Azure and applies to the **vast majority** of resources which can be provisioned on all platforms as they all accept tags in a Key: Value pair format.

## Defining Tags as a Map

So let's take a look at how we might define some tags in Terraform with some data that we might expect to use in a typical environment. First we'll _declare_ a single input variable as a _map_:

```terraform
#--variables.tf

variable "default_tags" {
  type        = map
  description = "Map of Default Tags"
}
```

Then in an **[.auto.tfvars](https://www.terraform.io/docs/language/values/variables.html#assigning-values-to-root-module-variables)** we will define the contents of the map:

```terraform
default_tags = {
    Administrator       = "Andy Welsh"
    Department          = "IT"
    CostCentre          = "ABC123"
    ContactPerson       = "andy@tinfoilcipher.co.uk"
    ManagedByTerraform  = "True"
}
```

## Provisioning Resources

Now that we have defined a map, lets create some different resources, a couple of _EC2 Instances_ and an _S3 Bucket_, as we see below instead of defining individual maps of tags we can simply set the **tags** argument as the variable **var.default_tags**:

```terraform
#--main.tf

resource "aws_instance" "private_node" {
    count                       = 2
    availability_zone           = data.aws_availability_zones.available.names[0]
    ami                         = data.aws_ami.tinfoil_ubuntu.id
    instance_type               = "t2.micro"
    key_name                    = "tinfoilkey"
    subnet_id                   = local.subnet_private #--Remote State Lookup
    vpc_security_group_ids      = [local.sg_private] #--Remote State Lookup
    tags                        = var.default_tags
}

resource "aws_s3_bucket" "tinfoil_bucket" {
    bucket          = "tinfoilcipherstorage"
    force_destroy   = true
    server_side_encryption_configuration {
        rule {
            apply_server_side_encryption_by_default {
                sse_algorithm     = "AES256"
            }
        }
    }
    tags = var.default_tags
}
```

If we look at the AWS Console we can see that our tags have applied correctly upon running an **apply**:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-5-1024x216.png)

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-2.png)

## Remote States and Locals

This configuration is fine if we're working with simple resources, but it won't go too far in enterprise deployments when we need to work with proper [remote state lookups]({% post_url 2020-07-10-terraform-looking-up-previous-configuration-data-with-remote-states %}). For these scenarios we should work without defining our data as variable and configure the entire map within **[locals](https://www.terraform.io/docs/language/values/locals.html)**.

Lets take a look at an example of this, but work with some extra data looked up form _variables_:

```terraform
#--variables.tf
variable "environment" {
    type        = string
    description = "Current Environment"
    default     = "dev"
}

data "terraform_remote_state" "tinfoil" {
    backend = "s3"
    config = {
        bucket = "tinfoilbucket"
        key     = "tinfoil-backbone.tfstate"
        region  = "eu-west-1"
    }
}

#locals.tf
locals {
    #--Lookup Remote State Network Data
    subnet_private  = data.terraform_remote_state.tinfoil.outputs.private_subnet_id
    sg_private      = data.terraform_remote_state.tinfoil.outputs.private_sg_id
    tag_admin       = data.terraform_remote_state.tinfoil.outputs.admin_name
    tag_department  = data.terraform_remote_state.tinfoil.outputs.department
    tag_costcentre  = data.terraform_remote_state.tinfoil.outputs.costcentre
    tag_contact     = data.terraform_remote_state.tinfoil.outputs.contactemail

    #--Construct Tag Data
    default_tags = {
        Administrator       = local.tag_admin
        Department          = local.tag_department
        CostCentre          = local.tag_costcentre
        ContactPerson       = local.tag_contact
        ManagedByTerraform  = "True"
        Environment         = var.environment
    }
}
```

In the above example, most of our data is being looked up from a **remote state** and defined as locals (a single additional value named **environment** is being defined as a _variable_). All of these values are then being defined in a _map_ as a new _local_ named **default\_tags**. We can now pass this map directly to any _resource_ as a _local_:

```terraform
resource "aws_instance" "private_node" {
    count                       = 2
    availability_zone           = data.aws_availability_zones.available.names[0]
    ami                         = data.aws_ami.tinfoil_ubuntu.id
    instance_type               = "t2.micro"
    key_name                    = "tinfoilkey"
    subnet_id                   = local.subnet_tinfoil_private
    vpc_security_group_ids      = [local.sg_tinfoil_private]
    tags                        = local.default_tags
}
```

Once again, if we look at the console we can see our new _map_ has been applied correctly:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/04-2-1024x267.png)

## Merging Default Tags and Resource-Specific Tags

Another scenario we might encounter is where we want to want to apply our set of _Default Tags_ to all resources in an environment but also wish to include to some tags which are specific only to a certain resource. Terraform's **[merge](https://www.terraform.io/docs/language/functions/merge.html)** _function_ allows us to manage this additional functionality on a per-_resource_ basis.

The below shows an example of the _merge function_; demonstrating a merge of our existing **default\_tags** _map_ with another _map_ containing a single tag of **Name = "PrivateInstance-${count.index}"**.

```terraform
resource "aws_instance" "private_node" {
    count                       = 10
    availability_zone           = data.aws_availability_zones.available.names[0]
    ami                         = data.aws_ami.tinfoil_ubuntu.id
    instance_type               = "t2.micro"
    key_name                    = "tinfoilkey"
    subnet_id                   = local.subnet_tinfoil_private
    vpc_security_group_ids      = [local.sg_tinfoil_private]
    tags = merge(var.default_tags,{
        Name = "Instance-${var.environment_name}${count.index}"
        },
    )
}
```

This will create each instance with a unique **Name** tag with an incremental number in the suffix of it's _Value_:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/06.png)
