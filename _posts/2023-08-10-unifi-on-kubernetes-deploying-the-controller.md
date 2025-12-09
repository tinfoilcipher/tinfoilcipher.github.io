---
title: "UniFi on Kubernetes - Deploying the Controller"
layout: post
date: 2023-08-10
categories: 
  - "containers"
  - "devops"
  - "linux"
tags: 
  - "devops"
  - "helm"
  - "kubernetes"
  - "networking"
  - "projects"
  - "unifi"
---

Recently I've been having some fun moving my lab and home infrastructure to Kubernetes. I had a feeling that deploying the UniFi Controller was going to be a bit of a painful process but it's not so bad.


<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01.png" class="scaled-img-50">

## Has This Already Been Done?

Well, allegedly.

The UniFi Controller has long been a Linux application so theoretically there are no real issues in the way. My initial searching led me to the now long deprecated official [**Helm git repo**](https://github.com/helm/charts/blob/master/stable/unifi/README.md). This has clearly already been done before so that's a good sign!

I found a few guides going through this process, a good start, but they all stopped dead at the start of the setup wizard. Suspicious. Similarly problematic is that the official UniFi chart has been abandoned for some time in the old official Helm repo and doesn't seem to have ever been migrated.

I gave the old, orphaned chart a try to see if it was actually functional but ran in to some issues out of the gate:

- All of the services are ClusterIPs, so there is no way for _Access_ _Points_ to actually communicate with the _Controller_

- The first time setup wizard was impossible to complete and could not progress past the final screen (the **Complete** box was un-clickable

So I guess we know why all these guides don't go any further than the login screen now!

I found a few other charts kicking around online but they all had problems that stopped them being installed, so eventually I decided just to take the abandoned official chart and patch it up.

## Installing

Installation is a pretty painless process (now that a suitable chart exists). First we need to add the Helm repo:

```bash
helm repo add tinfoilcharts https://tinfoilcipher.github.io/tinfoilcharts
```

Then we'll need to define some values for configuration, importantly I've made sure that the relevant services are being exposed correctly on the _Controller_.

We'll also need to set a Timezone (after much debugging, this appears to be at the root of the issues with the setup wizard that I encountered). In the values below, pay particular attention to the two IP addresses for TCP and UDP services. **These can technically be the same address**, but if you have enough to spare you may as well split them up.

```yaml
#--values.yaml

environment:
  timezone: "Europe/London" #--Setting this stops the setup wizard breaking
  stdout: true

service:
  ports:
    webapi: 8443
  tcpLoadBalancerIP: 192.168.1.220 #--Set as relevant. This is the IP your Access Points will communicate over regularly
  udpLoadBalancerIP: 192.168.1.221 #--Set as relevant. This is the IP your Access Points will discover the Controller on

ingress:
    enabled: 'true'
    className: nginx #--If you're using nginx, I am
    annotations:
      cert-manager.io/cluster-issuer: ca-issuer #--Or whatever your issuer is called
      nginx.ingress.kubernetes.io/rewrite-target: /             #--This bunch of annotations are needed if you're
      nginx.ingress.kubernetes.io/proxy-body-size: "0"          #--using an nginx ingress controller, which I keep
      nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"     #--mentioning...I'm assuming you are
    hosts: 
      - host: unifi.tinfoilcipher.co.uk #--YOUR HOSTNAME HERE
        paths:
          - path: /
            pathType: ImplementationSpecific
    tls:
      - secretName: unifi-certificate
        hosts:
          - unifi.tinfoilcipher.co.uk #--YOUR HOSTNAME HERE

```

Add any other values as you see fit, then we can create a namespace and install:

```bash
kubectl create ns unifi
helm install unifi -n unifi tinfoilcharts/unifi-controller -f values.yaml

```

## Verifying

This should kick off the installation process, it's worth pointing out that the _Controller_ is at it's heart quite a large clunky Java application and it takes quite some time to start up, so don't be too concerned if it takes a while to come up. Monitor the process with the below commands:

```bash
#--Watch for the Pod becoming ready
kubectl get po -n unifi -w
# NAME                                READY   STATUS    RESTARTS  AGE
# unifi-controller-58d74d8b5f-dx4p9   0/1     Running   0         1m

#--Tail the logs to watch the boot process:
kubectl logs -n unifi unifi-controller-58d74d8b5f-dx4p9 -f

```

Once the _Controller_ has booted, we can access the UI via the hostname we provided in the **values.yaml** (assuming there are no issues with the setup wizard) and we will be asked to provide credentials on the first setup.

For the sake of completeness and to confirm everything works:

![Don't judge me for running such an old controller. I have a very old access point](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/04-1024x332.png)

Fairly short and straight forward post. In the next one we'll get more complex and look at how we can leverage _cert-manager_'s PKI and combine it with _freeradius_ in order to deliver an enterprise grade WPA-TLS wireless network from inside Kubernetes.
