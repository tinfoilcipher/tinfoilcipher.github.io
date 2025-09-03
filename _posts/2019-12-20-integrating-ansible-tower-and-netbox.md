---
title: "Integrating Ansible Tower and Netbox"
layout: post
date: 2019-12-20
categories: 
  - "automation"
  - "devops"
  - "integrations"
tags: 
  - "ansible"
  - "ansibletower"
  - "api"
  - "automation"
  - "devops"
  - "integration"
  - "netbox"
  - "rest"
---

**UPDATE**: At the time I wrote this the Netbox Collection was still pretty immature, it isn't anymore. If you're trying to do a simple task then you probably just want to go and install the **[Netbox Collection](https://galaxy.ansible.com/netbox/netbox)** from Ansible Galaxy and use the native Modules. You can find the Collection **[here](https://galaxy.ansible.com/netbox/netbox)**!

[**Ansible Tower**](https://www.ansible.com/products/tower) and [**Netbox**](https://netbox.readthedocs.io/en/stable/) are two of my favourite tools, and their integration is **seemingly** painless on the surface (and really it isn't all that bad) but there is a little nuance to it.

Both application stacks provide a RESTful API so sending data between the two should be as simple as firing some JSON between them right? Even with Ansible being a YAML focused platform those data structures are easily parsable to JSON (in fact that happens under the hood on every execution) so where does the issue occur?

The [Ansible Tower](https://docs.ansible.com/ansible-tower/latest/html/towerapi/index.html) and [Netbox](https://netbox.readthedocs.io/en/stable/api/overview/) APIs are very well documented (though the Netbox _Swagger_ documentation is a little wonky when it comes to some specifics, it gets the job pretty well.

The method for interacting with RESTful services within Ansible is via the [**uri**](https://docs.ansible.com/ansible/latest/modules/uri_module.html) module, which specifies that we can send either an external JSON payload or an inline JSON structure (that being a full structure) without much issue, in this sense we should be able to send key value pairs:

```yaml{% raw %}
uri:
  url: https://your_netbox_endpoint
  method: POST
  body_format: form-urlencoded
  headers:
    bearer-token: "{{ YOUR_TOKEN }}"
  body:
    "KEY": VALUE
    "KEY": VALUE
  status_code: 302
```
{% endraw %}

From everything we've **SEEN** of how Ansible handles YAML/JSON parsing, this should work....except it doesn't, the fun of solving an API.

This one seems to require that the entire JSON payload be built as either a file for lookup (using the [**lookup**](https://docs.ansible.com/ansible/latest/plugins/lookup.html) _plugin_), but I'm no fan of that as it leaves a bunch of rogue files lying around that can easily be lost or left out of source control, best to keep everything in code.

So what we really need is a one line JSON structure, that leaves us with:

```yaml{% raw %}
uri:
  url: https://your_netbox_endpoint
  method: POST
  body_format: form-urlencoded
  headers:
    bearer-token: "{{ YOUR_TOKEN }}"
  body: '{ "KEY": "VALUE", "KEY": "VALUE"}'
  status_code: 302
```
{% endraw %}

The more you know...
