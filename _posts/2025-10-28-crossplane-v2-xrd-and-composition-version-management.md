---
title: "Crossplane v2 - XRD and Composition Version Management"
layout: post
date: 2025-10-28
categories: 
  - "automation"
  - "aws"
  - "containers"
  - "devops"
  - "integrations"
tags: 
  - "automation"
  - "aws"
  - "cloud"
  - "crossplane"
  - "integration"
  - "kubernetes"
  - "microservices"
---

[Recently we got to grips with **Crossplane** v2 and how to create both bespoke *CRDs* and make use of it's *Composition* system to create bespoke provisioning workflows]({% post_url 2025-10-14-crossplane-v2-iac-for-k8s-platform-teams-part-2 %}). What we didn't get to cover though is how to manage *XRDs* and *Compositions* on an ongoing basis, adding new features and offering to them to our platforms as the need arises.

A lot of systems like this get quickly deployed without much foresight and platform teams are unlikely to build a perfect system on the first try. It's easy to run in to a wall when you have to start adding new features that you didn't anticipate only to realise that the release of a new version doesn't work in the way that you first thought. Crossplane does offer several versioning systems which are pretty flexible but this flexibility means there are multiple ways of doing things that you should be aware of when getting started. So...let's look at that.

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01.png" class="scaled-img-75">

## What Are We Working With?

In a [previous article]({% post_url 2025-10-08-crossplane-v2-iac-for-k8s-platform-teams-part-1 %}) we looked at how *XRDs* and *Compositions* work. I'm going to assume you already know that and not get bogged down in the fundamentals here. If you don't, the [previous article]({% post_url 2025-10-08-crossplane-v2-iac-for-k8s-platform-teams-part-1 %}) covers this off in detail.

Below is a basic example of an *XRD* being offered on our platform. A single version is currently offered; **v1alpha1** and a set of string inputs are accepted as part of the schema:

```yaml
apiVersion: apiextensions.crossplane.io/v2
kind: CompositeResourceDefinition
metadata:
  name: s3buckets.aws.platform.tinfoilcipher.com #--Platform specific endpoint
spec:
  group: aws.platform.tinfoilcipher.com
  names:
    kind: S3Bucket
    plural: s3buckets
  scope: Namespaced
  versions:
    #--Current version and spec
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                parameters:
                  type: object
                  properties:
                    bucketName:
                      type: string
                    region:
                      type: string
                    versioning:
                      type: string
                  required:
                    - bucketName
                    - region
                    - versioning
              required:
                - parameters
```

## XRD Versions - The Devil Is In The Detail

Generally, it is a bad idea to update something once it already has a version defined and released. There are exceptions to this however. A still in-development version of a system can be considered prone to change at any time, but stable releases should not see sudden change.

The [Crossplane Documentation on XRD Versions](https://docs.crossplane.io/latest/composition/composite-resource-definitions/#xrd-versions) suggests aligning *XRD* versions and evolution to the broader [Kubernetes API Versioning guidelines](https://kubernetes.io/docs/reference/using-api/#api-versioning), where:

- **alpha** (I.E. **v1alpha1**) releases are unstable, and prone to change at any time
- **beta** (I.E. **v1beta1**) releases are "considered stable and breaking changes are strongly discouraged", sensible if pretty vague language
- **stable** (I.E. **v1**) - releases are stable, and should not have breaking changes introduced

This model gives us something to work with, but how does this map on to *XRDs* in reality? *The *XRD* spec allows for the serving of multiple *Versions* at the same time as part of a single object:

```yaml
apiVersion: apiextensions.crossplane.io/v2
kind: CompositeResourceDefinition
metadata:
  name: s3buckets.aws.platform.tinfoilcipher.com
spec:
  group: aws.platform.tinfoilcipher.com
  names:
    kind: S3Bucket
    plural: s3buckets
  scope: Namespaced
  #--Available XRD Versions
  versions:
    - name: v1alpha1
    ...
    - name: v1alpha2
    ...
    - name: v1alpha3
    ...
```

Makes sense. But if you just try and add multiple new version schemas you will probably run in to a whole host of errors very quickly.

I found it useful to read this [Crossplane Blog Post](https://blog.upbound.io/crossplane-xr-best-practices), which discusses in detail the underlying theory of how Kubernetes objects are actually read and written to the Kubernetes data store ([etcd](https://etcd.io/)). I would strongly recommend reading this as it will serve as a good basis for understanding **WHY** we face the limitations we do here, it doesn't however touch on the realities of trying to release a new version of an *XRD* and how they interact with *Compositions*, so let's take a look at that.

## Introducing A New XRD Version

If for example we were to modify the inputs on **v1alpha1** to include or remove a new mandatory parameter; any *XRs* which are calling **s3buckets.aws.platform.tinfoilcipher.com/v1alpha1** would stop working and workflows would be interrupted. Much more importantly, any associated *Compositions* may cease to function, as expected input parameters may no longer exist.

Whilst *XRDs* **do** allow for the offering of multiple available versions at the same time, only a single version can act as the *Referenceable* version. There is some [complexity to how this works under the hood](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/#writing-reading-and-updating-versioned-customresourcedefinition-objects), but for brevity we can think of the *Referenceable* version as the preferred, standard version that should be used when calling an *XRD*, in simpler language we might call this the *Default* version.

In the example below, we will leave version **v1alpha1** available, and add a new version **v1alpha2** which adds a new **optional** input parameter:

```yaml
apiVersion: apiextensions.crossplane.io/v2
kind: CompositeResourceDefinition
metadata:
  name: s3buckets.aws.platform.tinfoilcipher.com #--Platform specific endpoint
spec:
  group: aws.platform.tinfoilcipher.com
  names:
    kind: S3Bucket
    plural: s3buckets
  scope: Namespaced
  versions:
    - name: v1alpha1 #--Current version, being deprecated
      served: true #--Available for Compositions
      referenceable: false #--Available, not default. This must be set to false to release a new version
      schema:
        openAPIV3Schema:
        ...
          type: object
          properties:
            spec:
              type: object
              properties:
                parameters:
                  type: object
                  properties:
                    bucketName: #--User input
                      type: string
                    region: #--User input
                      type: string
                  required: #--Mandatory inputs
                    - bucketName
                    - region
    - name: v1alpha2 #--New version
      served: true #--Available for Compositions
      referenceable: true #--Available, default
      schema:
        openAPIV3Schema:
        ...
          type: object
          properties:
            spec:
              type: object
              properties:
                parameters:
                  type: object
                  properties:
                    bucketName:
                      type: string
                    region:
                      type: string
                    #--Newly added changes to schema
                    forceDeletion:
                      type: boolean
                  required: #--Mandatory inputs, unchanged
                    - bucketName
                    - region
```

After configuring and saving, we can see the available versions with:

```bash
kubectl get crd s3buckets.aws.platform.tinfoilcipher.com -o=jsonpath='{.spec.versions[*].name}'
# v1alpha1, v1alpha2
```

It is important to point out here that the [Crossplane Documentation](https://docs.crossplane.io/latest/composition/composite-resource-definitions/#multiple-schema-versions) is very clear on the handling of breaking changes. Any attempt to introduce a breaking change should not be handled as a new version on the same *XRD*, this **MUST** be done as an entirely separate **XRD**. There are some interesting technical reasons for this, but I won't cover them here, they are [detailed in this post if you are interested](https://blog.upbound.io/crossplane-xr-best-practices).

## XRD Versions and Compositions Are Joined At The Hip

So we know how to introduce a new version to our *XRD*, but this runs in to a serious limitation when we start working with an existing *Composition*. If we look at the below *Composition* we can see that the *XRD* being called is referenced via the field `spec.compositeTypeRef.apiVersion`.

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: s3bucket.aws.platform.tinfoilcipher.com
spec:
  compositeTypeRef:
    apiVersion: aws.platform.tinfoilcipher.com/v1alpha1 #--XRD Endpoint
    kind: S3Bucket
  mode: Pipeline
  pipeline:
    ...
```

This field is immutable, meaning that it cannot be changed following the creation of a *Composition*. The only option we have here is to create a **NEW** *Composition* and point it to our new *XRD Version*. This of course has downstream effects, as depending on the configuration of our *XRs* they might suddenly start reading a newly deployed *Composition*, or they might know nothing about it.

## Composition and Referenceable XRD Versions - There Can Be Only One!

Another complexity here that I could not find documented, is that *Compositions* only appear to be able to make use of the current *Referenceable XRD version*. Making the idea of serving multiple *XRD Versions* of limited use to some degree.

The practical effect here is that if you change the *Referenceable Version* of an *XRD*, any *XRs* which are currently managed by a *Composition* linked to a previous version will become de-synchronised and future changes will fail to be written back to your cloud provider until they are migrated to a newer *Composition*. This could have serious consequences if not properly understood and managed.

In the below example, we have an *XRs* which is using a *Composition* linked to *XRD version* `v1alpha1`:

```yaml
apiVersion: aws.platform.tinfoilcipher.com/v1alpha1
kind: S3Bucket
metadata:
  name: tinfoil-example-bucket-12-11-25-tenant-1
  namespace: tenant1
spec:
  crossplane:
    compositionRef:
      name: v1alpha1-composition #--Point an XR to a specific composition
      ...
```

If we examine the *XR*, we can see it is healthy:

```bash
kubectl get s3bucket --all-namespaces
# NAME                                       SYNCED   READY   COMPOSITION            AGE
# tinfoil-example-bucket-12-11-25-tenant-1   True     True    v1alpha1-composition   1m2s
```

Now if we set `v1alpha2` as the *Referenceable XRD Version*, we will see our *XR* remains in a *READY* state, meaning that they are still in existence in our cloud provider, but that they are not *SYNCED*. This means we can read the object, but not write changes back to our cloud provider:

```bash
kubectl get s3bucket --all-namespaces
# NAME                                       SYNCED   READY   COMPOSITION            AGE
# tinfoil-example-bucket-12-11-25-tenant-1   False    True    v1alpha1-composition   1m10s
```

Our *XR* will remain in this state until we reconfigure it to:

- Use a *Composition* that calls the *XRD version* `v1alpha2`
- Use `v1Alpha2` as it's own `apiVersion`

At which point they will become *SYNCED* again:

```yaml
apiVersion: aws.platform.tinfoilcipher.com/v1alpha2
kind: S3Bucket
metadata:
  name: tinfoil-example-bucket-12-11-25-tenant-1
  namespace: tenant1
spec:
  crossplane:
    compositionRef:
      name: v1alpha2-composition #--Point an XR to a specific composition
      ...
```

Checking, we can see the *XR* is not back in sync:

```bash
kubectl get s3bucket --all-namespaces
# NAME                                       SYNCED   READY   COMPOSITION            AGE
# tinfoil-example-bucket-12-11-25-tenant-1   True     True    v1alpha2-composition   3m12s
```

With all this in mind, it should go without saying that managing *Composition* changes needs to be done with serious care and consideration for impact. **By default, a change made to an active *Composition* will be automatically served to all *XRs* which are consuming it**. If, for example, you make a change in your *Composition* to provision (or worse, destroy) some *Managed Resources*, you might quickly find yourself in a situation where all of your platform tenants are creating or destroying real world infrastructure.

There are systems in place to version control *Compositions* and avoid these situations and it will probably be no surprise by now that this introduces even more complexity and can be approached in a few different ways which we'll try to cover below.

## Compositions Have Their Own Versions?

*Compositions* have their own internal versioning system called *Revisions*. Every time a change is made to an individual *Composition*, a new version is automatically created and maintained as a separate object.

In the below example, I have modified the **s3bucket.aws.platform.tinfoilcipher.com** *Composition* to add a new *Patch Function*:

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: s3bucket.aws.platform.tinfoilcipher.com
  #--Custom labels
  labels:
    aws.platform.tinfoilcipher.com/composition: s3
spec:
  compositeTypeRef:
    apiVersion: aws.platform.tinfoilcipher.com/v1alpha1 #--New version
    kind: S3Bucket
  mode: Pipeline
  pipeline:
    ...
          patches:
            ...
            #--Newly added patch
            - fromFieldPath: "spec.tags"
              toFieldPath: "spec.forProvider.tags"
```

Revisions are managed using their own object *Kind*; **CompositionRevisions**. We can see that a new *Revision* has been created if we look these up. We look up the  object using a label automatically added by crossplane:

```bash
kubectl get compositionrevision -l crossplane.io/composition-name=s3bucket.aws.platform.tinfoilcipher.com
# NAME                                              REVISION   XR-KIND    XR-APIVERSION                             AGE
# s3bucket.aws.platform.tinfoilcipher.com-794c340   2          S3Bucket   aws.platform.tinfoilcipher.com/v1alpha1   5s
# s3bucket.aws.platform.tinfoilcipher.com-f1d929b   1          S3Bucket   aws.platform.tinfoilcipher.com/v1alpha1   23m
```

## Managing Composition Versions - Pinning an XR To A Composition Version

*Crossplane* offers the ability to pin not only to a specific *Composition*, but to a specific *Composition Revision* as shown below. When provisioning an *XR* we can pass a *crossplane* configuration in our *spec*:

```yaml
apiVersion: aws.platform.tinfoilcipher.com/v1alpha1
kind: S3Bucket
metadata:
  name: tinfoil-example-bucket-12-11-25-tenant-1
  namespace: tenant1
spec:
  crossplane:
    compositionUpdatePolicy: Manual #--If not set, XRs will take changes WHEN A COMPOSITE IS CHANGED    
    compositionRef:
      name: s3bucket.aws.platform.tinfoilcipher.com #--Specific Composition
    compositionRevisionRef:
      name: s3bucket.aws.platform.tinfoilcipher.com-f1d929b #--Pin to a specific version
```

Personally I wouldn't recommend this for long term use, it feels messy and the *Revision* refs aren't very user friendly for platform consumers. It is however an ideal candidate for debugging and testing out new *Compositions*.

## Managing Composition Versions - Subscribing to Update Channels

Crossplane's preferred way of doing most things is by leveraging [Kubernetes Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) and *Composition* versioning can be handled in this same manner. In the above example of a *Composition* I have added some *Custom Labels* that we can use to "subscribe" to updates as they are released, the same way as we would subscribe to an update channel for application updates.

In the example below, we are adding a label to our *Composition*, **release_channel**:

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: s3bucket.aws.platform.tinfoilcipher.com #--Composition names need to be unique
  labels:
    release_channel: unstable
spec:
  compositeTypeRef:
    apiVersion: aws.platform.tinfoilcipher.com/v1alpha2 #--Uses new XRD
```

This is useful in *gitflow* or similar rapid change environments where we have multiple available versions of the same *Composition*, with different *Composition Pipelines*. One version could be offered for example as **unstable**, another as **stable** and another as **experimental**. What is critical to understand here is that the *Composition* remains a **SINGLE OBJECT** with multiple *Revision* objects. The YAML manifest used to configure the *Composition* is still ultimately a single file so it is critical to consider your approach to source control before trying to use this system.

Such a configuration will give us something like:

```bash
kubectl get compositionrevisions -o="custom-columns=NAME:.metadata.name,REVISION:.spec.revision,CHANNEL:.metadata.labels.release_channel"
# NAME                                              REVISION   CHANNEL
# s3bucket.aws.platform.tinfoilcipher.com-acd735a   7          experimental
# s3bucket.aws.platform.tinfoilcipher.com-bb720ba   6          experimental
# s3bucket.aws.platform.tinfoilcipher.com-38cba24   5          experimental
# s3bucket.aws.platform.tinfoilcipher.com-381653c   4          experimental
# s3bucket.aws.platform.tinfoilcipher.com-137dcff   3          unstable
# s3bucket.aws.platform.tinfoilcipher.com-794c340   2          stable
# s3bucket.aws.platform.tinfoilcipher.com-f1d929b   1          stable
```

Based on this, we can then subscribe several different *XRs* to separate *Revisions* based on labels as shown below. The *compositionUpdatePolicy* must be set to *Automatic* (which is the default) in order to subscribe to updates like this, as this method relies on watching for incoming *Revisions* on a *Composition*.

```yaml
#--XR created from unstable channel
apiVersion: aws.platform.tinfoilcipher.com/v1alpha2
kind: S3Bucket
metadata:
  name: tinfoil-example-bucket-28-10-25-tenant-1
  namespace: tenant1
spec:
  crossplane:
    compositionUpdatePolicy: Automatic #--Set by default
    compositionRevisionSelector:
      matchLabels:
        release_channel: experimental

#--XR Created from stable channel
apiVersion: aws.platform.tinfoilcipher.com/v1alpha1
kind: S3Bucket
metadata:
  name: tinfoil-example-bucket-28-10-25-tenant-2
  namespace: tenant2
spec:
  crossplane:
    compositionUpdatePolicy: Automatic #--Set by default
    compositionRevisionSelector:
      matchLabels:
        release_channel: stable
```
