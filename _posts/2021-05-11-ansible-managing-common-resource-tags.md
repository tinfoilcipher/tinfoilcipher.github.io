---
title: "Ansible - Managing Common Resource Tags"
layout: post
date: 2021-05-11
categories: 
  - "automation"
  - "aws"
  - "azure"
  - "devops"
  - "gcp"
tags: 
  - "ansible"
  - "automation"
  - "aws"
  - "azure"
  - "cloud"
  - "devops"
  - "gcp"
---

**[Ansible](https://www.ansible.com/)** is a big favourite of mine as anyone that knows me will tell you and has become one of the biggest players in the DevOps world, inevitably if you're going to use it at any real scale you'll need to start thinking about _tags_. **[Tags](https://en.wikipedia.org/wiki/Tag_\(metadata\))** are an essential part of life in the cloud, given the scale and complexity we can encounter they really become the only way to track ownership, billing and other arbitrary metadata. Managing them on a per-resource basis is not only cumbersome it's the height of inefficiency. In this post we'll take a look at how to use pre-constructed _dictionaries_ of _tags_ to assign the same metadata to all resources in a given environment.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01.png)

## A Note on Clouds and Input

Most Ansible _modules_ already allow for the input of _tags_ on provision, our aim here is to remove the manual input of the same _tags_ for provisioning each different instance of every different resource and replace that with a single standard set.

We'll be working with AWS in these examples, though the same logic also applies to GCP and Azure and applies to the **vast majority** of resources which can be provisioned on all platforms as they all accept tags in a Key: Value pair format. AWS does however present an oddity with Ansible in terms of handling tags after provisioning in that _tags_ can be manipulated without modifying their resource (see the [**AWS EC2 Tag**](https://docs.ansible.com/ansible/latest/collections/amazon/aws/ec2_tag_module.html) module for an example).

## Defining Tags as a Dictionary

So let's take a look at how we might define some tags in Ansible with some data that we might expect to use in a typical environment. First we'll create a YAML file named **vars.yaml** which we'll use to inject _input variables_ to our _playbooks_:

```yaml
#--vars.yaml

---
default_tags:
  Administrator: Welsh
  Department: IT
  CostCentre: ABC123
  ContactPerson: andy@tinfoilcipher.co.uk
  ManagedByAnsible: True
...
```

In the above example we're creating a single _dictionary_ named **default\_tags** which contains a set of Key: Value pairs which will act as our tag data.

## Provisioning Resources

{% raw %}
Now that we have defined a _dictionary_ lets create some resources. For the sake of simplicity we'll just make some VPCs. As we see below instead of defining individual tags we can simply pass the **"{{ default_tags }}"** variable to the **tags** argument:
{% endraw %}

```yaml{% raw %}
---
- name: Create AWS Components
  hosts: localhost
  gather_facts: false
  connection: local
  tasks:
    - name: Create VPCs
      amazon.aws.ec2_vpc_net:
        name: "{{ item[0] }}"
        cidr_block: "{{ item[1] }}"
        region: eu-west-1
        tags: "{{ default_tags }}"
        tenancy: dedicated
      with_together:
        - ['VPC-Dev', 'VPC-Test', 'VPC-Prod']
        - ['10.1.0.0/16', '10.2.0.0/16', '10.3.0.0/16']
...
```
{% endraw %}

We can then run our _playbook_ and pass in the extra variable file using the **\--extra-vars** argument:

```bash
sudo ansible-playbook build_aws.yaml --extra-vars="@vars.yaml"

# PLAY [Create AWS Components] ****************************************************************

# TASK [Create VPCs] **************************************************************************
# changed: [localhost] => (item=[u'VPC-Dev', u'10.1.0.0/16'])
# changed: [localhost] => (item=[u'VPC-Test', u'10.2.0.0/16'])
# changed: [localhost] => (item=[u'VPC-Prod', u'10.3.0.0/16'])

# PLAY RECAP **********************************************************************************
# localhost                  : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

Looking at the console we can see that our VPCs have provisioned and their tags have been correctly applied:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-1-1024x200.png)

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-1.png)

## Merging Multiple Tag Sets

Another scenario we might encounter is where we want to want to apply two or more sets of _tags_ to resources in the same environment but limit the application of these tags to specific provisioned resources. Ansible's _[**combine**](https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters.html#combining-hashes-dictionaries)_ _filter_ allows us to manage this additional functionality on a per-_module_ basis.

Let's modify our **vars.yaml** to include an additional _dictionary_ named **optional\_tags**:

```yaml
#--vars.yaml

---
default_tags:
  Administrator: Welsh
  Department: IT
  CostCentre: ABC123
  ContactPerson: andy@tinfoilcipher.co.uk
  ManagedByAnsible: True
optional_tags:
  DevelopmentEnvironment: True
  SecondaryContact: welsh@tinfoilcipher.co.uk
...
```

In the _playbook_ below we can then change the input to the **tags** argument to merge both of these _dictionaries_ using the _combine filter_ and create some _VPC Subnets_ using both sets of tags:

```yaml{% raw %}
---
- name: Create AWS Components
  hosts: localhost
  gather_facts: false
  connection: local
  tasks:
    - name: Create Subnets
      amazon.aws.ec2_vpc_subnet:
        state: present    
        vpc_id: vpc-04b75b47f5d954322
        cidr: "{{ item }}"
        tags: "{{ default_tags | combine(optional_tags) }}"
      with_items:
        - '10.1.1.0/24'
        - '10.1.2.0/24'
        - '10.1.3.0/24'
...
```
{% endraw %}

Executing the _playbook_ again, we can see that our subnets have been created and both _dictionaries_ of _tags_ have been applied:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/06.png)

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/05.png)
