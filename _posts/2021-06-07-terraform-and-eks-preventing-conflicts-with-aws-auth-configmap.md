---
title: "Terraform and Elastic Kubernetes Service - Preventing Conflicts with the aws-auth ConfigMap"
layout: post
date: 2021-06-07
categories: 
  - "automation"
  - "aws"
  - "containers"
  - "devops"
tags: 
  - "automation"
  - "aws"
  - "containers"
  - "devops"
  - "eks"
  - "integration"
  - "kubernetes"
  - "security"
  - "terraform"
---

Last year I wrote about [**automating Elastic Kubernetes Service role configuration**]({% post_url 2020-07-08-elastic-kubernetes-service-automating-secure-role-configuration-with-terraform-and-vault %}) (direct modification of the **aws-auth** _ConfigMap_) using Terraform, and a somewhat clunky method of injecting ARN data by looking it up from a secret management service (in this case [**Hashicorp Vault**](https://www.vaultproject.io/)).  
  
Whilst the solution works well it comes with a serious built in issue when we want to provision a new deployment from scratch, namely the need to _import_ the _configMap_ before it can be edited which isn't very helpful for an idempotent deployment. If you're here you've probably run in to a lovely error along the lines of '**configmaps "aws-auth" already exists**' when trying to configure your cluster. This is a very quick post to take a deeper look at the cause and see how this issue can be overcome.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01.png)

## If It Works...What's The Problem?

So first of all, technically the solution we presented in the [earlier post]({% post_url 2020-07-08-elastic-kubernetes-service-automating-secure-role-configuration-with-terraform-and-vault %}) does work, but it leaves us with a very fussy, very manual step. The **aws-auth** _configMap_ provides a YAML mapping of which AWS _IAM_ roles and users can authenticate and interact with our Kubernetes cluster running in EKS. As this is a system managed object however we cannot simply start writing to it when it already exists, Terraform is only able to control the objects is creates or existing objects imported in to it's _state_.

Whilst _importing_ is a valuable function of Terraform we cannot really rely on it in an automated provisioning pipeline, especially as it's a one off function for any environment, we could perform it manually but our goal in the long term should really be to try and remove the human element from any provisioning. Working with this in mind we should be trying to avoid _imports_.

## When Does the ConfigMap Get Created?

Initially I had believed that the configMap was created on provisioning of the Kubernetes Control Plane (I.E. the EKS "Cluster", however it is actually created on provisioning of the Data Plane (I.E. the first provisioned EKS "Nodegroup"). With this bit of vital information in mind we should be able to overcome our problem with careful management of build order and dependencies. Let's have a look at the high level steps:

1. Provision our EKS IAM components
2. Provision the EKS Cluster (provisioning the Kubernetes Control Plane and API).
3. Look up the Kubernetes authentication data as a _Data Source_.
4. Use the returned data to connect to the Kubernetes API and create an **aws-auth** _configMap_.
5. Create one or more _Nodegroups_ (provisioning the Kubernetes Data Plane), as the **aws-auth** configMap now already exists, this will be used by the cluster going forward.

## A Note on Resources Vs Modules

The examples we'll be looking at in this post are only for illustrative purposes, I'll be writing out Terraform _resources_ only for the sake of brevity, however in real deployments it is **very strongly recommended** to split the management of the EKS infrastructure components and configuration of Kubernetes resources in to separate modules and not try and work with _resources_ in this manner. This is to say that the creation of our EKS _Cluster_, EKS _Nodegroup_ and **aws-auth** _configMap_ should be carried out in their own, separate modules (rather than _resources_ in a single _module_).

As stated in the Terraform documentation:

<blockquote>
  When using interpolation to pass credentials to the Kubernetes provider from other resources, these resources SHOULD NOT be created in the same Terraform module where Kubernetes provider resources are also used. This will lead to intermittent and unpredictable errors which are hard to debug and diagnose. The root issue lies with the order in which Terraform itself evaluates the provider blocks vs. actual resources.
  <footer>- https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs</footer>
</blockquote>

In my experience, these conditions are rare but can occur, the documentation is well laid out and does explain clearly how to avoid these potential nightmares.

## Building In Dependencies

So let's take a look at how we can force some dependencies to be built in. We'll be building a simple EKS cluster using the configurations from a previous article [here]({% post_url 2020-07-06-creating-authenticating-and-configuring-elastic-kubernetes-service-using-terraform %}) (for the sake of not re-writing all the same code again).

Below is a revised version of our providers.tf in which we've added some hard dependencies to ensure that we won't try and look up the authentication _Data Sources_ until the EKS Cluster has already been provisioned:

```terraform
#--providers.tf snippet

# THIS DATA SOURCE WILL RETURN, AMONG OTHER THINGS, YOUR ENDPOINT AND AND CA CERTIFICATE
data "aws_eks_cluster" "tinfoil" {
    name        = "tinfoilcluster" #--EKS Cluster Name
    depends_on  = [aws_eks_cluster.tinfoil] #--Dependancy, don't attempt lookup until cluster is provisioned
}

# THIS DATA SOURCE WILL RETURN, AMONG OTHER THINGS, YOUR API TOKEN
data "aws_eks_cluster_auth" "tinfoil" {
    name        = "tinfoilcluster" #--EKS Cluster Name
    depends_on  = [aws_eks_cluster.tinfoil] #--Dependancy, don't attempt lookup until cluster is provisioned
}

# THE RETURN DATA FROM THE ABOVE DATA SOURCES CAN BE PASSED TO THE PROVIDER BELOW
provider "kubernetes" {
    host                   = data.aws_eks_cluster.tinfoilcluster.endpoint
    cluster_ca_certificate = base64decode(data.aws_eks_cluster.tinfoilcluster.certificate_authority.0.data)
    token                  = data.aws_eks_cluster_auth.tinfoilcluster.token
    load_config_file       = false
}
```

What we **cannot** do however is set dependencies on our _Data Sources_ for downstream provisioning, this is why it becomes essential in a real world deployment to start considering module design at this point.

Now that we have our data sources we can also create our configMap and Nodegroup with the below configuration, making use of some additional dependencies.

```terraform
#--Write the aws-auth configMap ahead of the first NodeGroup Provision
resource "kubernetes_config_map" "aws_auth" {
    metadata {
        name        = "aws-auth"
        namespace   = "kube-system"
    }
    data = {
        api_host = data.aws_eks_cluster.tinfoilcluster.endpoint
        mapRoles =<<YAML
- groups:
  - system:bootstrappers
  - system:nodes
  rolearn: ${var.node_role_arn}
  username: system:node:{{EC2PrivateDNSName}}
YAML
        mapUsers =<<YAML
- userarn: ${local.userarn}
  username: ${local.username}
  groups:
    - system:masters
YAML
    }
}

resource "aws_eks_node_group" "ng1" {
    cluster_name    = var.cluster_name
    version         = var.k8s_version
    node_group_name = "${var.cluster_name}-nodegroup"
    node_role_arn   = var.node_role_arn
    subnet_ids      = var.vpc_subnets
    instance_types  = var.nodegroup_ec2_size
    scaling_config {
        desired_size = 3
        max_size     = 5
        min_size     = 1
    }
    depends_on = [kubernetes_config_map.aws_auth] #--Ensure that aws-auth configMap is written before provision
}
```

## Conclusion

As stressed, a resource-only driven deployment is far from suitable and working with modules and sub-modules is essential when hopping form AWS to the innards of the EKS cluster.

Working with a proper module driven delivery, this method is fine for a true CI/CD delivery of provisioning and EKS Cluster and configuring it's RBAC and Nodegroup from a single click, but as usual the devil is in the detail and not all that well documented.
