---
title: "Secure Home VPN"
layout: post
date: 2019-11-15
categories: 
  - "linux"
  - "projects"
  - "security"
tags: 
  - "encryption"
  - "linux"
  - "networking"
  - "projects"
---

Once upon a time I used to rely on nothing but a Secure Shell for access to my internal network, however this became more and more impractical the more things I stood up on the network and the more things I needed access to from my phone the less time I spent carrying a laptop with me.

Given my long time favouritism for [OpenVPN](https://openvpn.net/) and how much the platform had come along in years, it was the logical choice, and the [Access Server](https://openvpn.net/vpn-server/) deployment is very attractive, especially for small scale, providing a free solution for 2 users and a very low cost model for business use as both virtual appliances, ready built packages and cloud one-click bundles.

Sadly the Access Server package is no longer in the Ubuntu native repos, but the package is readily available from OpenVPN.

**Operating System**: Ubuntu 16.04  
**Required Applications**: wget, certbot, letsencrypt

## Download and install the applications

```bash
sudo apt-get update
sudo apt-get install wget certbot letsencrypt
wget http://swupdate.openvpn.org/as/openvpn-as-2.1.2-Ubuntu16.amd_64.deb
sudo dpkg -i openvpn-as-2.1.2-Ubuntu16.amd_64.deb
```

The installation takes a while so get a coffee, but that's really all there is to it.

## Configuration

First, set the password for the admin user:

`sudo passwd openvpn`

Enter a new password for your admin user and confirm it.

Your server will now be accessible from **http://your\_ip\_address:943** and you can login using the username **openvpn** and the password you just set.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/1.png)

You will of course need to forward/NAT traffic from the internet to port 443 on your firewall(s) to the LAN IP of the VPN server and external DNS will need to be configured if you don't want to use an IP address (I personally use a DDNS service from [NoIP](https://www.noip.com/) which renews for free every 30 days and is supported by my [TP-Link W9980](https://www.tp-link.com/uk/service-provider/dsl-router/td-w9980/).

From this point, a VPN connection can be made from the internet using the **openvpn** account, though this is ill advised.

## Basic Security

Unless you're crazy or working behind multiple layers of security, you probably don't want to display the admin page to the internet. Once you've logged in; browse to **Configuration** \> **Server Network Settings** and set the protocol to run in **Multi Daemon** **Mode** (we won't be using both TCP and UDP for VPN connections, UDP is more efficient for OpenVPN connections), TCP is going to be used for the web server), and the **Hostname or IP Address** should be set to the **INTERNET FACING** hostname/IP of the VPN. Since we're going to be using TLS encryption for the web server, we'll use TCP port 443 for the TCP daemon and the VPN will connect on UDP port 1194:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/2.png)

Once this is done, set the TCP port for the Admin Web UI to 943 (this is the default) and leave the **Client Web Server** configuration set to **Use the same address and port as the Admin Web Server**, if you have multiple NICs this can be changed, but unless you're working on an advanced network with multiple subnets/VLANs you won't really need to do this. The reason for running the Admin Web UI on a different port is that we don't want to forward internet traffic to the Admin console and only to the Logon Web UI.

## Create a User

You'll want to create a personal user account rather than just use the local admin account, integration options exist for **PAM** (default), **LDAP** and **RADIUS** as well as **Local** authentication. If this is a standalone deployment, switch to local authentication under **Authentication** \> **General**:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/5.png)

Now that your accounts are defined locally in the OpenVPN Access Server database, you can create local accounts (obviously a RADIUS or LDAP deployment is a stronger option), even PAM is preferred over this as it allows for local security hardening on your Linux platform, for the purposes of this example however we'll be looking at local only.

Under **User Management** \> **User Permissions** you can add a new user locally (using any other authentication method you will need to synchronise from an external source):

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/6.png)

## Maximum Security

I strongly suggest purchasing a domain in order to secure the connection further, and setting the name of the VPN server accordingly within **Configuration** > **Network Settings**. Though this can be done using the DDNS configuration. Remote connection without either of these is not advised.

Browse to User Management > User Permissions and create a new user to use for standard logons.

Browse to **Configuration** > **Client Settings** and enable:

- **Google Authenticator Support** (also supports [Authy](https://authy.com/))
- **Disable API**
- **Disable any Operating Systems you do not intend to user**
- **Disable any profile methods you do not wish to use**

Finally, configure the webserver server for TLS with a free certificate from [LetsEncrypt](https://letsencrypt.org/):  
Log in to the shell and download a new certificate bundle using LetsEncrypt.

`sudo letsencrypt`

When prompted, select the option to create a temporary webserver and enter the name of your host (E.G. **vpn.tinfoilcipher.co.uk**). This will need to be resolvable over the internet.

The following will be downloaded to **/etc/letsencrypt/live/** e.g. **/etc/letsencrypt/live/vpn.tinfoilcipher.co.uk**.

- fullchain
- chain
- cert
- privkey

Extract COPIES (**do not delete the original files**) these files from the host and in the OpenVPN GUI browse to **Configuration** > **Web Server**  
Upload the files you extracted to the upload options at the bottom of the page as follows:

- **chain** = CA Chain
- **cert** = Certificate
- **privkey** = Private Key

Then click

1. **Verify**
2. **Update Running Server**

The server will restart and now be running at https://your\_hostname. You can now connect externally to an encrypted, MFA secured, free VPN server.

## Logging On

Now that you're ready to connect, you should be able to see an encrypted connection to the **Client Web UI** from the public internet at **https://<your\_hostname>** which will automatically redirect to **/?src=connect** and this should be properly encrypted:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/3.png)

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/4.png)

Once you log on you will be taken through the MFA enrollment process and taken to links to download the OpenVPN software (for whatever platforms you have allowed under **Client Settings**) as well as your configuration profiles:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/7.png)
