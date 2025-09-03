---
title: "Creating Kubernetes Clusters in GCP With Terraform"
layout: post
date: 2020-06-08
categories: 
  - "automation"
  - "containers"
  - "devops"
  - "gcp"
tags: 
  - "automation"
  - "cloud"
  - "containers"
  - "devops"
  - "gcp"
  - "gke"
  - "integration"
  - "kubernetes"
---

Google Cloud Platform tends to be forgotten from the conversation a lot when talking about public cloud offerings, however their hosted Kubernetes offering [GKE (Google Kubernetes Engine)](https://cloud.google.com/kubernetes-engine) has for me been the best of the major offerings for getting to grips with the platform and the best reason to use GCP at all. Without much issue we can get Terraform integrated with GCP, provision and scale out clusters as we need.

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-4.png" class="scaled-img-75">

## GCP Authentication - IAM Service Accounts

Before going any further, we're working with a GCP project named **tinfoilproject**, that's where we're going to make our cluster.

GCP has a particularly nice IAM setup and a dedicated section for **Service Accounts** to help to segregate programmatic access from real users. We'll need to start here to get access to GCP for external systems. We can access this in the GCP Console from **IAM & Admin** \> **Service Accounts**:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-5.png)

From here we can create a new **Service Account**. At **Step 1** we name the account, we're calling the Service Account **terraform-k8s**:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-4-1024x448.png)

Click **Create** and at **Step 2**, define which access the Service Account needs, we're going to grant full access to **GKE**:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/04-3-1024x415.png)

Finally, at **Step 3** click **Create Key** and select **JSON** for a key format, a JSON formatted key file will be downloaded. Be aware that this contains a Private Key and can be used to gain access to your GCP environment and should be handled as a secret. In our examples we have saved the file as **terraformk8s.json**.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/05-3.png)

## Terraform - Integrating the GCP Provider

Now that we have a means of authenticating, we can configure a provider in Terraform. In this example we will be using the **terraformk8s.json** file stored locally, this would not be advisable in most production scenarios unless permissions are incredibly secure (it appears that the secrets can be centrally managed either by using Vault or GCP's own Secrets Engine, but that's for another post).

Below is our configured **provider.tf** which will allow us to execute **terraform init** without issue:

```terraform
#--provider.tf
provider "google" {
    credentials = file("terraformk8s.json")
    project     = "tinfoilproject"
    region      = "europe-west2"
}
```

## Terraform - Cluster Parameters

Before we deploy anything, we'll need to know some basic parameters:

1. 3 Clusters, named **tinfoilcluster01**\-**03**.
2. Each cluster will have a /16 subnet, numbered from **10.1**.**0.0**\-**10.3.0.0**
3. Each Cluster will have a 3 node pool (named **tinfoilnodepool01**\-**03**, using machine type of **n1-standard-1** (1vCPU and 3.75gb RAM), this is pretty high and should be able to take a massive workload.

To allow for future growth or shrinking, we'll define all of this data in **variables.tf** so we can use Terraform's **count** function for iterations. This allows us a great degree of flexibility for future scaling, especially when we need to start considering CI/CD pipelines:

```terraform
#--variables.tf
variable "location" {
  type        = string
  description = "Primary Location"
  default     = "europe-west2-a"
}

variable "cluster_names" {
  description = "All GKE Clusters"
  type        = list(string)
  default = [
    "tinfoilcluster01",
    "tinfoilcluster02",
    "tinfoilcluster03",
  ]
}

variable "cluster_cidr_blocks" {
  description = "All GKE Clusters"
  type        = list(string)
  default = [
    "10.0.0.0/16",
    "10.1.0.0/16",
    "10.2.0.0/16",
  ]
}

variable "node_pool_names" {
  description = "All GKE Node Pools"
  type        = list(string)
  default = [
    "tinfoilnodepool01",
    "tinfoilnodepool02",
    "tinfoilnodepool03",
  ]
}
```

## Terraform - Immutable Cluster Deployment

Now that we know what we want to deploy, we can get the cluster deployed, below is our **main.tf**:

```terraform
#--main.tf
resource "google_container_cluster" "cluster" {
    name     = var.cluster_names[count.index]
    location = var.location

    remove_default_node_pool = true
    initial_node_count       = 1
    cluster_ipv4_cidr        = var.cluster_cidr_blocks[count.index]
    count                    = length(var.cluster_names)

    master_auth {
        username = ""
        password = ""

        client_certificate_config {
            issue_client_certificate = true
        }
    }
}

resource "google_container_node_pool" "nodes" {
    name       = var.node_pool_names[count.index]
    count      = length(var.cluster_names)
    location   = var.location
    cluster    = google_container_cluster.cluster[count.index].name
    node_count = 3

    node_config {
        preemptible  = true
        machine_type = "n1-standard-1"

        metadata = {
            disable-legacy-endpoints = "true"
        }

        oauth_scopes = [
            "https://www.googleapis.com/auth/logging.write",
            "https://www.googleapis.com/auth/monitoring",
        ]
    }
}
```

With all this in place, we can now run **terraform init**, however we can expect something of a delay as the provision of clusters isn't a swift process, however eventually we will see our clusters created:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/06-1.png)

If we drill in to one of the clusters, we can see that default version of Kubernetes has deployed and that the node pools have built:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/08-1.png)

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/07-1.png)

From here we can begin to deploy workloads to create containerised services we can also deploy load balancers and control ingress/egress, however this is all a bit manual so next projects are going to be looking in to other integrations, probably [Helm](https://helm.sh/).
