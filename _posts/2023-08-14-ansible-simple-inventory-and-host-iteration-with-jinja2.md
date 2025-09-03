---
title: "Ansible - Simple Inventory and Host Iteration with Jinja2"
layout: post
date: 2023-08-14
categories: 
  - "automation"
  - "devops"
tags: 
  - "ansible"
  - "automation"
  - "devops"
---

Recently I had cause to revisit a topic that often seems to cause problems for people coming to Ansible for the first time, especially for people (like me) who don't have a development background. How to iterative over inventory variables or facts using a simple Jinja2 template.

It can be a fussy task to get your head around and the documentation isn't the greatest to the newcomer, so this is going to be a very short post to try and break down the issue with as little jargon as possible.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01.png)

## What's The Problem? A Real World Scenario

One of the most common things we want to do with Ansible is build a configuration file for a remote server. Ideally, we want to do this dynamically based on data from our _Inventory_ but accessing and iterating over the inventory can be a little tricky to understand.

In this example, we'll be working with an inventory with 5 hosts named **webserver01** - **webserver05**:

```yaml
#--inventory.yaml

---
all:
  hosts:
    webserver[01:05]
    
  children:
    webservers:
      webserver[01:05]
    
  vars:
    system_domain: tinfoilcipher.co.uk
...
```

For our demonstration, we're going to be creating a **haproxy** configuration file, which is going to need to end up looking like this:

```ini
#--haproxy.cfg

defaults
    mode    http
    timeout check 10s

frontend http
    bind *:443
    mode tcp
    default_backend application

backend application
    mode tcp
    balance     roundrobin
    server webserver01.tinfoilcipher.co.uk 10.0.1.10:8080 check fall 3 rise 2
    server webserver02.tinfoilcipher.co.uk 10.0.1.11:8080 check fall 3 rise 2
    server webserver03.tinfoilcipher.co.uk 10.0.1.12:8080 check fall 3 rise 2
    server webserver04.tinfoilcipher.co.uk 10.0.1.13:8080 check fall 3 rise 2
    server webserver05.tinfoilcipher.co.uk 10.0.1.14:8080 check fall 3 rise 2
```

So how do we get this file?

## The Solution

Obviously we can look up the hostname of our servers using the ansible **host** variable. However we can't just use that in a play. On the surface it might look that way but if we do we'll just end up with a config file with file that has the same entry 5 times because we aren't looping over anything, this is where using a _Jinja2 template_ comes in. We can also look up the IP address of each node by making use of Ansible **facts**, allowing us to dynamically look up the IP address of each server at run time.

First we need to create a separate template file, named **haproxy.cfg.j2** (this is just the name of whatever file you're trying to create with an additional **.j2** suffix). In there we'll add some templating logic:

```yaml{% raw %}
#--haproxy.cfg.j2

defaults
    mode    http
    timeout check 10s

frontend http
    bind *:443
    mode tcp
    default_backend application

backend application
    mode tcp
    balance     roundrobin

{% for host in groups['webservers'] %}
    server {{ host }}.{{ system_domain }} {{ hostvars[host].ansible_default_ipv4.address }}:8080 check fall 3 rise 2
{% endfor %}
```
{% endraw %}

We can then reference this file using an Ansible task:

```yaml
---
- name: Build Config Files
  hosts: webservers
  gather_facts: true
  any_errors_fatal: 

  tasks:
  - name: Build haproxy.cfg
    template:
      src: haproxy.cfg.j2
      dest: haproxy.cfg
...

```

This will produce exactly the configuration file that we're looking for!
