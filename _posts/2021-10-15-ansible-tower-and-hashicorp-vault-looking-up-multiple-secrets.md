---
title: "Ansible Tower and Hashicorp Vault - Looking Up Multiple Secrets"
layout: post
date: 2021-10-15
categories: 
  - "devops"
  - "integrations"
  - "security"
tags: 
  - "ansible"
  - "ansibletower"
  - "devops"
  - "secops"
  - "secrets"
  - "security"
  - "vault"
---

A question that I've been approached about several times recently is how to lookup multiple _Hashicorp Vault Secrets_ and assign them to a single _Ansible Tower Credential_ for use in a _Playbook_. A while ago I looked at [the process of integrating Hashicorp Vault with Ansible Tower]({% post_url 2020-04-14-hashicorp-vault-integrating-with-ansible-tower %}) (a less that perfect process in the first place) but this has repeatedly led to the same question about multiple _Secrets_, so in this post we'll dive in to that scenario as well as mapping secrets from multiple _Vault Secret Engines_.

Please take a look at the previous article [HERE]({% post_url 2020-04-14-hashicorp-vault-integrating-with-ansible-tower %}) before diving in otherwise this might not be very clear.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-2.png)

## Tower Isn't Very Flexible Here!

The input specification for a _Tower_ _Custom Credential_ is incredibly rigid (it’s defined [here](https://docs.ansible.com/ansible-tower/latest/html/userguide/credential_types.html#create-a-new-credential-type) if you want to see it) and limits your input type to only **strings** and **booleans**. This means sending our secrets as a _list_ or _dictionary_ is out which would let us operate with some grace.

The only real way I’ve found to get around this and maintain some flexibility is to instead define multiple inputs in your _Custom Credential_ _Input Configuration_ and make only the first mandatory (using the _Required_ key). This will give us a _Credential_ type which offers us the ability to look up multiple secrets but only **requires** that we lookup a minimum of one. Below is an example of an input configuration we can use:

```yaml
--# INPUT CONFIGURATION

--# These keys define the GUI fields which will be presented in Tower. One is offered
--# FOR EACH SECRET LOOKUP. Strings  are named Secret1, Secret2 etc. and will hold
--# the Vault Secret data initially.
--# Each has a unique id (lookupsecret1, lookupsecret2 etc.) and their "type" is "secret"
--# meaning they will be obscured on input. As we will always need to work with a minimum
--# of a single secret; we have also set the first secret as "required" meaning it will
--a become a mandatory field in the GUI
fields:
  - id: lookupsecret1
    type: string
    label: Secret1
    secret: true
  - id: lookupsecret2
    type: string
    label: Secret2
    secret: true
  - id: lookupsecret3
    type: string
    label: Secret3
    secret: true
  - id: lookupsecret4
    type: string
    label: Secret4
    secret: true
  - id: lookupsecret5
    type: string
    label: Secret5
    secret: true
required:
  - lookupsecret1
```

The _Injector Configuration_‘s **extra_vars** key is prone to exactly the same rigidity and allows only the output of **strings**, so out goes the idea of constructing a _list_ or _dictionary_ of new _vars_ to pass directly to a _Playbook_ here either, we can however send each to it’s own _variable_. Inside the Playbook these will just be empty values if no secret has been passed.

```yaml{% raw %}
--# INJECTOR CONFIGURATION

--# Here the "lookupsecret" values are converted to a new variables named
--# vault_secret1, vault_secret2 etc., these values can be read in to
--# playbooks
extra_vars:
  vault_secret1: '{{ lookupsecret1 }}'
  vault_secret2: '{{ lookupsecret2 }}'
  vault_secret3: '{{ lookupsecret3 }}'
  vault_secret4: '{{ lookupsecret4 }}'
  vault_secret5: '{{ lookupsecret5 }}'
```
{% endraw %}

## Looking Up Multiple Secrets in Ansible Tower

As with our [previous example]({% post_url 2020-04-14-hashicorp-vault-integrating-with-ansible-tower %}) we can now load a new _Credential_ type in to Tower, however now we have the option to load multiple _secrets_ in to a single _Credential_:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-1-1024x456.png)

For clarity, we'll be using the below secret in _Vault_ our target:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-1.png)

We’re going to perform lookups exactly as in the previous article, but only for **SECRET1**, **SECRET2** and **SECRET3**, leaving the other two Secret Inputs empty (to ensure that this won’t cause any issues), then attach our new _Credential_ to a _Template_ as usual:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/04-2.png)

…we’re then going to run a very primitive _Playbook_ as a Sanity Test to confirm that the _secrets_ are being read by Ansible:

```yaml{% raw %}
---
- name: credentialdebug
  hosts: localhost
  connection: local
  gather_facts: false
  any_errors_fatal: true
  vars:
    vault_lookups:
      - '{{ vault_secret1 }}'
      - '{{ vault_secret2 }}'
      - '{{ vault_secret3 }}'
      - '{{ vault_secret4 }}'
      - '{{ vault_secret5 }}'
  tasks:
    - name: print credentials
      debug:
        msg: | 
          Secret1 is "{{ vault_lookups[0] }}".
          Secret2 is "{{ vault_lookups[1] }}".
          Secret3 is "{{ vault_lookups[2] }}".
          Secret4 is "{{ vault_lookups[3] }}".
          Secret5 is "{{ vault_lookups[4] }}".
...
```
{% endraw %}

Sure enough, we can see the _secrets_ are being read and that defining empty variables hasn’t caused us any problems:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/05-2.png)

## Multiple Secret Engines?

This should be implicit, but it's good to know that this solution also works if we want to look up _Secrets_ from two different _Secret Engines_ in the same _Vault_ deployment and the functionality was vastly enhanced in _Tower_ v3.6 (on which this guide is based).

Below we can see _Secrets_ in two different _Secret Engines_ as an example:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/06-3.png">
  <figcaption>Secret at path /passwords/welsh/pass</figcaption>
</figure>

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/07-2.png">
  <figcaption>Secret at path /tokens/welsh/bearer</figcaption>
</figure>

When populating our _Credential Metadata_ we can simply specify two different "backends" for each of the _Secrets_, as shown below:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/08-1-1024x384.png)

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/10.png)

If you haven't upgraded to _Tower_ 3.6 or later the **Backend** field is not present, simply entering the full path will work I.E. **/password/welsh/pass**

## Preferences Over Variables

If you happened to see my thoughts on the integration with Azure KeyVault then you'll know that I think this process is already ugly enough. I don’t much care for handling the _secrets_ inside _Playbooks_ as 5 separate _variables_ and prefer to convert them to a _list_, this however is down to your own preference and use cases (it’s just a little disappointing the option doesn’t exist out of the gate in the _Injection Configuration_, if anyone finds out a way let me know). A scenario you might want to consider before you rush in to an implementation however is looking up secrets from **multiple _KeyVaults** in your _Templates_ and how that’s going to work for your long term strategy.

If you want to convert your _variables_ in to a _list_ later in your workflow you can do so, working with our simple example we can use:

```{% raw %}
vars:
  vault_lookups:
    - {{ vault_secret1 }}
    - {{ vault_secret2 }}
    - {{ vault_secret3 }}
    - {{ vault_secret4 }}
    - {{ vault_secret5 }}
```
{% endraw %}

…which will allow for the referencing of your secrets at the _play_ level as:

```{% raw %}
---
- name: credentialdebug
  hosts: localhost
  connection: local
  gather_facts: false
  any_errors_fatal: true
  tasks:
    - name: print credentials
      debug:
        msg: | 
          Secret1 is "{{ vault_lookups[0] }}".
          Secret2 is "{{ vault_lookups[1] }}".
          Secret3 is "{{ vault_lookups[2] }}".
          Secret4 is "{{ vault_lookups[3] }}".
          Secret5 is "{{ vault_lookups[4] }}".
...
```
{% endraw %}

Semi-Painless!
