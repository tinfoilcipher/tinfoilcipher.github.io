---
title: "Terraform - Centralised State Locking with AWS DynamoDB"
layout: post
date: 2020-09-02
categories: 
  - "automation"
  - "aws"
  - "devops"
tags: 
  - "automation"
  - "aws"
  - "cloud"
  - "devops"
  - "dynamodb"
  - "integration"
  - "s3"
  - "terraform"
---

In a previous post we looked at [setting up centralised Terraform state management using S3]({% post_url 2020-06-01-terraform-vault-and-s3-centralised-iac-for-aws-cloud-provisioning %}) for AWS provisioning (as well as using [Azure Object Storage for the same solution in Azure]({% post_url 2020-04-23-terraform-vault-and-azure-storage-centralised-iac-for-cloud-provisioning %}) before that).
What our S3 solution lacked however is a means to achieve _State Locking_, I.E. any method to prevent two operators or systems from writing to a state at the same time and thus running the risk of corrupting it. In this post we'll be looking at how to solve this problem by creating _State Locks_ using AWS' NoSQL platform; [DynamoDB](https://aws.amazon.com/dynamodb/?trk=ps_a134p000003yOhzAAE&trkCampaign=pac_q2_0520_paidsearch_dynamodb_b_prosp_d_ukir&sc_channel=ps&sc_campaign=pac_q2_2020_paidsearch_dynamodb_ukir&sc_outcome=PaaS_Digital_Marketing&sc_geo=EMEA&sc_country=mult&sc_publisher=Google&sc_category=database&sc_detail=dynamodb&sc_matchtype=e&sc_segment=447305944533&sc_content=dynamodb_e&sc_medium=PAC-PaaS-P|PS-GO|Brand|Desktop|PA|Database|DynamoDB|UKIR|EN|Text&s_kwcid=AL!4422!3!447305944533!e!!g!!dynamodb&ef_id=EAIaIQobChMI-9GEi9-v6wIV5IBQBh2zEQlJEAAYAiAAEgLSAvD_BwE:G:s&s_kwcid=AL!4422!3!447305944533!e!!g!!dynamodb).

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-2.png" class="scaled-img-50">

## The Problem

As it stands our existing solution is pretty strong if we're the only person who's going to be configuring our infrastructures, but presents us with a major problem if multiple people (or in the cause of CI/CD multiple pipelines) need to start interacting with our configurations.

These scenarios present us with a situation where we could potentially see two entities attempting to write to a _State File_ for at the same time and since we have no way right now to prevent that...well we need to solve it.

## The Solution - State Locking

Luckily the problem has already been handled in the form of _State Locking_. If you're running terraform without a _Remote Backend_ you'll have seen the lock being created on your own file system. When a lock is created, an md5 is recorded for the _State File_ and for each lock action, a UID is generated which records the action being taken and matches it against the md5 hash of the _State File_.

This is fine on a local filesystem but when using a Remote Backend _State Locking_ must be carefully configured (in fact only [**some backends don't support _State Locking_ at all**](https://www.terraform.io/docs/backends/types))

## DynamoDB - The AWS Option

When using an S3 backend, Hashicorp suggest the use of a DynamoDB table for use as a means to store _State Lock_ records. The documentation explains the IAM permissions needed for DynamoDB but does assume a little prior knowledge. So let's look at how we can create the system we need, using Terraform for consistency.

For brevity, I won't include the **provider.tf** or **variables.tf** for this configuration, simply we need to cover the _Resource_ configuration for a DynamoDB table with some specific configurations:

```terraform
#--main.tf

resource "aws_dynamodb_table" "tinfoil_tf_state_locking" {
    name                    = "tinfoil_tf_state_locking"
    billing_mode            = "PROVISIONED"
    read_capacity           = 25 #--Max Free Tier
    write_capacity          = 25 #--Max Free Tier
    hash_key                = "LockID"
    attribute {
        name = "LockID"
        type = "S"
    }
    server_side_encryption {
        enabled = true
    }
}
```

A few points to cover here:

- **Line 5**: We're setting the billing mode to **Provisioned** rather than **On-Demand**. For a tedious break down of costing see the **[AWS Documentation](https://aws.amazon.com/dynamodb/pricing/)**.
- **Line 6**\-**7**: Capping **[Read/Write Capacity Units](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadWriteCapacityMode.html)** to 25 a piece should keep our table within AWS' _Free Tier_. Unless you're building enormous infrastructure you shouldn't be seeing too much throughput and your table shouldn't be growing anywhere near the size of the _Free Tier_ limits.
- **Line 8**: Terraform will seek a **Primary Key** named **LockID** to write it's _State Lock IDs_ which we are creating here.
- **Lines 9-12**: Defining the **LockID** key as a **String**
- **Lines 13-15**: Enabling table encryption, because of course

Applying this configuration in Terraform we can now see the table created:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-2.png)

## Implementing DyanmoDB for State Locking

Now that we have our table, we can configure our backend configurations for other infrastructure we have to leverage this table by adding the **dynamodb\_table** value to the _backend stanza_.

If we take a look at the below example, we'll configure our infrastructure to build some EC2 instances and configure the backend to use S3 with our Dynamo _State Locking_ table:

```terraform
#--BACKEND
terraform {
    backend "s3" {
        bucket          = "tinfoil-terraform-backend"
        key             = "ec2_build.tfstate"
        region          = "eu-west-2"
        dynamodb_table  = "tinfoil_tf_state_locking"
    }
}

#--PROVISIONING
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

resource "aws_instance" "tinfoil" {
    ami                     = data.aws_ami.tinfoil.id
    instance_type           = "t2.micro"
    key_name                = "tinfoil-key"
    count                   =  2
    tags = {
        Resource = "Compute"
    }
}
```

If we now try and _apply_ this configuration we should see a _State Lock_ appear in the **DynamoDB** Table:

```bash
terraform apply

# Do you want to perform these actions?
#   Terraform will perform the actions described above.
#   Only 'yes' will be accepted to approve.

Enter a value: yes
# Acquiring state lock. This may take a few moments...
```

During the _apply_ operation, if we look at the table, sure enough we see that the _State Lock_ has been generated:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-3-1024x412.png)

Finally if we look back at our _apply_ operation, we can see in the console that the _State Lock_ has been released and the operation has completed:

```bash
# Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
# Releasing state lock. This may take a few moments...
```

...and we can see that the _State Lock_ is now gone from the Table:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/04-2.png)

Simple as that!
