---
title: "Creating, Authenticating and Configuring Elastic Kubernetes Service using Terraform"
layout: post
date: 2020-07-06
categories: 
  - "automation"
  - "aws"
  - "containers"
  - "devops"
tags: 
  - "automation"
  - "aws"
  - "cloud"
  - "containers"
  - "devops"
  - "eks"
  - "integration"
  - "kubernetes"
  - "microservices"
  - "terraform"
---

**NOTE**: The sample code used here is hosted in my GitHub **[here](https://github.com/tinfoilcipher/blogexamples/tree/main/eks-terraform-example)**.

Recently I've been getting my hands dirtier and dirtier with Kubernetes but there's some interesting oddities that only occur in [**Elastic Kubernetes Service (EKS)**](https://aws.amazon.com/eks/), the AWS PaaS Kubernetes platform, especially when it comes to how you can authenticate. As Kubernetes is strongly driven by a declarative (and by extension Infrastructure as Code) philosophy, it makes perfect sense that our deployment and configuration should leverage the correct tools to avoid manual work and Terraform is the perfect tool in the arsenal. In this post I'm going to be looking at how we build, authenticate against and configure our EKS hosted clusters as well as the very specific foibles that we encounter.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-1.png)

## Building the Cluster

Building the cluster itself proves to be a very painless process, almost identical to the other major cloud providers, a standard process of initiating the terraform _provider_ for AWS and provisioning both the **Master** and **Worker Nodes** stands up the environment as we would expect (with some added pain that comes with AWS IAM of course).

In the example below we're going to assume that we've already created:

1. The appropriate IAM roles for **Cluster** and **Nodegroup** permissions, we'll just enter their values at run time (this is broken down in the AWS documentation [here](https://docs.aws.amazon.com/eks/latest/userguide/security_iam_service-with-iam.html))
2. An appropriate IAM Service Account to which we have an **Access Key** and **Secret Access Key**, which we will also enter at run time (we'll need admin rights to EKS and EC2)

```terraform
#--provider.tf

provider "aws" {
    region          = var.region
    access_key      = var.access_key
    secret_key      = var.secret_key
}
```

```terraform
#--variables.tf

#--AWS Authentication
variable "access_key" {
    type        = string
    description = "Access Key"
}

variable "secret_key" {
    type        = string
    description = "Secret Key"
}

variable "cluster_arn" {
    type        = string
    description = "Cluster ARN"
}

variable "node_arn" {
    type        = string
    description = "Node ARN"
}

#--Default AWS Region
variable "region" {
    type        = string
    description = "Primary Location"
    default     = "eu-west-2"
}

#--Subnet IDs, be sure to use some real subnet IDs
variable "subnet_ids" {
    description = "All Subnet IDs"
    type        = list(string)
    default     = ["subnet-XXXXXXXXXXXXXXXX",
                  "subnet-XXXXXXXXXXXXXXXX"]
}

#--All Cluster Names
variable "cluster_group_names" {
    description = "All Cluster Names"
    type        = list(string)
    default     = ["tinfoilcluster"]
}

#--Node Group Names (Source Cluster)
variable "node_group_names" {
    description = "All Node Group Names"
    type        = list(string)
    default     = ["tinfoil_node_group_1",
                  "tinfoil_node_group_2"]
}
```

```terraform
#--main.tf

#--Master Node
resource "aws_eks_cluster" "tinfoil" {
    name     = var.cluster_group_names[0]
    role_arn = var.cluster_arn
    version  = "1.15"
    vpc_config {
        subnet_ids = var.subnet_ids
    }
}

#--Worker Nodes
resource "aws_eks_node_group" "tinfoil" {
    cluster_name    = aws_eks_cluster.tinfoil.name
    count           = length(var.node_group_names)
    node_group_name = var.node_group_names[count.index]
    node_role_arn   = var.node_arn
    subnet_ids      = var.subnet_ids
    instance_types  = ["t3.medium"]

    scaling_config {
        desired_size = 1
        max_size     = 2
        min_size     = 1
    }
}
```

So we have a pretty simple configuration here, when we run **terraform apply** we'll be prompted for the **Cluster** and **Nodegroup Role _ARNs** and the **Access Key** and **Secret Access Key** that we'll use to authenticate with AWS. This configuration will create us a 2 **Worker Node EKS** cluster with some very basic scaling balanced over two subnets (which need to be HA over two **availability zones**).

## EKS Authentication - It's Not Obvious

The first issue here is that we created the cluster (which is EKS) but all of the Terraform modules at our disposal to perform any meaningful configuration are **kubernetes** modules (designed to work directly with kubernetes itself and not to go through the additional layer of abstraction presented by EKS).

So we need to actually get **inside** the cluster. In order to do this **and** since we want the solution to be automated we need to be able to create and configure our cluster in a single, unattended action. This is where EKS begins to stray from the path presented by other PaaS offerings. The solution **is** **technically** documented but it isn't exactly clear and the various documents are spread apart. After a good while of hunting I couldn't really turn up a good guide that explained what needed to be done.

According to the [terraform documentation](https://www.terraform.io/docs/providers/kubernetes/index.html) we at least need to work with two configurations when working with a PaaS kubernetes platform, one for cluster **creation** and then a second when changing authentication context, according to the docs:

<blockquote>
  When using interpolation to pass credentials to the Kubernetes provider from other resources, these resources SHOULD NOT be created in the same `apply` operation where Kubernetes provider resources are also used. This will lead to intermittent and unpredictable errors which are hard to debug and diagnose
  <footer>- https://www.terraform.io/docs/providers/kubernetes/index.html</footer>
</blockquote>

...and you don't want chaotic bugs that suddenly cause errors in production. Also, that is one mouthful of a warning!

So what are our authentication options, according to both the Terraform and Kubernetes official documentation we can either:

- Use a local **kube config** file
- Use **Basic Authentication** (username and password)
- Use an **API Token**
- Use certificate authentication using a **Client Certificate**, **Private Key** and the Clusters' **CA Certificate**

However none of these methods actually work when trying to authenticate against an EKS cluster. Let's get to the bottom of that.

## Configuring Terraform for EKS Cluster Authentication

Passing authentication to an EKS cluster uses webhook token authentication via an endpoint presented from your new EKS cluster and handles the authentication of that token using an implementation of **[aws-iam-authenticator](https://github.com/kubernetes-sigs/aws-iam-authenticator)**.

To make this work, we will need to make sure that we have:

- An appropriate IAM Service Account to which we have an **Access Key** and **Secret Access Key**, which we will also enter at run time (the same account from earlier should be fine)
- The **CA Certificate**, **Endpoint** and **API Token** for our newly created cluster

Those last 3 items are key, as they are what will allow us to directly authenticate using the **kubernetes** provider and from this point use the entire **kubernetes** resource suite to provision and configure kubernetes directly.

Using the rather hidden **aws\_eks\_cluster\_auth** _Data Source_ in conjunction with the **awks\_eks\_cluster** _Data Source_ we can pass the return values we need directly in to the **kubernetes** provider. Let's see how that looks in a **provider.tf** file:

```terraform
#--provider.tf

# THIS PROVIDER WILL BE USED FOR AUTHENTICATION WITH THE DATA SOURCES BELOW
provider "aws" {
    region      = var.region
    access_key  = var.access_key
    secret_key  = var.secret_key
}

# THIS DATA SOURCE WILL RETURN, AMONG OTHER THINGS, YOUR ENDPOINT AND AND CA CERTIFICATE
data "aws_eks_cluster" "tinfoil" {
    name = "tinfoilcluster" #--DEFINE YOUR CLUSTER NAME HERE
}

# THIS DATA SOURCE WILL RETURN, AMONG OTHER THINGS, YOUR API TOKEN
data "aws_eks_cluster_auth" "tinfoil" {
    name = "tinfoilcluster" #--DEFINE YOUR CLUSTER NAME HERE
}

# THE RETURN DATA FROM THE ABOVE DATA SOURCES CAN BE PASSED TO THE PROVIDER BELOW
provider "kubernetes" {
    host                   = data.aws_eks_cluster.tinfoilcluster.endpoint
    cluster_ca_certificate = base64decode(data.aws_eks_cluster.tinfoilcluster.certificate_authority.0.data)
    token                  = data.aws_eks_cluster_auth.tinfoilcluster.token
    load_config_file       = false
}
```

Now when we run **terraform init**, authentication will be completed first with AWS and then directly inside your EKS cluster, from here **kubernetes** resources can be used directly. If we look at the below we can see an example of this in creating some kubernetes _namespaces**:**

```terraform
#--main.tf

#--Build Namespaces
resource "kubernetes_namespace" "tinfoilcluster" {
    count    = length(var.namespace_names)
    metadata {
        name = var.namespace_names[count.index]
        labels = {
          name = var.namespace_names[count.index]
        }
    }
}
```

Now that we have authentication, I'll be looking in some future posts how to manipulate our EKS cluster further.
