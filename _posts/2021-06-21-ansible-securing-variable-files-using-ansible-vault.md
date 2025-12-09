---
title: "Ansible - Securing Variable Files Using Ansible Vault"
layout: post
date: 2021-06-21
categories: 
  - "automation"
  - "devops"
  - "linux"
  - "security"
tags: 
  - "ansible"
  - "ansiblevault"
  - "automation"
  - "devops"
  - "secops"
  - "secrets"
  - "security"
---

**[Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html)** isn't, if I'm honest, a solution that I've ever found **much** use for in my day to day work. I prefer to use a centralised _Secrets Management_ solution wherever it's practical (particularly favouring **[Hashicorp Vault](https://www.vaultproject.io/)**). These systems however are time consuming to properly deploy have a steep learning curve, depending on the scale of your deployments and integration requirements _Ansible Vault_ might serve you just fine and I often find myself using in it in a pinch. In this short post we're going to look at securing multiple variables for your _Ansible_ project.

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-9-832x1024.png" class="scaled-img-50">

## How Does It Work?

Obviously, we never want to save our _Secret_ data inside our _Playbooks_ in the clear so we need a method to pass in these values from an external source, ideally from one which is _Encrypted At Rest_ and can only be accessed by an operator who possess the decryption passphrase. _Ansible Vault_ ticks these boxes (the encryption specification is impressive and is defined in detail **[here](https://docs.ansible.com/ansible/latest/user_guide/vault.html#ansible-vault-payload-format-1-1-1-2)**). Lets take a look at managing some secrets.

## Working With Multiple Secrets?

I'm not going to look at _[**encrypting single secrets using Ansible Vault**](https://docs.ansible.com/ansible/latest/user_guide/vault.html#encrypting-individual-variables-with-ansible-vault)_ and the rest of the components in that process, honestly I find the method of managing single encrypted secrets within _Ansible Vault_ too cumbersome and not very friendly to work with when given the option to use an entire file.

Often this problem is solved with the use of **Environment Variables** but I fine that this often proves brittle in local development as we need to manage a lot of values manually and gets messy quickly within a _Automation Pipeline_ as more values get added along the way.

Lets take a look at an example _Variables_ file:

```yaml
#--vars.yaml

---
authentication_data:
  username: api_service
  password: Sup3rS3cr3tP@55w0rD!
  api_token: aosd8910mfm02m3mflwi3yu4921j39f4f932o8hdeo243nfmp23d
...
```

Pretty simple, this is going to serve as the authentication data for a basic API endpoint, but all of that information is pretty sensitive, so let's get it encrypted. We'll need to enter an encryption passphrase for which we'll use **D3crypt10nPassphras3!**:

```yaml
#--Encrypt file vars.yaml with ansible-vault
sudo ansible-vault encrypt vars.yaml
New Vault password:                     #-- Output Obscured
Confirm New Vault password:                      #-- Output Obscured
# Encryption successful
```

If we try to read the file, we can now see that it's been encrypted:

```yaml
sudo cat vars.yaml 
# $ANSIBLE_VAULT;1.1;AES256
# 31326533303166326138343033346366313030326539323765316165636331396365663434623135
# 6564363365333136626262343964396539326631303964380a316566396632323435663538303538
# 39326362343633333262663738353264303639376662666532303938666239383061313833623132
# 6339646630613334310a346631356663653231363562363733313236373938356162376132663933
# 61386133346634343432356635353133346339333639653161313962343736643062633738393431
# 66386637336366396133613266663661366666623434373335653930346666663736626234383164
# 39663766323635663031663533623061303530376131376637616633643265663964643235656332
# 66383830343639303131333064396662616633616531333435366132386661383765333063333366
# 36623666316631376332613333633834306661633130353434326532646136373438336364623161
# 30323637633739346230383039363463633734386637323734393333626236633539383236316462
# 34306430623336323063386533653830336532383766386163323363646638666133656166343436
# 64393561346561376663613733373561653664636132373235623936316138356664383833316339
# 66393036363038626663656362373734376230613731636663343263333466313264
```

We can edit this file to add further secrets at a later date provided that we have the _Encryption Passphrase_:

```bash
sudo ansible-vault edit vars.yaml
Vault password:                     #-- Output Obscured

```

This opens the file in memory (in _Vi_ by default) where you can add additional secrets:

```bash
# ---
# authentication_data:
#   username: api_service
#   password: Sup3rS3cr3tP@55w0rD!
#   api_token: aosd8910mfm02m3mflw;i3yu4921j39f4f932o8hdeo243nfmp23d
# ...
# ~                                                                                                                        
# ~                                                                                                                        
# ~                                                                                                                        
# ~                                                                                                                        
# "~/.ansible/tmp/ansible-local-jD97HFVP3a/tmpF3a0f1.yaml" 7 lines, 184 characters
```

## Using The Secrets File

So lets take a look at passing these secrets to a _Playbook_.

First let's look at an example _Play_book with a single _Play_ and _Task_; it's going to be a simple GET request to an API endpoint that uses our username, password and token from the variable file:

```yaml{% raw %}
#--api_play.yaml

---
- name: API Query Example
  hosts: localhost
  gather_facts: false
  connection: local
  tasks:
  - name: Get App Server Auth Data 
    uri:
      url: "https://apps.madcaplaughs.co.uk/auth?token={{ authentication_data.api_token }}"
      user: "{{ authentication_data.username }}"
      password: "{{ authentication_data.password }}"
      method: GET
      status_code: 201
      register: api_data
...
```
{% endraw %}

As we see on **Lines 9-11** our secret values are being references from **vars.yaml**

We can execute with the below command, note the use of the **\--extra-vars** argument to lookups **vars.yaml** as external variables and **\--ask-vault**\-**password** to request the _ansible-vault_ password at run time:

```bash
sudo ansible-playbook api_play.yaml --extra-vars="@vars.yaml" --ask-vault-password
Vault password:                     #-- Output Obscured
# PLAY [API Query Example] *****************************************************************************************
# 
# TASK [Get App Server Auth Data ] *********************************************************************************
# 
# PLAY RECAP *******************************************************************************************************
# localhost                  : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

## Automation - Removing Manual Input

So this works great for an interactive solution but it's not exactly suited to _Automation Pipeline_ (such as we'd find in a _CI/CD_ solution) and automating things is the name of the game. The **[ansible-vault CLI reference](https://docs.ansible.com/ansible/latest/cli/ansible-vault.html)** gives us another option for authenticating in the form of the **\--vault-pass-file** argument, this can be as simple as a flat file which contains our decryption passphrase.

Obviously we don't really want to do this day to day or on a local machine but typically a build container is fine for a process like this as they're usually ephemeral and non-interactive. Loosely we can do something like:

```bash
#--Export our encryption passphrase to an Environment Variable. Automation Pipeline (CI/CD) solutions provide
#--this function internally and this step does not usually need to be performed, this is an illustration only
export ANSIBLE_VAR_VAULT_PASSWORD=Sup3rS3cr3tP@55w0rD!

#--Write value to a temporary file
echo $ANSIBLE_VAR_VAULT_PASSWORD >> vaultpass.temp

#--Set Read Only Permissions on temporary file for the file owner, the exection bit must NOT be set
chmod 400 vaultpass.temp

#--Run ansible-playbook, using the --vault-pass-file
ansible-playbook api_play.yaml --extra-vars="@vars.yaml" --vault-password-file="vaultpass.temp"
# PLAY [API Query Example] *****************************************************************************************
# 
# TASK [Get App Server Auth Data ] *********************************************************************************
# 
# PLAY RECAP *******************************************************************************************************
# localhost                  : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

#--Delete temporary file
rm -f vaultpass.temp
```

Not perfect, but it does work, as ever if anyone has a better implementation don't keep it to yourself!
