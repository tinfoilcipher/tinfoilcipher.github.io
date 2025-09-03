---
title: "Dynamically Manage Route 53 Records for Istio Gateways using Terraform"
layout: post
date: 2021-02-23
categories: 
  - "automation"
  - "aws"
  - "containers"
  - "devops"
tags: 
  - "cloud"
  - "containers"
  - "devops"
  - "dns"
  - "eks"
  - "integration"
  - "istio"
  - "kubernetes"
  - "netops"
  - "networking"
  - "route53"
---

In the days of cloud we're often called on to integrate a lot of technologies together (as the somewhat messy title of this post suggests). One of the more recent systems I've encountered is Istio, popular Kubernetes [**Service Mesh**](https://istio.io/latest/docs/concepts/what-is-istio/), which in _EKS_ tends to rely on an **[Elastic Load Balancer](https://aws.amazon.com/elasticloadbalancing/)** of one flavour or another as the point of access to it's **[Gateway](https://istio.io/latest/docs/concepts/what-is-istio/)**.

In this post we'll look at how to dynamically provision DNS records in to **[AWS Route 53](https://aws.amazon.com/route53/)** for these Load Balancers in a couple of common configurations using Terraform. I won't be diving in to the deep functions of Istio, if that's what you're looking for you start with the [**Istio High Level Architecture**](https://istio.io/latest/docs/).

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-1.png" class="scaled-img-75">

## What's The Problem?

When we deploy our Load Balancer it's going to be provisioned with a rather unpleasant DNS name that we can't do much about. In and of itself this isn't a huge problem since we can just front this with a DNS CNAME and we'll be using Route 53 to provide our DNS services.

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-16.png">
  <figcaption>Rolls right off the tongue</figcaption>
</figure>

A quick glance at the [**Istio Gateway overview**](https://istio.io/latest/docs/concepts/traffic-management/#gateway-example) shows us that we need to explicitly define the hostname(s) that we want to allow ingress for, this will need to be reflected in our DNS CNAME, if we don't have appropriate CNAME entries pointing to the Load Balancer matching the hostname(s) defined in our Istio Gateway, then nothing is going to work.

This of course gets more complicated when we consider that we could possibly see re-provisioning of these services when a configuration is changed at a later date and we want to be able to simply reconfigure from a single command or pipeline. The last thing we want to be doing is defining a static value anywhere in our Terraform configurations, we need to be able to look up the **current** value of the Load Balancer CNAME at any time and then pass it to Route 53.

## Configuring with Terraform - ELB Classic

In my experience to date ELB Classic is the far more pervasive LB solution and leverages the standard **[LoadBalancer Service](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer)**.

Terraform can leverage the [**Kubernetes _provider**](/creating-authenticating-and-configuring-elastic-kubernetes-service-using-terraform/) to look up the current value assigned to an ELB Classic Istio LoadBalancer. The below example performs this lookup and writes the value to a Route 53 CNAME named **myapp.tinfoilcipher.co.uk**:

```terraform
#--Lookup ELB DNS Name
data "kubernetes_service" "istio_ingress_gateway_elb" {
    metadata {
        name        = "istio-ingressgateway"
        namespace   = "istio-system"
    }
}

#--Write new CNAME to Route 53
resource "aws_route53_record" "tinfoil_elb_cname" {
    zone_id = aws_route53_zone.tinfoil.zone_id
    name    = "myapp"
    type    = "CNAME"
    ttl     = "300"
    records = [data.kubernetes_service.istio_ingress_gateway_elb.load_balancer_ingress.0.hostname]
}
```

## Configuring with Terraform - Application Load Balancer

The configuration of an ALB is much more involved and appears to be much less common. The most popular implementation replaces the **[LoadBalancer Service](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer)** with an **[Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)** and creation of the Load Balancer is handled by a separate _Controller_ named **[aws-load-balancer-controller](https://github.com/kubernetes-sigs/aws-load-balancer-controller)**. That's not really a topic for this post though it's far too complicated to get in to here, one for a later post.

Here we can get the same results from Terraform if we change the Kubernetes _data source_ to lookup the _Ingress_ rather than the _Service_:

```terraform
#--Lookup ALB DNS Name
data "kubernetes_ingress" "istio_ingress_alb" {
    metadata {
        name        = "istio-ingress"
        namespace   = "istio-system"
    }
}

#--Write new CNAME to Route 53
resource "aws_route53_record" "tinfoil_alb_cname" {
    zone_id = aws_route53_zone.tinfoil.zone_id
    name    = "myapp"
    type    = "CNAME"
    ttl     = "300"
    records = [data.kubernetes_ingress.tinfoil_alb_cname.load_balancer_ingress.0.hostname]
}
```

As we can see, use those couple of very quick _resources_ can dynamically reconfigure our DNS and very easily save you a lot of tedious manual reconfiguration.
