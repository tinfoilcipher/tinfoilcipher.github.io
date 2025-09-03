---
title: "Azure Key Vault - Integrating With Ansible Tower"
layout: post
date: 2020-04-02
categories: 
  - "azure"
  - "devops"
  - "integrations"
  - "security"
tags: 
  - "ansible"
  - "ansibletower"
  - "azure"
  - "azurekeyvault"
  - "cloud"
  - "devops"
  - "integration"
  - "netbox"
  - "secops"
  - "secrets"
  - "security"
---

Recently we looked at [integrating Ansible Tower with Hashicorp Vault](/hashicorp-vault-integrating-with-ansible-tower/), but I thought it would be worth taking a look at another popular Secrets management system, [Azure Key Vault](https://azure.microsoft.com/en-gb/services/key-vault/). Whilst the solution isn't exactly the same using Azure Key Vault and Tower was my first time trying to integrate Ansible with a centralised Secrets repository, so let's take a look at how to achieve the integration as it's not very well documented and the specifics (like some of the handiest functions of Tower) are nice and buried inside the internal Red Hat knowledge base.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-3.png)

## Ansible Tower Credentials – Native Integration

Ahead of time, I've got my Key Vault in Azure named **tinfoil-keyvault** and have granted appropriate permissions for an existing _Service Principal_ to access the Key Vault and have pre-populated it with some secrets:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-4-1024x268.png)

Ansible Tower has a native Credential type for _Azure Key Vault_, if we add a new Credential in Tower we can select **Microsoft Azure Key Vault**, this is the integration with a the Key Vault API:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-3.png)

Setting up the Credential further, we will need to specify:

- The URL of the _Azure Key Vault_ instance.
- The **Client ID** and **Client Secret** of the _Service Principal_ which will be used to access the instance.
- The **Tenant ID** of the Azure _tenancy_

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/04-3-1024x459.png)

## Key Vault, But No Secrets?

So now we have an integration with our _Key Vault_ instance, but we don’t actually have a means to look up the individual secrets to Tower, this could be problematic.

Within Tower, we will define a custom **Credential Type** to map custom metadata from the _Key Vault_ integration and then inject this data in to playbooks when required.

Within Tower, we can browse to **Credential Types** and add a new YAML structure to define our new Credential, this will be used as a template for any credentials that we look up from our _Key Vault_:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/05-1-1024x510.png)

Creating this Custom Secret will create a new **Credential Type** named **Microsoft Azure Key Vault Secret**.

The structures are broken down in detail below:

```yaml{% raw %}
--# INPUT CONFIGURATION

--# The fields define the GUI fields which will be presented, only one is provided
--# and it is mandatory, a string named secret which will hold the Secret data initially
--# it's unique id is "secret" and it's type is "secret" meaning it will be obscured on
--# input, we have also defined it as "required" meaning it has to be entered
fields:
  - id: secret 
    type: string
    label: Secret
    secret: true
required:
  - secret

--# INJECTOR CONFIGURATION

--# Here the "secret" value is converted to a new variable named
--# azurekv_secret, this value can be read in to playbooks
extra_vars:
  azurekv_secret: '{{ secret }}'
```
{% endraw %}

## Looking Up an Individual Secret

Now that we have this means of looking up individual secrets, we still need to load them in to Tower so that they can be assigned to _Templates_. In this example, I’m going to look up a Secret named **secret1** and obtain it’s secret data held in a key named **password** for use in Tower _Templates_:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/06-2.png)

When attempting to create the new _Credential_, the option is present to enter the secret data manually, but we want to look up from our _Key Vault_ instance which we have already authenticated with, clicking on the “search” icon next to the Secret field will allow us to perform a lookup action against our _Key Vault_:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/07-1.png)

The lookup will first ask **which** external system we wish to lookup from (we only have one):

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/08.png)

On the next page we need to define metadata for this lookup. This is where we define the data for our specific _Secret_, defining the name and optionally; the version:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/09-1.png)

The **Test** button can be used to validate that a successful lookup has been made and the **OK** button will save the new _Credential**:**

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/10-1.png)

As we see, the full credential is now created with nested lookup data:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/11.png)

## Using the Credential in Ansible

Now that a Credential has been passed in to Tower, it can be assigned to any Template in the same manner as any other _Credential_ and used in a _Playbook_. In the below example, I’m using one of the great community modules for Netbox from [**@FragmentedPacket**](https://github.com/FragmentedPacket) and [**@amb1s1**](https://github.com/amb1s1) to create a new Netbox Device:

```yaml{% raw %}
---
- name: "Netbox Demo"
  connection: local
  hosts: localhost
  gather_facts: False

  tasks:
    - name: Create Netbox Device
      netbox_device:
        netbox_url: https://mc-apps.madcaplaughs.co.uk
        netbox_token: "{{ vaultkv_secret }}"
        data:
          name: Test Switch
          device_type: TLSG105S
          device_role: Managed Switch
          site: madcaplaughs
        state: present
...
```
{% endraw %}

The variable being called on line 11 can be read (provided that the Credential **secret1** has been assigned to the _Template_ calling this _Playbook_) as it will be passed from the **Injector Configuration** of the _Custom Credential Type_ we created earlier.

Using this method we can still store our data within Tower’s own Fernet encrypted database, however the secrets can still remain managed outside of Ansible in _Azure Key Vault_.

## Looking Up Multiple Secrets

I was eventually asked about the scenario of how to look up multiple _Secrets_ and assign them to the same _Credential_, see the later article [**here**](/ansible-tower-and-azure-keyvault-managing-multiple-secrets/) for an answer on that scenario :).
