---
title: "Hashicorp Vault - End Key Cycling with OTP SSH"
layout: post
date: 2020-05-04
categories: 
  - "devops"
  - "integrations"
  - "linux"
  - "security"
tags: 
  - "devops"
  - "integration"
  - "linux"
  - "secops"
  - "security"
  - "ssh"
  - "vault"
---

In a complex Linux environment where multiple administrators have a requirement to manage countless machines (or even a small amount of machines), there is inevitably a requirement to manage SSH Private Keys, as well as the large administrative overhead that comes with cycling them when they expire, or new admins join or move teams. Vault offers us a method to remove the churn of key cycling.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/00-2.png)

A fantastic feature of Vault is the ability to integrate directly with PAM to remove the requirement to use Private Keys internally, instead providing OTPs (One Time Passwords) via direct integration with Vault. The below diagram lays out the high level of how this is implemented:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-11.png">
  <figcaption>Credit: https://learn.hashicorp.com</figcaption>
</figure>

## Some Quick Preamble

If you aren't at all familiar with Vault, I would first recommend taking a look at my earlier posts on the topic, as I'll be working in an existing environment and making some references to some components, though I'll be making links where relevant. The configuration in brief is as follows:

- A single Vault server at https://mc-vault.madcaplaughs.co.uk:8200
- Both UI and API offered at the same address
- A single Vault _Policy_ named **apiaccess** is configured to allow fairly liberal access, which we will be editing
- The Vault instance is Active Directory integrated using the **ldap** _Access Method_
- The CA certificate is already located on the Vault server at **/etc/ssl/certs/mc-strongbox-ca.cer**
- OTP will be tested via a single endpoint named **mc-client**

The configuration of this policy is detailed under a [previous post found here](/hashicorp-vault-tokens-and-the-rest-api/).

We'll be doing the configuration in the Vault UI for the entirety of this guide for the sake of simplicity, though all of these actions can also be performed using the Vault CLI and API when it comes to performing workflow, integration and automation actions.

## Configuring Policies

Before we can set anything up, we will need to ensure that the account you want to work with has suitable permissions in the **Policy** assigned to it. I have done this ahead of time by editing the **apiaccess** policy which is assigned to my account (by making this change from the root account). At a minimum, the account you are going to use for setup must have the following rights assigned to it's **Policy**:

```terraform
# Allow viewing OTP/Secrets Engine in UI
path "sys/mounts" {
    capabilities = [ "read", "update" ]
}

# Allow Configuring SSH Secrets Engine
path "ssh/*" {
    capabilities = [ "create", "read", "update", "delete", "list" ]
}

# Allow Enabling Secrets Engines
path "sys/mounts/*" {
    capabilities = [ "create", "read", "update", "delete" ]
}

# Grant Access to SSH OTP Role
path "ssh/creds/otp_key_role" {
    capabilities = ["create", "read", "update"]
}
```

Adding these permissions to the **apiaccess** policy (which is also assigned to my personal admin account) means that my Active Directory integrated account can perform the setup and provide proper auditing, rather than rely on the root account which we should not be using for day to day use. It also means that tasks can be alternatively performed over the API or CLI should we so choose.

## Configuring the SSH Secrets Engine

First of all, we'll need to configure a Secrets Engine for SSH. When we log in to Vault we browse to **Secrets** > **Secrets Engines** and we see the option to **Enable New Engine**:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-6-1024x122.png)

From here we will select **SSH** as the **Secrets Engine** to enable, leaving the options as default we can enable the engine.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-4.png)

Once the **SSH Secrets Engine** is enabled, we can now browse in to it and see the option to create a **Role**:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/04-5-1024x134.png)

In the new **Role** we will define the method that we will connect to our endpoints to request OTP, as well as the subnets that can perform the action. We can also define the **default username**, this will be used if a username is not provided when requesting an OTP:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/05-5-1024x650.png">
  <figcaption>SSH OTP Role Configuration</figcaption>
</figure>

**Depending on your environment, you probably want to create multiple roles, I.E. one per user, for the purposes of this example I have allowed all users to request an OTP, this is not really recommended for a production system!**

## Configuring an Endpoint

As each endpoint requires manual configuration, this is not really practical to do for all future deployments. This will cover a manual deployment but a real world deployment should be done via an automated method (I would personally recommend [Ansible](https://www.ansible.com/) for existing nodes and building a gold image with [Packer](https://www.packer.io/) for future deployments which already contains this configuration).

First we need to download the **vault-ssh-helper** binary on to the **mc-client** endpoint, unzip it and correctly permission it:

```bash
# Install unzip if not installed
sudo apt-get install unzip

# Download vault-ssh-helper
wget https://releases.hashicorp.com/vault-ssh-helper/0.1.4/vault-ssh-helper_0.1.4_linux_amd64.zip

# Unzip binary and copy to /usr/local/bin
sudo unzip -q vault-ssh-helper_0.1.4_linux_amd64.zip -d /usr/local/bin

# Correctly permission and set ownership on binary
sudo chmod 0755 /usr/local/bin/vault-ssh-helper
sudo chown root:root /usr/local/bin/vault-ssh-helper
```

We now need to create a configuration file, predictably written in HCL, which will be used to inform both PAM and SSH how to interact with Vault:

```bash
# Create directory and config file for vault-ssh-helper config
sudo mkdir /etc/vault-ssh-helper.d
sudo touch /etc/vault-ssh-helper.d/config.hcl
```

Edit the file with **sudo nano /etc/vault-ssh-helper.d/config.hcl** and complete as below, substituting the **vault\_Addr** and **ca\_cert** for your own values:

```ini
vault_addr = "https://mc-vault.madcaplaughs.co.uk:8200"
ssh_mount_point = "ssh"
ca_cert = "/etc/ssl/certs/mc-strongbox-ca.cer"
tls_skip_verify = false
allowed_roles = "*"
```

Save the file with **CTRL+O** and exit with **CTRL+X**

Now we need to reconfigure PAM Daemon to ensure that it's config is reading the vault-ssh-helper configuration file. Edit the PAM Daemon config file with **sudo nano /etc/pam.d/sshd** and configure the _Standard Unix authentication_ block as below, ensuring to comment out the **@include common-auth** option:

```ini
# Standard Un*x authentication.
#@include common-auth
auth requisite pam_exec.so quiet expose_authtok log=/tmp/vaultssh.log /usr/local/bin/vault-ssh-helper -config=/etc/vault-ssh-helper.d/config.hcl
auth optional pam_unix.so not_set_pass use_first_pass nodelay
```

Save the file with **CTRL+O** and exit with **CTRL+X**

Finally, we need to reconfigure the SSH Daemon to ensure that it is observing PAM, Challenge Response and **not** accepting password authentication. Edit the SSH Daemon config file with **sudo nano /etc/ssh/sshd\_config** and set the following values:

```ini
ChallengeResponseAuthentication yes
PasswordAuthentication no
UsePAM yes
```

Save the file with **CTRL+O** and exit with **CTRL+X**

At this point, we should have a functioning system and need to restart the SSH Daemon, **be aware before doing this that if you have made a mistake you could lose access to SSH completely so ensure that you have one of the following as a route back in**:

- At least one active SSH session to the endpoint
- Private Key based access, if this is the first time you're setting this up
- Console/Physical access to the endpoint

If you're comfortable, then restart the service:

```bash
sudo systemctl restart sshd
```

## OTP Access - Testing it Out

Now that everything is in place, let's see how it works.

Browsing back in to the **otp_key_role** we created earlier, we are given the option to **Generate SSH Credentials**, specifying a username and password:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/07-4-1024x226.png">
  <figcaption>If you don't specify a username here, the default for the role will be used</figcaption>
</figure>

Clicking **Generate** will create an **OTP**:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/08-2.png">
  <figcaption>One Time Password, ready for use</figcaption>
</figure>

## But, Does It Work?

Attempting a login from my desktop, let's first try to log on using a normal username and password, to first establish that normal _password_ authentication indeed isn't being authenticated:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/09-2.png">
  <figcaption>Password authentication, denied!</figcaption>
</figure>

Trying again, using the **OTP** value when prompted for a password, we can see we are allowed in to the remote system:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/10-5.png">
  <figcaption>PAM Authentication, using OTP, allowed!</figcaption>
</figure>

To prove that the OTPs are indeed **one time only**, if we log out and attempt to reuse the same OTP, we see that it is rejected and we are not able to log in using the same OTP a second time:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/11-4.png">
  <figcaption>You'll have to take my word for it...or try it yourself :)</figcaption>
</figure>
