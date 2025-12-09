---
title: "Netbox - 502 Bad Gateway During Large Changes"
layout: post
date: 2019-11-11
categories: 
  - "devops"
  - "linux"
tags: 
  - "gunicorn"
  - "linux"
  - "netbox"
  - "nginx"
  - "postgresql"
---

[**Netbox**](https://netbox.readthedocs.io/en/stable/) is an incredible tool and I'll happily say I don't know how I worked before I was introduced to it, scrabbling around in leviathan (non version controlled) spreadsheets and SharePoint pages that try to perform IP address management, or even worse the notes on a scrap of paper or book on someone's desk.

There are [**other tools on the market**](https://www.ittsystems.com/best-ipam-tools-ip-address-tracking-management/), but they cost an arm and a leg for the same solution, as is often the case with open source software however, you get the support that you pay for, which is what the documentation provides you and what you're able to gleam from blog posts, wikis and bug trackers.

Given that outside of the core application Netbox relies on an application stack of [**Linux**](http://NGINX, gunicorn, Django, Python, PostgreSQL, django-pq/Redis and NAPALM), [**NGINX**](https://www.nginx.com/), [**gunicorn**](https://gunicorn.org/), [**Django**](https://www.djangoproject.com/), [**Python**](https://www.python.org/), [**PostgreSQL**](https://www.postgresql.org/), [**django-pq**](https://github.com/rq/django-rq)/[**Redis**](https://redis.io/) and [**NAPALM**](https://napalm-automation.net/), that's a lot of moving parts where something can theoretically go wrong, and in an error you could be forgiven for not understanding all of those technologies, especially when the guide makes the installation _fairly_ idiot proof and doesn't require you to understand the moving parts in depth.

## Troubleshooting - My old friend

My issue came in the form of a very unhelpful **502 Bad Gateway** when trying to upload some big data sets to the bulk uploader, the VM would choke and the web server would die. The VM is running with a single vCPU so I first look at **[top](https://linux.die.net/man/1/top)** and see that as soon as I fire a large data set in, the core runs up to 100% load and then stays there until death. Not good.

So the logical thing here is to add another vCPU right? Doing that doesn't help, the first vCPU hits a 100% while the seconds stays at around 1% load. Well at least we're making [**progress**](http://www.commitstrip.com/en/2018/05/09/progress/). A small point in the Netbox HTTP Daemon setup documentation does suggest referring to the [**gunicorn documentation**](https://docs.gunicorn.org/en/stable/) for tuning gunicorn, but in order to do this you really need to understand how the actual application stack works otherwise it's like looking through a phone book for a name you don't know.

## RTFM

This is where it's important to actually read the first page of a manual and understand what the components of the stack do, I had initially assumed that since Netbox was writing to the database, the issue lay with PostgreSQL and lost a bunch of time reading up on tuning the DB, this had nothing to do with anything.

When a request is sent from the page, it first goes to gunicorn as the WSGI (Web Server Gateway Interface) and is then proxied to NGINX (as the actual web server). This I would have understood if I'd have been reading the installation manual and not blindly bashing away at commands. This then leads me to realise that gunicorn is the weakness.

Finally going in to the gunicorn documentation, I find that it was written for some kind of gigantic brain that reads raw data in to usable information, not really my cup of tea, back to google. I found a blog post that explained a lot better that gunicorn treats it's threads as workers and threads and that the default Netbox configuration handles these as if you were using a single CPU if you don't edit the default config, this will always act as if you have a single CPU, even when you don't.  
A standard config file looks like:

```ini
command = '/usr/bin/gunicorn'
pythonpath = <your_pythonpath>
bind = '127.0.0.1:8001'
workers = 3
user = 'www-data'
max_requests = 5000
max_requests_jitter = 500
```

## What's the deal with workers?

That workers comand is a real issue, the root workers process is spawned against the first CPU and then forked in a child processes which can **then** be spread over the other CPUs, but this requires that gunicorn know about them, this calculation works as (#of_cpus * 2 + 1). E.G. a 2 CPU system gets 5 workers. Threads should also be set to match. Great right? No.  
There's also the problem of timeout. There's two problems we've been seeing here and one of them is timeout, we need to tell gunicorn how long it should allow a connection to stay open before it considers it timed out in seconds, so your final config should really look more like:

```ini
command = '/usr/bin/gunicorn'
pythonpath = 
bind = '127.0.0.1:8001'
workers = 5
threads = 5
timeout = 5000 
user = 'www-data'
max_requests = 5000
max_requests_jitter = 500
```

## What about the web server?

...and that should give us a perfect working deployment right, that 5000 second timeout is huge and more time than anyone could ever possibly need? No. Not at all. Because I still wasn't thinking how the stack worked. it's all good and well thinking that requests to gunicorn have a 500 second timeout, but NGINX is the webserver, and it has a default timeout of 30 seconds and we've still not told it anything about these changes. So when NGINX closes it's session, gunicorn can do whatever it likes, but there's no open session

The last bit of the puzzle is extending NGINX's timeout window to meet gunicorn, which is edited inside the site config file for your site (usually **/etc/nginx/sites-available/netbox**) if you're following the recommended locations.

A few lines should be added to the **location** block to read:

```bash
location {
    ...
    proxy_connect_timeout       5000;
    proxy_send_timeout          5000;
    proxy_read_timeout          5000;
    send_timeout                5000;
    ...
}
```

Finally a restart of all services in the stack:

```
sudo systemctl restart nginx
sudo supervisorctl restart netbox
```

This should see you with a much better tuned deployment that can handle gigantic data sets being uploaded:
