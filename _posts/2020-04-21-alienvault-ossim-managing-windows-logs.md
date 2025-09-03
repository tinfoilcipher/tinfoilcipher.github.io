---
title: "AlienVault OSSIM - Managing Windows Logs"
layout: post
date: 2020-04-21
categories: 
  - "integrations"
  - "linux"
  - "security"
  - "windows"
tags: 
  - "alienvault"
  - "integration"
  - "linux"
  - "ossim"
  - "secops"
  - "security"
  - "siem"
  - "windows"
---

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-4.png)

In a [previous post](/alienvault-ossim-building-a-siem-on-a-shoestring/) we looked at building AlienVault OSSIM, but the setup of a SIEM is pretty Spartan without any data sources feeding it. The Operating System integration for AlienVault is surprisingly Windows-centric for a Linux platform, but let's take a look at it.

## Windows Log Management

For this configuration, we'll be using the existing **mc-ossim** OSSIM server set up previously and capturing logs from a Domain Controller named **mc-dc01** at IP Address **192.168.1.10**. We will of course also be using a _Service Account_ for our integrations, this account is **svc_ossim**.

AlienVault captures logs and remote information most effectively using it's HIDS (Host-based IDS) agent, which relays information back to OSSIM.

Windows Logs don't conform to any existing log format (such as SYSLOG or CLF, defined clearly in RFCs 5424 and 6872 respectively), instead they use their own unpleasant format which tends to lead to some handling problems in other systems, OSSIM manages fairly well with this, but more to the point it handles the agent deployment very smoothly.

## Deploying HIDS to Windows

Within the OSSIM web console, browse to **Environment** > **Detection** > **Agents** where we should see only a single host (for the local host):

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-1-1024x244.png)

Click **Add Agent** and we should be able to browse from any Assets defined under **Assets and Groups**, in this case it's the assets set up during the Setup Wizard and I'm selecting the Domain Controller **mc-dc01**. It is important that the **Operating System** value has been set appropriately to Windows as automated deployment can only be performed against Windows endpoints.

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02.png">
  <figcaption>Agent Name and IP/CIDR should automatically populate</figcaption>
</figure>

After saving, you should be able to click the **Automatic HIDS Deployment** button to define credentials for the deployment:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03.png)

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/04.png">
  <figcaption>Your account will need to have the appropriate permissions to install software on the remote machine</figcaption>
</figure>

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/05.png">
  <figcaption>After a minute or so, we can confirm on the remote machine that the agent has indeed installed:</figcaption>
</figure>

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/06.png">
  <figcaption>Successful deployment</figcaption>
</figure>

## Do We Have Data?

If we browse back to the main dashboard, we can now see that that our agent is present as a node (under the **Top 10** section):

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/07-3-1024x718.png)

If we click on the IP we can drill further and see that events are indeed being recorded from the Windows Event Logs for the Domain Controller:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/08-1024x481.png">
  <figcaption>Windows Event Logs, made less horrible</figcaption>
</figure>

If we drill in to any of these events, we can read the raw event data and see more detailed information such as the event ID and associated network:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/09-1-1006x1024.png)
