---
title: "EKS - Configuring IAM Roles for Service Accounts Using Terraform"
layout: post
date: 2021-09-12
categories: 
  - "automation"
  - "aws"
  - "containers"
  - "devops"
  - "security"
tags: 
  - "automation"
  - "aws"
  - "cloud"
  - "devops"
  - "eks"
  - "kubernetes"
  - "microservices"
  - "s3"
  - "secops"
  - "security"
  - "terraform"
---

This article was going to be a look at how to configure IAM roles to work with EKS _Service Accounts_, however that topic is already well documented in the AWS docs right **[here](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)**. Whilst there's nothing wrong with it in a technical sense, I can't help find it a little clunky, using the **[AWS CLI](https://aws.amazon.com/cli/)** and **[eksctl](https://eksctl.io/)** to get the job done.

I've been pretty unattracted to **eksctl** (though it does offer an easy way to support small deployments) as it has a tendency to leave rogue artifacts in your infrastructure. Personally I favour a Terraform solution to crack this nut so I've written a simple module to deliver the functionality.

The finished module can be found **[on the Terraform Registry here](https://registry.terraform.io/modules/tinfoilcipher/eks-service-account-with-oidc-iam-role/aws/latest)** and this post is going to take a brief look at how it works.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01.png)

## Understanding The Auth Flow

So before we dive in to this, let's understand how the authentication actually works. We'll be using an **OIDC** **Identity Provider** for authentication (which was **[_introduced in EKS Version 1.16_](https://aws.amazon.com/blogs/containers/introducing-oidc-identity-provider-authentication-amazon-eks/)**). The diagram below breaks down how our finished product behaves at a high level and what security is enforced:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-1.png)

1. A Kubernetes _workload_ sends a request to an AWS Service (in this example **S3**) via a _Service Account_.
2. The _Service Account_ uses AWS _STS_ to send a request to an _OIDC Provider_ associated with the EKS Cluster. This issues a short term access token.
3. This token is requested via an AWS _IAM Role_. This role can only be _assumed_ by the Kubernetes _Service Account_, within the _Namespace_ that the request originated.
4. The _IAM Role_ is limited to whatever access is granted by it's associated _IAM Policies_.
5. Access is granted to the requested AWS Service (in this example **S3**) once permissions have been verified.
6. The requested access is granted to the Kubernetes _workload_ that initiated the request.

This approach is very attractive and lets us enforce nice strict security without having to do anything as unpleasant as bake credentials in to our _container_, but the setup process is pretty cumbersome and there's a lot of things that could go wrong when manually creating something with such complexity, so lets take a look at a Terraform option for some templating and automation.

## Getting The OIDC URLs

When an EKS Cluster is created an _OIDC Issuer_ URL is created along with it. This might not be immediately obvious but it's visible in the AWS Console after the creation of a cluster. When creating a cluster via Terraform using the **aws_eks_cluster** resource this URL can be obtained as the return value **aws_eks_cluster.cluster.identity.0.oidc.0.issuer**.

This URL is essential for the creation of the _OIDC Provider_ that we'll be using in our authentication process:

```terraform
resource "aws_iam_openid_connect_provider" "eks" {
    client_id_list  = ["sts.amazonaws.com"]
    thumbprint_list = ["9e99a48a9960b14926bb7f3b02e22da2b0ab7280"]
    url             = aws_eks_cluster.cluster.identity.0.oidc.0.issuer
}
```

**NOTE:** Pay particular attention to the Certificate Thumbprint being passed in the **thumbprint\_list** argument, this is the thumbprint for the AWS Certificate Authority, without this your authentication is going to fall apart!

This resource provides two important return attributes in the form of **arn** and **url** which we can use to inform our new module.

## Creating The IAM Components and Kubernetes Service Account

I won't dive in to the minutia of how the module works for the sake of brevity (the source is [here](https://github.com/tinfoilcipher/terraform-aws-eks-oidc-service-account)), but we can now produce the final product using:

```terraform
module "tinfoil_sa" {
    source                      = "github.com/tinfoilcipher/terraform-aws-eks-oidc-service-account"
    service_account_name        = "tinfoil-sa"
    iam_policy_arns             = ["arn:aws:iam::123456789012:policy/tinfoil-limited-access"]
    kubernetes_namespace        = "tinfoil"
    enabled_sts_services        = ["ec2", "rds", "s3"]
    openid_connect_provider_arn = aws_iam_openid_connect_provider.eks.arn #--Return values from earlier resource
    openid_connect_provider_url = aws_iam_openid_connect_provider.eks.url #--Return values from earlier resource
}
```

The attached policy **tinfoil-limited-access** grants a very limited amount of access to specific _EC2_, _RDS_ and _S3_ services, further limiting the access of our service account.

We need now only run a **terraform apply** to create our resources. As we can see the _IAM Role_ has been created correctly with the appropriate _Trust Relationship_ established. The AWS console shows a friendly breakdown of the generated _Policy Document_:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/04-1-1024x436.png">
  <figcaption>1. The _AWS Services_ that this _Policy Document_ grants access to<br/>2. The ARN of the _OIDC Provider_ used for authentication<br/>3. The allowed k8s _Service Account_ and _Namespace_ allowed to use this OIDC Provider</figcaption>
</figure>

A Terraform **[Dynamic Block](https://www.terraform.io/docs/language/expressions/dynamic-blocks.html)** is leveraged to ensure that a new _IAM Statement_ is generated for each _AWS Service_ defined against the **enabled_sts_services** argument, the below snippet shows this broken down:

```terraform
data "aws_iam_policy_document" "this" {
    ...
    dynamic "statement" {
        for_each = var.enabled_sts_services #--A for_each loop iterates over the enabled_sts_services list
        iterator = enabled_sts_services #--enabled_sts_services is defined as the loop iterator
        content {
            sid = "GrantSTS${upper(enabled_sts_services.value)}" #--The iteration is switched to uppercase for the statement SID            actions = ["sts:AssumeRole"]
            principals {
                type        = "Service"
                identifiers = ["${enabled_sts_services.value}.amazonaws.com"] #--The iteration is used to grant STS access
            }
            effect = "Allow"
        }
    }
    ...
}
```

Finally, if we examine the Kubernetes Service Account, we can see that the IAM Role has been properly assigned:

```bash
kubectl get serviceaccount tinfoil-sa -n tinfoil -o yaml
```

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/05.png">
  <figcaption>Account ID redacted for the sake of...me</figcaption>
</figure>

Any workload using this Service Account can now implicitly access the relevant AWS Services.

Simple!
