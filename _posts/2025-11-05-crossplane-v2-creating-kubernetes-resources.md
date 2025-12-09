---
title: "Crossplane v2 - Creating Kubernetes Managed Resources"
layout: post
date: 2025-11-05
categories: 
  - "automation"
  - "containers"
  - "devops"
  - "integrations"
tags: 
  - "automation"
  - "cloud"
  - "crossplane"
  - "integration"
  - "kubernetes"
  - "microservices"
---

I've been playing a lot with [Crossplane](https://www.crossplane.io/) recently and found myself a little stuck trying to create additional Kubernetes resources using Crossplane itself as the orchestration layer. This is pretty well documented in disparate places, but I haven't seen it covered anywhere top to bottom so in this short post I'll try and cover the topic off.

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01.png" class="scaled-img-75">

## Installing and Configuring the Kubernetes Provider

In this example I am installing the Kubernetes *Provider* and creating a *ClusterProviderConfig* to serve it out to all tenants. Since we don't have to think about authenticating with a cloud provider like AWS, we can keep everything inside the cluster and that should make our lives a lot easier. For the most part this installation behaves the same as all others, with the exception of how the *credential* source is handled:

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-kubernetes
spec:
  package: xpkg.upbound.io/upbound/provider-kubernetes:v1.1.0
---
apiVersion: kubernetes.m.crossplane.io/v1alpha1 #--Namespaced endpoint
kind: ClusterProviderConfig #--Cluster wide
metadata:
  name: kubernetes
spec:
  credentials:
    source: InjectedIdentity #--Use the provider's existing SA
```

The *credential source* above is configured to use an *InjectedIdentity*. What this means is that the *Service Account* attached to the *Kubernetes Provider Pod* is used as the source of auth when attempting to create resources. When a provider is installed the Crossplane RBAC Manager automatically creates a *SA* and binds it to a *ClusterRole* with a suitable set of permissions (it's a pretty cool system).

With the Kubernetes *Provider* this is quite limited in what you can do by default, limiting you to creating *ConfigMaps*, *Secrets* and managing other *Crossplane* resources. But if you want to widen the scope of this you only have to create your own *SA* and attach it to the provider as shown below:

```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: DeploymentRuntimeConfig
metadata:
  name: kubernetes
spec:
  serviceAccountTemplate:
    metadata:
      name: provider-k8s-enhanced #--Your SA name here
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-kubernetes
spec:
  package: xpkg.upbound.io/upbound/provider-kubernetes:v1.1.0
  runtimeConfigRef:
    name: kubernetes #--Links to the Runtime Config above
```

With a *Provider* installed and configured, you can created Kubernetes resources by passing manifests nested within the Crossplane *MR* type `object.kubernetes.m.crossplane.io` as shown below:

```yaml
apiVersion: kubernetes.m.crossplane.io/v1alpha1 #--Namespaced resource
kind: Object
metadata:
  name: tfc-example-secret
spec:
  forProvider:
    manifest:
      #--Begin Kubernetes object manifest
      apiVersion: v1
      kind: Secret
      metadata:
        name: tfc-example-secret
        namespace: tenant1
      data:
        secret_information: c29tZXJ1YmJpc2gK
      #--End kubernetes object manifest
  providerConfigRef:
    name: kubernetes #--Created above
    kind: ClusterProviderConfig
```

We can see the resource has created and can be looked up using regular Kubernetes verbs:

```bash
kuebctl get secrets -n tenant1
NAME                TYPE           DATA   AGE
tfc-example-secret  Opaque         1      1m
```
