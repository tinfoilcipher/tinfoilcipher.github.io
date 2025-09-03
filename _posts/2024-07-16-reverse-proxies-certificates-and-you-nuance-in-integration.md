---
title: "Reverse Proxies, Certificates and You - Nuance in Integration"
layout: post
date: 2024-07-16
categories: 
  - "devops"
  - "opinions"
  - "security"
tags: 
  - "devops"
  - "integration"
  - "networking"
  - "nginx"
  - "opinions"
  - "pki"
  - "security"
---

A while ago I wrote an article breaking down how to deploy [_Hashicorp Vault_ using NGINX as a reverse proxy](/hashicorp-vault-reverse-proxy-with-nginx/). It has been a popular article but after it had been up for a couple of years I got some comments that my proposed method wasn't recommended and that using an _HTTP Reverse Proxy_ generally is insecure for a few reasons.

I don't like the idea of putting bad information out in to the world, especially information that gets seen so much by people who are obviously trying to do the same thing as me, so I set some time aside to try and investigate if using an HTTP proxy is such a bad idea. In the process I realised that there is really quite a lot of confusion going on and a lot of misconceptions flying around, so I thought I'd talk a bit about it and maybe someone will find that useful.

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-1.png" class="scaled-img-50">

## What Even Is A Reverse Proxy?

_Reverse Proxy_ is a complicated **sounding** name for a simple idea. As the name implies they act as a "proxy" for clients making requests to a regular web service. As a client you might think you're interacting with a regular web server but in reality you are speaking to a _Reverse Proxy_ sitting in front, which forwards your traffic on to an _upstream_ service as appropriate, the application that eventually receives this traffic is referred to as the **origin**. Usually _Reverse Proxy_ applications are bundled with other widely-needed features; such as load balancing or auth services.

Generally, most of our applications run on [Unprivileged Ports](https://www.w3.org/Daemon/User/Installation/PrivilegedPorts.html) ([MySQL](https://www.mysql.com/)** runs on TCP/3306, [Postgres](https://www.postgresql.org/) runs on TCP/5432). These ports are _Unprivileged_ in the sense that any _user_ can bind one of them to a service and create a [socket](https://en.wikipedia.org/wiki/Network_socket), offering it up for connections. Long ago in the original designs of the Unix Kernel a design decision was taken to earmark the first 1024 ports as _Privileged Ports_, reserved for use by legitimate and well known services.

A service can only be bound to one of these _Privileged Ports_ by _root_ (or at least a superuser), so in order for your application to present itself directly on one of them it too would need to run as root. It is generally undesirable to give an application such high privileges or to expose them fully to consumers given the impact that being compromised might have.

This is where our _Reverse Proxy_ steps in. it often does run on a _Privileged Port_ (specifically TCP/443 to accept HTTPS connections) and forwards traffic to your _origin_ application at a specific port and address. This has the advantage that a single proxy may manage traffic for multiple logical _origins_, forwarding traffic to them as appropriate. With this separation of frontend and backend, the _proxy_ and _origins_ can be hardened and managed as separate elements.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-proxy.png)

Other than port translation and process separation, there are some more practical reasons to consider a _Reverse Proxy_ for your application:

- **Traffic Shaping and Manipulation**: Various forms of traffic management before reaching your _origin_

- **High Availability**: Various forms of _Load Balancing_ can be implemented before traffic hits your _origin_

- **Authentication and Authorisation**: Admission control can be handled in front of your __origin__.

- **Strict Firewall Requirements**: Even if you wanted to, you may not be able to open yet another port on your _downstream_ firewall

- **Compatibility**: Exposing applications on _Unprivileged Ports_ can run in to strange configuration issues with some external firewalls, usually because you're encouraged not to use them, best to do things properly and avoid that.

## Layer 4 and Layer 7 Reverse Proxies

There are two main types of _Reverse Proxy_ out there, **[Layer 4 (TCP/UDP)](https://en.wikipedia.org/wiki/OSI_model)_ [and](https://en.wikipedia.org/wiki/OSI_model) _[Layer 7 (HTTP)](https://en.wikipedia.org/wiki/OSI_model)**. I have noticed a tendency to oversimplify the differences between them and be quite prescriptive about which one is "best" for different circumstances. This isn't helped by the fact that several of the biggest products can be run in both Layer 4 and Layer 7 modes and have big, vocal fan bases. In an effort to try and clarify things, the table below mentions some of the popular options and how you can use them:


| Application               | Layer Modes | Platform                   |
|---------------------------|-------------|----------------------------|
| NGINX                     | 4, 7        | Linux, Windows, Kubernetes |
| Apache                    | 7           | Linux, Windows             |
| HAProxy                   | 4, 7        | Linux, Kubernetes          |
| Squid                     | 7           | Linux, Windows             |
| IIS ARR                   | 7           | Microsoft Windows          |
| AWS ALB                   | 7           | Amazon Web Services        |
| Azure Application Gateway | 7           | Microsoft Azure            |

<div class="center-text">
  Other brands are available, consult the internet for bickering.
</div>

There is often a lot of confusion online when it comes to discussing the use of _TCP/UDP_ and _HTTP_ as if they are exclusive, but really TCP/UDP and HTTP work together as part of the process of **[encapsulation](https://en.wikipedia.org/wiki/Encapsulation_\(networking\))**. When you make an _HTTP_ request, the application data is ultimately sliced up in to TCP segments (or UDP datagrams), which are in turn used to transport the original data to your application.

Let's take a look at how these two types of _Reverse Proxy_ fundamentally differ:

### **Layer 4: (Transport)**

These will handle inbound _TCP_ or _UDP_ connections and forward them to your application. This can be used for any application that uses TCP/UDP as it's _Transport_ layer, including HTTP.

- This can be a very fast and efficient arrangement and the proxy often has much less work to do. We can just forward TCP segments directly to our __origin__ in one continuous stream which is pretty inexpensive for the proxy. The down side is that your application will always have a 1:1 relationship between the amount of client connections coming in the amount of connections your _origin_ has to handle.

- TLS encrypted data can be forwarded transparently to the _origin_ (a process called _TLS passthrough_). Despite what you read, Layer 4 _Reverse Proxies_ can terminate TLS just fine, but it's uncommon and not very practical.

- Caching isn't usually an option since responses tend to be handled by the _origin_.

- Whilst fast, this is a rigid system. The _proxy_ can't inspect the application traffic it's forwarding, it can only see the TCP/UDP headers and so can only make routing decisions based on that information, practically this boils down to source and destination IP. This is potentially very expensive for the _origin_ as you cannot rewrite data before it reaches the application which might then have to do a lot of heavy lifting.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-1-rproxy.png)

### **Layer 7: (Application)**

These will handle inbound HTTP connections up front and then establish a TCP session with the __origin__ (typically using an **[HTTP tunnel](https://en.wikipedia.org/wiki/HTTP_tunnel)**). Technically this means that every connection to the _proxy_ results in two TCP sessions (one on the frontend and one on the backend) and the application traffic is relayed between them by the _proxy_ (the specifics of how this works differs between HTTP versions, just to make life harder).

- This arrangement provides us with a lot of flexility to make routing decisions based on any of the other layers down to and including _Layer 4_. This means things like hostnames, paths, certificate metadata, origin are now all on the table.

- HTTP requests can be modified before reaching the application as required, headers can be added, stripped etc. This is attractive for flexility and application performance but has security implications that shouldn't be ignored. This could be a juicy target for someone trying to attack your system so hardening is essential.

- Effective request caching is often built in to _Reverse Proxies_ that can mean less work for your application by serving up content before requests reach the _origin_.

- Connection management can be a lot smarter, multiple frontend requests can often be worked through less backend requests (a concept called **[Connection Pooling](https://en.wikipedia.org/wiki/Connection_pool)**).

- The trade off here is that all this extra processing is expensive for the proxy, but potentially takes a lot of cost away from the __origin__ application.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/06-01-rproxy.png)

## The Crazy World of TLS Certificates

OK, that was complicated enough, so let's make it worse...

Certificates have a tendency to make everything a lot more complicated. In my experience they are a very misunderstood topic (with good reason, they are built on decades of jank, oblique systems and legitimate complexity). Their inclusion in any system tends to lead to a lot of misconceptions and there are a couple points of confusion that I keep seeing over and over again that relate to _Reverse Proxies_:

### HTTPS Proxies - Insecure On The Inside?

The most common thing I keep hearing is that you should avoid using an _HTTP Reverse Proxy_ since you will be forced to terminate TLS at the proxy and not at your __origin__. This will lead to a situation where you have an insecure channel between your proxy and application. Is that true?

Well, it could be true, it depends on your configuration. Most example configurations you see in documentation suggest such a configuration and this would indeed leave your traffic beyond the proxy visible to anyone with superuser permissions on the host. The traffic will be will be visible to a packet sniffer including the responses from the application.

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/rproxy-07-1.png">
  <figcaption>In this configuration any traffic sent to the origin will be IN THE CLEAR and visible to a packet sniffer running on the reverse proxy host. Traffic is also potentially visible to a number of other methods, such as reading data from memory, caches or application logs. Any these methods however will require superuser permissions.</figcaption>
</figure>

A potential way to avoid this is to encrypt your traffic to the __origin__ target. In this method another session is started which itself is TLS encrypted, meaning that no data has to transmitted in the clear. I have heard claims that because traffic is decrypted before being sent to the __origin__ that a design like this is inherently insecure, but I disagree with this.

Under such a configuration traffic is never **transmitted** in the clear. When a TCP session is established with your __origin__ a new TLS handshake is completed before any data is actually sent. It **is** true however that data may be stored in a request cache or in memory, so it is critical that the HTTP proxy be properly configured to prevent these being accessed without a serious effort, but this is true of any secure host.

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/09-rproxy.png">
  <figcaption>In this configuration, any traffic sent to the origin will be encrypted and not visible to a packet sniffer running on the Reverse Proxy host. Traffic is still potentially visible in all the same ways to an attacker on an improperly hardened host (such as caches, memory, logs etc.). There is also the potential that an attacker may attempt to simply redirect or mirror traffic to an insecure origin. Such attacks still require superuser permissions.</figcaption>
</figure>

Another benefit offered here is that your proxy and __origin__ application will likely be using two different TLS implementations. This is attractive in high security environments as a vulnerability in one would be impossible to leverage because of protection by the other.

### The Cost of Encryption

I often see posts where people are trying to secure all connections in their system with TLS. Whilst this is perhaps attractive for security purposes it carries a performance overhead that should not be ignored. Every session that needs to use TLS also needs to use a little of CPU for both the encryption and decryption processes.

Depending on how you design your network that overhead doesn't need to be huge but sticking to a rigid mindset of "everything needs to be encrypted in transit" does have costs. It is perhaps more useful to consider instead is the data being transmitted behind a proxy even worth encrypting? Or is it being transmitted inside an environment secure enough that this risk can be sacrificed or mitigated for the sake of high performance?

If you find yourself in an industry or environment where this decision is taken our of your hands, it is usually wise to employ _TLS Offloading_ if your proxies are struggling with CPU use, this is the process of offloading cryptographic tasks to dedicated hardware and letting the proxy/loadbalancer worry about their day jobs.

## Building Secure Systems in Reality

When working with the kind of complex integrations discussed in this article, there is a tendency to idealise the systems we're building and picture them in a mythical greenfield deployment. This mindset is nice, but in reality the constraints you might find yourself under in the field will probably look nothing like the lab.

Security in the real world is a messy business and often you will need to know how to make the most secure product you can, using the systems you have access to whilst delivering the best performance. Delivering the best solution only from understanding the nuance of how they work for individual scenarios.

As we can see, both _Layer 4_ and _Layer 7_ proxies have their place. Often there is not really much of a technical barrier to using either for the same application, you **could** stand up either but that decision should be based on:

- What your application actually does and how it needs to process data

- The type and volumes of data your application processes and how sensitive it is

- How secure the rest of your environment is and if you are in a position to harden it

- What kind of company you work at and what their policies are for encryption

Hopefully my point is coming across here that there is no "right" way of designing an integration like this and I find myself suspicious of anyone claiming to have all of those answers. In fact, the methods I've talked about here are just the tip of the iceberg, there are countless other ways you could expose your application in a secure manner and plenty of other good ways you could leverage a _Reverse Proxy_, these are just the broad strokes. Designing a solution like this requires a good understanding of several overlapping, complicated disciplines and the system you build is going to look completely different depending on your needs.
