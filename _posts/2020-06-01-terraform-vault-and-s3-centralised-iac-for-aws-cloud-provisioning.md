---
title: "Terraform, Vault and S3 - Secure, Centralised IaC for AWS Cloud Provisioning"
layout: post
date: 2020-06-01
categories: 
  - "automation"
  - "aws"
  - "devops"
  - "integrations"
  - "security"
tags: 
  - "aws"
  - "cloud"
  - "devops"
  - "integration"
  - "s3"
  - "secops"
  - "secrets"
  - "security"
  - "terraform"
  - "vault"
---

In a [**previous post**]({% post_url 2020-04-23-terraform-vault-and-azure-storage-centralised-iac-for-cloud-provisioning %}) we've looked at how to build Azure infrastructure with Terraform, handle sensitive secrets by storing them within Vault and centrally manage states within Azure Object Storage (confusingly called **[Containers](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-blobs-introduction)**).

In this post we'll take a look at the same solution but leverage the same technology within AWS, making use of AWS S3 object storage platform and using Terraform to provision further AWS resources.

Sample code for this post can be found in my GitHub at **[here](https://github.com/tinfoilcipher/blogexamples/tree/main/terraform-remote-backend-vault-aws-example)**.

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-1.png" class="scaled-img-75">

## What are the requirements?

_State Files_ are used by Terraform to check what has already been created and ratify what actions should and shouldn't be taken on the next apply/plan/graph/destroy action taken. To that end it is essential that states be treated with the utmost care and be available when any action is undertaken, a missing (or incorrect) state could mean the difference between altering or destroying an entire environment.

Since secrets are going to end up stored in the _State File_ it is essential that they are stored with the following considerations:

- In an encrypted file system
- With strict access controls
- With soft delete/file recovery or version controls

AWS S3 offers all of these options and being an AWS native component it makes perfect sense if we're provisioning AWS infrastructure we should go with it. Versioning is also very robust and offers an edge over Azure Storage Containers which only provides soft deletion (though that will get you far enough in most cases).

So our ultimate design should look like:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-1.png)

## Making it Happen - S3

In order to get this in place, we will first need an S3 _Bucket_ created outside of Terraform which is going to save our _State Files_. Whilst we could technically create the resource with Terraform and migrate the state in to our newly created _Bucket_ after the fact, lets try and keep things simple and create it manually in the GUI for now:

Within in the AWS Console we can browse to the S3 _Service_ and select **Create Bucket** to start the creation wizard:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-1.png)

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/04-1024x405.png">
  <figcaption>In Step 1, provide a name for your _Bucket_. A globally unique name must be entered for your bucket</figcaption>
</figure>

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/05-1024x174.png">
  <figcaption>In Step 2 be sure to enable _Object Versioning_...</figcaption>
</figure>

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/06-1024x266.png">
  <figcaption>...and Object Encryption</figcaption>
</figure>

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/07-1024x311.png">
  <figcaption>In Step 3, unless you have strict requirements, block all public access.</figcaption>
</figure>

Finally in **Step 4**, confirm the creation of your bucket and you should see that it is now created and ready for use:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/09-1024x250.png)

We will of course need to ensure that whatever IAM user we attempt to access AWS with has the relevant permissions to this S3 bucket. The precise details of the granular IAM attributes needed see the Terraform official documentation **[here](https://www.terraform.io/docs/backends/types/s3.html)**.

## Terraform - Backend Integration

Now that a suitable object storage backend in place, we can leverage an existing IAM User (which should be appropriately stored in a Vault KV Secret Engine as a number of Key Value Pairs) to authenticate. Configuring this in any existing Terraform **main.tf** can be done by adding an additional stanza to the top (or by adding an additional **backend.tf** to the working directory).

Below is the **main.tf** that we will be using to create our environment of 10 EC2 instances. The top stanza defines our backend.

Within the _backend_ stanza we need to define the **bucket**, **key** and **region** for our S3 configuration. The **key** value will be the name of our newly created state file:

```terraform
###########
# main.tf #
###########

#--BACKEND
terraform {
    backend "s3" {
        bucket = "tinfoil-terraform-backend"
        key    = "ec2_build.tfstate"
        region = "eu-west-2"
    }
}

# Lookup Vault Secrets - Provided to provider.tf
data "vault_generic_secret" "aws_credentials" {
    path = "kv/aws_credentials"
}

#--PROVISIONING
data "aws_ami" "tinfoil" {
    most_recent = true
    filter {
        name   = "name"
        values = ["ubuntu/images/hvm-ssd/ubuntu-xenial-16.04-amd64-server-*"]
    }
    filter {
        name   = "virtualization-type"
        values = ["hvm"]
    }
    owners = ["099720109477"] # Canonical
}

resource "aws_instance" "tinfoil" {
    ami                     = data.aws_ami.tinfoil.id
    instance_type           = "t2.micro"
    key_name                = "tinfoil-key"
    count                   =  10
    tags = {
        Resource = "Compute"
    }
}
```

Beyond this, we have our **variables.tf** containing a single item and our **provider.tf** configured to look up our AWS credentials from our Vault instance. For flexiblity we have exported both the **$VAULT_ADDR** and **$VAULT_TOKEN** _Environment_ _Variables_ which will be consumed by the Vault _Provider_.

```terraform
###############
# provider.tf #
###############

provider "aws" {
    region          = var.region
    access_key      = data.vault_generic_secret.aws_credentials["aws_access_key_id"]
    secret_key      = data.vault_generic_secret.aws_credentials["aws_secret_access_key"]
}

provider "vault" {
    skip_tls_verify = false
}

################
# variables.tf #
################

#--Region
variable "region" {
    type        = "string"
    description = "Primary Location"
    default      = "eu-west-2"
}
```

## Execution and Outcome

Running **terraform apply** now reads our _Vault_ _Environment Variables_ and the Secrets are looked up. Terraform provisions our environment and a _State File_ is written to our centralised backend as expected:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/10.png)

It is obviously critical that the _Bucket_ is properly permissioned to ensure that only appropriate administrators who can already access the secrets in Vault can access the objects held within, otherwise this is all for nothing :)
