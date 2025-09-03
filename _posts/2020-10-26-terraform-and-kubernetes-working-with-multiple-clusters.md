---
title: "Terraform and Kubernetes - Working with Multiple Clusters"
layout: post
date: 2020-10-26
categories: 
  - "automation"
  - "aws"
  - "containers"
  - "devops"
  - "gcp"
tags: 
  - "automation"
  - "aws"
  - "cloud"
  - "containers"
  - "devops"
  - "eks"
  - "gke"
  - "integration"
  - "kubernetes"
  - "terraform"
---

In a previous post we looked at the basics of [working with multiple instances of Terraform _providers_](/terraform-tricks-working-with-multiple-provider-instances/), however as usual, Kubernetes presents some slight variations on this theme due to it's varied options for authentication. In this post we're looking at how to handle authentication for multiple Kubernetes clusters in Terraform.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-4.png)

## Provider Aliases

Underpinning all concepts of working with multiple instances of a provider is the concept of working with _provider aliases_. We've looked at _aliases_ in a bit more depth [here](/terraform-tricks-working-with-multiple-provider-instances/) In a nutshell, _aliases_ allow us to run multiple instances of the same _provider_ and reference them against _resources_ that consume that _provider_.

Referencing an alias is best achieved by declaring the cluster name in the _provider_ and referencing it against the _resource_ as **kubernetes**.**<cluster_name>**, I.E. in the below example for creating a Kubernetes _namespace_:

```terraform
resource "kubernetes_namespace" "tinfoilcluster1_services_namespace" {
    provider = kubernetes.tinfoilcluster1
    metadata {
        name = "services"
    }
}
```

Where this becomes interesting with Kubernetes are the myriad ways that Kubernetes can be authenticated against, especially with the various flavours of Kubernetes platforms out there.

## Bare Metal

All flavours of Kubernetes allow for interaction using the local _kube context_, however I've found that this is typically only suited for interactive configuration as it will read the local _KUBECONFIG_. So unless you write an external method to configure a _kube context_ prior to executing your Terraform configuration this is typically a weak solution, however it is supported across the board by configuring the provider as:

```terraform
provider "kubernetes" {
    alias                  = "tinfoilcluster1"
    load_config_file       = true
    config_context         = "tinfoilcluster1"
}

provider "kubernetes" {
    alias                  = "tinfoilcluster2"
    load_config_file       = true
    config_context         = "tinfoilcluster2"
}
```

By default, this assumes your _kube context_ file is located in the default location of **~/.kube/config**, if for some reason this isn't the case, or you're running with two separate config files for safety this can be defined per-provider with the additional setting of **load_config_file**.

The option also exists to authenticate using both username and password (which can be very handy if you're working in a quick dev environment like Minikube):

```terraform
provider "kubernetes" {
    alias            = "tinfoilcluster1"
    host             = "https://192.168.1.10"
    username         = "minikube"
    password         = "minikube"
    load_config_file = false
}

provider "kubernetes" {
    alias            = "tinfoilcluster2"
    host             = "https://192.168.1.11"
    username         = "minikube"
    password         = "minikube"
    load_config_file = false
}
```

Much more preferable is the option to authenticate using a certificate bundle:

```terraform
provider "kubernetes" {
    alias                  = "tinfoilcluster1"
    host                   = "https://192.168.1.10"
    client_certificate     = file("~/.kube/cert.pem")
    client_key             = file("~/.kube/key.pem")
    cluster_ca_certificate = file("~/.kube/ca-cert.pem")
    load_config_file       = false
}

provider "kubernetes" {
    alias                  = "tinfoilcluster2"
    host                   = "https://192.168.1.11"
    client_certificate     = file("~/.kube/cert.pem")
    client_key             = file("~/.kube/key.pem")
    cluster_ca_certificate = file("~/.kube/ca-cert.pem")
    load_config_file       = false
}
```

## AWS - Elastic Kubernetes Service

EKS is particularly fussy in it's authentication **[as we've discussed previously](/creating-authenticating-and-configuring-elastic-kubernetes-service-using-terraform/)**, only one method is supported for authentication using a combination of **Cluster CA Certificate** and **Token**. The below example shows two _provider_ instances connecting to EKS clusters:

```terraform
##############################
# Lookup host and token data #
##############################

#--EKS Cluster Lookups
data "aws_eks_cluster" "tinfoilcluster1" {
    name = "tinfoilcluster1"
}

data "aws_eks_cluster" "tinfoilcluster2" {
    name = "tinfoilcluster2"
}

#--EKS Token Lookups
data "aws_eks_cluster_auth" "tinfoilcluster1" {
    name = "tinfoilcluster1"
}

data "aws_eks_cluster_auth" "tinfoilcluster2" {
    name = "tinfoilcluster2"
}

###########################################
# Configure Providers using Return Values #
###########################################

provider "kubernetes" {
    alias                  = "tinfoilcluster1"
    host                   = data.aws_eks_cluster.tinfoilcluster1.endpoint
    cluster_ca_certificate = base64decode(data.aws_eks_cluster.tinfoilcluster1.certificate_authority.0.data)
    token                  = data.aws_eks_cluster_auth.tinfoilcluster1.token
    load_config_file       = false
}

provider "kubernetes" {
    alias                  = "tinfoilcluster2"
    host                   = data.aws_eks_cluster.tinfoilcluster2.endpoint
    cluster_ca_certificate = base64decode(data.aws_eks_cluster.tinfoilcluster2.certificate_authority.0.data)
    token                  = data.aws_eks_cluster_auth.tinfoilcluster2tinfoilcluster1.token
    load_config_file       = false
}
```

Lookup from the local _kube context_ must be forcible overidden by setting **load\_config\_file** to **false** otherwise a conflict may emerge where both configurations attempt to authenticate at the same time.

## Google Kubernetes Engine

GKE is almost identical in it's authentication method to EKS in that it authenticates using a combination of Cluster **CA Certificate** and **Token**, however the return data structure differs significantly using the Google data sources:

```terraform
##############################
# Lookup host and token data #
##############################

#--GKE Cluster Lookups
data "google_client_config" "tinfoilcluster1" {}

data "google_client_config" "tinfoilcluster2" {}

#--GKE Token Lookups
data "google_container_cluster" "tinfoilcluster1" {
    name     = "tinfoilcluster1"
    location = "europe-west2"
}

data "google_container_cluster" "tinfoilcluster2" {
    name     = "tinfoilcluster2"
    location = "europe-west2"
}

###########################################
# Configure Providers using Return Values #
###########################################

provider "kubernetes" {
  host                   = "https://${data.google_container_cluster.tinfoilcluster1.endpoint}"
  token                  = data.google_client_config.tinfoilcluster1.access_token
  cluster_ca_certificate = base64decode(data.google_container_cluster.tinfoilcluster1.master_auth[0].cluster_ca_certificate,)
  load_config_file       = false
}

provider "kubernetes" {
  host                   = "https://${data.google_container_cluster.tinfoilcluster2.endpoint}"
  token                  = data.google_client_config.tinfoilcluster2.access_token
  cluster_ca_certificate = base64decode(data.google_container_cluster.tinfoilcluster2.master_auth[0].cluster_ca_certificate,)
  load_config_file       = false
}
```
