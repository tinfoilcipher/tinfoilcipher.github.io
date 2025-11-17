---
title: "Crossplane v2 – Infrastructure as Code for Kubernetes Platform Teams Part 2"
layout: post
date: 2025-10-14
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

In the [last post]({% post_url 2025-10-08-crossplane-v2-iac-for-k8s-platform-teams-part-1 %}) we took a look at some of the fundamentals of **[Crossplane](https://crossplane.io/)** v2, specifically how authentication and integration with AWS (and cloud providers more broadly) is handled now. There is a lot more to cover though, last time we only got around to creating some basic resources. In this post we're going to look at how we can leverage the main offering of *Crossplane*; *Composition*, giving us the ability to create our own [CRDs](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) and *Managed Resource* templates.

The sample code for this article can be found **[here](https://github.com/tinfoilcipher/blogexamples/tree/main/crossplane-v2-example/part-2)**.

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01.png" class="scaled-img-75">

## What Do We Have So Far

Throughout this article we will be using several of the components that we set up in [Part 1 of this guide]({% post_url 2025-10-08-crossplane-v2-iac-for-k8s-platform-teams-part-1 %}):

- A *DeploymentRuntimeConfig* called *aws* which uses an existing *IRSA* role
- A *Provider* called *provider-aws-s3*, using the *provider-aws-s3:v2.1.0* *Package* which consumes the *aws DeploymentRuntimeConfig*
- A *ClusterRole* and *ClusterRoleBinding* to handle *ServiceAccount* permissions for our *Provider*
- Namespaces for multiple tenants, named *tenant1* and *tenant2*
- A *ClusterProviderConfig* which will use the *provider-aws-s3 Provider* and authenticate using *IRSA*

We won't cover what each of these is for, it's all in the [previous article]({% post_url 2025-10-08-crossplane-v2-iac-for-k8s-platform-teams-part-1 %}).

## Composition. It's Templates All The Way Down!

So what is Composition? From the *Crossplane* documentation:

<blockquote>
  Crossplane’s key value is that it unlocks the benefits of building your own Kubernetes custom resources without having to write controllers for them.
  - https://docs.crossplane.io/latest/whats-crossplane/
</blockquote>

This "key value" is achieved via a templating system called [Composition](https://docs.crossplane.io/latest/composition/) which allows us, as platform administrators, to create our own pre-templated, resource sets and serve them to tenants as custom CRDs. The below diagram breaks this down in a bit more clarity:

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02.png">

As the diagram above shows, there are 4 key concepts to think about when working with *Composition* and they introduce a lot of new terminology:

- **Composite Resource Definitions** - These objects define the spec and schema for new platform-specific CRDs , which *Crossplane* called **XRDs**. These can be either Cluster-wide or Namespaced.
- **Compositions** - These objects define the spec for calling an *XRD*, including how we will handle inputs and templating. It also defines the exact *Managed Resources* to be provisioned by our *Provider*.
- **Composite Resources** - These objects, which *Crossplane* also calls **XRs**, are used to call an *XRD* and their creation will result in the provisioning of the *Managed Resources* defined in an available *Composition*. Unlike the other objects above, these are intended to be provisioned by tenants of your platform.
- **Functions** - Like *Packages*, these are external packages which offer extended functionality to *Crossplane*. Whilst their purpose is not solely part of *Composition*, we cannot practically use a *Composition* without them. Functions are used here for templating purposes, allowing us to *Patch* YAML manifests using input values from an *XR*

That's a lot to take in, so let's go through how to create each one of these components in a real world example. Our goal is going to be to offer a single *Composite Resource* to our platform tenants which:

- Creates an *S3 Bucket*
- Encrypts the *Bucket* using a provided *KMS Key ID*
- Tags the *Bucket*, using a provided set of tags
- Enables or disables versioning on request

## Composite Resource Definition

The below manifest contains our *XRD*, additional comments have been added to break down it's complexity but broadly this configuration is creating an input spec for a new kind of custom resource offering which will be served at a new endpoint specific to our platform; **s3buckets.aws.platform.tinfoilcipher.com**, with an object kind of **S3Bucket** and a strict set of allowed inputs:

```yaml
# 01-xrd.yaml

apiVersion: apiextensions.crossplane.io/v2
kind: CompositeResourceDefinition
metadata:
  name: s3buckets.aws.platform.tinfoilcipher.com #--Platform specific endpoint
spec:
  group: aws.platform.tinfoilcipher.com
  names:
    kind: S3Bucket #--Object Kind, E.G. kubectl get S3Bucket
    plural: s3buckets #--Kind plural. E.G. kubectl get s3buckets
  scope: Namespaced #--Namespaced or cluster-wide. For a multi tenant system, this should be offered to tenant namespaces
  versions:
    - name: v1alpha1 #--Current version you are releasing
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                parameters: #--Paramaters object, defines input params
                  type: object
                  properties:
                    bucketName: #--User input
                      type: string
                    region: #--User input
                      type: string
                    versioning: #--User input
                      type: string
                    kmsKeyID: #--User input
                      type: number
                  required: #--Mandatory inputs
                    - bucketName
                    - region
                    - kmsKeyID
                tags: #--Tags objects, defines a blank map for tags
                  type: object
                  additionalProperties:
                    type: string
              required: #--Mandatory objects. Tags are optional
                - parameters
```

Apply with:

```bash
kubectl apply -f 01-xrd.yaml
# compositeresourcedefinition.apiextensions.crossplane.io/s3buckets.aws.platform.tinfoilcipher.com created

kubectl get compositeresourcedefinition
# NAME                                       ESTABLISHED   OFFERED   AGE
# s3buckets.aws.platform.tinfoilcipher.com   True                    1m
```

## Function and Composition

The below manifest contains our *Function*. This will download an external package which will be used to patch in templated values to our *Composition*:

```yaml
#--02-function.yaml

apiVersion: pkg.crossplane.io/v1
kind: Function
metadata:
  name: patch-and-transform #--Name of your choice
spec:
  package: xpkg.upbound.io/upbound/function-patch-and-transform:v0.9.0 #--https://marketplace.upbound.io
```

Apply with:

```bash
kubectl apply -f 02-function.yaml
# function.pkg.crossplane.io/patch-and-transform created

kubectl get function
# NAME                  INSTALLED   HEALTHY   PACKAGE                                                          AGE
# patch-and-transform   True        True      xpkg.upbound.io/upbound/function-patch-and-transform:v0.9.0   2m
```

With the *Function* installed we can create our *Composition*. When you send an *XR* request to your newly created *XRD* with some input values:

- The *XRD* will pass the request to a *Composition* in order to actually provision *Managed Resources*.
- The *Managed Resources* are provisioned based on a serious of steps defined within a *Pipeline* within the *Composition*, in a similar manner to a CI/CD pipeline.
- Your chosen *Function* is used to handle your input values and *Patch* them in to the YAML spec for your *Compositions Managed Resources* (we've already touched on how *Managed Resources* work generally in [last post]({% post_url 2025-10-08-crossplane-v2-iac-for-k8s-platform-teams-part-1 %}), so we won't cover them here again).

All of this together creates a system where multiple *Managed Resources can be templated in one action from a single resource.

Let's take a look at how our *Composition* manifest looks, with comments to expand on it's functionality:

```yaml
# 03-composition.yaml

apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: s3bucket.aws.platform.tinfoilcipher.com #--Composition name
  labels:
    aws.platform.tinfoilcipher.com/composition: s3bucket #--Label to identify this Composition
spec:
  compositeTypeRef:
    apiVersion: aws.platform.tinfoilcipher.com/v1alpha1 #--XRD Endpoint
    kind: S3Bucket #--XRD Kind
  mode: Pipeline
  pipeline: #--Pipeline of steps to create Managed Resources
    - step: create-bucket #--Pipeline step (everything in this example is handled in one big step)
      functionRef:
        name: patch-and-transform #--Previously created function
      input:
        apiVersion: pt.fn.crossplane.io/v1beta1 #--Function (patch-and-transform) endpoint
        kind: Resources
        resources:

        #--Managed resources to be created--#
        #--https://marketplace.upbound.io/providers/upbound/provider-aws-s3/v2.1.1/resources/s3.aws.m.upbound.io/Bucket/v1beta1
        - name: s3-bucket
          base:
            apiVersion: s3.aws.m.upbound.io/v1beta1  #--Namespaced S3 endpoint
            kind: Bucket #--S3 Bucket resource
            spec:
              forProvider:
                #--Static configuration elements
                forceDestroy: false
              providerConfigRef:
                kind: ClusterProviderConfig
                name: aws-s3 #--Existing ClusterProviderConfig
          #--Patches take input values from an XR and pass them through
          #--the XRD spec and in to the Composition.
          #--E.G. the XRs input value in spec.parameters.region is used to
          #--patch spec.forProvider.region in the Composition
          patches:
            - fromFieldPath: "spec.parameters.bucketName"
              toFieldPath: "metadata.name"
            #--Applies the label aws.platform.tinfoilcipher.com/bucket_name based on input value to the newly created bucket object.
            #--Since Crossplane Managed Resources are just regular k8s objects, they can have labels applied to them like any
            #--other object.
            - fromFieldPath: "spec.parameters.bucketName"
              toFieldPath: "metadata.labels['aws.platform.tinfoilcipher.com/bucket_name']"
            - fromFieldPath: "spec.parameters.region"
              toFieldPath: "spec.forProvider.region"
            - fromFieldPath: "spec.tags"
              toFieldPath: "spec.forProvider.tags"

        #--A second resource is created in the pipeline. Configuring versioning
        #--https://marketplace.upbound.io/providers/upbound/provider-aws-s3/v2.1.1/resources/s3.aws.m.upbound.io/BucketVersioning/v1beta1
        - name: s3-bucket-versioning
          base:
            apiVersion: s3.aws.m.upbound.io/v1beta1
            kind: BucketVersioning
            spec:
              providerConfigRef:
                kind: ClusterProviderConfig
                name: aws-s3
          patches:
            - fromFieldPath: "spec.parameters.region"
              toFieldPath: "spec.forProvider.region"
            - fromFieldPath: "spec.parameters.versioning"
              toFieldPath: "spec.forProvider.versioningConfiguration.status"
            - fromFieldPath: "spec.parameters.bucketName"
              toFieldPath: "metadata.labels['aws.platform.tinfoilcipher.com/bucket_name']"
            #--The bucketSelector function matches the label for our existing Bucket resource created in the previous
            #--action. Labels are the preferred method for looking up existing or newly provisioned Managed Resources.
            - fromFieldPath: "spec.parameters.bucketName"
              toFieldPath: "spec.forProvider.bucketSelector.matchLabels['aws.platform.tinfoilcipher.com/bucket_name']"

        #--A third resource is created in the pipeline. Configuring encryption with a provided key
        #--https://marketplace.upbound.io/providers/upbound/provider-aws-s3/v2.1.1/resources/s3.aws.m.upbound.io/BucketServerSideEncryptionConfiguration/v1beta1
        - name: s3-bucket-encryption
          base:
            apiVersion: s3.aws.m.upbound.io/v1beta1
            kind: BucketServerSideEncryptionConfiguration
            spec:
              forProvider:
                #--Static configuration elements
                rule:
                  - applyServerSideEncryptionByDefault:
                      sseAlgorithm: aws:kms
              providerConfigRef:
                kind: ClusterProviderConfig
                name: aws-s3
          patches:
            - fromFieldPath: "spec.parameters.region"
              toFieldPath: "spec.forProvider.region"
            - fromFieldPath: "spec.parameters.bucketName"
              toFieldPath: "metadata.labels['aws.platform.tinfoilcipher.com/bucket_name']"
            - fromFieldPath: "spec.parameters.kmsKeyID"
              toFieldPath: "spec.forProvider.rule[0].applyServerSideEncryptionByDefault.kmsMasterKeyId" #--KMS Key Input
            #--The same label lookup as before to locate the correct Bucket
            - fromFieldPath: "spec.parameters.bucketName"
              toFieldPath: "spec.forProvider.bucketSelector.matchLabels['aws.platform.tinfoilcipher.com/bucket_name']"
```

Apply with:

```bash
kubectl apply -f 03-composition.yaml
# composition.apiextensions.crossplane.io/s3bucket.aws.platform.tinfoilcipher.com created

kubectl get composition
# NAME                                         XR-KIND    XR-APIVERSION                             AGE
# s3bucket.aws.platform.tinfoilcipher.com      S3Bucket   aws.platform.tinfoilcipher.com/v1alpha1   2m
```

## Resource Dependencies, Labels and Selectors

That's a lot of YAML, comments and templating to take in. One thing that might get lost is the important role that [Kubernetes Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) play in making this *Pipeline* work. Since *Managed Resources* are just the same as any other Kubernetes object, they can have *Labels* applied to them and can be looked up using *Selectors*. This lookup method is how *Crossplane* handles forward dependencies during *Composition*.

*Crossplane* leverages this existing *Labeling* functionality in it's *Composition Pipeline* by applying *Labels* to each *Managed Resources* it creates and then allowing downstream *Managed Resources* to look up existing ones via *Selectors*. This allows for *XRDs* where *Managed Resources* can be created that rely on the outputs of other *Managed Resources* earlier in the same pipeline, including relying on outputs which are not yet known at run time.

The below diagram breaks down how this is done in the example *Composition* above, for the sake of making things easier to read, I have fully rendered the YAML and removed the patches:

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03.png">

## Creating a Composite Resource (XR)

With all of this understood, we can finally start making some resources. The manifest below shows two *Composite Resources* or *XRs* which call our *XRD*. Applying this will start off our *Composition Pipeline*:

```yaml
# 04-xr.yaml

apiVersion: aws.platform.tinfoilcipher.com/v1alpha1 #--XRD endpoint
kind: S3Bucket #--XRD Kind
metadata:
  name: tinfoil-example-bucket-14-10-25-tenant-1 #-Bucket name
  namespace: tenant1 #--Tenant namespace
spec:
  #--Only explicit input values can be passed to "parameters"
  #--as defined in the XRD
  parameters:
    bucketName: tinfoil-example-bucket-14-10-25-tenant-1
    region: eu-west-2
    versioning: Enabled
    kmsKeyID: 15abcf4-15c5-1fe6-afbd-584bcfe57168
  #--Any list of tags can be passed as input values to
  #--"tags", as defined in the XRD
  tags:
    environment: dev
    department: devops
---
apiVersion: aws.platform.tinfoilcipher.com/v1alpha1
kind: S3Bucket
metadata:
  name: tinfoil-example-bucket-14-10-25-tenant-2
  namespace: tenant2
spec:
  #--Omitted for brevity
  ...
```

Apply with:

```bash
kubectl apply -f 04-xr.yaml
# s3bucket.aws.platform.tinfoilcipher.com/tinfoil-example-bucket-14-10-25-tenant-1 created
# s3bucket.aws.platform.tinfoilcipher.com/tinfoil-example-bucket-14-10-25-tenant-2 created
```

Whilst the Kubernetes objects have been created immediately, the *Composition Pipeline* will take a short while to run, so the AWS resources will not be ready for a minute or so. After a short wait we can check:

```bash
kubectl get s3bucket --all-namespaces
# NAME                                       SYNCED   READY   COMPOSITION                               AGE
# tinfoil-example-bucket-14-10-25-tenant-1   True     True    s3bucket.aws.platform.tinfoilcipher.com   2m03s
# tinfoil-example-bucket-14-10-25-tenant-2   True     True    s3bucket.aws.platform.tinfoilcipher.com   2m06s
```

Looking in the console, we can see that the *Buckets* have been created, and that our configurations have been passed as expected:

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/04.png">

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/05.png">

## How Do Compositions Get Selected?

Something that is easy to miss in that whole dance is that there is no implicit link between the *XR* and *Composition*. When we create our *XR* it calls the *XRD* endpoint and the *Composition Pipeline* is triggered, but we have not made any kind of direct request to do that.

When an *XR* is created, the *XRD* will attempt to automatically locate a suitable *Composition*; if there is only a single *Composition* which calls the *XRD* then this will work correctly. However if there are several *Compositions* calling a single *XRD* (as may be the case in more advanced platforms), or if you just want to be explicit in your intentions, you can pass a **Composition Ref** as part of your XR.

In the example below, two different *XRs* are being created, explicitly using different *Compositions* on the same *XRD*:

```yaml
apiVersion: aws.platform.tinfoilcipher.com/v1alpha1
kind: S3Bucket
metadata:
  name: tinfoil-example-bucket-14-10-25-tenant-1
  namespace: tenant1
spec:
  #--Explicitly define 
  crossplane:
    compositionRef:
      name: s3bucket.aws.platform.tinfoilcipher.com
...
---
apiVersion: aws.platform.tinfoilcipher.com/v1alpha1
kind: S3Bucket
metadata:
  name: tinfoil-example-bucket-14-10-25-tenant-2
  namespace: tenant2
spec:
  crossplane:
    compositionRef:
      name: s3bucket-alternative.aws.platform.tinfoilcipher.com
...
```

If you are working in a scenario where you have multiple *Compositions* calling the same *XRD* and you don't pass a *Composition Ref*, you will run in to some messy behavior where the *XR* appears to select a *Composition* at random without consistency. I cannot be sure but I suspect that this is due to the way Kubernetes returns results from an endpoint as a list, to avoid such chaotic behaviour, it would be advisable to be as specific as possible about the *Composition* you want to work with unless you know for sure there is only one to be selected!

## More Functionality?

There is still a lot more lurking inside Crossplane. I've not yet touched on scheduling jobs, executing custom code, hardening workflows, passing data between providers or the additional power that comes from the existing library of *Functions*, but this is more than enough for one post. We'll try to visit all of this in future posts if there is any interest in it.
