---
title: "Terraform Tricks - Working With Multiple Provider Instances"
layout: post
date: 2020-10-23
categories: 
  - "automation"
  - "aws"
  - "containers"
  - "devops"
tags: 
  - "automation"
  - "cloud"
  - "containers"
  - "devops"
  - "integration"
  - "terraform"
---

One of the lesser known functions of Terraform is the ability to operate multiple instances of the same _provider_ within the same configuration. The uses of this are various though as it's not always needed it's one of those things that doesn't always leap out. It's pretty easy to get to grips with so this is a short post to take a look at how to get started.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-3.png)

## Providers - A Quick Recap

In the world of Terraform, a _Provider_ is used to allow Terraform interface with the API for a Cloud (or other infrastructure) platform and expose it's resources to allow provisioning and configuration from a Terraform configuration. Countless _providers_ exist now both officially and in the community world.

By default when Terraform is initialised providers are automatically installed, however by default a provider is automatically configured with a single static set of configurations, let's take a look at a standard provider configuration for AWS which is looking up it's secret credentials from Vault:

```terraform
provider "vault" {
    skip_tls_verify = false
}

provider "aws" {
    region          = "eu-west-2" #--London
    access_key      = data.vault_generic_secret.aws_credentials_london.data["aws_access_key_id"]
    secret_key      = data.vault_generic_secret.aws_credentials_london.data["aws_secret_access_key"]
}
```

By default any Terraform configurations which use AWS _resources_ will now be created/modified within the **eu-west-2** region (London).

This is all fine for a single region configuration, but what happens if we have some resources in London and some in Ireland (**eu-west-1**), we don't want to have to keep switching between configurations. Luckily the solution already exists in the form of _provider aliases**.**

## Provider Aliases

By using _aliases_ within our _provider_ configurations we can set up multiple instances of the same provider, in the below example we are setting up two AWS providers, one for **eu-west-1** with the alias **Ireland** and one for **eu-west-2** with the alias **London**:

```terraform
provider "vault" {
    skip_tls_verify = false
}

provider "aws" {
    alias           = "Ireland"
    region          = "eu-west-1"
    access_key      = data.vault_generic_secret.aws_credentials_london.data["aws_access_key_id"]
    secret_key      = data.vault_generic_secret.aws_credentials_london.data["aws_secret_access_key"]
}

provider "aws" {
    alias           = "London"
    region          = "eu-west-2"
    access_key      = data.vault_generic_secret.aws_credentials_ireland.data["aws_access_key_id"]
    secret_key      = data.vault_generic_secret.aws_credentials_ireland.data["aws_secret_access_key"]
}
```

Now we can specify which instance of these _provider_ instances we wish to use when provisioning resources. In the below example we'll create a VPC in each region:

```terraform
resource "aws_vpc" "tinfoilvpc_ireland" {
    provider   = aws.ireland
    cidr_block = "10.1.0.0/16"
}

resource "aws_vpc" "tinfoilvpc_london" {
    provider   = aws.london
    cidr_block = "10.2.0.0/16"
}
```

The specific provider is referenced within the main stanza as **<provider\_name>**.**<provider\_instance>**, in this example **aws.ireland** and **aws.london**.
