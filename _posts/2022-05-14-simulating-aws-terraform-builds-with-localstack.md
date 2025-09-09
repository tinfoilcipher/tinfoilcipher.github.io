---
title: "Simulating AWS Terraform Builds With Localstack"
layout: post
date: 2022-05-14
categories: 
  - "aws"
  - "devops"
  - "integrations"
tags: 
  - "aws"
  - "cloud"
  - "devops"
  - "integration"
  - "localstack"
  - "terraform"
---

Previously we looked at [using _Localstack_ to emulate AWS services and speed up the feedback loop during development]({% post_url 2022-02-17-emulating-aws-services-using-localstack %}). In this short post we're going to look at how to integrate this tool with _Terraform_ to perform some simple testing that can emulate our builds for free and give us some confidence in our code before running it.

This post will assume that you have _Localstack_ installed and running. If not, see the previous [previous post]({% post_url 2022-02-17-emulating-aws-services-using-localstack %}) for advice on doing so!

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01.png)

## The Problem

The biggest issue I tend to run in to with testing _Terraform_ modules and configurations is that in order to be sure that they really work (and work well), you need to execute them in anger and put them through their paces of testing. Not only is this inevitably a time consuming process, it can be an expensive one, cloud resources aren't free after all.

This might not be the end of the world if you're fairly sure that your code is ready to go, but this is the real world and most of us spend a more time chasing down bugs than we want to admit. All of this debugging comes at a financial cost, not to mention the time it can take. Integrating _Localstack_ in to _Terraform_ allows us to emulate our builds and perform a lot of the debugging faster and at no cost.

## Configuring Terraform (The Manual Solution)

There are two methods of integrating _Terraform_ and _Localstack_. The first (which I prefer) is to reconfigure our AWS _Provider_ file, specifying mock credentials and redirecting all calls:

```terraform
provider "aws" {
    
    #--Standard Provider Configurations
    region     = "eu-west-1"
    default_tags {
        tags = {
            Environment = "dev"
        }
    }
    
    #--Mocked Credentials
    access_key = "test"
    secret_key = "test"

    #--Forces proper S3 path validation. Hopefully this is in use throughout your code!
    s3_force_path_style         = true
    
    #--These settings allow for authentication and other validations which are enforced
    #--in the AWS provider to be bypassed by Localstack.
    skip_credentials_validation = true
    skip_metadata_api_check     = true
    skip_requesting_account_id  = true
    
    #--Redirect Service Endpoints to Localstack. Whilst we won't be any of these it's good
    #--to see how they work and one should be specified to avoid rogue creations.
    #--See the Localstack docs for a full list of suitable endpoints):
    #--https://registry.terraform.io/providers/hashicorp/aws/latest/docs/guides/custom-service-endpoints#available-endpoint-customizations
    endpoints {
        apigateway     = "http://localhost:4566"
        cloudwatch     = "http://localhost:4566"
        ec2            = "http://localhost:4566"
        elasticache    = "http://localhost:4566"
        route53        = "http://localhost:4566"
        s3             = "http://s3.localhost.localstack.cloud:4566" #--This format required for s3_force_path_style = true
    }
}
```

With this configuration in place, we'll "create" a few VPC resources (I won't post the code for the sake of brevity but the example is in GitHub **[here](https://github.com/tinfoilcipher/blogexamples/tree/main/localstack-terraform-example)**). With this ready we can run _Terraform_:

```bash
terraform init
# ...
# Terraform has been successfully initialized!
# ...
terraform validate
# ...
# Success! The configuration is valid.
# ...
terraform apply
# ...
# Apply complete! Resources: 6 added, 0 changed, 0 destroyed.
# ...
terraform destroy
# ...
# Destroy complete! Resources: 6 destroyed.
# ...
```

As we can see, _Terraform_ has "created" and "destroyed" our VPC resources without any real resources being provisioned:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-1024x197.png)

## Using tflocal (The Automated Solution)

If you prefer a simplified solution, the _Localstack_ project now provides a wrapper in the form of **tflocal_** which provides much the same functionality but is not as configurable. Using _tflocal_ we can use a standard provider file without any special configurations, and all localstack modifications should be injected at run time:

```terraform
provider "aws" {
    region  = "eu-west-1"
}
```

To install and run **tflocal**:

```bash
#--Install tflocal
pip install terraform-local

#--Initialise Terraform with Localstack Params
tflocal init
# ...
# Terraform has been successfully initialized!
# ...

#--Apply Terraform with Localstack Params
tflocal apply
# ...
# Apply complete! Resources: 6 added, 0 changed, 0 destroyed.
# ...
tflocal destroy
# ...
# Destroy complete! Resources: 6 destroyed.
# ...
```

As with the manual configuration, _Terraform_ has "created" and "destroyed" our resources.

## Backend Behaviour - An Important Consideration

An important consideration to be aware of is that _Terraform's_ management of resources is tracked using a [State File](https://www.terraform.io/language/state) and this behaviour does not change just because _Localstack_ is introduced. _Localstack_ does have it's own concept of local data and persistence but this does not extend in to the _Terraform Backend_.

As the _Terraform_ _Backend Configuration_ will be observed as usual this must be considered in any workflow you have (for example if you're running _Localstack_ in an automated testing pipeline) it may be much more practical for you to use a separate _State File_ for the purposes of testing (depending on your scenarios or configure a separate _backend_ for this case).

For more information of Terraform Backend configurations, take a look at some previous articles [here]({% post_url 2020-09-02-terraform-centralised-state-locking-with-dynamodb %}) and [here]({% post_url 2020-06-01-terraform-vault-and-s3-centralised-iac-for-aws-cloud-provisioning %}) as well as the extensive [Terraform Documentation.](https://www.terraform.io/language/settings/backends/configuration)
