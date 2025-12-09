---
title: "Ansible - Calling REST APIs and Sending to Webhooks"
layout: post
date: 2020-05-01
categories: 
  - "devops"
  - "linux"
tags: 
  - "ansible"
  - "api"
  - "devops"
  - "linux"
  - "rest"
  - "webhooks"
---

A useful function nested within Ansible is the ability to query remote REST APIs, return the JSON data, parse it and perform subsequent actions based on the data that your get back.

When we make the subsequent action sending to a remote Webhook we can then make the function even more powerful (most of the time that is going to be sending a notification to a remote system to let you know that something has happened), in that sense we can create a rudimentary alerting system **inside the Playbook**.

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-9.png" class="scaled-img-75">

## A Basic Scenario

Lets take a basic scenario where we want to query a REST API, and if it contains a certain value, we want to send an alert to a remote Webhook. We **could** simply say do this if the task fails, but this may not be what we want to happen, and the criteria that is needed to be considered a failure might not be triggered, this would also preclude us from performing a number of API queries in a single playbook.

In this example we're going to send an alert when the remote API returns a _Status_ value of anything other than **Operational**. We'll also assume that the API is accepting authentication by means of a Bearer Token.

## Performing the API Query

Let's take a look at the _Task_ below which we can use to query the API using the Ansible **uri** Module:

```yaml{% raw %}
- name: Perform API GET to return the status of remote service
  uri:
    url: "{{ your_api_endpoint }}"
    method: GET
    return_content: yes
    body_format: json
    status_code: 200
    headers:
      Authorization: "Bearer {{ your_bearer_token }}"
  register: results_of_get
  no_log: True
```
{% endraw %}

A few things going on here:

- **Line 7**: We're defining the return code as **HTTP 200**, any other return code will result in failure
- **Lines 8-9**: We're defining that a **Bearer Token** be sent as a header
- **Line 10**: The results of the GET will be saved in a new variable called **results_of_get**
- **Line 11**: We're specifying that no log be displayed in the console, as the token will be displayed in the clear and as this is secret data we probably don't want it to be displayed

## Manipulating the Returned Data and Sending to a Webhook

Now that we have queried the endpoint, we need to determine what we actually want to do with it, and if we want to send anything out to our Webhook or not. Below is the task that we'll use for this. Once again we will be using the **uri** module:

```yaml{% raw %}
- name: Read the return data and POST to webhook if anything other than Opeartional is found in Status
  uri:
    url: "{{ your_incoming_webhook }}"
    method: POST
    return_content: yes
    body: '{ YOUR BODY HERE }'
    body_format: json
  when: 
    - results_of_get.json.value[0].Status != 'Operational'
```
{% endraw %}

## A Practical Example - Microsoft Teams Webhooks

Now that we've seen some snippets, let's look at a functional example of a full playbook, sending to a Microsoft Teams Webhook:

```yaml{% raw %}
---
- name: GET_and_POST
  hosts: localhost
  gather_facts: false
  any_errors_fatal: true
  vars:
    your_api_endpoint: "https://some.imaginaryorg.com/fictional/endpoint"
    your_incoming_webhook: "https://outlook.office.com/webhook/<TEAMS_WEBHOOK_UID>"
  
  tasks:
  - name: Perform API GET to return the status of remote service
    uri:
      url: "{{ your_api_endpoint }}"
      method: GET
      return_content: yes
      body_format: json
      status_code: 200
      headers:
        Authorization: "Bearer {{ your_bearer_token }}"
    register: results_of_get
    no_log: True
    
  - name: Read the return data and POST to webhook if anything other than Opeartional is found in Status
    uri:
      url: "{{ your_incoming_webhook }}"
      method: POST
      return_content: yes
      body: '{"@type":"MessageCard","@context":"http://schema.org/extensions","themeColor":"0076D7","summary":"FAILURE"}'
      body_format: json
    when: 
      - results_of_get.json.value[0].Status != 'Operational'
...
```
{% endraw %}

This will ensure that a Message Card notification is generated in Teams with the summary **FAILURE** when the Status is returned with anything other than **Operational**. This is a very basic message which can be spruced up significantly, a fantastic utility for designing this payload exists at [MessageCard Playground](https://messagecardplayground.azurewebsites.net/).

This can then be linked to a schedule either with **cron** or if using **Ansible Tower**, via the native _beat_ scheduler.
