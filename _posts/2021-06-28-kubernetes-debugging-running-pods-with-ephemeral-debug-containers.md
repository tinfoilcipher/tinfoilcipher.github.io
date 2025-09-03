---
title: "Kubernetes - Debugging Running Pods with Ephemeral Debug Containers"
layout: post
date: 2021-06-28
categories: 
  - "aws"
  - "containers"
  - "devops"
tags: 
  - "cloud"
  - "containers"
  - "devops"
  - "docker"
  - "kubernetes"
  - "microservices"
  - "networking"
---

At the end of last year I wrote about some basic methods for [debugging networking issues inside a Kubernetes Cluster](/kubernetes-tips-basic-network-debugging/). In that article we very briefly mentioned a then-_alpha_ feature (with a complicated sounding name) called **[Ephemeral Debug Containers](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-running-pod/#ephemeral-container)** first introduced back in Kubernetes v1.16. 
  
This looks to be the real future of debugging in Kubernetes and as of v1.20 it's finally in beta. This great feature really strengthens a lot of the issues we have with out current toolset for debugging, but on managed platforms which are really my wheelhouse it's still a mysteriously lacking feature. None the less, let's look at how we can use it in development environments.

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-5.png" class="scaled-img-50">

## Use Cases and The Old Fashioned Way

So we're going to work with a simple single node microk8s development environment, deployed to the cluster is a simple deployment of _Prometheus_ (deployed using _Prometheus-Operator)_ in the _Default Namespace_. The application isn't so important we're just going to use it as an example.

Let's have a quick look at the _Pods_ and grab the name of a running _Container_ inside one of them:

```bash
microk8s kubectl get pods
# NAME                                                     READY   STATUS    RESTARTS
# prometheus-node-exporter-z8fpq                           1/1     Running   0
# prometheus-kube-state-metrics-bbd98b855-tm8bm            1/1     Running   0
# prometheus-prometheus-oper-operator-7c98ddcd74-c2mrb     1/1     Running   0
# alertmanager-prometheus-prometheus-oper-alertmanager-0   2/2     Running   0
# prometheus-prometheus-prometheus-oper-prometheus-0       3/3     Running   0

microk8s kubectl get pods prometheus-kube-state-metrics-bbd98b855-tm8bm -o jsonpath='{.spec.containers[*].name}'
# kube-state-metrics

```

Now how are we going to get inside one of these to do some debugging? Traditionally we'd use some form of _kubectl exec_ to send arbitrary commands in to a container which is running a shell (which can get pretty sketchy, and it's not really viable on a production system), or we end up trying to work inside an adjacent pod which isn't ideal, so let's try out a better option.

## Debug Containers - First We Need To Enable

Even in _microk8s_ we will need to enable the **[Kubernetes Feature Gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)** in the _kube api-server_ before we can use it, this is the case for all _Experimental_ features until they become stable. We'll need to stop the service, edit the config file and then start up the service again:

```bash
#--Stop microk8s
microk8s stop
# Run service command "stop" for services ["daemon-apiserver" "daemon-apiserver-kicker" "daemon-cluster-agent" "daemon-containerd" "daemon-control-plane-kicker" "daemon-controller-manager" "daem…
# Stopped.

#--Edit the Kube api-server config file
sudo nano /var/snap/microk8s/current/args/kube-apiserver

```

To this file, we need to edit the **\--FeatureGates** argument and add the **EphemeralContainers** Key=Value pair, set to **true**:

```bash
...
--feature-gates=RemoveSelfLink=false,EphemeralContainers=true
...
```

Save the file with **CTRL + O** and start _microk8s_ again:

```bash
#--Start microk8s
microk8s start
# Run service command "start" for services ["daemon-apiserver" "daemon-apiserver-kicker" "daemon-cluster-agent" "daemon-containerd" "daemon-control-plane-kicker" "daemon-controller-manager" "daem…
# Started.
```

## Debug Containers To The Rescue

A _Debug Container_ is created as a **[Sidecar](https://kubernetes.io/docs/concepts/workloads/pods/#how-pods-manage-multiple-containers)** within an existing _Pod_. For our purposes we're going to work with **busybox,** always a favourite toolkit troubleshooting.

We get a few great advantages using this method:

1. The network topology inside a _Pod_ is completely flat, meaning that once we're running inside we can capture whatever traffic we like if we need to do network troubleshooting, This is especially helpful if we're using a _Service Mesh_ such as [Istio](https://istio.io/) and we need to capture traffic between an application _Container_ and it's _Sidecar Proxy_.
2. Since we're going to be inside a standalone _Container_ and know that it's going to be destroyed at the end of it's use, we can use it with a lot less worry than our previous shaky choices for debugging.
3. Most importantly, we can perform advanced troubleshooting against _distroless_ images (which are getting more and more popular), our previous options with _exec_ do not offer us this ability and need the presence of a shell.

To deploy a _Debug Container_ we simply need to:

```bash
#--Inject a busybox container inside the prometheus-kube-state-metrics-bbd98b855-tm8bm Pod targeted at the kube-state-metrics Container
microk8s kubectl debug -it prometheus-kube-state-metrics-bbd98b855-tm8bm --image=busybox --target=kube-state-metrics
# Defaulting debug container name to debugger-kwjf4
# If you don't see a command prompt, try pressing enter.
root:/#
```

This drops us in to an interactive shell where we can perform all of our usual tasks with _busybox._

## Cloning A Running Pod and Add a Debug Container

In the **[last post on debugging Kubernetes](/kubernetes-tips-basic-network-debugging/)** we looked at a very sketchy method of debugging by using _exec_ to enter a running container and then installing packages in to it on the fly. This is a very worrying proposition and could wreak all kinds of havoc. _Debug_ offer us a better solution in the form of cloning a running _Pod_ and including a _Debug_ container running inside the new _Pod_.

Let's take a look at this in practice, we'll work again with our **prometheus-kube-state-metrics** _Pod_:

```bash
microk8s kubectl debug prometheus-kube-state-metrics-bbd98b855-tm8bm -it --image=busybox --share-processes --copy-to=debug-pod
# Defaulting debug container name to debugger-s8jqd.
root:/#
```

This will drop us in to our _Busybox_ shell in our new _Pod_ named **debug-pod**. Let's take a closer look:

```bash
#--Confirm Running Debug Pod
microk8s kubectl get pod debug-pod
# NAME                               READY   STATUS    RESTARTS
# debug-pod                          2/2     Ready     0

microk8s kubectl describe pod debug-pod
# ...
# Ephemeral Containers:
#   debugger-s8jqd:
#     Image:        busybox
#     Port:         <none>
#     Host Port:    <none>
#     Environment:  <none>
#     Mounts:       <none>
# ...
```

## So You're Using A Managed Kubernetes Platform

Well it looks like you're probably out of luck, and so am I since I spend most of my time in _EKS_. looking at the big 3 players (**[_GKE_](https://cloud.google.com/kubernetes-engine)**, **[EKS](https://aws.amazon.com/eks/)** and **[AKS](https://azure.microsoft.com/en-gb/services/kubernetes-service/)**) it's not turned on in _GKE_ or _EKS_ with no way to get it to turn on and at the time of writing AKS isn't offering Kubernetes 1.20.

As using this functionality requires enabling a _Feature Gate_ on the _kube api-server_ (itself part of the _Control Plane_) and the _Control Plane_ is fully managed, we don't really have a way to turn it on.

Statements on enabling Feature Gates and functions still in Alpha and Beta are either vague, misleading or outright wrong, often stating that you can ask for support but with no explanation of who to ask, sometimes that beta features are always enabled (but it turns out they aren't if they're behind _Feature Gates_) and in the case of EKS stating that the feature is present when the Feature Gate is still not enabled (if anyone has gotten this working I'd **love** to hear about it). I suspect this will be a case of getting it when it's in GA and not before:

<blockquote>
  Features in public preview are fall under 'best effort' support as these features are in preview and not meant for production and are supported by the AKS technical support teams during business hours only.
  <footer>- https://docs.microsoft.com/en-us/azure/aks/support-policies#preview-features-or-feature-flags</footer>
</blockquote>

<blockquote>
  The following Kubernetes features are now supported in Kubernetes 1.20 Amazon EKS clusters:…  
  ...kubectl debug has reached beta status. kubectl debug provides support for common debugging workflows directly from kubectl.
  <footer>– https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html</footer>
  
</blockquote>

<blockquote>
  Beta features are _enabled_ by default and _can be disabled by GKE_ for a particular version. In some cases, GKE may have disabled a beta feature for a specific GKE control plane version. Contact Cloud Customer Care for assistance in verifying if the beta feature is enabled on that GKE control plane version.
  <footer>– https://cloud.google.com/kubernetes-engine/docs/concepts/feature-gates</footer>
  
</blockquote>

For certaintly, I tested this on both GKE and EKS using their latest release platforms (this means using the _Rapid_ release channel in GKE) and if we take a look at the current flags enabled on the **kube api-server** we unfortunatley see nothing is enabled:

```bash
#--GKE v1.21 (Rapid Channel, Kubernetes 1.21.1)

#--Dumping the cluster-info and seraching for the EphemeralContainers flag gets us nothing
kubectl cluster-info dump | grep EphemeralContainers

#--Attempt to spawn a debug container
kubectl debug -it prometheus-kube-state-metrics-1cd57c819-zv1lp --image=busybox --target=kube-state-metrics
# error: ephemeral containers are disabled for this cluster (error from server: "the server could not find the requested resource").
```

```bash
#--EKS v1.20 (k8s 1.20.4)
kubectl cluster-info dump | grep feature-gate

#--Attempt to spawn a debug container
kubectl debug -it prometheus-kube-state-metrics-ba337a199-js82l --image=busybox --target=kube-state-metrics
# error: ephemeral containers are disabled for this cluster (error from server: "the server could not find the requested resource").
```

So I guess we're waiting for that one!
