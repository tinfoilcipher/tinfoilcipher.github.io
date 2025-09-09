---
title: "Hashicorp Vault - Secure Installation and Setup"
layout: post
date: 2020-04-09
categories: 
  - "devops"
  - "linux"
  - "security"
tags: 
  - "devops"
  - "linux"
  - "secops"
  - "security"
  - "vault"
---

Recently I've been working with Hashicorp's [Vault](https://www.vaultproject.io/), a product that I'd played with a little in the past but never really gotten stuck in to. Vault provides a centralised Secret Management platform, including some really cool features like IDAM, cross platform support, dynamic secret management and a fully fledged enterprise offering. It also boasts some pretty fantastic out of the box back-end integrations, Hashicorp's own Consul is a big favourite, but Amazon's S3 and Azure's Container Storage are now included as well a host of other platforms. Following the footsteps of some of the best products, Vault now offers functionality over CLI, GUI and a REST driven API.

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/0-1.png" class="scaled-img-75">

## Getting Started

I found as I was getting set up that there were plenty of guides, but few that took in to account some specifics that had to be scraped together from a bit of experience or some nuances that had cropped up over version changes, there were also some glaring security holes that could be handled out of the box that should really be addressed, this is a product all about security after all!

I'm installing on a clean Ubuntu 20.04 server named **mc-vault** with an IP Address of **192.168.1.43**, I have also issued a certificate from my internal CA ahead of time and placed the certificate and private key in to **/etc/ssl/certs/mc-vault-backend.crt** and **/etc/ssl/private/mc-vault-backend.key** respectively. My host is part of my internal **madcaplaughs.co.uk** domain.

**NOTE:** It is critical that your client certificate (in this case **mc-vault.cer**) be constructed as a bundle, including any intermediary and root CA certificates, as Vault will check the entire certificate chain.

## Installation

Vault itself is a single binary and can be downloaded for free from the Hashicorp website, since we're working with the latest 1.4 version, we can use wget to download the latest version from [https://releases.hashicorp.com/vault/](https://releases.hashicorp.com/vault/1.4.0/vault_1.4.0_linux_amd64.zip)

```bash
# Download latest Vault binary
wget https://releases.hashicorp.com/vault/1.4.0/vault_1.4.0_linux_amd64.zip #--Use your OS and Arch as relevant

# Install unzip
sudo apt-get install unzip

# Unzip the Vault archive
unzip vault_1.4.0_linux_amd64.zip

# Move the Vault binary to an executable path
sudo cp vault /usr/local/bin/
```

Running **vault** from the shell should now return auto complete commands, but there is still much more to do, however we can now verify that Vault is being detected in the system $PATH by running **vault --version**

```bash
vault --version
# Vault v1.4.0
```

## Security First

Now we can get started with configuring Vault, for the purposes of this build I'm going to be looking at a simple, one server configuration with a local filesystem back end. This **IS NOT** suitable for HA or other critical deployments, for something like this we really want a better back end (Consul, S3, Azure Containers etc.). In these examples my back end file system is going to be located at **/opt/vault**.

Now that we have Vault installed, we should make a secure base to work from, starting creating a service account to allow vault to run under, ours will just be called **vault**, we're going to make this user non-usable by assigning the nologin shell (removing the ability to use the STDIN shell and hardening further):

```bash
# Create user "vault", -d sets home directory as /opt/vault, -r defines this as a system account,
# -s defines the shell as /bin/nologon
sudo mkdir /opt/vault
sudo useradd -r -d /opt/vault -s /usr/sbin/nologin vault
```

Now, we set **user** and **group** permissions for **vault** to /opt/vault with **exclusive** permissions:

```bash
# Don't let use of the "install" command throw you, it's used for setting attributes
# -o sets user ownership, -g group ownership, -d sets directory with a mode (-m) of 750
sudo install -o vault -g vault -m 750 -d /opt/vault
```

Now, we give Vault the ability to use the memory lock (mlock) syscall without having to elevate to root. This allows Vault to perform this call without having to swap memory to disk:

```bash
sudo setcap cap_ipc_lock=+ep /usr/local/bin/vault
```

Now we need to configure permissions on the certificate and key we created earlier so our service account can access them, to do this we'll create a group which has access (named **tls**), and then add our service account (**vault**) to it:

```bash
# Create the "tls" group
sudo groupadd tls

# Change the "group" assigned to /etc/ssl/certs and /etc/ssl/private
sudo chgrp tls /etc/ssl/{certs,private}

# Grant read and execute permissions to the assigned group, now "tls"
# to /etc/ssl/certs and /etc/ssl/private
sudo chmod g+rx /etc/ssl/{certs,private}

# Add the "vault" account to the "tls" group
sudo gpasswd -a vault tls
```

## Configuring Vault Options

Now that security is reasonably hardened, we can put the configuration of Vault in place, the file should be placed in the root of **/etc** as **/etc/vault.hcl**. As the name implies, the file is written in HCL (Hashicorp Configuration Language) and then parsed to JSON, to this end it is also possible to create the file as **vault.json** if preferred, however I would advise sticking to HCL as support for JSON may be dropped in the future (as we have already seen with the likes of Puppet):

Create the file with **sudo nano /etc/vault.hcl**:

```terraform
storage "file" {
        path = "/opt/vault"
}

ui = true

listener "tcp" {
        address = "192.168.1.43:8200"
        tls_disable = 0
        tls_cert_file = "/etc/ssl/certs/mc-vault-backend.crt"
        tls_key_file = "/etc/ssl/private/mc-vault-backend.key"
}

max_lease_ttl = "10h"
default_lease_ttl = "10h"
api_addr = "https://192.168.1.43:8200"

```

The configuration is broken in to several stanzas, anyone familiar with Terraform will recognise the HCL syntax, some key points are:

- We're using the **storage** type **file** (previously termed back end), that **ui** has been set to **true**, without this we will have only the CLI and APIs

- The **Listener** configuration defines our specific configurations for TCP settings, without the hard IP configuration we will listen on all IP addresses which is problematic for a multi IP system. By default we will also not enable TLS and will not look to use a certificate and private key.

- The **api_addr** argument has been defined, this needs a full URL definition, including port, if this is not set, one is assumed from the IP set in the listener

- The two lease settings define how long issued tokens from Vault to clients are valid for

Be sure to change the IP Addresses where relevant to match your own and save the file using **CTRL+O** and exit with **CTRL+X**.

We should now also set the permissions on the **vault.hcl** file correctly so the service account can interact with it:

```bash
# Set user and group ownership
sudo chown vault:vault /etc/vault.hcl

# Set file system permissions, mode 640
sudo chmod 640 /etc/vault.hcl
```

## Daemonising With systemd

Now that we have a running system and a valid config, we should create a **systemd** Unit Service in order that Vault can be stopped and started as a service, critically, by our service account.

To do this, we'll need to create a unit file to define the service:

```bash
# Create a new unit file for the new vault.service Unit Service
sudo nano /etc/systemd/system/vault.service
```

In the new file, paste in the below:

```ini
[Unit]
Description=Hashicorp Vault Service
Documentation=https://vaultproject.io/docs/
After=network.target
ConditionFileNotEmpty=/etc/vault.hcl

[Service]
User=vault
Group=vault
ExecStart=/usr/local/bin/vault server -config=/etc/vault.hcl
ExecReload=/usr/local/bin/kill --signal HUP $MAINPID
CapabilityBoundingSet=CAP_SYSLOG CAP_IPC_LOCK
AmbientCapabilities=CAP_IPC_LOCK
SecureBits=keep-caps
NoNewPrivileges=yes
KillSignal=SIGINT

[Install]
WantedBy=multi-user.target
```

If you read the file line by line you should see the relevance of the changes we've made so far and how it ties up with the file locations we've used, if you have used any other locations make sure they're reflected. Save the file using **CTRL+O** and exit with **CTRL+X**.

We can now verify if the service runs as expected:

```bash
# Reload systemd daemon
sudo systemctl daemon-reload

# Start vault service
sudo systemctl start vault

# Verify status
sudo systemctl status vault

# vault.service - Hashicorp Vault Service
#   Loaded: loaded (/etc/systemd/system/vault.service; disabled; vendor preset: enabled)
#   Active: active (running) since Thu 2020-04-09 14:24:06 BST; 3h 23min ago
#     Docs: https://vaultproject.io/docs/
# Main PID: xxxx (vault)

```

All going well, you should see an output as above, if a **FAILED** message is shown, read the debug to determine where the error is occurring, if you've followed these steps to the letter you should have no issues.

## Accessing Vault and Unsealing

Vault can now be accessed via the GUI or CLI for it's first initialisation, for a change of pace I'm going to use the GUI here, browse to https://<your\_ip\_or\_hostname>:8200/ui and you should be taken to the Vault setup:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/2.png">
  <figcaption>It's Alive!</figcaption>
</figure>

Initial setup requests that you go through the initiation procedure. By default, Vault is in a **Sealed** state until it is initialised. [Sealing and Unsealing](https://www.vaultproject.io/docs/concepts/seal/) are core concepts behind Vault and ones to understand before you try and dive in so make sure you understand them.

In order to initialise the Vault you will first need to generate it's Master Key and define a quorum of "key shares" (segments of the key) which will need to be submitted in order to gain access in an emergency (I.E. no authorised users being available to unseal the Vault again). For example if you define 3 **Key Shares** and set the **Key Threshold** as 2, then you will be given a key split in to 3 parts, any two of which can be used to Unseal the Vault.

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/3.png">
  <figcaption>Choose Carefully...</figcaption>
</figure>

Once you have **Initialised**, you will be given the option to download a JSON representation of your master key segments along with your root login token **ONE TIME ONLY**, don't skip this, you're going to need it:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/4.png">
  <figcaption>The most secret of secrets</figcaption>
</figure>

These keys should be stored outside of the Vault and ideally held in separate locations. You will need these whenever the Vault is **stopped**, **restarted** or manually **sealed**.

As the Vault is started in a **Sealed** state, you will come face to face with this instantly and be asked to provide the minimum amount of key shares you defined in your setup:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/6.png">
  <figcaption>Unsealing</figcaption>
</figure>

Now you will need to login for the first time, as no identity providers exist yet, you will need to use your root token to gain access (this is included in the file you downloaded when setting up your master key shares):

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/7.png">
  <figcaption>Root token for now, this shouldn't be a permanent feature</figcaption>
</figure>

Now Vault is ready for use, an initial [Cubbyhole](https://www.vaultproject.io/docs/secrets/cubbyhole/) Secrets Engine is configured out of the box (for arbitrary TTL-linked secret storage), though dozens of other integrated secrets engines are also ready to use at a few clicks:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/10-1024x305.png">
  <figcaption>Ready to go!</figcaption>
</figure>

This covers the installation and setup of Vault. Next post we'll start looking at deeper use cases once I've had more time to get my hands dirtier :)

**UPDATE**: A later post found [here]({% post_url 2020-06-27-hashicorp-vault-reverse-proxy-with-nginx %}) covers a slightly revised approach of the deployment using NGINX as a reverse proxy to remove the need to expose Vault directly and allowing the ability to terminate TLS on NGINX rather than Vault.
