---
title: "AlienVault OSSIM - Building a SIEM on a shoestring"
layout: post
date: 2020-04-20
categories: 
  - "integrations"
  - "linux"
  - "security"
tags: 
  - "alienvault"
  - "integration"
  - "linux"
  - "ossim"
  - "secops"
  - "security"
  - "siem"
---

I noticed around 2015 that SIEM became the new buzzword that IT consultancies started throwing around to sell things that sensible admins had already been doing for decades, namely a centralised platform for the storing and management of logs.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/0-7.png)

The king of these solutions is unarguably ELK (now known as [ElasticStack](https://www.elastic.co/products/)), however ELK is a dark art and scares a lot of people away, when we're talking about a SIEM platform. ELK requires an incredibly deep knowledge of it's components, log manipulation and of Linux in general, I've found that it's a bit more practical to stick with another old favourite, [AlienVault](https://cybersecurity.att.com/).

AlienVault (now known by it's much less cool name of AT&T Cybersecurity since the buyout) was born from it's Open Source roots of the OSSIM project, despite some misleading information online there is still a thriving open source project running in the form of [AlienVault OSSIM](https://cybersecurity.att.com/products/ossim/download) which for most requirements will make a perfect SIEM solution, at a net cost of **zero**.

## Installation - Some Planning

Before getting started, let's be aware of what we're working with. We're building on a single VM with the following specs:

- 2 vCPUS
- 4GB RAM
- 250GB Storage, dynamically expanding
- 3 vNICs (one of which is connected to a VMWare Port Group configured to allow [promiscuous traffic](https://kb.vmware.com/s/article/1004099))
- The VM is going to be called **mc-ossim.madcaplaughs.co.uk** with an IP of **192.168.1.19/24**
- A TLS Certificate has been generated from my internal CA, and the corresponding private key as well as the CA certificate are on standby for use later

## Installing AlienVault OSSIM

When first booting we are presented with an installation screen looking much like the Debian installer, the first hint that we're working with modified Debian:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/1-4.png">
  <figcaption>Select Install AlienVault OSSIM and press Return</figcaption>
</figure>

Set the language and keyboard settings as required and set **eth0** (our first NIC) as the Management interface:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/2-1-1.png)

Then set the IP, netmask, gateway and DNS configuration for **eth0**:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/3-4.png)

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/4-6.png)

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/5-3.png)

Set an appropriately strong root password:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/6-5.png)

Then put the kettle on while the OS and platform install:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/7-2.png">
  <figcaption>Seriously, get comfortable</figcaption>
</figure>

## Initial Configuration - Name and Domain

After forever, the installation will complete and the OSSIM instance will boot, eventually presenting a console for logon. Log on for the first time using the **root** account and the password you created earlier:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/8-1.png)

By default, the OSSIM instance has a generic name of **alienvault.alienvault** and can only be reconfigured via the VM console. To change the name browse to **0 System Preferences** \> **1 Configure Hostname** and enter the new _hostname_ for the server:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/9.png)

Then browse to **0 System Preferences** > **0 Configure Network** \> **5 Network Domain** and configure the domain, the two together will complete the FQDN for the server:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/10-1.png)

Once done, the settings will need to be applied and the server will need to be rebooted. To apply the settings select to **8 Apply all Changes** which will apply all config changes:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/10-2.png">
  <figcaption>...and get comfortable again</figcaption>
</figure>

finally, select **6 Reboot Appliance** to restart the server.

## OSSIM - Administrator Configuration

We can now browse to **https://mc-ossim.madcaplaughs.co.uk** and will be greeted with the initial **admin** user configuration, enter the name, password, contact and company details for the **admin** user:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/11-1.png)

Click **Start Using AlienVault** and you will be taken to the login page, enter the **admin** user credentials to enter the **Setup Wizard**:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/12.png)

## AlienVault - Setup Wizard

I prefer to setup parts of the installation automatically via the wizard and skip others, it makes the process quicker for most configurations.

The first step presents our NICs and asks what we want to dedicate each one to. As we're already using **eth0** for management we need to use another NIC for Log Collection (**eth1**) and another for Network Monitoring (**eth2**, this is the NIC that we configured against a _promiscuous_ Port Group in VMWare) We won't be using Network Monitoring yet but we'll look at it in a later post, but we will get the NIC configured:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/14-1-2.png">
  <figcaption>NOTE: Be sure to set eth2 to Not in Use, as setting up Network Monitoring in the Wizard appears to have a bug. We can set this up manually below</figcaption>
</figure>

For the next step we have the option to scan network(s) for assets, we can scan for assets from any routable networks or enter assets manually:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/15.png)

By default your local subnet will be added to available networks that you can scan:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/16.png)

As I'm only working with this single network I'm not going to widen the scope anymore and just keep this one network. Since I also just want to work with a few specific assets I'm not going to scan, I'm going to manually enter a set of hosts manually, below is my final list of assets, a mix of Linux, Windows and Network platforms:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/19.png)

Skip the setup of **HIDS** deployment and **Network Log Management** (I'll be covering these in different posts and you get a much better understanding of them if you do them manually).

You can then optionally sign up to the [Open Threat Exchange](https://otx.alienvault.com/) which integrates with the system and provides active feedback on vulnerabilities on your endpoints.

Completing this should present you with a basic setup SIEM:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/07-1-1024x718.png">
  <figcaption>First time use</figcaption>
</figure>

## AlienVault - Set up Network Monitoring

Since we skipped setting up a Network Monitoring interface in the Setup Wizard, we still need to do it now. The Setup Wizard configuration appears to have a fatal bug.

To set this up manually, we can connect to the VM Console again, either via the hypervisor or via SSH, and browse to **1 Configure Sensor** > **0 Configure Network Monitoring**, here we will set **eth2** to the **Network Monitoring** interface:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/40.png">
  <figcaption>Bug free changes</figcaption>
</figure>

Once again, we will need to run **8 Apply all Changes** so get comfortable and let the config update again.

## AlienVault - Webserver TLS Configuration

Now that the initial configuration is complete we can address a final piece of admin. We're presently accessing OSSIM at **https://mc-ossim.madcaplaughs.co.uk** we will see an HTTPS error as the webserver is currently running with a self-signed certificate. This obviously won't do.

In the web interface, browse to **Configuration** \> **Administration** \> **Main** and collapse the **OSSIM Framework** section of the configuration:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/24.png)

Upload a copy of the Certificate, Private Key and CA Certificate (PEM Format) and click the **Update Configuration** button. This will run the reconfiguration task from earlier again in the background so the change may take a few minutes to apply. Be sure to set the **Resolve IPs** flag to **Yes** if you are allowing IP SANs in your certificate.

In the next posts on OSSIM I'll be looking configuration of Linux, Windows, Cisco and Juniper devices to send logs to AlienVault OSSIM.
