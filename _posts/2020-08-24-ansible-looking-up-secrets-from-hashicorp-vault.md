---
title: "Ansible - Looking Up Secrets from Hashicorp Vault"
layout: post
date: 2020-08-24
categories: 
  - "aws"
  - "devops"
  - "linux"
  - "security"
tags: 
  - "ansible"
  - "aws"
  - "devops"
  - "integration"
  - "linux"
  - "secops"
  - "security"
  - "vault"
---

Previously I've looked at how to **[lookup secrets from Hashicorp Vault using Ansible Tower](/hashicorp-vault-integrating-with-ansible-tower/)** however whilst that functionality is incredibly valuable it doesn't really tackle the issue of how to write _Playbooks_ which can interact with Vault. In this post we'll look at how we can use some excellent lookup functionality provided as part of the ansible which provides this functionality.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01.png)

## Some Assumptions

For this article, I'm going to be working with the same Vault instance set up [here](/hashicorp-vault-secure-installation-and-setup/) and hardened behind an NGINX reverse proxy [here](/hashicorp-vault-reverse-proxy-with-nginx/). What we're working with will be:

- A single Vault instance running behind a TLS reverse proxy running at **https://mc-vault.madcaplaughs.co.uk**.
- This Vault has a single _kv_ _Secrets Engine_ containing a single secret for the purposes of an example, the Secret contains several Key Value Pairs. This credential will be used to handle our AWS Access Credentials.
- We're going to use a _Playbook_ which returns some facts on already provisioned AWS infrastructure and nothing more, the theory is identical for working with making changes.
- The _kv Secrets Engine_ has been created as a v1 engine, meaning that secrets versioning is not enabled (though we will look at using v2).

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-1024x273.png">
  <figcaption>Our example credential as shown in the Vault GUI</figcaption>
</figure>

## The hashi_vault Plugin and Configuration

Integration is provided by the excellent **hashi_vault** community plugin, however this module does require that the _hvac_ python library be installed before use:

```bash
sudo pip install hvac
```

If we take a look at the **hashi_vault** _plugin_ for Ansible, we see some simple examples that should suit us, however they can be a bit cumbersome and the examples given in the documentation lack specific context.

Before getting started, lets be sure to configure some Environment Variables for both VAULT\_TOKEN and VAULT\_ADDR. If these aren't set then we will need to define them inside the playbook which isn't a secure configuration:

```bash
export VAULT_ADDR=https://mc-vault.madcaplaughs.co.uk
export VAULT_TOKEN=<VALID_API_TOKEN>
```

## Using the hashi_vault Plugin

The below examples show the use of the plugin to retrieve a secret and write it out to the console:

```yaml{% raw %}
---
#--Retrieve a secret and all Key Value Pairs

- name: Lookup and Output Entire Secret from Vault
  hosts: localhost
  gather_facts: false
  connection: local
  tasks:
    - name: Lookup and Output Entire Secret from Vault
      debug:
        msg: "{{ lookup('hashi_vault', 'secret=kv/aws_credentials')}}"
...
```
{% endraw %}

```yaml{% raw %}
---
#--Retrieve a secret and a specific Value

- name: Lookup a Secret from Vault and Output a Specific Value
  hosts: localhost
  gather_facts: false
  connection: local
  tasks:
    - name: Lookup a Secret from Vault and Output a Specific Value
      debug:
        msg: "{{ lookup('hashi_vault', 'secret=kv/aws_credentials:aws_secret_access_key')}}"
...
```
{% endraw %}

In the above example, **Line 11** is configured to use the lookup to search a different path and return either the entire secret, or a specific value, the below shows the return data:

```bash
TASK [Lookup and Output Entire Secret from Vault] ********************************************************************
ok: [127.0.0.1] => {
    "msg": {
        "aws_access_key_id": "AKIAIOSFODNN7EXAMPLE", 
        "aws_default_region": "eu-west-2", 
        "aws_secret_access_key": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
    }
}
```

```bash
TASK [Lookup and Output Specific Secret from Vault] *****************************************************************
ok: [127.0.0.1] => {
    "msg": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
}
```

## The Trouble With Versions

So we've been looking at v1, but it's rapidly becoming less popular than v2 now, using this code just won't work.

A different syntax is required between v1 and v2 _Secrets Engines_, this is covered (in some incredibly light detail) in the plugin documentation, but this is no great surprise and most platforms encounter the same issue (including basic REST queries) due to the structure of the API of v2 _Engines_ being totally different in order to handle versioning. We need to pass some additional parameters to state which version of the secret we want to retrieve, it's often not enough to simply assume that if we state no version that the **latest** secret will be retrieved.

The below examples cover the same secret lookup and output for a v2 engine. The key difference being that the secret path has "data" inserted between the name of the _Secret Engine_ and the _Secret_ we're trying to look up:

```yaml{% raw %}
---
#--Retrieve a secret and all Key Value Pairs from a v2 Secret Engine

- name: Lookup and Output Specific Secret from Vault
  hosts: localhost
  gather_facts: false
  connection: local
  tasks:
    - name: Lookup and Output Specific Secret from Vault
      debug:
        msg: "{{ lookup('hashi_vault', 'secret=kv2/data/aws_credentials') }}"
...
```
{% endraw %}

As we see on **Line 11**, we can perform a simple lookup however the plugin only returns an entire _Secret_ and despite some suggestions in the bugtracker that returning individual secrets should be possible I've never had any success. If you have I'd love to know!

As we see below, the return data is more complex (including metadata as found in v2 _Secrets_) but the structure is by and large the same:

```bash
TASK [Lookup and Output Specific Secret from Vault] *****************************************************************
ok: [127.0.0.1] => {
    "msg": {
        "data": {
            "aws_access_key_id": "AKIAIOSFODNN7EXAMPLE", 
            "aws_default_region": "eu-west-2", 
            "aws_secret_access_key": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
        }, 
        "metadata": {
            "created_time": "2020-08-22T15:49:24.270265844Z", 
            "deletion_time": "", 
            "destroyed": false, 
            "version": 1
        }
    }
}
```

## Using The hashi_vault Plugin - A Practical Example

Returning secret information to the console is all good and well, but it's not really a useful function, especially for secrets, so let's try and do something a little more practical.

Now that we've seen how to interact with data and the structures it can be returned in, lets see how we can use it to securely connect to AWS. The below example is going to:

- Fetch our credentials from Vault
- Use those credentials to connect to AWS
- Search AWS for any _running_ EC2 instances we have and tell us their IP public IP addresses

All going well, this should find us data for 5 instances:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-1024x192.png)

```yaml{% raw %}
---
- name: Lookup EC2 Instance Data
  hosts: localhost
  gather_facts: false
  connection: local
  vars:
    aws_credential: "{{ lookup('hashi_vault', 'secret=kv/aws_credentials')}}"
  tasks:
    - name: Gather EC2 Facts
      ec2_instance_info:
        region: "{{ aws_credential.aws_default_region }}"
        filters:
          instance-state-name: [ "running" ]
        aws_access_key: "{{ aws_credential.aws_access_key_id }}"
        aws_secret_key: "{{ aws_credential.aws_secret_access_key }}"
      delegate_to: localhost
      register: ec2_instances
    
    - name: Determine Public IPs from EC2 Facts
      set_fact:
        ec2_public_ips: "{{ ec2_instances | json_query('instances[*].public_ip_address') }} "
      
    - name: Output Running EC2 Public IPs
      debug:
        var: ec2_public_ips
...
```
{% endraw %}

Let's step through a few key lines:

- **Line 7**: The _hashi_vault_ Plugin is used to fetch all data from the _aws_credentials_ secret in Vault, this data is then stored in the variable **aws_credential**.
- **Lines 11**, **15** and **15**, the values held within **aws_credential** are passed to the various authentication variables within the **ec2_instance_module**. **aws_access_key** and **aws_secret_key** are secure input variables, meaning that any output that they return will not be displayed in the console.
- **Line 13**: A filter of _running_ is applied to EC2 to ensure that only running instances are located.
- **Line 17**: All located running EC2 instances are recorded in a variable named **ec2_instances**.
- **Line 21**: A JSON query is applied to **ec2_instances** to filter out only the public IP addresses of each instance, these are then recorded in a new variable named **ec2_public_ips**
- **Lines 23**-**25**: The final data in **ec2_public_ips** is output to the console

Finally, let's confirm that this works:

```bash
PLAY [Lookup EC2 Instance Data] *****************************************************************************

TASK [Gather EC2 Facts] *************************************************************************************
ok: [127.0.0.1 -> localhost]

TASK [Determine Public IPs from EC2 Facts] ******************************************************************
ok: [127.0.0.1]

TASK [Output Running EC2 Public IPs] ************************************************************************
ok: [127.0.0.1] => {
    "ec2_public_ips": [
        "3.10.144.95", 
        "18.133.117.66", 
        "18.133.204.188", 
        "18.132.204.51", 
        "3.8.233.232"
    ]
}
```

## In Conclusion

As we see, the **hasi_vault** plugin provides us a very flexible and simple way to secure our secrets for use in Playbooks and a very simple example of how those secrets can be leveraged. This is of course just a simple example, this return data can then be used if needed to create an ephemeral inventory for node configuration or configuration can be undertaken directly against PaaS components, and of course we aren't limited to AWS or any cloud provider, secrets management is as flexible as Ansible itself.
