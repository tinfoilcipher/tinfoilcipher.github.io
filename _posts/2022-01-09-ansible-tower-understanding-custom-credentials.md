---
title: "Ansible Tower - Understanding Custom Credentials"
layout: post
date: 2022-01-09
categories: 
  - "devops"
  - "integrations"
  - "security"
tags: 
  - "ansible"
  - "ansibletower"
  - "cloud"
  - "devops"
  - "secops"
  - "secrets"
  - "security"
---

Without a doubt the topic that seems to confuse people the most when using Ansible Tower is working with Credentials. Especially how to pass multiple credentials from either an external _Secret Management_ source (which we've looked at a few times here) or just defining some arbitrary set of credentials and using them in a template.

I get emails about this topic from readers on a fairly regular basis and professionally I've run in to plenty of people that haven't found the system very intuitive, none of this is exactly aided by the **[_Tower_ documentation on the topic](https://docs.ansible.com/ansible-tower/latest/html/userguide/credential_types.html#create-a-new-credential-type)** being a little brief and unclear to the uninitiated. In this post we'll be looking at how to create _Custom Credentials_ in _Tower_ and how to employ them within _Playbooks_.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-3.png)

## What Do We Want?

Let's look at a straight forward scenario, let's say we want to pass a REST API token to Ansible _Task_. _Tower_ probably has a _Credential Type_ for something like this already right? After all there's _Modules_ that allow you to use simple token authentication as an input parameter, but a look at the _Tower_ [_Credential Type_ documentation](https://docs.ansible.com/ansible-tower/latest/html/userguide/credentials.html#credential-types) doesn't show us any such thing. In fact there's little over a dozen _Credential Types_ full stop, that won't go too far. This is where _Custom Credential Types_ come in to play.

## Creating Custom Credentials

_Custom Credentials_ allow us to create a custom set of inputs, optionally obscured at input and then remap them for use in a _Task_, this allows our same _Credential Type_ to be re-used for any number of _Credentials_. Think of _Credential Types_ as a template for _Credentials_ that _Tower_ knows nothing about yet.

Within Tower, we will browse to **Credential Types** and create a new _Credential_:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-2.png)

The structures are broken down in detail below:

```yaml{% raw %}
--# INPUT CONFIGURATION

--# "fields" defines the GUI fields which will be presented, only one is provided
--# and it is mandatory, a string named token which will hold the Secret data initially.
--# It's type is "secret" meaning it will be obscured on input, we have also defined
--# it as "required" meaning it has to be entered. The label "API Tokd en" will be presented
--# in the GUI when entering the secret data.
fields:
  - id: token
    type: string
    label: API Token
    secret: true
required:
  - token

--# INJECTOR CONFIGURATION

--# Here the "token" value is converted to a new variable named rest_api_token.
--# The data defined in the Injector Configuration is available in any Playbook
--# that our Secret is connected to by a Tower Template
extra_vars:
  rest_api_token: '{{ token }}'
```
{% endraw %}

## Creating and Using A Simple Credential

Now that we have a _Credential Type_ defined, let's create a _Credential_ to leverage it. In _Tower_ we can browse to **Credentials** and create a new _Credential_ (in this example I'm going to create an API Token to communicate with my [Netbox](https://netbox.readthedocs.io/en/stable/) deployment):

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-2-1024x381.png)

Now that our _Credential_ has been created we can create a simple _Playbook_ to test that it works:

```yaml{% raw %}
---
- name: "Netbox Demo"
  connection: local
  hosts: localhost
  gather_facts: False

  tasks:
    - name: Create Netbox Device
      netbox_device:
        netbox_url: https://netbox.madcaplaughs.co.uk
        netbox_token: "{{ rest_api_token }}"
        data:
          name: Test Firewall
          device_type: SRX 100B
          device_role: Firewalls
          site: madcaplaughs
        state: present
...
```
{% endraw %}

{% raw %}
...and call the _Playbook_ from a _Template_ in _Tower_. Provided that we attach our new **Netbox-API-Token** _Credential_ to the _Template_, we can implicitly call the new **"{{ rest_api_token }}"** variable from within our _Task_:
{% endraw %}

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/04-3-1024x290.png)

## What About Multiple Secrets?

OK, so this is all fine if we only have a single secret value we ever want to pass, but eventually we'll need to go beyond that. Let's consider another scenario, say we want to pass a username and password for Basic HTTP authentication to an Ansible Task. Hopefully we aren't forced to do that too much but this is the real world and there's plenty of Module support out there. So let's take a look at how we can accommodate that by extending our _Custom Credential_ configuration:


```yaml{% raw %}
--# INPUT CONFIGURATION

--# This time our "fields" will present two input fields showing as "Username" and "Password".
--# Both of these will obscure input for security and both will be mandatory.
fields:
  - id: username
    type: string
    label: Username
    secret: true
  - id: password
    type: string
    label: Password
    secret: true
required:
  - username
  - password

--# INJECTOR CONFIGURATION

--# Here the Injector Configuration will remap our values to two new strings, allowing
--# them to be looked up at runtime as "{{ http_basic_username }}" and "{{ http_basic_password }}"
extra_vars:
  http_basic_username: '{{ username }}'
  http_basic_password: '{{ password }}'
```
{% endraw %}

{% raw %}
...and now when we try to create a new _Credential_ using this _Credential Type_, we will be required to enter two secret values which will be implicitly available in our _Playbooks_ as **"{{ http_basic_username }}"** and **"{{ http_basic_password }}"**.
{% endraw %}

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/05-3-1024x330.png)

A final point that is useful to know. You can add as many fields to you _Credential Types_ as you like and provided that they aren't set as **Required**, you can just leave them blank when creating a _Credential_ without running in to any issues with you _Plays_. This can come in useful when trying to develop your credential management strategy as the limitations of _Custom Credentials_ quickly show themselves and proper planning is essential to avoiding such problems.

Hopefully that clears up some confusion!
