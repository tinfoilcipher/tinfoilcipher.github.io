---
title: "Ansible - Querying Structured JSON From A Local File"
layout: post
date: 2021-04-21
categories: 
  - "devops"
  - "linux"
tags: 
  - "ansible"
  - "devops"
  - "integration"
  - "json"
  - "linux"
---

When I first started using Ansible, querying JSON was a source of constant frustration. Most of the articles I could find on the topic seem particularly interested in a long lesson on the topic of how JSON is structured. Whilst that is important to understand I couldn't really find a guide that just broke down a few simple queries like I wanted. I'm not even going to attempt to talk theory on JSON, there's better places than here to read about that, instead let's take a look at how to perform some straight forward JSON tasks in Ansible.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-2.png)

## Sample Data and Goals

As a sample dataset, I'm going to use an extract from an [lshw](https://linux.die.net/man/1/lshw) output:

```json
{
  "interfaces": [
    "docker0",
    "wlp3s0",
    "lo",
    "enp0s25"
  ],
  "lo": {
    "device": "lo",
    "mtu": 65536,
    "active": true,
    "type": "loopback",
    "promisc": false,
    "ipv4": {
      "address": "127.0.0.1",
      "broadcast": "",
      "netmask": "255.0.0.0",
      "network": "127.0.0.0"
    }
  },
  "docker0": {
    "device": "docker0",
    "macaddress": "01:92:41:6a:10:c1",
    "mtu": 1500,
    "active": false,
    "type": "bridge",
    "interfaces": [],
    "id": "8000.0693286d9779",
    "stp": false,
    "promisc": false,
    "ipv4": {
      "address": "172.17.0.1",
      "broadcast": "172.17.255.255",
      "netmask": "255.255.0.0",
      "network": "172.17.0.0"
    }
  },
  "wlp3s0": {
    "device": "wlp3s0",
    "macaddress": "1a:52:4b:a4:08:ab",
    "mtu": 1500,
    "active": true,
    "module": "iwlwifi",
    "type": "ether",
    "pciid": "0000:01:00.0",
    "promisc": false,
    "ipv4": {
      "address": "10.0.0.71",
      "broadcast": "10.0.0.255",
      "netmask": "255.255.255.0",
      "network": "10.0.0.0"
    }
  },
  "enp0s25": {
    "device": "enp0s25",
    "macaddress": "12:d2:be:51:6b:0b",
    "mtu": 1500,
    "active": false,
    "module": "e1000e",
    "type": "ether",
    "pciid": "0000:00:11.0",
    "speed": -1,
    "promisc": false
  }
}
```

From this data, we have an array of Interfaces, and then a set of further arrays containing data on each of these interfaces. Our goals are going to be to:

1. Work with the understanding that this dataset may change on subsequent hosts, meaning that there may be a larger or smaller number of interfaces
2. Write a query which will first check the contents of the **Interfaces** array and then use that return data as the basis for subsequent queries
3. Ensure that subsequent queries return the value in the **pciid** key of each interface, where present

This should be fairly easy to achieve with a couple of queries.

## Loading A JSON File

Before we do anything, we'll need to load the file in to memory using Ansible, whilst this can be achieved using the shell module and use of **[cat](https://man7.org/linux/man-pages/man1/cat.1.html)**, it's always best to use native functionality of Ansible. The below example uses the _file lookup_ to load the contents of our JSON:

```yaml{% raw %}
---
- name: Query Arbitrary JSON
  hosts: localhost
  gather_facts: false
  connection: local
  tasks:
    #--Load JSON file from a lookup
    - name: Load JSON
      set_fact:
        loaded_json: "{{ ( lookup('file', 'lshw.json') | from_json ) }}"
```
{% endraw %}

This loads the contents of our JSON to a fact named **loaded_json** using the _from\_json filter_. Now that the data is loaded we can start to query it.

## Performing A Simple Query

Now that our JSON has been loaded in to a fact we can query it in order to load the contents of the _interfaces_ array in to it's own fact named **loaded_json_interfaces**._ We can do this by _piping_ the existing **loaded_json** fact in to the **json_query** filter and applying the query **('interfaces')**. To make the return data easier to handle with Ansible, we'll pipe it back to YAML using the **to_yaml** _filter_ after the query has been executed:

```yaml{% raw %}
- name: Define Interfaces Array
  set_fact:
    loaded_json_interfaces: "{{ loaded_json | json_query('interfaces') | to_yaml }} "
```
{% endraw %}

This will leave us a our new fact, which now contains a **list** of all interfaces.

## Performing A Query With Iterations

Since we now need to work with a list of interfaces we'll need to think about how to iterate our query over multiple items. Due to the messy nature of jinja2 filters, we're better served by defining our query in a playbook variable before we execute it and then pass that variable to the _json query filter_ by calling that variable name. We'll define our query as **query\_pciids** and run our task to set the **loaded\_json\_pciids** fact which will contain the final item we're looking for:

```yaml{% raw %}
---
- name: Query Arbitrary JSON
  hosts: localhost
  gather_facts: false
  connection: local
  vars:
    query_pciids: '{{ item }}[*].pciid'
    
    ...

    - name: Define pccids Array
      set_fact:
        loaded_json_pciids: "{{ loaded_json | json_query (query_pciids) }} "
      with_items: "{{ loaded_json_interfaces }}"
```
{% endraw %}

There's a little more going on here so let's break it down. The query as defined in **query_pciids** is using the special variable **"{{ item }}"** which iterates over any list passed to it with the **with_items** statement. In this example, **with_items** is reading the list of interfaces contained in **loaded_json_interfaces**. We've also wildcarded the query pattern, so we're actually executing the following 4 queries one after the other:

1. **('lo.pciid')**
2. **('docker0.pciid')**
3. **('wlp3s0.pciid')**
4. **('enp0s35.pciid')**

## Putting It Together - Does It Work

So let's see the whole thing together in a _Playbook_ with some of the moving parts cleaned up:

```yaml{% raw %}
---
- name: Query Arbitrary JSON
  hosts: localhost
  gather_facts: true
  connection: local
  vars:
    json_input_file: lshw.json #--Input file
    query_interfaces: 'interfaces' #--JSON Query to look up all interfaces
    query_pciids: '{{ item }}[*].pciid' #--JSON Query to look up pciids
  tasks:
    #--Load JSON file from a lookup
    - name: Load JSON file
      set_fact:
        loaded_json: "{{ ( lookup('file', '{{ json_input_file }}') | from_json ) }}"
    - name: Define Interfaces Array
    #--Run first JSON query, lookup all interfaces
      set_fact:
        loaded_json_interfaces: "{{ loaded_json | json_query(query_interfaces) | to_yaml }} "
    #--Run second JSON query, lookup all pciids
    - name: Define pccids Array
      set_fact:
        loaded_json_pciids: "{{ loaded_json | json_query (query_pciids) }} "
      with_items: "{{ loaded_json_interfaces }}"
    #--Output the PCIIDs in the given space
    - name: Print pciids
      debug:
        msg: "{{ loaded_json_pciids }}"
```
{% endraw %}

...and let's take a look at the output:

```bash
$ sudo ansible-playbook lookup.yaml

# PLAY [Query Arbitrary JSON] ***************************************************************************************

# TASK [Load JSON] **************************************************************************************************
# ok: [localhost]

# TASK [Define Interfaces Array] ************************************************************************************
# ok: [localhost]

# TASK [Define pccids Array] ****************************************************************************************
# ok: [localhost] => (item=[docker0, wlp3s0, lo, enp0s25]
#  )

# TASK [Print JSON] *************************************************************************************************
# ok: [localhost] => {
#     "msg": [
#         "0000:01:00.0",
#         "0000:00:11.0"
#     ]
# }

# PLAY RECAP ********************************************************************************************************
# localhost                  : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
```

As we see, pciids are returned (where they exist) for all physical NICs.
