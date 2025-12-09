---
title: "Automating Docker deployments in Azure App Services with Terraform"
layout: post
date: 2020-02-29
categories: 
  - "automation"
  - "azure"
  - "containers"
  - "devops"
tags: 
  - "automation"
  - "azure"
  - "cloud"
  - "containers"
  - "devops"
  - "docker"
  - "terraform"
---

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/dockerazure.png)

If you're anything like me, you probably spent years hearing about the wonders of containerisation and didn't know where to start. [Docker](https://www.docker.com/), [Kubernetes](https://kubernetes.io/), [Swarm](https://docs.docker.com/engine/swarm/), [ECS](https://aws.amazon.com/ecs/), [App Services](https://azure.microsoft.com/en-gb/services/app-service/) and [Containers](https://www.docker.com/resources/what-container) are thrown around as almost interchangeable terms and to the uninitiated it's just another wall of terms that means nothing (spoiler: the terms aren't interchangeable and Docker isn't the only game in town, it's just the most popular form of container).

## What the hell is a container anyway?

In a nutshell, a container is a a packaged up deployment of some software that provides:

- The packaged up code that the application is comprised of
- The applications dependencies (libraries, other pieces of dependant software etc.)
- Preconfigured settings and runtime libraries
- Any other tools and files that need to be included

All of this together gives you a perfect standalone application bundle that can run on just about every modern platform (most enterprise Linuxes, Windows and the popular cloud platforms). It should be stressed that while Docker is the most popular of the container platforms, it isn't the only one under the sun, but lets face it, it's the one that everyone loves.

## Isn't that just a VM?

Well in a nutshell no, a great post at StackOverflow [here](https://stackoverflow.com/questions/16047306/how-is-docker-different-from-a-virtual-machine) explains it far better than I ever could, but the ability to destroy, re-provision and critically **scale** containers in a way that VM never could is the killer function, as well as allowing for highly reusable, application level segregation and security that you just can't get in a VM with a fraction of the footprint.

## Deployment - Infrastructure IS Code

It's really more realistic when talking about public clouds to say that Infrastructure **IS** Code. In the example I'm going to use here we're not going to delve in to more advanced concepts like setting up private **Container Registry** (effectively a repository for pre-made container images) and we're just going to use one of the gold standards ([Docker Hub](https://hub.docker.com/)), this is Docker's own Container Registry, and we're going to use a container provide by Docker for testing purposes a simple node.js application called **node-bulletin-board** provided as part of the [Getting Started](https://docs.docker.com/get-started/) guide. For the sake of ease I've added this to my own Docker Hub repository [here](https://hub.docker.com/repository/docker/tinfoilcipher/bulletinboard).

We could set this up in the GUI, but what we want is something fully reusable and with as little human input as possible. The key is immutability and something we can get in source control from the word go.

## Terraform - Doing it all in code

Azure offers cloud-native deployments of **Kubernetes** and **Docker Swarm**, however these strike me as a little over engineered, especially when Azure already offers it's own **App Services** platform which will provide container hosting, scaling and monitoring for less and with the same results, it's also Docker friendly and allows native integration with Docker Hub, Azure's own native [Container Registry](https://azure.microsoft.com/en-gb/services/container-registry/) and the all the other main players for [Private Registries](https://azure.microsoft.com/en-gb/services/container-registry/).

Our end goal should be define the end state in Terraform and achieve, at the execution of a single command:

- A secure connection to azure
- Create a new App Service
- Deploy our container in to the app service
- Host it, securely, to the public internet

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/workflow.png)

This is of course a basic example and lends itself to a scenario that can be scaled for several, if not hundreds of microservice deployments.

## Show me the code

Terraform should be broken down in to 3 files (we can break it down in to fewer or more), but I like to break down in to:

1. **main.tf** (the main Terraform execution)
2. **variables.tf** (defined variables for lookup in **main.tf**)
3. **provider.tf** (definitions of the cloud provider to connect to, in this case, **AzureRM**)

```terraform
#provider.tf
provider "azurerm" {
    subscription_id = var.subscription_id
    tenant_id       = var.tenant_id
    client_id       = var.client_id
    client_secret   = var.client_secret
}
```

```terraform
#variables.tf
###############################
#---Azure Connection Params---#
###############################

#--Primary Location
variable "location" {
    type        = string
    description = "Primary Location"
    default      = "uksouth"
}

#--Subscription
variable "subscription_id" {
    type        = string
    description = "Subscription id"
}

#--Tenant
variable "tenant_id" {
    type        = string
    description = "Tenant id"
}

##############################
#---Auth and Secret Params---#
##############################

#--Service Principle AppID
variable "client_id" {
    type        = string
    description = "Client id"
}

#--Service Principle Secret
variable "client_secret" {
    type        = string
    description = "Client secret"
}

####################
#---Build Params---#
####################

#--Resource Groups
variable "resource_groups" {
    description = "All resource groups"
    type        = list(string)
    default     = ["tinfoil_network_rg",
                 "tinfoil_compute_rg",
                 "tinfoil_storage_rg",
                 "tinfoil_images_rg",
                 "tinfoil_database_rg",
                 "tinfoil_kubernetes_rg",
                 "tinfoil_containers_rg"]
}

#--App Prefix
variable "container_prefix" {
    description = "Common Prefix for Container Resources"
    type        = string
    default     = "tinfoil-app"
}
```

```terraform
#main.tf
resource "azurerm_app_service_plan" "tinfoilcontainers" {
    name                = "${var.container_prefix}-asp"
    location            = var.location
    resource_group_name = var.resource_groups[6]
    kind                = "Linux"
    reserved            = true

    sku {
        tier = "Standard"
        size = "S1"
    }
}

resource "azurerm_app_service" "tinfoilcontainers" {
    name                = var.container_prefix}-appservice"
    location            = var.location
    resource_group_name = var.resource_groups[6]
    app_service_plan_id = azurerm_app_service_plan.tinfoilcontainers.id

    site_config {
        app_command_line = ""
        linux_fx_version = "DOCKER|tinfoilcipher/bulletinboard:1.0"
    }

    app_settings = {
        "WEBSITES_ENABLE_APP_SERVICE_STORAGE" = "false"
        "DOCKER_REGISTRY_SERVER_URL"          = "https://index.docker.io"
    }
}

resource "azurerm_app_service_custom_hostname_binding" "tinfoilcontainers" {
    hostname            = "board.tinfoilcipher.co.uk"
    app_service_name    = "${var.container_prefix}-appservice"
    resource_group_name = var.resource_groups[6]
}
```

There's a few things going on in **main.tf** that are worth pointing out:

- **Line 28**: This is the URL for **Docker Hub**, this will be used as the base location to pull the container image, since we will be using a public repository, we have no need to provide authentication, however options do exist to provide numerous authentication methods.
- **Line** **23**: This is the standard container format for pulling from a container registry, this tells us what container and with what tag we're going to pull, this says we are pulling a **DOCKER** container, from the **tinfoilcipher** user, under the repository **bulletinboard** using the **tag: 1.0**.
- **Lines 32-36**: These define a custom CNAME that will be applied to the website for browsing on the public internet. Azure will perform an internet lookup against the CNAME that you set up in an external DNS system so this record needs to exist **prior** to your execution. By default the URL will be based on the variable **"${var.container\_prefix}-appservice"** meaning the URL will be **https://tinfoil-app-appservice.azurewebsites.net**. However you don't want that on the internet unless you want an abysmal looking website.

Now that we have this, we simply need to initiate terraform (execute the terraform backend plugin(s) as defined in the **provider** block). In this case this is in **provider.tf**, and then apply the configuration as defined in **main.tf**.

```bash
terraform init

terraform apply
```

## What do we get?

Execution completed after 22 seconds and we now have a new **App Service Plan** and **App Service** (which has built based on our container) sitting in out Azure Subscription:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/app-services.png)

Looking at the **App Service** in detail, we can now see that the deployment of the web app built without issue, as well as the log of the application being built from Docker Hub:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/docker.png)

Going further in to **Custom Domains**, we can see that our Custom URL has applied without issue, however it is not TLS secured, being Microsoft we have to provide this in a .pfx format, will cover this in a later post, obviously you never want this in a production system and the generation of a certificate bundle should really be handled as part of a deployment pipeline.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/bindings.png)

Finally, browsing to both URLs, we can see that the site has correctly deployed and is available:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/itsalive-1-1024x492.png">
  <figcaption>It's aliiiiiive!</figcaption>
</figure>

Now wasn't that easier than building a server?
