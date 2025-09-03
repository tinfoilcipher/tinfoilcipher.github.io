---
title: "Managing Kubernetes Secrets with Terraform and Hashicorp Vault"
layout: post
date: 2020-07-24
categories: 
  - "automation"
  - "containers"
  - "devops"
  - "integrations"
  - "linux"
tags: 
  - "automation"
  - "cloud"
  - "containers"
  - "devops"
  - "integration"
  - "kubernetes"
  - "microservices"
  - "secops"
  - "secrets"
  - "security"
  - "terraform"
  - "vault"
---

Recently I've spent a good amount of time looking at options for managing [**Kubernetes Secrets**](https://kubernetes.io/docs/concepts/configuration/secret/) with Vault. Hashicorp being a great supporter of the **[Cloud Native](https://www.hashicorp.com/blog/hashicorp-joins-the-cncf/)** philosophy, it's little surprise to find that they provide a multitude of options to integrate with Kubernetes and provide extensive documentation [**here**](https://www.vaultproject.io/docs/platform/k8s). for my needs I found that the suggested configurations were either unsuitable or required a degree of over-engineering so I'm going to dive in my own solution.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-5.png)

Much documentation is provided to run Vault **within** a cluster, however for operational simplicity I wanted to use a single instance of Vault external to any Kubernetes cluster (even hacking environments such as Minikube or Compose).

## The Suggested Method

Hashicorp's suggested method for this conundrum is documented in detail [**here**](https://learn.hashicorp.com/vault/kubernetes/external-vault), and loosely involves:

- Installing a Vault agent in to each Kubernetes cluster
- Creating a Service Account and configMap on each cluster for Vault
- Configuring Authentication between Vault and each Cluster (based on the allowed authentication method for your implementation in order to request a [Vault API Token](/hashicorp-vault-tokens-and-the-rest-api/))
- Configuring ACLs and Policies in Vault to allow the Vault agent to access the relevant _Secrets Engines_
- Adding Vault-specific annotations to your services/pods which need to access the relevant Secrets

...and if all that goes right, you still don't have your Secrets ingested. All this has let us do is introduce the Secret data in to the pod, we still need to either map the data to the YAML from the injected JSON as detailed [**here**](https://learn.hashicorp.com/vault/kubernetes/sidecar) or configure an additional package as detailed [**here**](https://learn.hashicorp.com/vault/kubernetes/secret-store-driver).

This seems a little painful.

## Kubernetes Secrets - The Work Is Already Done

Kubernetes provides a reasonable method for managing secrets natively, it's not perfect, you still need to ensure your ACLs are locked right down, that TLS transport is being enforced and ensure that **[etcd](https://etcd.io/)** is encrypted at rest. That said as long as proper security precautions are being taken it's at least a serviceable method and the native solution is always a little friendlier if you can use it.

So we still have the problem of where are we going to store our secrets, we can't just write them all in to a cluster from paper every time we want to provision one, let's inject them using Vault using Terraform and get some automation going.

## Vault and Terraform to Create Secrets

A couple of pre-requisites:

- We have an existing Vault instance, running behind a TLS reverse proxy as broken down [here](/hashicorp-vault-secure-installation-and-setup/) and [here](/hashicorp-vault-reverse-proxy-with-nginx/), running at **https://mc-vault.madcaplaughs.co.uk**
- This Vault has a single _kv_ _Secrets Engine_ containing two secrets, each containing different numbers of values
- We're going to be creating secrets in two different namespaces which **already exist**:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-1.png)

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-4.png)

So let's get started by looking at the Terraform Provider configuration and variables files to see how we might connect to both Vault and our Kubernetes cluster:

```terraform
#--provider.tf

provider "vault" {
    address         = var.vault_address
    skip_tls_verify = false
    token           = var.vault_api_token
}

provider "kubernetes" {
    config_context = "minikube"
    load_config_file = "true"
    host = "http://127.0.0.1"
}

#--variables.tf

variable "vault_address" {
    description = "Vault URL"
    type        = string
    default     = "https://mc-vault.madcaplaughs.co.uk"
}

variable "vault_api_token" {
    description = "Vault API Token"
    type        = string
}
```

**NOTE**: I'm using Minikube here, for details on the Kubernetes provider see the documentation [**here**](https://www.terraform.io/docs/providers/kubernetes/index.html) for a real cluster.

Finally, let's consume those secrets from Vault and apply them in a **main.tf** file:

```terraform
#--Lookup Vault Data Sources
data "vault_generic_secret" "mysql" {
    path = "kv/mysql_admin"
}

data "vault_generic_secret" "prometheus" {
    path = "kv/prometheus_secrets"
}

#--Create Kubernetes Secrets
resource "kubernetes_secret" "mysql" {
    metadata {
        name      = "mysql"
        namespace = "dev"
    }
    data = {
        mysqlPassword = data.vault_generic_secret.mysql.data["mysqlPassword"]
    }
    type = "Opaque"
}

resource "kubernetes_secret" "prometheus" {
    metadata {
        name      = "prometheus"
        namespace = "monitoring"
    }

    data = {
        password     = data.vault_generic_secret.prometheus.data["password"]
        bearer_token = data.vault_generic_secret.prometheus.data["bearer_token"]
    }
    type = "Opaque"
}
```

As we see, between lines **2** - **8** we see the Vault _endpoints_ as being looked up as _Data Sources_ and on lines **17**, **29** and **30** we look up the values from these _Data Sources_ to provide to the **kubernetes\_secret** _Resource_.

## Verification

Running **terraform apply** should now create our secrets (and keep them maintained if and when when any changes are made), we should now be able to see this data in our cluster if these Secrets exist.

```bash
terraform apply -auto-approve
# var.vault_api_token
#   Vault API Token

  Enter a value: ***************************
# ...
# ...
# Apply complete! Resources: 2 added, 0 changed, 0 destroyed

kubectl get secrets -n dev mysql -o yaml
# apiVersion: v1
# data:
#   mysqlPassword: d2hhdGtpbmRvZnBlcnNvbgo=
# kind: Secret
# metadata:
#   creationTimestamp: "2020-07-23T18:09:31Z"
#   name: mysql
#   namespace: dev
#   resourceVersion: '10051'
#   selfLink: /api/v1/namespaces/dev/mysql
#   uid: 35a0d91b-4f21-42cc-2214-f21668387556
# type: Opaque

kubectl get secrets -n monitoring prometheus -o yaml
# apiVersion: v1
# data:
#   password: cHV0c2FwYXNzd29yZAo=
#   bearer_token: b250aGVpbnRlcm5ldAo=
# kind: Secret
# metadata:
#   creationTimestamp: "2020-07-23T18:09:33Z"
#   name: prometheus
#   namespace: monitoring
#   resourceVersion: '10052'
#   selfLink: /api/v1/namespaces/monitoring/prometheus
#   uid: 3110a9bb-1f15-1643-ff31-b221f1940528
# type: Opaque
```

As we see, the secrets are now created within the cluster within in their respective _Namespaces_ with the actual secret _Values_ visually obfuscated in base64 format.
