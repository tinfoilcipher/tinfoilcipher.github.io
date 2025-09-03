---
title: "Hashicorp Vault - Integrating with Ansible Tower"
layout: post
date: 2020-04-14
categories: 
  - "devops"
  - "integrations"
  - "linux"
  - "security"
tags: 
  - "ansible"
  - "ansibletower"
  - "api"
  - "devops"
  - "integration"
  - "linux"
  - "rest"
  - "secops"
  - "secrets"
  - "security"
  - "vault"
---

In my recent posts I've covered the [hardened setup of Vault](/hashicorp-vault-secure-installation-and-setup/) and covered the basics of [using the REST API](/hashicorp-vault-tokens-and-the-rest-api/). As we've seen so far, Vault is primarily designed for programmatic interactions from external systems via the API, so lets take a look a favourite of mine; _Ansible Tower_, which is a prime candidate as a third party system which often has a requirement to call secrets from external systems.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-1.png)

For the purposes of this post, we'll be using the KV (Key Value) Secrets Engine (one of two supported with Ansible Tower, the other being Signed SSH) and accessing the Vault **mc-vault.madcaplaughs.co.uk** over TCP port 8200 using HTTPS.

## Ansible Tower Credentials - Native Integration

Ansible Tower has two native Credential types for Hashicorp, if we add a new Credential in Tower we can select **HashiCorp Vault Secret Lookup**, this is the integration with a KV Secret Engine:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/1-2.png">
  <figcaption>KV Integration with Ansible Tower</figcaption>
</figure>

Setting up the Credential further, we will need to specify:

- The URL of the Vault Server
- A **Token** which has appropriate rights to both the Secrets Engine and the API Endpoint **/YOUR\_SECRET\_ENGINE/config**, in our case **/kv/config**
- A definition of if the Secret Engine is KV Version 1 or Version 2

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-3.png)

## Secrets Engine, But no Secrets?

So now we have an integration with the Secrets Engine, but we don't actually have individual secrets, this could be problematic.

Within Tower, we will define a custom **Credential Type** to map custom metadata from the Vault integration and then inject this data in to playbooks when required.

Within Tower, we can browse to **Credential Types** and add a new YAML structure to define our new Credential, this will be used as a template for any credentials that we look up from this Vault:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/3-1-1024x493.png)

Creating this Custom Secret will create a new **Credential Type** named **Hashicorp Vault KV Lookup**.

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
--# vaultkv_secret, this value can be read in to playbooks
extra_vars:
  vaultkv_secret: '{{ secret }}'
```
{% endraw %}

## Looking Up an Individual Secret

Now that we have this means of looking up individual secrets, we still need to load them in to Tower so that they can be assigned to Templates. In this example, I'm going to look up a Secret named **secret1** and obtain it's secret data held in a key named **password** for use in Tower Templates:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/4-2.png)

When attempting to create the new Credential, the option is present to enter the secret data manually, but we want to look up from the Vault which we have already authenticated with, clicking on the "search" icon next to the Secret field will allow us to perform a lookup action against the Vault:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/5-1.png)

The lookup will first ask **which** external system we wish to lookup from (we only have one):

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/6-2.png">
  <figcaption>So many choices...</figcaption>
</figure>

On the next page we need to define metadata for this lookup, and this is where we define data for the lookup:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/7-1.png">
  <figcaption>The path must follow the API location (appropriate to the version of KV in use), the key should be one of the keys held in the Secret. If a version is not specified, the current version will be used by default.</figcaption>
</figure>

The **TEST** button can be used to validate that the configuration is correct and then saved with **OK**.

## Using the Credential in Ansible

Now that a Credential has been passed in to Tower, it can be assigned to any Template in the same manner as any other Credential and used in a playbook, in the below example, I'm using one of the great community modules for Netbox from [@FragmentedPacket](https://github.com/FragmentedPacket) and [@amb1s1](https://github.com/amb1s1) to create a new Device:

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

The variable being called on line 11 can be read (provided that the Credential **secret1-password** has been assigned to the Template calling this _Playbook_) as it will be passed from the **Injector Configuration** of the customer Credential Type we created earlier.

For a look at how to expand on this design to use multiple _Secrets_ take a look at the follow up post [here](/ansible-tower-and-hashicorp-vault-looking-up-multiple-secrets/) which covers that scenario.
