---
title: "AlienVault OSSIM – Managing Network SYSLOGs"
layout: post
date: 2020-04-24
categories: 
  - "integrations"
  - "linux"
  - "security"
tags: 
  - "alienvault"
  - "cisco"
  - "integration"
  - "juniper"
  - "linux"
  - "networking"
  - "ossim"
  - "secops"
  - "security"
  - "siem"
  - "vmware"
---

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-6.png)

In previous posts I've looked at the [setup of AlienVault OSSIM]({% post_url 2020-04-20-alienvault-ossim-building-a-siem-on-a-shoestring %}) and managing logs from both [Windows]({% post_url 2020-04-21-alienvault-ossim-managing-windows-logs %}) and [Linux]({% post_url 2020-04-22-alienvault-ossim-managing-linux-logs %}) Operating Systems. However as any admin knows dealing with servers is only half the battle when it comes to logs, network devices are arguably the most important part. In this post we'll be looking at log management for **Juniper JUNOS**, **Cisco IOS** and **VMware EXSi** devices in particular, all of which share a common and widely used logging standard.

## Different Logs, Different Method

Unlike with Operating System logs, Windows in particular, we use a SIEM to passively accept logs from _most_ network appliances. Due to the restrictions we usually find around network devices we often cannot install an agent on the network Operating System, in this post we'll be using a Juniper device, specifically on my long in the tooth SRX100 firewall, for some of the setup examples, but will be looking at configs for other platforms as well.

_Most_ network devices send logs in the widely used SYSLOG format (as meticulously defined under RFC 5424) which operates over either UDP port 514 or TCP port 6514 (AlientVault accepts both but different platforms use different implementations).

Since we aren't working with an agent we cannot rely on the firewall simply finding it's way to us, it will need to be configured.

## AlienVault - Configuring a Log Management Interface

We configured a port during the initial setup of AlientVault, however if that wasn't done or if something has changed along the way, this can be set up manually or changed/reconfigured at any time by accessing the VM Console either via the hypervisor or via SSH.

First we will need to browse to **0 System Preferences** > **0 Configure Network** > **1 Setup Network Interface**:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-5.png)

This menu defines which NICs are being used for **Log Management**, select the NIC(s) you wish to use and define an IP address and netmask:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-2.png)

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-2.png)

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/04-2.png)

Once again, we will need to browse to **8 Apply all Changes** and apply the configuration, this takes a while so get comfortable.

## AlienVault - Configuring Plugins

In order to properly receive and parse the log data, AlienVault must be configured to use an appropriate **plugin** which is then linked to an **Asset**. Below we can see an existing device, which is a my Juniper firewall, configured as a **Network Device** type asset, but with no further configuration.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-7.png)

Clicking the _Information_ at the end of the row will take us to the details of the asset, where we see no logs are being gathered. If we browse to the **Plugins** tab we can use the **Edit Plugins** button to add a new Plugin:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-3.png)

Here we simply need to define the type of device we're working with, in this case **Juniper/SRX Series**:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-3.png)

A similar configuration should be applied to any other devices using the appropriate Vendor/Model details, for the purposes of Cisco IOS and VMware ESXi these are:

- **Cisco IOS**
    - **Vendor**: Cisco
    
    - **Model**: Cisco Router (This is the case for any device running IOS, Switches and Routers)

- **VMware ESXi**
    - **Vendor**: VMware
    
    - **Model**: ESXi

## Configuring Devices to send SYSLOG data

There is a wealth of documentation provided from AT&T for configuring each platform which you might want to send logs from, a valiant effort but it's variable in quality when it comes to configuring the endpoints (for example the [Juniper configuration examples](https://cybersecurity.att.com/documentation/usm-appliance/supported-plugins/configuring-juniper-srx.htm) and [Cisco configuration examples](https://cybersecurity.att.com/documentation/usm-anywhere/supported-plugins/configuring-cisco-ios.htm) are missing some arguments that the configuration won't work without), but the vendor shouldn't have to do all the work for you! They're more than enough to get you started.

The fastest way in my experience to configure the syslog on any device is to log on the shell and set up the configuration. Below are the configs on all 3 platforms.

NOTE: All of these assume that we're sending logs to the AlienVault interface of **192.168.1.7**:

```bash
###---Set up Remote SYSLOG for Juniper JUNOS---###

# Enter configuration mode
configure

# Configure remote syslog using UDP, where 192.168.1.1 is the Juniper device
set system syslog host 192.168.1.7 source-address 192.168.1.1 port 514 any any

# Apply changes
commit
```

```bash
###---Set up Remote SYSLOG for Cisco IOS---###

# Enter configuration terminal
conf t

# Configure remote syslog using TCP, UDP doesn't appear to work with AlienVault
logging host 192.168.1.7 transport tcp

# Apply changes
end
wr mem
```

```bash
###---Set up Remote SYSLOG for VMware ESXi---###
# NOTE: You will need to manually enable SSH on the ESXi host first!

# Configure syslog, yes the strange use of tcp and 514 does work, I don't get it either
esxcli system syslog config set --loghost='tcp://192.168.1.7:514'

# Reload syslog config
esxcli system syslog reload
```

## Does it Work?

If we return to the AlienVault web console, we can see that there are logs being received from the devices we've since added:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/06-1.png">
  <figcaption>Juniper JUNOS SYSLOGs</figcaption>
</figure>

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/05-3.png">
  <figcaption>VMware ESXi SYSLOGs</figcaption>
</figure>

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/04-3.png">
  <figcaption>Cisco IOS SYSLOGs</figcaption>
</figure>

Drilling in to any event from these devices, we can see more detail on the SYSLOG event in question, in the below example it was myself reading the SYSLOG configuration from the shell on the Juniper firewall:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/11-3-1024x388.png)

## Conclusion

These last few posts cover the build, configuration and deployment of a full SIEM and log management solution for zero cost, with so much data being readily available in most networks which can almost instantly point to errors and trends before they become an issue, there really is no reason not to aggregate your logs in to a single place.

Unless you **really** like making things hard for yourself :)
