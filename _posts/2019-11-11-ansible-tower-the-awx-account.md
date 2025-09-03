---
title: "Ansible Tower - The awx Account"
layout: post
date: 2019-11-11
categories: 
  - "devops"
  - "linux"
tags: 
  - "ansible"
  - "ansibletower"
  - "linux"
  - "secops"
  - "security"
---

One of Ansible's most brilliant features is [Privilege Escalation](https://docs.ansible.com/ansible/latest/user_guide/become.html), the ability to enter the context of a more privileged user following an initial connection to either your local or remote node, however a bizarre little caveat in _Tower_ I haven't been able find documented anywhere and it refers to the use of a system account (by default named **awx**) on the localhost.

#### What Is AWX Anyway?

Floating around all over the Ansible Tower deployment is references to _AWX_, this makes sense as [**AWX**](https://www.ansible.com/products/awx-project/faq) is Red Hat's upstream, rapidly changing and full Open Source release of _Tower_, given is rate of rapid change it is highly not recommended for release in to mission critical or production environments (speaking from experience this is a smart decision unless you're **very** used to supporting a potentially unstable _Microservices_ environment). It does tend to work well but even the installation can be a bit of a painful process to the un-initiated and support is relegated to IRC and the official Mailing List, whilst this is nothing new for long time FOSS users, it's not the smartest decision for a critical system.

#### The AWX Account In Ansible Tower

When an installation of _Tower_ is completed using the standard installation _Playbook_ (unless you made some **SEVERE** modifications), the system account which will be created on the server is named **awx**, this account assumes responsibility for all tasks undertaken on the localhost when connecting, and will be the default user account you see in all of your _STDOUT_ and _STDERR_ data streams (which led me to some serious confusion as it isn't mentioned anywhere in the setup or administration documentation).

#### **Privilege** Escalation and Connection Methods

So, you have your Tower instance built using the nice and easy to use installation Playbook and followed the fool proof setup guide as provided in the [**Ansible Tower Quick Setup Guide**](https://docs.ansible.com/ansible-tower/latest/html/quickstart/index.html), you've configured SSH keys on your localhost and inside Tower and should now be ready to escalate your account, the documentation is pretty clear on how that [**works**](https://docs.ansible.com/ansible/latest/user_guide/become.html)?

Nominally, when using Ansible _Engine_, escalation uses Ansible's own system called **become** which uses it's own set of directives, this can simply look like:

```yaml
---
- name: some_name
  ansible_connection: local
  become: yes
  become_method: sudo
  become_user: your_priv_user
  tasks: 
    <TASK_DATA>
...
```

When executing from _Tower_, this is the same system being used, it's simply the case that we don't need to actually define the commands in code, as the arguments are passed as additional variables from Tower. So this should present no issue, when creating our credential in Tower we should just need to add a become method, username and password, easy.

Except it doesn't work when connecting to the localhost, if we try and enable escalation we get an error:

`sudo: effective uid is not 0, is sudo installed setuid root`

...glad that cleared things up...

Oh dear, well I suppose the awx account just doesn't have sudo rights? However granting these rights just yields the same results, what's going on?

#### What's Going On?

It should be noted that the suggested method for connecting to the localhost is using the **ansible\_connection: local** method, this is going to take more reading. The answer comes in the form of a article explaining the nature of the awx account a little more (sadly hidden inside the Red Hat Knowledge Base):

<blockquote>
  It isn't possible to use Tower with local action to escalate to the root user. It will be necessary to alter your task to connect via SSH and then escalate to root using another user (not AWX).
  This is done purposefully to avoid security risks associated with our user having root level access to the system.
  <footer>- https://access.redhat.com/solutions/3223501</footer>
</blockquote>

Well that makes sense now, the article also stresses **NOT** to give AWX _sudo_ rights, better undo that too.

So the solution? Whatever account you were escalating to, you need to make that the same account that you're connecting as (or connect as a different account and elevate to an appropriate account). Basically don't connect using the **ansible_connetion: local** method if you're ever going to need to elevate on the localhost, switch this up for SSH (which is the default if you don't define local).

As it turns out, doing nothing would have solved this one, but the documentation is trying to protect you, as any admin knows, sometimes you actually do need to become root.
