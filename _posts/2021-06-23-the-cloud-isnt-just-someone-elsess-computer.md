---
title: "The Cloud Isn't \"Just Someone Elses's Computer\""
layout: post
date: 2021-06-23
categories: 
  - "devops"
  - "opinions"
tags: 
  - "cloud"
  - "devops"
  - "opinions"
---

I have a t-shirt that says "There Is No Cloud, It's Just Someone Elses's Computer", I also have that same quote on a sticker on the laptop I'm writing this on. It's a good gag and it's a view I **used to** subscribe to but it's not really true. It's fair to say that public clouds **run** on someone elses's computer but that's a big distinction.

There's a million articles out there breaking down the differences between _Public_, _Private_ and _Hybrid Clouds_, none of which I'm going to get bogged down in, really when most people talk about "_The Cloud_" they're almost always talking about _Public Cloud_ and most of the time they're talking about one of the 3 biggest players; **[Amazon Web Services](https://aws.amazon.com/)**, **[Microsoft Azure](https://azure.microsoft.com/en-gb/)** and **[Google Cloud Platform](https://cloud.google.com/)**.

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01.jpg" class="scaled-img-50">
  <figcaption>Technically yes, but no, sort of...</figcaption>
</figure>

## My Own Experience (In Brief)

Around 2009 I first started to really hear about some technologies gaining traction from Amazon called **[EC2](https://aws.amazon.com/ec2/)**, [**S3**](https://aws.amazon.com/s3/) and **[EBS](https://aws.amazon.com/ebs/)** and I couldn't quite work out what the attraction was (nevermind the confusing names, thanks Amazon). To my eye they just looked a way to run your normal VMs on someone else's _Hypervisor_ without having to buy your own physical servers but they ended up costing more anyway...so who was going for this madness?

I wouldn't actually use any cloud technologies for a while, getting my first cloud exposure with _Azure_ a few years down the road where the first thing we went ahead and did was go and build VMs in the cloud...and it was as we know, expensive.

In my experience, many Sysadmins went on to do the same thing, starting out with not only this incorrect belief that public cloud represents little more that running your VMs on "someone elses computer" but also that there are no more features that a public cloud can offer beyond just running VMs. A lot of people never seem to get over the fence in realising this. Far from being hyperbole I've watched this happen first hand and heard these views presented very strongly, indeed it was my opinion for some time.

## Everything As A Service

Public clouds offer us the ability to use different components previously offered as functions on a server as _services_ offered directly out of the cloud, such as Object Storage, Databases or Message Queues to name a few, without the need to build any (or at least very little) surrounding infrastructure.

Personally, I think the **["As a Service"](https://en.wikipedia.org/wiki/As_a_service)** moniker has become farcical now, with snakeoil salesmen seemingly willing to offer anything you can imagine "As a Service", but it does have a real power in **[Platform as a Service](https://en.wikipedia.org/wiki/Platform_as_a_service)**; one of the great features of public clouds that is easy to overlook to begin with. At first glance, the previous point about running your VMs elsewhere could seem to be true, especially if you come from a Sysadmin background and your usual wheelhouse is running everything on VMs.

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-1024x678.jpg" class="scaled-img-75">
  <figcaption>Can I interest you in cabling as a service?</figcaption>
</figure>

For example, once upon a time when we built everything on servers (physical or virtual), if we wanted to build a brand new environment that needed a database component this would mean building a powerful server, installing and configuring a relational database platform (MS SQL, Postgres etc.), tuning it for performance and **then** configuring our actual databases. Building the server and installing your SQL of choice isn't a quick process and can take several hours. Using public cloud and a database _Platform as a Service_ model makes a mockery of this, usually providing the entire RDMS with _High Availability_, _Backups_ and _Encryption_ (as well as many other bells and whistles) out of the box and providing you with the endpoints to plug your application in to. These days if it takes more than a few minutes to create a database instance or even an entire cluster, I find myself tapping my watch.

As the complexity of current generation technology grows, as well as the requirements for automated deployment and testing, the idea of manually building a VM and installing your entire stack manually is not an attractive one, public clouds allow us to remove these painful stages (or at the very least significantly lower their overheads).

## Scalability, Metrics and Economics

Debatably, the biggest efficiency we're offered in public cloud is the ability to carry out [**both vertical and horizontal scaling**](https://www.section.io/blog/scaling-horizontally-vs-vertically/). This becomes far more valuable when make these scaling actions dynamic based on the state of performance metrics; automatic scaling (either vertical or horizontal) is after all not a very attractive proposition if someone needs to sit watching the CPU load 24 hours a day with their finger hovering over the big red **SCALE OUT** button. When we begin to enhance this concept with modern [**Application Performance Management**](https://en.wikipedia.org/wiki/Application_performance_management) platforms (APM) such as Prometheus and it's ilk, we really begin to see the potential; dynamically scaling our services to higher and lower compute depending on real world demand.

This of course, is not a feature exclusive to public cloud. You could do this on the ground using a decent virtualisation platform like **[OpenStack](https://www.openstack.org/)**, **[KVM](https://www.linux-kvm.org/page/Main_Page)** or even **[VMWare](https://www.vmware.com/uk.html)** in your own server room or a reasonably resourced datacentre but you're far more likely to run in to the constraints of finite compute or storage, you can after all only get so many CPUs, RAM and disks in to the space you have. Public clouds are, as we've discussed, running on "someone else's computer" in the sense that they're running over innumerable large datacentres but with the advantage that they can scale your workloads in a way that would require an incredible amount of configuration and maintenance to do yourself.

## High Availability and Disaster Recovery

As we just mentioned, High Availability is present out of the box. I enjoy designing HA and DR systems and it was always fun to do in on-premise deployments, but it's always quite common to run up against the cost of hardware when you're doing this (especially if you're working on a limited budget). If you want to make a system truly highly available it's going to cost you twice as much to have a standby system. Beyond this, you have logistical problems to think about, for real DR we need to think about putting our DR site in a suitable geographic location.

Public clouds usually let us solve this problem straight out of the gate and usually make it very painless, often as simple as a tickbox or setting a boolean to true. Datacentres are constructed with HA and DR in mind with physical location already considered and the innate ability to load balance services over different geographic locations with fast failover. This is a hard proposition to turn down.

## APIs...Everywhere

Public clouds wrap their services in REST APIs to allow for programmatic interaction with their services. At the most basic level this is usually seen as using a Command Line Interface (CLI) bespoke to the cloud provider (**[Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)**, **[AWS CLI](https://aws.amazon.com/cli/)**, **[GCloud CLI](https://cloud.google.com/sdk/gcloud)** etc.) which allows the operator to perform _CRUD_ operations (Create, Read, Update, Delete). This however is only the tip of the iceberg.

The real power of public cloud APIs is realised when we begin to look at _Infrastructure as Code_ and _Configuration Management_ tools, **[Terraform](https://www.terraform.io/)** and **[Ansible](https://www.ansible.com/)** being particular favourites of mine (though many others exist). When we combine these tools with a public cloud providers APIs we can suddenly create entire environments from templated code which "describes" said environment from a single click. Sounds good already but we can also maintain the state of these environments _[**idempotently**](https://en.wikipedia.org/wiki/Idempotence)_ throughout their lifetime, ensuring that our environments and the configurations within them remain _immutable_.

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03.jpg" class="scaled-img-75">

If we go one step further and combine this functionality with an automated CI/CD Pipeline, we can reduce more and more the manual effort needed to build and maintain our infrastructure day to day and focus our efforts on something that's actually productive instead of wasting time putting out fires because someone broke the infrastructure with a rogue configuration.

Again, all of this **could** be done without a public cloud, but I wouldn't like to go through that kind of pain just to build in my own limitations.

## Microservices

Yes, we're talking about _Microservices_ again, everyone's using them now after all, including me, so let's take a look.

First let's be clear that _Containers_ aren't the be all and end all of _Microservices_. _Microservices_ after all are after all an architectural concept that are often conflated with the idea of _Containers_. That said, when we talk about _Microservices_ in this day and age...we're usually talking about _Containers_ (and specifically we're usually talking about _Docker_ and probably _Kubernetes_ as the orchestrator that hosts our _Containers_. We do, however, have other options, such as just running our services on small, ephemeral VMs which can be scaled out or up.

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/04.png" class="scaled-img-75">

Whatever the strategy we want to work with for _Microservices_, really the only game in town to host them is using a public cloud. At the very least we're almost certainly going to need scope for automatic scalability based on metrics, high resiliency and availability, the ability to implement a CI/CD solution, a robust monitoring and logging solution, a reliable strategy for _Persistence_, a method for inter-service communication and HA load balancing. Whilst all of this can be theoretically implemented without a public cloud...I wouldn't like to the be the guy doing it.

When we come to the question of _Kubernetes_, the big players all offer us managed solutions (in the form of **[GKE](https://cloud.google.com/kubernetes-engine)**, **[EKS](https://aws.amazon.com/eks/)** and **[AKS](https://azure.microsoft.com/en-gb/services/kubernetes-service/)**). Trade-offs are made vs managing your own bare metal deployment (which could still be done on VMs in a public cloud) in terms of the features you can use, but for me the gains are a lot greater than the losses and managing a bare metal _Kubernetes_ deployment is a full time job.

## Conclusion

This article is obviously far from exhaustive but hopefully it makes a reasonable argument.

These days I work almost exclusively with cloud systems and creating automation solutions for cloud deployments (though I do still enjoy getting my hands dirty from time to time with an on-premise deployment). The landscape has changed significantly in the last decade or so and it's important to realise that the cloud is far from just "someone else's computer", offering a lot of benefits that we just can't get very easily from traditional infrastructure.

The paradigm shift we face by trying to work **efficiently** in public cloud is a big one and for many people (including myself) that came from a Sysadmin background this means no small amount of work, learning and discarding a lot of the approaches we previously took when designing infrastructure.

I'm not suggesting that public clouds are perfect by any stretch of the imagination, there are issues presented for sure; sometimes we're hindered by seriously steep learning curves, the inability to do what we think should be simple things, waiting for feature releases and I have my own concerns over encryption, but for the most part I think the trade-offs are acceptable for the gains.

...but I think I'll keep the t-shirt, it's still a pretty funny gag.
