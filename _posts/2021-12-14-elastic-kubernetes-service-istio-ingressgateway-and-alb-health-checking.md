---
title: "Elastic Kubernetes Service, Istio IngressGateway and ALB - Health Checking"
layout: post
date: 2021-12-14
categories: 
  - "aws"
  - "containers"
  - "devops"
  - "integrations"
tags: 
  - "aws"
  - "cloud"
  - "devops"
  - "eks"
  - "integration"
  - "istio"
  - "kubernetes"
  - "microservices"
---

In a previous article we took a look at [**the very unwieldy integration of the Istio IngressGateway with an AWS Application Load Balancer**]({% post_url 2021-06-14-elastic-kubernetes-service-securely-integrating-aws-alb-with-istio-ingressgateway %}), however we didn't look at any _Health Check_ options to monitor the the _ALB_ via it's _Target Group_. A dig around the usual forums suggests that this is confusing a lot of people and it threw me the first time I looked. In post we'll have a quick look at how to get a _Target Group Health Check_ properly configured on our ALB and why it doesn't work by default in the first place.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01.png)

## Unhealthy By Default?

As a quick reminder, our _ALB_s work in this deployment by forwarding requests to a _Target Group_ (which contains all the _EC2 Instances_ making up our _EKS Data Plane)_. Using a combination of the **[aws-load-balancer-controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/)** and normal Kubernetes Ingresses this can all be created and managed dynamically as we covered in the **[previous article]({% post_url 2021-06-14-elastic-kubernetes-service-securely-integrating-aws-alb-with-istio-ingressgateway %})**.

For a recap, our ALB is deployed using the below _Ingress_ configuration:

```yaml
#--ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: istio-ingress
  namespace: istio-system
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internal
    alb.ingress.kubernetes.io/subnets: subnet-12345678, subnet-09876543
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS": 443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:eu-west-1:123456789012:certificate/b1f931cb-de10-43af-bfa6-a9a12e12b4c7 #--ARN of ACM Certificate
    alb.ingress.kubernetes.io/backend-protocol: HTTPS
spec:
  tls:
  - hosts:
       - "cluster.tinfoil.private"
       - "*.cluster.tinfoil.private"
    secretName: aws-load-balancer-tls
  rules:
  - host: "cluster.tinfoil.private"
    http:
      paths:
      - path: /*
        backend:
          serviceName: istio-ingressgateway
          servicePort: 443
  - host: "*.cluster.tinfoil.private"
    http:
      paths:
      - path: /*
        backend:
          serviceName: istio-ingressgateway
          servicePort: 443
```

With this default configuration, our _Target Group_'s _Health Check_ will be misconfigured, with all _Instances_ reporting as _Unhealthy_:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/05-1-1024x587.png)

We know they really are available so, what's the problem? Let's take a look at how the _Health Check_ is configured out of the gate:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/06-1-1024x241.png)

What's going on, exactly?

## Understanding the Architecture

It's important to realise when working with **[NodePorts](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport)**,_ as we are in this scenario, that the TCP ports are being exposed via each of our _Data Plane_ (or _Worker_) nodes themselves and then routing traffic to the the relevant _Service_ (in our case the _Istio IngressGateway_). These _NodePorts_ are the very ports that the inside of our _ALB_ needs to connect to.

As our _Nodes_ are EC2 instances, our architecture looks something like this:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/05.png)

In the above example _NodePort_ represents the specific TCP port number of the _Istio IngressGateway_'s _traffic-port_, which will forward traffic to TCP port **443** on the _IngressGateway_ _Service_.

But all this is a little theoretical, how to do we actually get the _Health Check_ to work?

## Gathering the Health Check Details

The _Istio IngressGateway_ is serving several ports. The **traffic-port** that we mentioned earlier is currently being used for _Health Checks_ by default, but this port is being using to serve content only and the _Health Check_ endpoint is not available on this port . The _IngressGateway_ has a separate **status-port** for this very purpose.

So let's gather the correct _NodePort_ to run a _Health Check_ against:

```bash
kubectl get svc istio-ingressgateway -n istio-system -o json | jq .spec.ports[]

{
  "name": "status-port",
  "nodePort": 31127,
  "port": 15021,
  "protocol": "TCP",
  "targetPort": 15021
}

#--http, https and tls ports omitted for brevity
```

...and the _endpoint_ that Kubernetes actually offers us as a **[Readiness Probe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-readiness-probes)** that we can run a _Health Check_ against:

```bash
kubectl describe deployment istio-ingressgateway -n istio-system

Name:                   istio-ingressgateway
Namespace:              istio-system
Containers:
  ...
  Readiness:  http-get http://:15021/healthz/ready delay=1s timeout=1s period=2s #success=1 #failure=30
  ...
...

#--Most output omitted for brevity
```

From this we can see that the proper endpoint we need to send out _Health Check_ to is **/healthz/ready** using TCP port **31127** over **HTTP**. This will map to **TCP/15021** within the **istio-ingressgateway** _Pod_ and tell us if the _IngressGateway_ is still available.

Be aware that this _NodePort_ value is dynamic, be sure to look up your own and don't just copy and paste from here! This value will also **change on re-deployment** and this should be taken in to consideration when automating deployments.

## Missing Annotations

As we briefly mentioned in the previously article, the list of annotations for _ALB Ingress_ is not widely documented. But a comprehensive list is available **[here](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/guide/ingress/annotations/)**. A couple more of these need to be explicitly set in order to make the _Health Check_ work. We can complete these with the information we've already looked up:

```yaml
#--ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: istio-ingress
  namespace: istio-system
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internal #--internet-facing for public
    alb.ingress.kubernetes.io/subnets: subnet-12345678, subnet-09876543
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS": 443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:eu-west-1:123456789012:certificate/b1f931cb-de10-43af-bfa6-a9a12e12b4c7 #--ARN of ACM Certificate
    alb.ingress.kubernetes.io/backend-protocol: HTTPS
    #--ADDITIONAL ANNOTATIONS BELOW
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTP #--HTTPS by default
    alb.ingress.kubernetes.io/healthcheck-port: "31127" #--traffic-port by default
    alb.ingress.kubernetes.io/healthcheck-path: "/healthz/ready" #--/ by default
spec:
    ...
```

## Does It Work?

```
kubectl apply -f ingress.yaml
```

We can see now that these settings have applied to the _ALB_:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/07-1024x272.png)

...and that the _Health Check_ is now showing a happy load balancer:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/08-1024x387.png)

Simple!
