---
title: "SSH and SFTP Public Key Authentication - Doing it Right"
layout: post
date: 2020-03-08
categories: 
  - "linux"
  - "security"
tags: 
  - "linux"
  - "networking"
  - "secops"
  - "security"
  - "ssh"
---

Secure Shell might be the greatest component of Linux and the best gem to come from the Open Source community, enabling countless systems to connect to one-another and allowing the secure communication of systems both manually and programmatically with very little complexity, yet despite this people still appear to struggle with it, especially admins from a Windows background.

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/keep-calm-and-ssh-2.png" class="scaled-img-50">

## Keys Vs Passwords

There's a significant downside to using a username and password for access to a system and this concern is amplified by a gigantic factor when you start exposing services to the internet, however in this age of interconnected internet services and cloud providers there is a certain inevitability of making some of your systems available to and accessible from the internet.

It's a false assumption to assume that your internal network doesn't deserve the same love and attention regards it's security as your internet services, SSH out of the box doesn't overcome that (though it's not half bad), but as any good sysadmin or security professional knows, public key based (asymmetric encryption) authentication solves a number of major security weaknesses:

- Attackers cannot (realistically) brute force a strong key, a bad password can be blown away with a trivial attack

- Even a "good" password is often pretty poor in terms of entropy, human beings make pretty bad random character generators

- Password policies are nice, but system administrators can go blue in the face trying to enforce them and the end user will always take the path of least resistance ([Microsoft's Complexity Policy](https://docs.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/password-must-meet-complexity-requirements) for passwords still considers **Password123!** as a "strong" password) and they're not alone

- Programmatic integrations are far more efficient when they don't rely on usernames and passwords which are prone to being changed at some point down the road (not to mention being shared with someone else)

## How does it work?

SSH is a remarkably simply system and in a nutshell follows a 4 step process:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-1024x684.png">
  <figcaption>description</figcaption>
</figure>

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/SSH_simplified_protocol_diagram.png">
  <figcaption>SSH Authentication Flow<br/>Source: https://www.ssh.com/ssh</figcaption>
</figure>

This method allows us to establish **Steps 1 and 2** in the clear over a public network without exposing any sensitive secrets, and all following traffic is achieved over a secure channel and away from the prying eyes of an observer who may be sitting in the middle. SSH.com lists a number of implementations [here](https://www.ssh.com/ssh#list-of-ssh-implementations), for both standard **SSH**, **SCP** (Secure Copy) and **SFTP** (SSH File Transfer Protocol). All of these operate over TCP port 22.

## Out of the box setup?

**Most** Linux servers launch with SSH out of the box (or at least include the option to install the incredibly popular OpenSSH Server at installation). This is also true of Linux-based platforms, though a lot of the bigger ones at least don't have it turned on out of the box (such as **VMWare ESXi** and **Cisco NX-OS**).

In these examples, I'm looking at Linux only; by default, OpenSSH servers authenticate using username and password for any admin user (anyone in the **sudo** group or defined in the **sudoers** file). This is usually a bad idea and if your system is going to be exposed to the internet, it's an **awful** idea.

## Locking down the system

To understand how to lock down using keys, we should first understand how asymmetric encryption works, this is encryption which requires keeping the **Private Key**....well, private.

So this happens (most efficiently, assuming a new server which is not yet configured) as follows:

#### Step 1 - Generating Keys

On your local machine you generate a pair of keys (a **Public Key** and a **Private Key**). The **Private Key** stays with you and is kept secret, we'll be adding the **Public Key** to the remote system(s) and it will be used to identify the **Private Key** when you authenticate.

The keys are generated at the same time and are intrinsically linked, one key identifies the other and no other key can be substituted (I.E. you cannot use a different **Private Key** to authenticate with an existing **Public Key**).

```bash
ssh-keygen

#Generating public/private rsa key pair.
Enter file in which to save the key (/home/welsh/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
#Your identification has been saved in /home/welsh/.ssh/id_rsa.
#Your public key has been saved in /home/welsh/.ssh/id_rsa.pub.
#The key fingerprint is:
#SHA256:I0YZ1yFVGLNnUpnrTZutFcIQ2zmHsCqJHQLwcPbbvaw welsh@lautrec
#The key's randomart image is:
#+---[RSA 2048]----+
#| o.+  . oo=B+o   |
#|  = o  + .o=B o  |
#|   . oo.  oo=* . |
#|     .* + .+.o+. |
#|     oo=So . o.+.|
#|     . .o.. . + o|
#|         o     o |
#|        .     .  |
#|       E         |
#+----[SHA256]-----+

```

As you see, you are prompted for locations to save the key pair (pressing return will use the default which I'd recommend) and for a passphrase to protect the **Private Key**, this is optional but is something I would also recommend, however some automation systems may not support this so always RTFM before you add one :).

By default you will end up with two files in your home folder under **~/.ssh** (if this directory doesn't yet exist it will be created):

1. **id_rsa** - Private Key
2. **id_rsa.pub** - Public Key

#### **Step 2 - Sending your key to the remote system**

The **Public Key** should now be added to the remote system. This needs to be added to the **authorized\_keys** file (yes, by default that American spelling is correct) within the remote servers **~/.ssh** directory.

Assuming we're working with a new server, we can use the following commands to simplify the process (**NOTE:** this server will already need to have an admin account for your user created locally):

```bash
#Create a .ssh directory in the home folder on the remote host
ssh user_name@remote_host 'mkdir .ssh'

#Create authorized_keys file on the remote host within ~/.ssh
ssh user_name@remote_host 'touch .ssh/authorized_keys'

#Add the Public Key to the authorized_keys file
cat .ssh/id_rsa.pub | ssh user_name@remote_host 'cat >> .ssh/authorized_keys'
```

This may not always be viable, especially with cloud systems or a server that you did not build, if this is the case you can always log on to the server manually and execute the following:

```bash
#Create a .ssh directory
cd ~
mkdir ~/.ssh

#Create authorized_keys file
touch ~/.ssh/authorized_keys

#Add key
nano ~/.ssh/authorized_keys

#Correctly Permission the folder and file
chmod 700 ~/.shh
chmod 600 ~/.ssh/authorized_keys

#Once Nano opens, paste in the Prive Key, this should be in the format of ssh-rsa <key> user@host
#Save with CTRL+O
#Exit with CTRL+X
```

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/keys.png">
  <figcaption>Proper authorized_keys format</figcaption>
</figure>

**Step 3 - Configuring the SSHD Config File on the Remote Host**

For the final steps of security within SSH, you should configure the **sshd\_config** (Secure Shell Daemon Configuration) file on the remote host with some extra hardening. At present you will now be allowing public key authentication, but this is being undermined by the fact that you are **still** allowing password authentication. This is a common mis-configuration and one that you shouldn't fall prey to.

Log on to the remote server, and edit the **/etc/ssh/sshd\_config** file

```bash
sudo nano /etc/ssh/sshd_config

#there are innumerable configurations you can make here, but the critical ones for now are:

UsePrivilegeSeparation yes #--Handles incoming network traffic as an unprivileged child process
LoginGraceTime xx #--Time in seconds for login timeout, use this to harden against attacks, 120 seconds by default
StrictMode yes #--verifies key ownership before allowing login, prevents users from leaving their keys open to the world
PermitEmtpyPasswords no #--Obvious
PasswordAuthentication no #--This will be overidden by public key auth
PubkeyAuthentication yes #--Allow public key authentication
AuthorizedKeysFile %h/.ssh/authorized_keys #--Path to public keys, %h is relative to any users home directory
PermitRootLogin no #--Root should NEVER be using SSH

#Once this is done CTRL+O to save
#CTRL+X to exit

#Restart the SSH Daemon
sudo /etc/init.d/ssh restart
```

## Permissions on your Private Key

Your Private Key also needs proper permissions since you're enforcing **StrictMode**, this is just a couple of commands to execute on your local machine:

```bash
#Permission the Private Key
chmod 644 ~/.ssh/id_rsa
```

This will avoid hard errors when attempting to connect to the SSH shell on a remote host. It's not uncommon to see people get stuck here and just disable StrictMode, it's a step you'll thank yourself for learning about properly.

You should now be able to connect via SSH to your remote host and issue the **Public Key** to any other host(s) you wish to connect to correctly by repeating **Step 2**. If you set a passphrase you will be prompted for this at connection time.

## Additional Hardening - Firewall

As we only presently wanting traffic over TCP port 22 (for SSH), lets make sure the soft firewall (I'm assuming **iptables**) can allow that traffic, let's assume your firewall is otherwise properly hardened and is dropping all non-explicit.

Obviously, traffic should **only ever** be allowed from the destinations that you actually want it come from, opening the floodgates to everywhere is just asking for trouble and this is especially true is you're working on internet facing services.

```bash
#Where 10.0.1.12/32 is a specific host that you want to allow SSH access for, could be a host or subnet

sudo iptables -A INPUT -p tcp -s 10.0.1.12/32 --dport 22 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -p tcp --sport 22 -m conntrack --ctstate ESTABLISHED -j ACCEPT
```

Your upstream firewalls will also need to allow TCP port 22 (and only TCP port 22) to get traffic to SSH and should be similarly ACL'd to allow traffic only from appropriate sources.

## Additional Hardening - Automatic blacklisting with fail2ban

[fail2ban](http://www.fail2ban.org/wiki/index.php/Main_Page) is a must have for an SSH server, especially if it's going to be internet facing, it will automatically detect failed authentications against your SSH shell and blacklist any IP addresses which fail to authenticate a number of times. The default ban time is **600 seconds** after **5 authentication failures**.

To install and configure:

```bash
sudo apt-get install fail2ban -y

#Configure:
sudo nano /etc/fail2ban/jail.conf

#Uncomment
[sshd]
enabled = true

#Edit the following values if you want any tweaks to the defaults
bantime = 10m #--Amount of time a banned IP is blacklisted for
findtime = 10m #--Cooldown time after the max retry time has been breached on a per-IP basis
maxretry = 5 #--number of attempts before a ban occurs

#Save the file with CTRL+O
#Exit the file with CTRL+X

#Restart the fail2ban service
sudo /etc/init.d/fail2ban restart
```

Logs are automatically saved for fail2ban events in **/var/log/fail2ban.log**

## Conclusion

Password authentication just won't cut it, especially on internet facing services and out of the box configurations will always need hardening no matter what system you go with.

SSH is a fantastic solution with the agility to provide you with the levels of security you need, but always make sure it's properly configured and that you understand the fine details of your deployment. There is much more that can be done with SSH certificate signing, but that's for another day. The next time someone tells you that a username and password is enough, think hard about it!
