---
title: "Terraform, Vault and Azure Storage - Secure, Centralised IaC for Azure Cloud Provisioning"
layout: post
date: 2020-04-23
categories: 
  - "automation"
  - "azure"
  - "devops"
  - "security"
tags: 
  - "azure"
  - "azurestorage"
  - "cloud"
  - "devops"
  - "integration"
  - "secops"
  - "secrets"
  - "security"
  - "terraform"
  - "vault"
---

In a [previous post](/terraform-vault-and-azure-secure-automated-cloud-deployments/) we've looked at how to build Azure infrastructure with Terraform and handle sensitive secrets by storing them within Vault and looking them up at run time. This however still poses a problem if we're using the default **local** [backend](https://www.terraform.io/docs/backends/index.html) for Terraform; particularly that these secrets will be stored in plain text in the resulting state files and in a local backend they will be absorbed in to source control and visible to any prying eyes. The solution? A remote backend which can be better governed.

The sample code for the this post is hosted in my GitHub **[here](https://github.com/tinfoilcipher/blogexamples/tree/main/terraform-remote-backend-vault-example)**.

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/00-1.png" class="scaled-img-75">

## What are the requirements?

State files are used by terraform to check what has already been created and ratify what actions should and shouldn't be taken on the next apply/plan/graph action taken. To that end it is essential that states be treated with the utmost care and be available when any action is undertaken, a missing (or incorrect) state could mean the difference between altering or destroying an entire environment.

Since secrets are going to end up stored in the state file it is essential that the state files are stored with the following considerations:

- In an encrypted file system
- With strict access controls
- With soft delete/file recovery or version controls

Azure Storage offers all of these via it's **Containers** which allows for the creation of items as BLOBs in an encrypted state with strict access controls with optional soft deletion.

So our ultimate design should look like:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/how-it-should-work-1.png">
  <figcaption>Our ultimate design</figcaption>
</figure>

## Making it happen - Azure Storage

In order to get this in place, we will first need an Azure Storage Account and Storage Container created outside of Terraform.

In this example I'm using the existing Resource Group **tinfoil_storage_rg**, my Container is going to be called **tfstate** and my Storage Account is going to be called **tinfoilterraformbackend**, this isn't a great example for a production Storage Account, and if you're using an environment with a lot of moving parts and multiple states it would serve you better to use some pseudo RNG (in fact the Azure Shell provides this in the form of the **$RANDOM** function E.G. **STORAGE_ACCOUNT_NAME=terraform$RANDOM**).

Below is the code to create the Storage Account and Container using the Azure Shell, either via a remote connection or via the Azure RM integrated shell:

```bash
# Define Variables
RESOURCE_GROUP_NAME=tinfoil_storage_rg
STORAGE_ACCOUNT_NAME=tinfoilterraformbackend
CONTAINER_NAME=tstate

# Create Storage Account with Global Replication for HA
az storage account create --resource-group $RESOURCE_GROUP_NAME --name $STORAGE_ACCOUNT_NAME --sku Standard_GRS --encryption-services blob

# Extract Storage Account Key
ACCOUNT_KEY=$(az storage account keys list --resource-group $RESOURCE_GROUP_NAME --account-name $STORAGE_ACCOUNT_NAME --query [0].value -o tsv)

# Enable soft deletion on a 7 day rolling basis
az storage blob service-properties delete-policy update --days-retained 7 --account-name tinfoilterraformbackend --enable true

#Create Storage Container using Key
az storage container create --name $CONTAINER_NAME --account-name $STORAGE_ACCOUNT_NAME --account-key $ACCOUNT_KEY
```

Once executed, we can now see that the **Storage Account** and **Container** have been created:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/04-4.png">
  <figcaption>As if by magic</figcaption>
</figure>

## Terraform - Backend Integration

Now that a suitable container is in place, we can leverage an existing Service Principal (which should be appropriately stored in a Vault KV Secret Engine as a number of Key Value Pairs) to authenticate. Configuring this in any existing Terraform **main.tf** can be done by adding an additional stanza to the top.

Below is the **main.tf** that we will be using to create the environment. We need only define the **Resource Group**, **Storage Account** and **Container Name**. The **key** value is the name of the state file which we will be creating:

```terraform
###########
# main.tf #
###########

terraform {
    backend "azurerm" {
        resource_group_name   = "tinfoil_storage_rg"
        storage_account_name  = "tinfoilterraformbackend"
        container_name        = "tstate"
        key                   = "terraform.tfstate"
    }
}

# Lookup Vault Secrets - Intended to lookup secrets named azure_terraform_service_principal and
# azure_new_db from a KV type Secret Engine named kv. Each secret contains multiple KVPs
data "vault_generic_secret" "kv-azuresp" {
    path = "kv/azure_terraform_service_principal"
}

data "vault_generic_secret" "kv-azuredb" {
    path = "kv/azure_newdb_creds"
}

# Create Resource Group
resource "azurerm_resource_group" "tinfoil" {
    name        = var.resource_group
    location    = var.location
    tags = {
        Resource = "Group"
    }
}

#--Create Database Servers
resource "azurerm_sql_server" "tinfoil" {
    name                         = var.sql_database_server
    resource_group_name          = var.resource_group
    location                     = var.location
    version                      = "12.0"
    administrator_login          = data.vault_generic_secret.kv-azuredb.data["username"]
    administrator_login_password = data.vault_generic_secret.kv-azuredb.data["password"]
    tags = {
        Resource = "Database"
    }
}

#--Create SQL Databases
resource "azurerm_sql_database" "tinfoil" {
    name                  = var.sql_databases[count.index]
    count                 = length(var.sql_databases)
    resource_group_name   = var.resource_group
    location              = var.location
    server_name           = var.sql_database_server
    tags = {
        Resource = "Database"
    }
}
```

For the sake of inclusion, the **variables.tf** and **provider.tf** are below (these will be critical for completing Vault lookups)

```terraform
################
# variables.tf #
################

#--Azure Location
variable "location" {
    description = "Primary Azure Location"
    type        = string
    default     = "eastus"
}

# Vault URL
variable "vault_url" {
    description = "Vault URL"
    type        = string
    default     = "http://mc-vault.madcaplaughs.co.uk:8200"
}

# Vault Token (For Runtime Input)
variable "vault_token" {
    type        = string
    description = "Vault Token"
}

#--Resource Group
variable "resource_group" {
    description = "All resource groups"
    type        = string
    default     = "tinfoil_database_rg"
}

#--Database Server
variable "sql_database_server" {
    description = "Database Server"
    type        = string
    default     = "tinfoil-database-server"
}

#--Databases
variable "sql_databases" {
    description = "All Databases"
    type        = list(string)
    default     = ["tinfoil_database_prod",
                 "tinfoil_database_dev"]
}

```

```terraform
###############
# provider.tf #
###############

# Define Vault Connection Params
provider "vault" {
    address         = var.vault_url
    skip_tls_verify = false
    token           = var.vault_token
}

# Define Azure Connection Params
provider "azurerm" {
    version = "=2.1.0"
    features {}
    subscription_id = data.vault_generic_secret.kv-azuresp.data["subscription_id"]
    tenant_id       = data.vault_generic_secret.kv-azuresp.data["tenant_id"]
    client_id       = data.vault_generic_secret.kv-azuresp.data["client_id"]
    client_secret   = data.vault_generic_secret.kv-azuresp.data["client_secret"]
}
```

## Execution and Outcome

Running **terraform apply** now prompts for a **Vault** Token and the Secrets are looked up and written to the State File as expected:

```terraform
terraform apply
# var.vault_token
#   Vault Token

  Enter a value: ***********************
```

However the State File is **not** written back in to source control as usual, this time we see it is correctly written in to the Azure Storage backend as a new BLOB, just as we have configured:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/05-4.png">
  <figcaption>State Files redirected to Azure</figcaption>
</figure>

It is obviously critical that the Storage Account and access to the Container are properly permissioned to ensure that only appropriate administrators who can already access the secrets in Vault can access the Azure Storage, otherwise this is all for nothing :)
