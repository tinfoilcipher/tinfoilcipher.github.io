---
title: "Ansible - Looking Up Secrets From Azure Key Vault"
layout: post
date: 2020-08-27
categories: 
  - "automation"
  - "azure"
  - "devops"
  - "linux"
  - "security"
tags: 
  - "ansible"
  - "azure"
  - "azurekeyvault"
  - "cloud"
  - "devops"
  - "integration"
  - "linux"
  - "netbox"
  - "secops"
  - "secrets"
  - "security"
---

In previous posts we've looked at [**how to look up Secrets from Hashicorp Vault using Ansible**](/ansible-looking-up-secrets-from-hashicorp-vault/) and [Ansible Tower](/hashicorp-vault-integrating-with-ansible-tower/). We've also taken a look at how to integrate [Azure Key Vault with Ansible Tower](/azure-key-vault-integrating-with-ansible-tower/), however I've never gotten round to taking a look at how to integrate Ansible itself with Azure Key Vault (without the use of Tower).

Whilst I've largley moved away from using Azure Key Vault in favour of Hashicorp Vault I did find myself using it for some tinkering recently so in this post we'll be looking at how to do the integration, luckily it's a pretty painless process.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-4.png)

## Some Assumptions

For this article, I’m going to be working with an existing _Azure Key Vault_ instance and have already configured an appropriate _Service Principal_ to allow programatic access. What we’re working with will be:

1. A single _Azure Key Vault_ instance located at **https://tinfoil-keyvault.vault.azure.net**.
2. This Vault has a single contains a single secret for the purposes of an example, the Secret contains an API token which we'll use to access another platform (in this case Netbox)
3. We’re going to use a _Playbook_ which looks up this token, and then uses this token to create an infrastructure record within Netbox

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-4-1024x268.png)

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-5.png)

## The azure\_keyvault\_secret Plugin and Configuration

Unlike the _hashi\_vault_ lookup _Plugin_ there is no native lookup _Plugin_ shipped with Ansible to interact with _Azure Key Vault_, Microsoft do however provide one which can be either installed via the [**Ansible Galaxy**](https://galaxy.ansible.com/) community repo along with the rest of the Azure Preview Modules or pulled directly from GitHub:

```bash
#--Download Lookup Plugin from GitHub
wget https://github.com/Azure/azure_preview_modules/blob/master/lookup_plugins/azure_keyvault_secret.py

#--Install all Azure Preview Modules from Ansible Galaxy
ansible-galaxy install azure.azure_preview_modules

```

If you have not configured a custom location for your _Lookup Plugins_ in your **ansible.cfg** this file can just be placed alongside your playbook in a directory named **lookup\_plugins**, so your file structure should look like:

```bash
├── _playbook.yaml
├── /lookup_plugins
│   ├── azure_keyvault_secret.py
```

The _Plugin_ also requires that a couple of additional python libraries be installed before use which can be obtained as part of the **azure-keyvault** and **azure-common** packages:

```bash
pip install azure-keyvault azure-common
```

Before getting started, lets be sure to configure some Environment Variables for AZURE\_CLIENT\_ID, AZURE\_SECRET and AZURE\_TENANT. If these aren’t set then we will need to define them inside the playbook which isn’t a secure configuration:

```bash
export AZURE_CLIENT_ID=<Service principal Client ID>
export AZURE_SECRET=<Service principal Secret>
export AZURE_TENANT=<Azure Tenancy ID>
```

## Using The azure_keyvault_secret Plugin

The below examples show the use of the plugin to retrieve a secret and write it out to the console:

```yaml{% raw %}
---
- name: Lookup secret from Azure Key Vault
  hosts: localhost
  gather_facts: false
  connection: local
  tasks:
    - name: Look up secret from Azure Key Vault and output to Console
      vars:
        url: 'https://tinfoil-keyvault.vault.azure.net/'
        secretname: 'netbox-token'
      debug:
        msg: "{{ lookup('azure_keyvault_secret', secretname, vault_url=url) }}"
...
```
{% endraw %}

In the above example, **Line 1**2 is configured to use the lookup to search the _Key Vault_, select a named secret and then print the _Secret_ to the console.

This is of course fine for theory, but returning secret information to the console is just about the worst thing you can do in real world scenarios, so let’s try and do something a little more practical.

## Using The azure_keyvault_secret Plugin – A Practical Example

Now that we’ve seen how to interact with _Secrets_ from our _Key Vault_, lets see how we can use our _Secret_ to perform tasks against another platform, in this case we're going to connect to a Netbox deployment by looking up an API token from our _Key Vault_ and using it to connect to the Netbox API. I’m using one of the great community modules for Netbox from [**@FragmentedPacket**](https://github.com/FragmentedPacket) and [**@amb1s1**](https://github.com/amb1s1) to create a new Netbox Device:

```{% raw %}
---
- name: "Secure Netbox Demo"
  connection: local
  hosts: localhost
  gather_facts: False
    vars:
      url: 'https://tinfoil-keyvault.vault.azure.net/'
      secretname: 'netbox-token'
      api_token: "{{ lookup('azure_keyvault_secret', secretname, vault_url=url) }}"
  tasks:
    - name: Create Netbox Device
      netbox_device:
        netbox_url: https://netbox.madcaplaughs.co.uk
        netbox_token: "{{ api_token }}"
        data:
          name: Test Switch
          device_type: TLSG105S
          device_role: Managed Switch
          site: madcaplaughs
        state: present
...
```
{% endraw %}

The variable being defined on **Line 9** will now perform the lookup against our _Key Vault_ and contain our API token, it can then be used in the **netbox_token** argument on **Line 14**. As both of these are secure inputs our _Secret_ will not be displayed in any return data.
