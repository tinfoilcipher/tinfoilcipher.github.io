---
title: "Terraform and Vault - Automating Secret Creation"
layout: post
date: 2020-05-27
categories: 
  - "automation"
  - "devops"
  - "linux"
  - "security"
tags: 
  - "automation"
  - "devops"
  - "linux"
  - "secops"
  - "secrets"
  - "security"
  - "terraform"
  - "vault"
---

When working at scale with secret creation we can employ Vault's [Dynamic Secrets](https://learn.hashicorp.com/vault/getting-started/dynamic-secrets) functions, however another less used and sometimes more flexible option is to leverage Terraform to create secrets at run time, allowing the injection of your secrets from _pseudorandom_ secret generation in to Vault and then using these newly minted secrets further on in the creation process when creating resources in your cloud platform.

Example code for this post can be found in my GitHub [**here**](https://github.com/tinfoilcipher/blogexamples/tree/main/terraform-vault-example).

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-7.png" class="scaled-img-75">

## Some Prerequisites and Gotchas

We're writing secrets in to a _kv_ Secrets Engine over TLS to an existing Vault deployment located at **https://mc-vault.madcaplaughs.co.uk** running over TCP port 8200 (see [here]({% post_url 2020-04-09-hashicorp-vault-secure-installation-and-setup %}) for setting up a secure Vault deployment). The _kv_ Secrets Engine is named **kvstore** and is running as a **Version 1** vault, this is intentional as the Terraform Resource _vault\_generic\_secret_ appears to be restricted to using Version 1 Secrets Engines (if this is not the case and I've just missed something I'd love to know)! We're going to write our Key Value Pairs directly in to an existing _Secret_ named **project _secrets**.

We're not going to be saving an API token anywhere in the Terraform files as this would be a major security no-no, so we'll leave this to be prompted at run time, however if this was going to be executed in a pipeline for full automation we could just as easily do this using an _Environment Variable_.

Finally, this is an example only, due to the way that Terraform stores secrets in **State Files** we want to always ensure that we're saving the states to a secure remote backend (see [here]({% post_url 2020-04-23-terraform-vault-and-azure-storage-centralised-iac-for-cloud-provisioning %}) for notes on setting up a secure remote backend), this example is not using a remote backend and that consideration should be made in the real world.

## Provider and Variables

The first thing we'll need is the Provider configured to access Vault so we can authenticate against an endpoint, we'll also be configuring Variables so we can lookup the _Key_ names we want to write to this endpoint:

```terraform
#--provider.tf

provider "vault" {
    address         = var.vault_url
    skip_tls_verify = false
    token           = var.vault_token
}
```

```terraform
#--variables.tf

# Vault URL
variable "vault_url" {
    description = "Vault URL"
    type        = string
    default     = "https://mc-vault.madcaplaughs.co.uk:8200"
}

# Vault Token
variable "vault_token" {
    description = "Vault Token"
    type        = string
}

#--Resource Groups
variable "secret_keys" {
    description = "Keys (Names) For Secrets"
    type        = list(string)
    default     = ["DefaultLinuxAdmin",
                 "DefaultWindowsAdmin",
                 "DefaultDBAdmin",
                 "DefaultFWAdmin",
                 "ServicePrincipalSecret"]
}
```

We now have enough to work with, we're defining a variable (**vault_url**) which will carry the URL of the _Vault_ instance, another (**vault_token**) which will gather the API token at run time (as it has no _default_ value and a _list_ (**secret_keys**) which will define our JSON Keys for the Key Value Pairs we will be setting for our Secret data.

## Generating Secrets and Sending to Vault

Now that we have Keys generated, we need Values in order to have Key Value Pairs that we can send to Vault in order to create secret data. You will have noticed by now that we have no actual secrets, rather than save these anywhere, we're just going to dynamically generate them using Terraform's **random_password** _Resource_. We can then iterate over both the **secret_keys** list and the returned **random_password** data and send the results to Vault using the **vault_generic_secret** _Resource_:

```terraform
resource "random_password" "azuresecrets" {
    length = 32
    special = true
    count   = "${length(var.secret_keys)}"
}

resource "vault_generic_secret" "azuresecrets" {
    path      = "kvstore/project_secrets"
    count     = "${length(var.secret_keys)}"
    data_json = <<EOT
    {
    "${var.secret_keys[0]}": "${random_password.azuresecrets.0.result}",
    "${var.secret_keys[1]}": "${random_password.azuresecrets.1.result}",
    "${var.secret_keys[2]}": "${random_password.azuresecrets.2.result}",
    "${var.secret_keys[3]}": "${random_password.azuresecrets.3.result}",
    "${var.secret_keys[4]}": "${random_password.azuresecrets.4.result}"
    }
    EOT
}
```

This could probably be templated a lot better, but working within the constraints of JSON and using EOT is always a challenge, answers on a postcard please.

Running **terraform init** will now initialise the backends for both the **Random** and **Vault** providers:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-7.png)

Running **terraform apply** will now iterate over the lists we have created and create the new secrets in Vault.

If we look at the Secrets Engine in Vault, we can confirm that the secrets have indeed been created, using random values:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-6-1024x368.png)

A drawback of this approach is the lack of scaling presented in the manual JSON EOT input which doesn't appear to be overcome by use of _Count_ iteration, if you have a better way to do this then please do get in touch :).
