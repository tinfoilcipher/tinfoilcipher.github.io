---
title: "Building a Bare Metal Kubernetes Lab Part 2"
layout: post
date: 2023-03-17
categories: 
  - "automation"
  - "containers"
  - "devops"
  - "integrations"
  - "linux"
  - "projects"
tags: 
  - "automation"
  - "certificates"
  - "devops"
  - "helm"
  - "kubernetes"
  - "linux"
  - "metallb"
  - "networking"
  - "nginx"
  - "pki"
  - "projects"
  - "sysadmin"
---

In [Part 1]({% post_url 2023-01-20-building-a-bare-metal-kubernetes-lab-part-1 %}) of this project we covered building the infrastructure that underpins Kubernetes; the _Virtual Machines_ that make up it's _Control_ and _Data Planes_, implementing high availability, bootstrapping the core Kubernetes components and considerations for the various networking elements.

All of this is great, but after all of that all our cluster doesn't actually **do** very much yet. It's still in a pretty raw state and not ready to serve out applications in anything but a very basic manner. In this post we'll look at building out some internal components within a Kubernetes cluster that will be needed for most scenarios.

The full code for this post can be found in GitHub [here](https://github.com/tinfoilcipher/kubernetes-baremetal-lab).

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-1.png)

## What Do We Want To Cover?

We're going to deploy some of the most commonly used Kubernetes systems which can be implemented to serve a large amount of applications and provide a framework for serving web applications to the host network. Specifically we'll be covering:

- How to deploy applications and configure storage

- Exposing running applications to the host network

- Building a PKI and managing certificates

- Deploying an example application which uses these systems

- How this process can be automated

If you didn't already, you will need to clone the git repo for this project first:

```bash
git clone https://github.com/tinfoilcipher/kubernetes-baremetal-lab.git
cd kubernetes-baremetal-lab
```

If you’re following along, I’m going to assume you have both [**kubectl**](https://kubernetes.io/docs/tasks/tools/#kubectl), [**helm**](https://helm.sh/docs/intro/install/) and [**helmfile**](https://helmfile.readthedocs.io/en/latest/#installation) installed. If not, click the links for installation guides for all three.

## Choices For Deploying - Manifests Vs Helm

Kubernetes internal componentry is built up of independent _[**objects**](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)_, each of which represents an individual component in a wider system. These objects are "described" in a YAML specification and applied to a running cluster via the Kubernetes API, the cluster then does it's best to get itself in to that state.

Broadly speaking, the contents of your YAML is used to define what applications are and aren't running on your cluster and how they are configured.  
  
Whilst this is a good system for small deployments, it quickly starts to show it's limitations at scale. Complex deployments become unwieldy to manage when there is that much YAML flying around. [**Helm**](https://helm.sh/) presents us with a solution to this. Acting as a de-facto package manager for Kubernetes, it uses a powerful [**templating language**](https://helm.sh/docs/chart_template_guide/) to template YAML manifests from a set of pre-defined input values.

Public _Helm_ repositories are pretty common these days and packages _(Charts)_ are available for most of the biggest open source packages. In order to make our deployment as painless as possible, we'll be using _Helm_ as much as possible, but we'll go a step further and use [**Helmfi_le**](https://github.com/roboll/helmfile).

_Helmfile_ provides a templating system for deploying multiple charts at the same time, running pre and post installation operations, maintaining multiple environments and a bunch of other cool features.

I'm only going to point out the relevant commands for _Helmfile_ for this post, take a look at the [**readme**](https://github.com/tinfoilcipher/kubernetes-baremetal-lab) for more context.

## Creating Namespaces

**[Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)** are used by Kubernetes to isolate resources in to logical groups. As we're going to deploy a few resources here we'll get some _Namespaces_ set up in advance. The YAML spec below shows an example namespace we'll be using:

```yaml
---
kind: Namespace
apiVersion: v1
metadata:
  name: ingress-nginx
  labels:
    name: ingress-nginx
```

We'll be creating the following namespaces:

- nfs-provisioner

- metallb-system

- ingress-nginx

- cert-manager

- metrics-server

- kubernetes-dashboard

We can create them all with:

```bash
kubectl apply -f kubernetes/namespaces.yaml
# namespace/cert-manager created
# namespace/ingress-nginx created
# namespace/metallb-system created
# namespace/netbox created
# namespace/nfs-provisioner created

```

_Helm Charts_ often include the option to create namespaces but this isn't guaranteed, I'd always recommend creating and managing your own.

## Persistent Storage Using NFS

Kubernetes is a distributed system and by design it was meant to play host to stateless apps. One of the issues we face here is managing how we're going to work with storage, specifically how common storage can be accessed no matter what node your application is running on.

When deploying an application; Kubernetes mounts storage based on _[**StorageClass**](https://kubernetes.io/docs/concepts/storage/storage-classes/)_, this defines the type of storage you're going to use. In cloud environments we can use a native integration with a PaaS Storage _StorageClass_ such as S3, but running on bare metal we'll need to do something a little more generic. We're going to be using [**NFS**](https://en.wikipedia.org/wiki/Network_File_System) (I'll be connecting to an existing server and share). Just having an NFS share being offered still doesn't get us all the way there however.

Kubernetes applications access storage using _[**PersistantVolumes**](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)_ (_PVs_), which in the case of NFS will be subdirectories in our root NFS share (we can't have every application dumping it's files in one directory after all). A second mechanism called _[**PersistantVolumeClaims**](http://PersistantVolumes)_ (_PVCs_) represents a request from an application to access a _PV_.

The arrangement of trying to manually manage all of this will quickly become a nightmare, we really need the provisioning mechanism to be dynamic. To achieve this, we'll be using **[nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)**. This will create our _StorageClass_ and manage the provisioning of _PVs_ based on _PVCs_.

The below diagram breaks down the functionality at a high level:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/nfs.png)

These values shown above and below are defined in **[here](https://github.com/tinfoilcipher/kubernetes-baremetal-lab/blob/main/helmfile/values.yaml#L9)** in our **helmfile/values.yaml**. You will need to update these for your network before running:

```yaml
# helmfile/values.yaml

...
nfsSubdirProvisioner:
  imageTags:
    provisioner: v4.0.2
  host:
    server: "192.168.1.2" #--Host of your NFS Server
    path: "/your_vol" #--Path of your NFS Mount
  reclaimPolicy: Retain
  isDefault: true
  replicaCount: 2
...
```

With this updated, we can deploy the NFS Subdir Provisioner:

```yaml
helmfile -f helmfile/helmfile.yaml --environment default --selector app=nfs-provisioner sync
# ...
# ...
# UPDATED RELEASES:
# NAME      CHART                                                               VERSION
# metallb   nfs-subdir-external-provisioner/nfs-subdir-external-provisioner     4.0.9

kubectl get deployment -n nfs-provisioner
# NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
# nfs-subdir-external-provisioner   2/2     2            2           2m

```

With that in place, let's have a look at some of the networking challenges that we still have to tackle.

## Exposing Applications - What Are Services Anyway?

Before we get any more muddled with terminology, let's take a brief aside to discuss what _Services_ are in a Kubernetes context. We're not going to do an incredibly deep dive because this article is going to be long enough, but let's get some clarity.

In [**Part 1**]({% post_url 2023-01-20-building-a-bare-metal-kubernetes-lab-part-1 %}), we allowed kubeadm to set up a large subnet (10.96.0.0/12) to provide _Services_. _Services_ are Kubernetes _Objects_ which, broadly speaking, are used to provide _Service Dis_covery for _Pods_ and forward traffic in one of two ways:

1. Exposing pods outside of the cluster, making them accessible to the "outside world"

2. Exposing running pods to eachother and making them resolvable by internal DNS

In order to provide this in different ways; services come in 3 flavours:

1. **ClusterIP**: This is the default _Service_ type and is used to expose **[Pods](https://kubernetes.io/docs/concepts/workloads/pods/)** to other _Pods._ Despite the name, all lookups are done by internal DNS using the **$service.namespace.svc.cluster.local** syntax. Using a static IP address for anything in Kubernetes is generally a bad idea and IPs should be considered ephemeral.

2. **NodePort**: This type sets up a 1:1 NAT with the _Pod_'s TCP/UDP port and the underlying _Node_ using an _[**Unprivileged Port**](https://www.w3.org/Daemon/User/Installation/PrivilegedPorts.html)_ in the _30000-32767_ range. This is used to expose the application outside of the cluster but does not provide any load balancing between nodes. Once traffic reaches a node it is forwarded to the relevant _ClusterIP_ and then the relevant _Pod_.

3. **LoadBalancer**: A more common means of exposing an application outside of the cluster. This type sets up a loadbalancer on the cloud providers infrastructure and configures it to forward traffic to the relevant _Po_ds. The functionality of the cloud providers loadbalancer is defined in YAML and configured using _[**Annotations**](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/)_. Once traffic passes the _LoadBalancer_ it is sent to the underlying _NodePort_, then _ClusterIP_ and eventually to the relevant _Pod_.

For a more comprehensive breakdown, take a look at the Kubernetes docs [**here**](https://kubernetes.io/docs/concepts/services-networking/service/).

## MetalLB - LoadBalancer Services Without Public Clouds

As our quick outline of _Services_ just showed, we will need to use the _LoadBalancer_ _Service_ to reliably expose applications running in our cluster. This is another one of those things that the cloud vendors do a lot of the heavy lifting for us on and Kubernetes does not implement a bare metal loadbalancer out of the box. We'll have to come up with another solution.

We could implement our own solution by forwarding an external load balancer to _NodePorts_, but this sounds like a pain to manage and the _LoadBalancer_ _Service_ type is sitting right there. To solve this problem, we'll be using the **[MetalLB](https://metallb.universe.tf/)** project. Which provides the _LoadBalancer_ functionality offered on public clouds on bare metal.

The below diagram shows a high level outline of how this will work:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/metallb.png)

Something to make note of here is that we will be using _MetalLB_ in Layer 2 mode. **[As the documentation implies](https://metallb.universe.tf/concepts/layer2/)** this is really only useful for failover between loads and not load distribution, for out purposes it should serve us fine though.

Once we deploy MetalLB, _helmfile_ will automatically define a set of subnets which can be used to serve _LoadBalancers_ by applying a couple of YAML manifests. In [Part 1]({% post_url 2023-01-20-building-a-bare-metal-kubernetes-lab-part-1 %}) we reserved this range as **192.168.1.225 - 234** but Kubernetes needs to be informed of the range we intend to use. If you want to use a different range for your network be sure to edit the manifest at **helmfile/apps/metallb/manifests/loadbalancer\_bootstrap.yaml**:

```yaml
#--helmfile/apps/metallb/manifests/loadbalancer_bootstrap.yaml

apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: application-services
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.225-192.168.1.234 #--MAKE YOUR CHANGE HERE
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: application-services
  namespace: metallb-system
spec:
  ipAddressPools:
  - application-services

```

These are applied as an _IPAddressPool_ (the pool from which _LoadBalancers_ can take addresses) and a _L2Advertisment_ (which, predictably) is a Layer 2 advertisement for available addresses when a _LoadBalancer_ is requested.

With this understood, let's deploy MetalLB:

```bash
helmfile -f helmfile/helmfile.yaml --environment default --selector app=metallb sync
# ...
# ...
# hook[postsync] logs | ipaddresspool.metallb.io/application-services created
# hook[postsync] logs | l2advertisement.metallb.io/application-services created
# hook[postsync] logs | 
#
# UPDATED RELEASES:
# NAME      CHART               VERSION
# metallb   metalb/metalb       4.1.8

kubectl get deployment -n metallb-system
# NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
# metallb-controller   1/1     1            1           5m

kubectl get daemonset -n metallb-system
# NAME              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
# metallb-speaker   3         3         3       3            3           kubernetes.io/os=linux   5m

```

These are not the only available configurations for MetalLB and the [**documentation**](https://metallb.universe.tf/configuration/) is fairly comprehensive. This will do fine for our needs though.

With our _LoadBalancers_ understood, let's move on to thinking about HTTP/S traffic specifically.

## Ingress - Manging HTTP/S with ingress-nginx-controller

_[**Ingress**](https://kubernetes.io/docs/concepts/services-networking/ingress/)_ is the Kubernetes _object_ used to manage HTTP/S routes in to an application. In order to implement this we will also need a _controller_ to manage our ingresses _ingresses_, in our case we'll be using the **[ingress-nginx-controller](https://kubernetes.github.io/ingress-nginx)**.

Don't let these concepts confuse you if you've never encountered them, you can roughly think of the _controller_ as any other reverse proxy deployment you might have seen (like NGINX or Apache) and the _ingresses_ as HTTP/S routes in _sites-available_.

The _controller_ must still be attached to a _LoadBalancer_ _Service_ in order to expose it outside the cluster, the benefit of this arrangement is that multiple sites can be exposed on the same _LoadBalancer_ and routed based on their domain names. The below diagram shows an example of how this looks and will work in our infrastructure:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/ingress.png)

With that understood, let's deploy:

```bash
helmfile -f helmfile/helmfile.yaml --environment default --selector app=ingress-nginx sync
# ...
# ...
# UPDATED RELEASES:
# NAME              CHART                               VERSION
# ingress-nginx     ingress-nginx/ingress-nginx         4.3.0

kubectl get deployment -n ingress-nginx
# NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
# ingress-nginx-controller   1/1     1            1           3m
```

So at this point, we have the networking under control and we can serve out web applications over HTTP, but that's not great for the 21st century. As if things weren't complicated enough we'll need to bring certificates in to the mix to work with HTTPS.

## Implementing PKI with Cert-Manager

Unlike the other 99% of people, I'm a big fan of working with certificates. I'll be the first to admit that it's usually a pretty painful process though. Jetstack's **[cert-manager](https://cert-manager.io/)** offers a very pain free solution to PKI for Kubernetes and makes what used to be a pretty agonising and often very manual process very efficient and highly automated.

Just because it's automated though doesn't mean the process is any less important to understand. There's a lot of moving parts in the way we're going to deploy which is going to mean bootstrapping a _Certificate Authority_ that our applications can make use of.

This deployment roughly this has 4 steps:

1. _cert-manager_ is deployed using the official _helm chart_ using the values defined [**here**](https://github.com/tinfoilcipher/kubernetes-baremetal-lab/blob/main/helmfile/apps/cert-manager/values_cert_manager.yaml.gotmpl)

3. An **[ClusterIssuer](https://cert-manager.io/docs/concepts/issuer/)** is created (the _cert-manager object_ responsible for issuing cluster-wide certificates

5. A self-signed certificate is requested from the _ClusterIssuer_ which will act as our Root CA cert

7. A new _ClusterIssuer_ is created, using the newly created certificate

At a high level this process looks like:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/cert-manager-2.png)

With this understood, we can deploy the PKI with:

```bash
helmfile -f helmfile/helmfile.yaml --environment default --selector app=cert-manager sync
# ...
# ...
# UPDATED RELEASES:
# NAME              CHART                           VERSION
# ingress-nginx     jetstack/cert-manager           4.3.0

kubectl get deployment -n cert-manager
# NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
# cert-manager              1/1     1            1           21m
# cert-manager-cainjector   1/1     1            1           21m
# cert-manager-webhook      1/1     1            1           22m
```

Future applications will be able to use HTTPs or leverage any other services which need TLS from this point.

## Example Application - Netbox with PostgresDB

I've talked about Netbox a lot in the past and I'm using it as an example here as it will let us demonstrate a few things:

1. **MetalLB** and **ingress-nginx** will be working in tandem to expose the application over the _Ingress Controller Loadbalancer_

3. **cert-manager** will be issuing a valid certificate to our _Ingress_

5. **nfs-subdir-external-provisioner** will configure underlying storage _PersistentVolumes_ and _PersistentVolumeClaims_ for the applications Postgres database and static assets

With the relevant infrastructure in place, our deployment will do something along these lines:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/netbox-1024x495.png)

Deployment should now make use of these components if we run:

```bash
export NETBOX_DB_ADMIN_PASSWORD=StrongPassword123!
export NETBOX_DB_ROOT_PASSWORD=VeryStrongPassword123!
export NETBOX_ADMIN_PASSWORD=JustAsStrongPassword123!

helmfile -f helmfile/helmfile.yaml --environment default --selector app=netbox sync
```

## Deploying Everything - Further Automation with Helmfile

Throughout we've deployed by running _helmfile_ and selecting specific applications with the **\--selector** argument to choose specific _labels_. If we remove this then all sub-helmfiles listed in **helmfile/helmfile.yaml** will be processed in the order they are provided:

```bash
#--Set these to something actually secure
export NETBOX_DB_ADMIN_PASSWORD=StrongPassword123!
export NETBOX_DB_ROOT_PASSWORD=VeryStrongPassword123!
export NETBOX_ADMIN_PASSWORD=JustAsStrongPassword123!

helmfile -f helmfile/helmfile.yaml --environment default sync

```

That will let out a wall of text and build all of the infrastructure we've discussed in the order we've discussed. This could have been done straight out of the gate but that misses the point a bit of trying to understand how a system works.

## Conclusion

Well! That was a lot, but hopefully it all made sense.

For anyone that made it this far, now we have a framework to which any future apps can theoretically be deployed. I've been running all of my home systems on a cluster like this for several months now and haven't had any real issues. In some future posts we'll take a look at leveraging the PKI further, and some unique application deployments.
