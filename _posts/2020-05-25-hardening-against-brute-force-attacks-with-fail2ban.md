---
title: "Hardening Against Brute Force Attacks With fail2ban"
layout: post
date: 2020-05-25
categories: 
  - "linux"
  - "security"
tags: 
  - "linux"
  - "security"
  - "sysadmin"
---

Even in 2020 (current year argument), it's woeful how prevalent Brute Force Attacks are, what's more worrying is how successful they are, whilst it might seem that the logical thing to do is just to harden password policies that's not really the way the tide is turning and I'd remind anyone to remember Kerckhoffs's principle of **The Enemy Knows The System**.

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-1-1024x800.jpg" class="scaled-img-50">

## What Is fail2ban?

I've briefly discussed the use of **fail2ban** as part of [proper configuration of SSH and SFTP]({% post_url 2020-03-08-ssh-and-sftp-public-key-authentication-doing-it-right %}) in the past, but it's use really does extend beyond SSH. In a nutshell fail2ban is a Unix-like log parser that looks at structured security logs for repeat patterns (say a repeated IP address or hostname appearing with failed authentication) and under such circumstances sending that offending IP to it's _jail_ for such a time as you see fit.

This process is highly automated and unless you have a broken automation where a service account has a bad credential or something of the like you shouldn't really notice what's happening. The ideal scenario would be that your services just don't come under attack, but the real world scenario is that your services are under attack, all of the time, if you aren't doing anything about it then it's just a ticking clock once you put something on the internet that it will eventually be compromised with no mitigation put in place.

## fail2ban - Installation and Basic Setup

Installation is pretty painless, for the sake of compatibility, let's look at some common Linux distros:

```bash
# Debian/Ubuntu
sudo apt-get install fail2ban

# CentOS/RHEL7
sudo yum install fail2ban

# Fedora/RHEL8
sudo dnf install fail2ban
```

Simple as that.

Now that we're installed, the first thing we really want to configure is SSH access, **especially if this server offers SSH to the internet**. Offering SSH to the public internet is as ugly business, and one would hope if it's being done at all it's being done properly with strong keys, trusted IPs and Firewall Rules, but even with those we want some other assurances, we after all cannot be sure of the security of a third parties network.

Configuration of **fail2ban** happens in the **jail.conf** file, so let's jump in to it:

```bash
# Edit jail.conf
sudo nano /etc/fail2ban/jail.conf

#Uncomment
[sshd]
enabled = true

#Edit the following values if you want any tweaks to the defaults for ALL JAILS
bantime = 10m #--Amount of time a banned IP is blacklisted for
findtime = 10m #--Cooldown time after the max retry time has been breached on a per-IP basis
maxretry = 5 #--number of attempts before a ban occurs

#Save the file with CTRL+O
#Exit the file with CTRL+X

#Restart the fail2ban service
sudo /etc/init.d/fail2ban restart
```

With this set we have hardened our servers SSH implementation, what's going on under the hood is that the **fail2ban** service is going to look through the logs for SSHD authentication events and pattern match them using a series of Regular Expressions which pertain to failures, upon 5 of these events matching the same source, an event will be met which places the source in _jail**,** below is an example of the SSH regex:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-6-1024x200.png)

## It's Not Just For SSH

Out of the box, **fail2ban** contains plugins for dozens of popular applications (and dozens more exist and can be written without much fuss), these configurations can be found in **/etc/fail2ban/filter.d** as seen below:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-5-1024x112.png)

Enabling any one of these can be applied by adding a configuration block to **jail.conf**, for example if we wish to add a **fail2ban** configuration for the popular Open Source PBX _Asterisk_ we can simply add the below to **jail.conf**:

```bash
# Edit jail.conf
sudo nano /etc/fail2ban/jail.conf

#Uncomment
[asterisk]
enabled = true

#Save the file with CTRL+O
#Exit the file with CTRL+X

#Restart the fail2ban service
sudo /etc/init.d/fail2ban restart
```

## This Isn't A Replacement For Good Security

On the surface this seems like a magic bullet for security, but of course it's not and no matter how much **fail2ban** can help to harden your security it's not a substitute for good hardened firewall rules and NACLs network access.

Keep in mind always that unless you need to access a specific resource, you should have no access to it, and this goes doubly for any systems which have access to the internet. Why on earth would you present SSH or any other remote access system to the internet unless it's absolutely needed?

It's also key to remember that attacks don't just come from the internet, they can just as easily come from inside your network and the same security practices should be considered inside the LAN as on the edge. It only takes one weak account inside the network to be exploited to blow a hole in your armour and tools like **fail2ban** were made to assist in defending against exactly those kind of weaknesses.
