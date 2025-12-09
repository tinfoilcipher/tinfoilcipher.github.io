---
title: "Ansible Tower - Integrating with Cisco Devices"
layout: post
date: 2020-03-30
categories: 
  - "devops"
  - "integrations"
  - "linux"
tags: 
  - "ansible"
  - "ansibletower"
  - "cisco"
  - "devops"
  - "integration"
  - "netops"
---

Following my look at integrating [Ansible Tower with Windows]({% post_url 2020-03-29-ansible-tower-and-windows-authentication %}), I thought I'd take a look at another common requirement that needs some slight tweaking (though not nearly to the extent of Windows), networking devices, specifically Cisco devices running IOS, ASA and NX-OS platforms.

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/1.jpeg" class="scaled-img-75">

## Networking - It's Built In

Unlike the additional layers of configuration that comes with Windows, the use of Cisco platforms is native to Ansible, however some steps can be a little less that clear in the [documentation](https://docs.ansible.com/ansible/latest/network/getting_started/network_differences.html), especially if you're jumping right in.

Typically in Tower we will map a **Credential** object to a **Template** and have that credential interact with our endpoint(s) at run time, however the nature of network Operating Systems in Ansible provides us with a few little quirks to be aware of.

## Configuring a Credential

Despite Tower offering a **Network** credential type, it is my experience that the **Machine** credential type is much more suitable for these kind of connections, especially if we're using a Firewall or Switch. This is due to this credential type accepting specific **Privilege Escalation** types. Since we typically aren't going to want to give accounts auto exec 15 if we can avoid it, we'll build an **enable** in to the Credential object:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/2-1-1024x570.jpg">
  <figcaption>A proper credential should use a Private Key for authentication with enable as the escalation method</figcaption>
</figure>

This credential should now be suitable to connect to an endpoint you define by hostname or IP address as being your Cisco Switch(es) or Firewall(s)? Right? Well, not yet.

## Execution on the Ansible Controller

Most network devices (except the newest and coolest) can't run python locally, so our commands need to be executed in a sane environment (on the Ansible controller) and then sent to a spawned console on the remote node(s), in this case your IOS/ASA/NX-OS endpoints.

In order to achieve this, we need to tell Ansible Tower what we're actually pointing at ahead of time, this can be done by setting some directives against either the **Inventory**, Group, Template or Playbook ( depending on your design):

```yaml
ansible_become: 'yes' #--Allows us to use the enable directive
ansible_become_method: enable #--Specified that the escalation method is "enable"
ansible_connection: network_cli #--Specifies that we wish to spawn a CLI on the remote node, overrides the default of ssh
ansible_network_os: ios #--The OS of the remote device
```

Keep in mind that the inheritance of variables and directives in Ansible Tower is top down **Inventory** \> **Group** \> **Template** \> **Playbook** and there is no need to set the same value twice if it has already been declared higher up.

## Other Options, beyond ASA and beyond Cisco

The above example is **only** suitable for a CLI on IOS devices of course (as the **ansible\_network\_os** value is set to **ios** and the **ansible\_connection** is set to **network\_cli**), there are other options that can be used however and these venture not only beyond ASA but beyond Cisco:

```yaml
#--Operating System Options
ansible_network_os: asa #--Cisco ASA (Firewalls)
ansible_network_os: iosxr #--Cisco IOS XR (Carrier Grade Routers)
ansible_network_os: nxos #--Cisco NX-OS (Nexus Switches)
ansible_network_os: junos #--Juniper JUNOS (Firewalls and Switches)
ansible_network_os: vyos #--VyOS
ansible_network_os: eos #--Arista EOS

#--Network Connection Options
ansible_connection: httpapi #--HTTP API (for devices with a suitable API)
ansible_connection: netconf #--XML Over SSH (for devices which support netconf)
```

## Running the Playbook

From here we can now run any modules (or arbitrary commands) without needing to expose authentication within the Playbook code.

For example we can run the below **Playbook** from a **Template** which is linked to the **Credential** we created earlier (assuming that the appropriate directives have been linked to the Inventory/Group/Template) to create a static route, as you see no credentials have been exposed in code:

```yaml
---
- name: Add_static_route
  hosts: 192.168.10.254
  gather_facts: false
  any_errors_fatal: true
  
  tasks:
  - name: Add static route
    ios_static_route:
      prefix: 192.168.20.0
      mask: 255.255.255.0
      next_hop: 10.0.0.1
...
```

Simple :)
