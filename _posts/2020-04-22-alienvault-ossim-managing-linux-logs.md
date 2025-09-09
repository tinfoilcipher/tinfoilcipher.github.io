---
title: "AlienVault OSSIM - Managing Linux Logs"
layout: post
date: 2020-04-22
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

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-linux.png)

In a [previous post]({% post_url 2020-04-20-alienvault-ossim-building-a-siem-on-a-shoestring %}) we looked at building AlienVault OSSIM, but the setup of a SIEM is pretty Spartan without any data sources feeding it. The Operating System integration for AlienVault is surprisingly Windows-centric for a Linux platform, so lets look at the somewhat involved process for gathering logs from Linux servers using AlienVault.

## Some Quick Setup

For this configuration, we'll be monitoring the existing Vault server **mc-vault** capturing the core Operating System logs and the main Vault log. The server is running at IP Address **192.168.1.43**. We will of course also be using a _Service Account_ for our integrations, this account is **svc\_ossim** and has SSH login rights. SSH has been allowed **inside the subnet** with password authentication.

AlienVault captures logs and remote information most effectively using it's HIDS (Host-based IDS) agent, which relays information back to OSSIM.

## Configuring an Linux Agent

We first need to create an agent within the AlienVault console, browse to **Environment** \> **Detection** \> **Agents** where we can select **Add Agent**:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-3-1024x244.png)

We should be able to browse from any Assets defined under **Assets and Groups**, in this case it's the assets set up during the Setup Wizard and I'm selecting the Vault server **mc-vault**. It is important that the **Operating System** value has been set appropriately to Linux as an automatic deployment cannot be used for Linux endpoint:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-1.png)

Once the agent is added, click the **Extract Key** icon next to the agent to view a Base64 encoded string containing the agent key, IP address and agent ID, copy this as you will need to to perform the configuration on the endpoint:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-1-1024x153.png">
  <figcaption>Expertly obscured</figcaption>
</figure>

## Installing HIDS on Linux

HIDS cannot be deployed from the AlienVault console, so it will need to be installed manually on the Linux endpoint.

First install the pre-requisite software and libraries, then download the HIDS agent from the OSSEC git repo:

```bash
# Download HIDS Agent for Linux and install Pre-Req libs/software
sudo apt-get install build-essential libevent-dev libpcre2-dev libz-dev libssl-dev
cd /tmp
wget https://github.com/ossec/ossec-hids/archive/3.6.0.tar.gz
sudo tar xzf 3.6.0.tar.gz
```

```bash
# Download HIDS Agent for Linux and install Pre-Req libs/software
sudo apt-get install build-essential libevent-dev libpcre2-dev libz-dev libssl-dev
cd /tmp
wget https://github.com/ossec/ossec-hids/archive/3.6.0.tar.gz
sudo tar xzf 3.6.0.tar.gz
```

Now we can install the HIDS agent using the installation script:

```bash
# Install HIDS Agent for Linux
cd /tmp/ossec-hids-3.6.0/
sudo ./install.sh

#  ** Para instalação em português, escolha [br].
#  ** 要使用中文进行安装, 请选择 [cn].
#  ** Fur eine deutsche Installation wohlen Sie [de].
#  ** Για εγκατάσταση στα Ελληνικά, επιλέξτε [el].
#  ** For installation in English, choose [en].
#  ** Para instalar en Español , eliga [es].
#  ** Pour une installation en français, choisissez [fr]
#  ** A Magyar nyelvű telepítéshez válassza [hu].
#  ** Per l'installazione in Italiano, scegli [it].
#  ** 日本語でインストールします．選択して下さい．[jp].
#  ** Voor installatie in het Nederlands, kies [nl].
#  ** Aby instalować w języku Polskim, wybierz [pl].
#  ** Для инструкций по установке на русском ,введите [ru].
#  ** Za instalaciju na srpskom, izaberi [sr].
#  ** Türkçe kurulum için seçin [tr].
  (en/br/cn/de/el/es/fr/hu/it/jp/nl/pl/ru/sr/tr) [en]: en
  
# OSSEC HIDS v3.6.0 Installation Script - http://www.ossec.net
 
# You are about to start the installation process of the OSSEC HIDS.
# You must have a C compiler pre-installed in your system.
 
#  - System: Linux mc-vault.madcaplaughs.co.uk 4.4.0-87-generic
#  - User: root
#  - Host: mc-vault.madcaplaughs.co.uk

  -- Press ENTER to continue or Ctrl-C to abort. --
  
# 1- What kind of installation do you want (server, agent, local, hybrid or help)?
  agent

# 2- Setting up the installation environment.
  - Choose where to install the OSSEC HIDS [/var/ossec]: 

# 3- Configuring the OSSEC HIDS.
  3.1 - What's the IP Address or hostname of the OSSEC HIDS server?: 192.168.1.19

  3.2- Do you want to run the integrity check daemon? (y/n) [y]: y

  3.3- Do you want to run the rootkit detection engine? (y/n) [y]: y

  3.4 - Do you want to enable active response? (y/n) [y]: y

#  3.5- Setting the configuration to analyze the following logs:
#    -- /var/log/auth.log
#    -- /var/log/syslog
#    -- /var/log/dpkg.log

# - If you want to monitor any other file, just change 
#   the ossec.conf and add a new localfile entry.
#   Any questions about the configuration can be answered
#   by visiting us online at http://www.ossec.net .
   
   
   --- Press ENTER to continue ---
```

At this point the HIDS agent will be compiled form source, takes a minute or so....

```bash
# - System is Debian (Ubuntu or derivative).
# - Init script modified to start OSSEC HIDS during boot.

# - Configuration finished properly.

# - To start OSSEC HIDS:
#      /var/ossec/bin/ossec-control start
#
# - To stop OSSEC HIDS:
#      /var/ossec/bin/ossec-control stop

# - The configuration can be viewed or modified at /var/ossec/etc/ossec.conf

#    Thanks for using the OSSEC HIDS.
#    If you have any question, suggestion or if you find any bug,
#    contact us at https://github.com/ossec/ossec-hids or using
#    our public maillist at  
#    https://groups.google.com/forum/#!forum/ossec-list

#    More information can be found at http://www.ossec.net

   --- Press ENTER to finish (maybe more information below). ---
```

By default HIDS is looking only at **auth.log**, **syslog** and **dpkg.log**, since we want to monitor the Vault log we want to configure this before we go any further.

## Adding Extra Logs

To add additional log sources, we need to edit the XML config file **ossec.conf**.

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add a new **localfile** node to the **ossec_config** node in the below format:

```xml
<ossec_config>
...
 <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/vault.log</location>
  </localfile>
...
</ossec_config>
```

Save the file with **CTRL**+**O** and close it with **CTRL**+**X**.

## Linking HIDS to AlienVault

Now that the HIDS agent is configured, it can be linked up to the AlienVault OSSIM. When prompted for your **key**, provide the Agent Key issued by the web console earlier:

```bash
sudo /var/ossec/bin/manage_agents

# ****************************************
# * OSSEC HIDS v3.6.0 Agent manager.     *
# * The following options are available: *
# ****************************************
#    (I)mport key from the server (I).
#    (Q)uit.
Choose your action: I or Q: I

# * Provide the Key generated by the server.
# * The best approach is to cut and paste it.
# *** OBS: Do not include spaces or new lines.

Paste it here (or '\q' to quit): *************************************

# Agent information:
#    ID:2
#    Name:mc-vault
#    IP Address:192.168.1.43

# Confirm adding it?(y/n): y
** Press ENTER to return to the main menu.

# ****************************************
# * OSSEC HIDS v3.6.0 Agent manager.     *
# * The following options are available: *
# ****************************************
#    (I)mport key from the server (I).
#    (Q)uit.
Choose your action: I or Q: Q

** You must restart OSSEC for your changes to take effect.
```

Now that the HIDS agent is properly integrated, we can start up the service:

```bash
sudo /var/ossec/bin/ossec-control start
# Starting OSSEC HIDS v3.6.0...
# Started ossec-execd...
# 2020/04/18 19:57:40 ossec-agentd: INFO: Using notify time: 600 and max time to reconnect: 1800
# 2020/04/18 19:57:40 going daemon
# Started ossec-agentd...
# Started ossec-logcollector...
# Started ossec-syscheckd...
# Completed.
```

Returning to the web console we can see that the new **192.168.1.43** agent is seen in the default dashboard:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/04-1.png)

Drilling in to this asset, we can see that logs are indeed being captured from the Linux endpoint:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/05-2-1024x371.png)
