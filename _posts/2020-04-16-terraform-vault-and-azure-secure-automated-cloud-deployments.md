---
title: "Terraform, Vault and Azure - Secure, Automated Cloud Deployments"
layout: post
date: 2020-04-16
categories: 
  - "automation"
  - "azure"
  - "devops"
  - "security"
tags: 
  - "api"
  - "automation"
  - "azure"
  - "cloud"
  - "devops"
  - "integration"
  - "linux"
  - "rest"
  - "secops"
  - "secrets"
  - "security"
  - "terraform"
  - "vault"
---

Previously I've looked in detail at the uses of two of Hashicorp's offering's; [Terraform](https://www.terraform.io/) and [Vault](https://www.vaultproject.io/). Predictably, the union of these two platforms allows for some ideal ways to further streamline the process of cloud provisioning, in this case by securely handling the myriad secrets needed for cloud shaping and configuration. In this post I'll be looking at a fairly simple configuration to get started.

The sample code for this post is hosted in my GitHub **[here](https://github.com/tinfoilcipher/blogexamples/tree/main/terraform-vault-lookup-example)**[](https://github.com/tinfoilcipher/terraform-vault-lookup-example).

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/0-5.png" class="scaled-img-75">

## Security First - An Important Proviso

When working this way, Terraform will by default store any secrets looked up in **plain text** in the state file, this should be considered carefully when storing states in Source Control and realistically if going down this path, states themselves should also be considered secret data and similarly stored in Vault.

If this is not practical however it may be more realistic to use Terraform's [Remote State](https://www.terraform.io/docs/providers/terraform/d/remote_state.html) functionality which allows for handling states in remote backends and handles states in memory only, this method will **NOT** save secrets in state files.

## Vault - Putting Secrets in the Right Place

Our ultimate aim is to ensure that all sensitive data be stored in Vault and looked up by Terraform, to that end we first need to know what secrets we'll need and make sure they exist in Vault in advance.

Our Terraform creation is going to be a simple creation of some SQL databases in Azure, so this will require two sets of credentials:

1. A Service Principal that we can use to connect to Azure
2. An Administrator username/password to initiate the databases

Within Vault, these Secrets have been created within a _kv_ **Secrets Engine**:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/1-3-1024x266.png">
  <figcaption>Two Secrets ready for use</figcaption>
</figure>

These Secrets each contain multiple Key Value Pairs, representing the various values we will need:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/2-5-1024x276.png">
  <figcaption>KVP entries for the Azure Service Principal</figcaption>
</figure>

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/3-3-1024x208.png">
  <figcaption>KVP entries for the Azure Database Admin Credentials</figcaption>
</figure>

## Terraform - Setting Up Providers

Now that we have the secrets in place, we need to first set up a Terraform's _providers_ to leverage the **vault** and **azurerm** provider modules.

The **vault** provider module is designed to write new Secrets **to** Vault, but by using Terraform's [Data Sources](https://www.terraform.io/docs/configuration/data-sources.html) functionality we can use this same functionality to read existing data from any _Provider._ Below is our **provider.tf** that we will use to integrate with both Vault and AzureRM:

```terraform
####################
#### provider.tf ###
####################

# Define Vault Connection Params
provider "vault" {
    address         = var.vault_url
    skip_tls_verify = true
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

A few key items to break down here so we can see exactly what's happening:

In the **vault** _provider_ stanza, we define the address and a token which has permissions to connect to our Vault instance. These values are being looked up from the **variables.tf** file (as indicated by the fact that they begin with **var.**).

In the **azurerm** _provider_ stanza, the 4 values are being defined from **data** lookups (which are defined in **main.tf** using the **data** source in conjunction with the Vault integrated **generic\_secret** module which allows for lookups from a KV type Secrets Engine). Within this data source we are selecting a Key Value Pair based on it's Key Name (E.G. **tenant\_id**, **client\_id**).

## Terraform - Variables

We now need to see how the **variables.tf** (Terraform variables) are constructed, as a rule where things can be parameterised, they should be:

```terraform
####################
### variables.tf ###
####################

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
    description = "Client secret"
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

Some values to be aware of:

- **Lines 13-17** - Define the URL to the Vault instance and is called in **provider.tf** to connect to Vault
- **Lines 20-23** - Provides a variable for the Vault API token which will be prompted for at **run time** since there is no **default** value provided
- **Lines 40-45** - A list is being used rather than a string so we can create two databases on the same server

## Terraform - Execution

Now we have the **main.tf** (Terraform's main execution file). What we define here is what will be run when we run the **terraform** command:

```terraform
###############
### main.tf ###
###############

# Lookup Vault Secrets
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

The important items to point out here are:

- **Lines 6-8** - The Azure Service Principal is looked up from Vault, using the **data** source, these are then used in **provider.tf** and are used to make the initial connection to AzureRM to perform all tasks
- **Lines 10-12** - The Azure Database Admin Credentials are looked up from Vault, using the **data** source
- **Line 29** - The Value from the **username** Key within the Azure Database Secret is used as the administrator username when creating new databases
- **Line 30** - The Value from the **password** Key within the Azure Database Secret is used as the password for the administrator on new databases

## Putting It All Together

So now that this code is all in place. we should be able to run a simple **terraform init** and **terraform apply**. However as we have intentionally added no **default** value for the **vault\_token** variable, we will now be prompted to enter one:

```bash
terraform init

# Initializing the backend...
# Initializing provider plugins...
# - Checking for available provider plugins...
# - Downloading plugin for provider "azurerm" (hashicorp/azurerm) 2.1.0...
# - Downloading plugin for provider "azurerm" (hashicorp/vault) 2.10.0...

# Terraform has been successfully initialized!

terraform apply
# var.vault_token
#   Vault Token

  Enter a value: ***********************
```

We first see the state file being updated as the Vault sources are looked up and refreshed, this checks in case the contents in Vault have been changed. Finally we will be prompted to proceed and after some time of applying, the resources will be created:

```bash
# data.vault_generic_secret.kv-azuresp: Refreshing state...
# data.vault_generic_secret.kv-azuredb: Refreshing state...

# Do you want to perform these actions?
#   Terraform will perform the actions described above.
#   Only 'yes' will be accepted to approve.

  Enter a value: yes

# Apply complete! Resources: 4 added, 0 changed, 0 destroyed.
```

Now when we look at Azure RM, we can see that the resources have provisioned, all without having to expose the secrets to the HCL files:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/4-5-1024x308.png">
  <figcaption>Success!</figcaption>
</figure>
