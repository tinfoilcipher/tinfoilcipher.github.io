---
title: "Chartmuseum Helm Repository - Reverse Proxy with NGINX"
layout: post
date: 2021-04-27
categories: 
  - "containers"
  - "devops"
  - "linux"
tags: 
  - "chartmuseum"
  - "cloud"
  - "devops"
  - "helm"
  - "kubernetes"
  - "microservices"
  - "nginx"
---

In the **[previous post we looked at how to build Chartmuseum on Ubuntu Linux with an S3 backend]({% post_url 2021-04-26-creating-a-private-helm-repo-with-chartmuseum-using-aws-s3 %})**, however out of the box this system presents a number of problems; specifically it isn't TLS encrypted and the service runs on an **[unprivileged TCP port](https://www.w3.org/Daemon/User/Installation/PrivilegedPorts.html)**. I could see no guides suggesting how to do this, so lets take a look at how to solve this problem by performing by proxying our connections through an NGINX reverse proxy with TLS termination.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-4.png)

## Unprivileged Ports?

The **[Chartmuseum documentation](https://chartmuseum.com/docs/)** suggests running the service on TCP port 8080, this is an _Unprivileged Port_, meaning that any user can bind it to a service. Long ago in the original designs of the Unix Kernel (and as a byproduct, the Linux Kernel) a design decision was taken to earmark the first 1024 ports as _Privileged Ports_ (this is why the IANA designate these for the most critical operations). The theory went that back in the days when only sysadmins operated computers you could be sure that if you were connecting to one of these ports over a network you could rest assured that it had been configured by a root user (at least a superuser).

Since we ultimatley want to serve out service over HTTPS using TCP port 443 we'll need to front the service with a proxy, as attempting to start the application using this port will result in failure, so let's take a look at what we need.

## Our Goal and Prerequstes

We'll be using the existing Chartmuseum deployment **[created in the previous post]({% post_url 2021-04-26-creating-a-private-helm-repo-with-chartmuseum-using-aws-s3 %})**, with the same _Service Account_ configuration.

**Technically**, we could perform TLS termination on the Chartmuseum instance and offer out the deployment at TCP port 8080, however if you have strict firewall requirements you may not want to open yet another port, and fewer ports is always nicer (as well as conforming to proper standards). So let’s look at how to use a reverse proxy to offer out Chartmuseum correctly over TCP port 443 using NGINX.

Our finished instance is going to look something like this:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-4.png">
  <figcaption>In a nutshell...</figcaption>
</figure>

## Implementation - NGINX

We’ll need to get NGINX installed and configured, the only installation needed is NGINX and that’s a single command, we're using Ubuntu 18.04 so we can use **apt-get**:

```bash
sudo apt-get install nginx
```

Ahead of time I've already created a TLS _Certificate_ and _Private Key_ and placed them in **/etc/ssl/certs** and **/etc/ssl/private** respectively. These have been created from my **[private certificate authority]({% post_url 2019-11-15-bind-dns-and-openssl-certificate-authority %})** but **[letsencrypt](https://letsencrypt.org/)** will do you just fine if you don't have one.

Now we can create our NGINX configuration:

```bash
sudo nano /etc/nginx/sites-enabled/default
```

...and paste in our configuration:

```bash
server {
        listen 443;
        server_name mc-chartmuseum.madcaplaughs.co.uk;

        ssl on;
        ssl_certificate /etc/ssl/certs/mc-chartmuseum.cer;
        ssl_certificate_key /etc/ssl/private/mc-chartmuseum.key;

        ssl_prefer_server_ciphers on;
        ssl_session_timeout 1d;
        ssl_session_cache shared:SSL:50m;
        ssl_session_tickets off;

        location / {
                proxy_pass http://127.0.0.1:8080;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto https;
        }
}
```

If we look closely at the **location** block, we can see that requests sent to our _server\_name_ (defined in the **server** block) to the host and port defined as **proxy\_pass** with some additional headers which will also be included.

In this example my server is named **mc-chartmuseum.madcaplaughs.co.uk**, yours should be substituted with the FQDN of your own host (which will also need to be reflected in the _CNAME_ on your Certificate). We are also specifying the path to the_Certificate_ and _Private Key_.

## Completion and Verification

Now that our configurations are in place we'll need to restart NGINX to bring everything up:

```bash
#--Restart NGINX
sudo /etc/init.d/nginx restart
# [ ok ] Restarting nginx (via systemctl): nginx.service.

#--Verify that NGINX is running
sudo /etc/init.d/nginx status
# nginx.service - A high performance web server and a reverse proxy server
#  Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
#  Active: active (running) since Sun 2021-04-25 10:47:01 UTC; 2s ago
#    Docs: man:nginx(8)
# Process: 4545 ExecStop=/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile /run/nginx.pid (code=exited, status=0/SUCCESS)
# Process: 4559 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
# Process: 4548 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
# Main PID: 4560 (nginx)

```

Now if we attempt to test our connection via a brower on TCP port 443 we should see a valid certificate presented:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-1.png">
  <figcaption>We're proxied!</figcaption>
</figure>

Finally, if we attempt to add our repository using _Helm_ we can also see that we are able to communicate using TCP port 443 without issue and without any TLS warnings being presented (as would be the case if we attempted to add a repository over TLS without a valid certificate chain present:

```bash
helm repo add mc-chartmuseum https://mc-chartmuseum.madcaplaughs.co.uk
# "mc-chartmuseum" has been added to your repositories
```

TLS sessions are now terminated at NGINX for all requests and proxied in to Chartmuseum.
