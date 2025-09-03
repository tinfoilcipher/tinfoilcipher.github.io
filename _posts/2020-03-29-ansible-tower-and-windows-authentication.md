---
title: "Ansible Tower and Windows Authentication"
layout: post
date: 2020-03-29
categories: 
  - "devops"
  - "integrations"
  - "linux"
  - "windows"
tags: 
  - "ansible"
  - "ansibletower"
  - "devops"
  - "integration"
  - "kerberos"
  - "linux"
  - "windows"
  - "winrm"
---

Since the release of Ansible 1.7, way back in the forgotten era of 2014, Ansible can connect to Windows (2008 and higher) using remote PowerShell over that most finicky of mechanisms, [WinRM](https://docs.microsoft.com/en-us/windows/win32/winrm/portal).

Red Hat are quick to sell the unilateral management capabilities of Ansible (which do exist), but under the hood we see a uniquely Windows problem. Ansible was built for SSH initially and because Microsoft as ever adopt a non-standard approach (with even WinRM itself being an implementation of another perfectly good open standard), we end up needing to wrestle with the stack to get it just to authenticate.

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/1.png">
  <figcaption>Ansible and Windows - It works, but...WHY</figcaption>
</figure>

## The Root of the Problem

At the root of the problem is the notion of the Windows domain, Linux machines have historically existed as fringe devices inside a Windows network, with sysadmins creating what looks like black magic to pure Windows admins just to get them to speak and authenticate, but dig a little deeper in to the stack and you'll see that the protocols never **really** change. Whilst it would be easy to sell this scenario as an "Ansible problem", we're not making _Ansible_ do anything, just the OS.

## Rewinding. How does Ansible Actually Work?

Keep in mind, Ansible is an **agentless** technology, for SSH this proves to be less difficult as virtually all Linux distros, network devices, appliances and hypervisors offer SSH natively, allowing for a common communication channel. A connection is opened using a Private Key that matches a corresponding Public Key on the remote host(s) and then a channel is established through which tasks will be executed and then closed.

However, Windows communication is performed using WinRM over PowerShell, so while this too is agentless we need to change how we approach the authentication. Keep in mind that **by default**, WinRM is neither enabled or configured on a Windows Host and to compound the problem Linux servers (I.E. your Ansible controller) are typically not configured out of the box to perform the authentication required to speak to a Windows node.

## Good Documentation, But Where to Start?

Ansible offers a number of authentication methods to overcome this limitation, but they make some big assumptions about prior knowledge, and trying to net Windows admins onboard can be tricky (an issue that needs to be overcome to see a wider proliferation of Ansible Tower). I don't think it's a stretch to say that a lot of Windows admins run screaming at the sight of a Linux Shell.

Ansible Documentation on this topic is covered in a length document; [Windows Remote Management](https://docs.ansible.com/ansible/latest/os_guide/windows_winrm.html#windows-winrm), however this covers huge topics and can be a little daunting, especially for non-Linux admins. So let's try and break down how to simplify them in a real world domain, and how to get started.

There are 5 ways to perform WinRM authentication with Windows nodes using Ansible, offering varying trade offs, at a high level they are:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/Screenshot_2020-03-29_15-51-56.png">
  <figcaption>Source: https://docs.ansible.com/</figcaption>
</figure>

For our purposes, we will be using **Kerberos**, though there are some issues with operating at **major** scale, these scales are unlikely to be encountered by all but the largest of enterprises and it's security is perfectly suitable for most applications, it is also the most compatible with Active Directory in my experience.

## Preparation - Deploying Service Account(s)

As with all other nodes, the use of service accounts is down to personal preference, you will probably wish to use more than one service account for different functions, these will need to be granted appropriate permissions on your target Windows hosts to execute whatever tasks you wish to use, if we're using Kerberos then these account(s) will need to exist within Active Directory ahead of time and be able to authenticate on the remote hosts.

## Preparation - Configuring Windows Host(s)

Your Windows hosts will also need to have WinRM configured ahead of time. Within the Ansible documentation are 4 scripts that perform some critical tasks. **Rather than installing these manually it would be prudent to run them against a gold image**. Personally I prefer to use Packer, but SCCM is just as suitable (if several thousand pounds more expensive).

The scripts that should be run on your remote host(s) are (in this order):

1. **[Upgrade-PowerShell.ps1](https://github.com/jborean93/ansible-windows/blob/master/scripts/Upgrade-PowerShell.ps1)** - Upgrades PowerShell and .NET Framework to a supported version (if not present)
2. **[Install-WMF3Hotfix.ps1](https://github.com/jborean93/ansible-windows/blob/master/scripts/Install-WMF3Hotfix.ps1)** - Installs a Windows HotFix for a known memory leak in WinRM (if not present)
3. **[ConfigureRemotingForAnsible.ps1](https://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1)** - Configures WinRM remote PowerShell for Ansible

## Configure Kerberos on Ansible Controller

The common Kerberos Agent to use on Linux is **krb5**, so let's get that installed. I'm going to assume that the platform is CentOS. I have tested this on Ubuntu 16.04 but have heard anecdotally that there are some significant problems getting **krb5** to play nicely on Ubuntu 18.04 and having to resort to **realmd** instead. This is a bit of a moot point as **Ansible Tower** **is no longer supported on Ubuntu following version 3.5**.

Installing krb5 and dependencies:

```bash
yum -y install python-devel krb5-devel krb5-libs krb5-workstation
```

This now gives us our completed **krb5** installation, but we need to configure it, so we will need to edit **/etc/krb5.conf**:

```bash
sudo nano /etc/krb5.conf
```

Delete any existing content and paste in the below, substituting **tinfoilcipher.co.uk** for the **FQDN** name of your own Active Directory domain and **dc1.tinfoilcipher.co.uk** for your own **Primary Domain Controller Emulator**, the reference to **kdc** is the Kerberos Key Distribution Centre as defined in [RFC 1510](https://tools.ietf.org/html/rfc1510):

```ini
[libdefaults]
        default_realm = TINFOILCIPHER.CO.UK
        krb4_config = /etc/krb.conf
        krb4_realms = /etc/krb.realms
        ticket_lifetime = 10h
        kdc_timesync = 1
        ccache_type = 4
        forwardable = true
        proxiable = true

[realms]
TINFOILCIPHER.CO.UK = {
        kdc = DC1.TINFOILCIPHER.CO.UK
        admin_server = DC1.TINFOILCIPHER.CO.UK
        default_domain = TINFOILCIPHER.CO.UK
}

[domain_realm]
        .tinfoilcipher.co.uk = TINFOILCIPHER.CO.UK
        tinfoilcipher.co.uk = TINFOILCIPHER.CO.UK

[login]
        krb4_convert = true
        krb4_get_tickets = false
```

Save the file with **CTRL+O** and close with **CTRL+X**.

Pay particular attention to case sensitivity and full stop placements in the above example, if I have used upper case or a full stop prefix then you should too, regardless of how your active directory presents the name of your domain.

Now we can confirm that our configuration is valid by using the krb5 client tools **kinit** and **klist** to obtain a ticket from the domain controller and confirm the validity of that ticket. In this example the service account is named **svc\_ansibletest** and the real start/expiry dates and times have been redacted:

```bash
#Request a new ticket
$ sudo kinit svc_ansibletest
Password for svc_ansibletest@tinfoilcipher.co.uk: *****************************

#List all active tickets
$ sudo klist
Default principal: svc_ansibletest@tinfoilcipher.co.uk

Valid starting          Expires              Service principal
DD/MM/YYYY mm:hh:ss     DD/MM/YYYY mm:hh:ss  krbtgt/tinfoilcipher.co.uk@tinfoilcipher.co.uk
```

This initial ticket will expire, but will be regenerated if/when required as requested from Ansible Tower.

## Ansible Tower - Credential Mapping

Now that the messy work is done on the Operating System, we can move in to Ansible Tower and create a **Machine** type credential object which will be called at run time. The credential can be applied to any template which is going to be executed against a Windows host(s) and should look along the lines of:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/2-1024x346.png">
  <figcaption>Creating a new credential</figcaption>
</figure>

As my account is also configured to be an administrator on the remote nodes, I have also configured the credential to allow privilege escalation to use the **same** username and password, though any username and password which has _Administrator_ rights can be used to bypass **UAC** **Prompts** using the **runas** method:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/3-1024x165.png">
  <figcaption>Privilege escalation parameters</figcaption>
</figure>

Credentials stored here are symmetrically encrypted using [a variation on the Fernet cipher](https://docs.ansible.com/ansible-tower/3.4.3/html/userguide/credentials.html#understanding-how-credentials-work), providing a secure method for saving credentials, though for working at a wider scale a centralised secret management system is a better bet.

## WinRM Variables - The Last Step

When your playbook runs, it's default behaviour is to attempt to connect using SSH over TCP port 22, but as we've just discussed at length this won't work. The following directives should be set (at **Inventory**, **Group**, **Template** or **Playbook** level depending on your design):

```yaml
ansible_connection: winrm #--Overrides the default connection method of SSH
ansible_port: 5985 #--Overrides the default TCP connection port of 22
```

Keep in mind that the inheritance of variables and directives in Ansible Tower is top down **Inventory** \> **Group** \> **Template** \> **Playbook** and there is no need to set the same value twice if it has already been declared higher up.

## Conclusion - Run Time

Now, when a template is executed against a Windows host(s) using the Credential object in Ansible Tower the Kerberos authentication method is invoked using **krb5** and should there be no valid ticket (as you would see using **klist**) a new one will be requested by running **kinit** under the hood, this process is kept secure as you will not send the password to the shell in the clear, rather the hash will be sent from the Ansible Tower database and a ticket returned. Once a valid ticket is returned Ansible can use this to complete the task(s) defined in your playbook.
