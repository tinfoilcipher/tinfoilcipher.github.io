---
title: "Ansible - Looping Over Lists and Dictionaries"
layout: post
date: 2022-04-06
categories: 
  - "automation"
  - "devops"
tags: 
  - "ansible"
  - "automation"
  - "devops"
---

Recently I've been writing some Ansible plays for a personal project and looking around online reminded me just how much people struggle with handling loops. It's not a huge surprise, whilst the **[documentation is pretty clear](https://docs.ansible.com/ansible/latest/user_guide/playbooks_loops.html#iterating-over-a-simple-list)** it's written in a slightly abstract way that can a little difficult to absorb if you're a newcomer to Ansible, this isn't aided by the fact that there are several options for looping and they all get muddled up together. In this very short post I'm going to take a look at how to handle simple looping.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01.png)

## One Function; _loop_

In the early days of Ansible, the only option to iterate over items was using the _with_items_ and _with_list_ functions, I've never been a big fan and their uses can get pretty messy. As of Ansible 2.5 the _loop_ function is stable and is the desired solution for all loops (though _with_ remains a supported solution).

Let's take a look at how _loop_ works in a simple debug operation:

```yaml{% raw %}
vars:
example_list_of_names:
  - Alice
  - Bob
  - Carol
  - Ted

tasks:
- name: Print out all user names
  debug:
    
    msg: "{{ item }}"
  loop: "{{ example_list_of_names }}"
```
{% endraw %}

{% raw %}
In the above example, the _loop_ function reads the contents of the **example_list_of_users** list and with each iteration passes the current value to the iterator **"{{ item }}"**. When executing we should then simply see all 4 names printed out:
{% endraw %}

```bash
TASK [Print out all user names] *********************************************************************************************************
ok: [localhost] => (item=Alice) => {
    "msg": "Alice"
}
ok: [localhost] => (item=Bob) => {
    "msg": "Bob"
}
ok: [localhost] => (item=Carol) => {
    "msg": "Carol"
}
ok: [localhost] => (item=Ted) => {
    "msg": "Ted"
}
```

So far this is analogous to using _with_items_ or _with_list_, so let's try and do something a little more advanced.

## Looping Over Dictionaries

Ideally the goal when creating _Plays_ should be to write a single templated _Task_ and have it be reusable via input variables. In the below example we're going to provide a set of _variables_ to create new local users on a remote system, providing the values as a simple _dictionary_:

```yaml
#--user_data.yaml

user_configs:
  alice:
    shell: /bin/bash    
    group: admin_users
  bob:
    shell: /bin/bash    
    group: standard_users
  carol:
    shell: /bin/zsh
    group: standard_users
```

These variables can then be consumed by a _Task_ as shown below:

```yaml{% raw %}
---
- name: Loop Example
  hosts: test_node
  gather_facts: false
  vars_files:
    - ../vars/user_data.yaml
  tasks:

  - name: Create Accounts
    user:
      name: "{{ item.key }}"
      comment: "{{ item.key }}"
      home: "/home/{{ item.key }}"
      shell: "{{ item.value.shell }}"
      group: "{{ item.value.group }}"
    loop: "{{ lookup('dict', user_configs) }}"
...
```
{% endraw %}

{% raw %}
In this _loop_, the _lookup plugin_ is used and each Key: Value pair in the dictionary is iterated over. For example the first item's _Key_ is **Alice** which can be called with **" {{ item.key }}"**, this item has two child _Values_ of **Shell** and **Group**, each of which can be called with **"{{ item.value.shell }}"** and **"{{ item.value.group }}"** respectively.
{% endraw %}

Ultimately, this _Task_ will produce our 3 new accounts (and any number of accounts we need) just by altering the input values:

```bash{% raw %}
PLAY [Loop Example] *********************************************************************************************************************

TASK [Create Accounts] ******************************************************************************************************************
ok: [localhost] => (item={u'value': {u'shell': u'/bin/bash', u'group': u'admin_users'}, u'key': u'carol'})
ok: [localhost] => (item={u'value': {u'shell': u'/bin/bash', u'group': u'standard_users'}, u'key': u'bob'})
ok: [localhost] => (item={u'value': {u'shell': u'/bin/zsh', u'group': u'standard_users'}, u'key': u'alice'})

PLAY RECAP ******************************************************************************************************************************
test_node   : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```
{% endraw %}
