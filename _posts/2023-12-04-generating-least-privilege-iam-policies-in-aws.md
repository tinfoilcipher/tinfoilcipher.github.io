---
title: "Generating Least Privilege IAM Policies in AWS"
layout: post
date: 2023-12-04
categories: 
  - "automation"
  - "aws"
  - "devops"
  - "security"
tags: 
  - "automation"
  - "aws"
  - "cloud"
  - "devops"
  - "iam"
  - "security"
---

If you've ever worked with AWS in the real world you are probably very used to seeing [**IAM Users and Roles**](https://docs.aws.amazon.com/IAM/latest/UserGuide/id.html) which are terrifyingly over-permissioned. In my experience it's pretty common to find them in the wild with access to every attribute of a specific service or just as often the native **[_AdministratorAccess_](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AdministratorAccess.html)** _Managed Policy_ assigned.

The **[principle of least privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege)** is a concept that you often **hear** about a lot but it's not uncommon to see people taking the path of least resistance in the wild and shuffling off security concerns until some point in the indeterminate future.

This week I've been working on some access policies myself so I thought I'd take a look at some of the tools that are out there to make our lives easier depending on the environments we're working in.

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01.webp" class="scaled-img-75">

## First...How Did We Get In This Mess?

The reasons for this whole situation are pretty easy to understand.

IAM isn't very user friendly, especially to newcomers. It's also built up quite a lot of cruft over the years, with some permissions confusingly crossing between services. This all creates a system where it's easy to lose time trying to work out which permissions are missing and just as easy to ostensibly get things working without realising there is even a problem.

On the other side of this, it's reasonable that **someone** needs to be an administrator in your organisation or at least have a very high level of access, after all we live in the real world and sooner or later someone is going to have to log in directly to delete some files. Our problem gets out of hand when we have automatically provisioned users and roles ending up with way too much access.

I read an article from Microsoft earlier this year suggesting that as many as 80% of cloud identities are inactive...imagine how many of them have highly privileged policies attached and are just waiting to be compromised. It doesn't give me a good feeling.

## iamlive - My Preferred Solution

**[iamlive](https://github.com/iann0036/iamlive)** is a tool for tracking outbound calls to the AWS API and then mapping them in to a JSON document ready for use as an IAM policy. Very cool stuff and I've found it to be the ideal answer to the least privilege problem. There a couple of ways to use it and they differ depending on your needs.

First let's download the application:

```bash
#--Ubuntu 
curl -s https://api.github.com/repos/iann0036/iamlive/releases/latest \
  | grep "linux-$(dpkg --print-architecture)" \
  | cut -d : -f 2,3 \
  | tr -d \" \
  | wget -qi -

tar zxvf iamlive* && rm iamlive*.tar.gz
mv iamlive /usr/bin

#--MacOS (Requires Homebrew)
brew install iann0036/iamlive/iamlive

```

Now that the application is installed, we'll need to at least have one highly privileged account which can already access AWS. I won't go in to the details of **how** to authenticate with the AWS CLI, if you're reading this then you probably already know that much (if you don't though feel free to drop me a line).

## iamlive - Client Side Monitoring (AWS CLI)

Let's take a look at using _iamlive_ first in **[Client Side Monitoring](https://github.com/iann0036/iamlive#csm-mode)** mode which captures local calls via UDP. This is pretty reliable when using the AWS CLI if that's the way you're doing things. We need to specify that we're using CSM with an environment variable in the shell that we're running our AWS CLI commands and then start up _iamlive_ in a separate terminal:

```bash
#--In the execution shell (this shell will already need to be authenticated with AWS with a privileged account)
export AWS_CSM_ENABLED=true
aws ec2 create-vpc --cidr-block "10.0.0.0/16"
aws ec2 describe-vpcs
aws ec2 delete-vpc --vpc-id vpc-0e3fb74409de95bea  #--Swap for your VPC ID

#--In the iamlive shell
iamlive --set-ini --output-file policy.json --force-wildcard-resource #--Forcing a wildcard is optional but
                                                                      #--much more practical for provisioning uses
```

The below screenshots show this in action:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-2.png">
  <figcaption>iamlive is now listening for any calls to AWS. We are running _iamlive_ with an output to a file named policy.json but changes will also be displayed live in the console. We are also using the --force-wildcard argument which will specify the wildcard on all resources. By default only the specific resources being targeted will be included but this is really too narrow if we're creating a policy for a role or account which will be used to provision resources of a specific type.</figcaption>
</figure>

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-1.png">
  <figcaption>Creating a VPC adds the first couple of statements to our policy.</figcaption>
</figure>

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/04.png">
  <figcaption>Likewise, describing the VPC adds it's own permission attribute to the policy.</figcaption>
</figure>

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/05.png">
  <figcaption>Finally, deleting the policy adds it's own attribute. This gives us the bare minimum policy statement to create, describe and destroy VPCs.</figcaption>
</figure>

## iamlive - Proxy Interception (Terraform, SDK, CDK)

So this is all fine to create policies with the AWS CLI, but most infrastructure doesn't really get created that way. Most of the time our resources are created with tooling that leverage the AWS SDK or CDK (in my case this is usually Terraform) and client side monitoring is quite inaccurate for this.

Frustratingly CSM mode does catch some of the calls to AWS so the problem might not be immediatley obvious but it can miss a good chunk of them (for example all of the VPC calls in this example get missed).

To use _iamlive_ this way we will need to use [_proxy_ mode](#--In the iamlive shell) which spins up a local HTTP(S) server and intercepts calls before they are sent to AWS.

I'm going to use some example code which you can obtain here which creates a VPC and a few subnets:

```bash
#--In the execution shell (this is where Terraform will be running)

terraform init #--It is important to init BEFORE setting the proxy as we will be using HTTPS to download our providers
export HTTP_PROXY=http://127.0.0.1:10080
export HTTPS_PROXY=http://127.0.0.1:10080
export AWS_CA_BUNDLE=~/.iamlive/ca.pem
terraform plan
terraform apply

#--In the iamlive shell
iamlive --mode proxy --output-file policy.json --force-wildcard-resource
```

The below screenshots show this in action:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/10-1024x240.png">
  <figcaption>iamlive is now listening for any calls to AWS via the proxy. All other configurations are identical to our previous run.</figcaption>
</figure>

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/11-1024x389.png">
  <figcaption>Running terraform apply will implicitly run terraform plan. We can see the minimum permissions being generated for a plan operation.</figcaption>
</figure>

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/12-1024x400.png">
  <figcaption>After running an apply operation, we can see a policy has been generated, as Terraform looks up data to record in it's state, the describe attributes have automatically been included.</figcaption>
</figure>

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/13-1024x403.png">
  <figcaption>Finally, running a terraform destroy adds the final statements to our policy.</figcaption>
</figure>

## Other Viable Options

So this is all great. I really like **iamlive**, over the years I have had to solve this problem a few other ways and I've found none of them to be this useful. I want to touch on a couple of them briefly.

- **AWS Access Advisor** - This is actually the way that AWS recommend that you do things, you can find out some more information [**here**](https://docs.aws.amazon.com/IAM/latest/UserGuide/access-analyzer-policy-generation.html) in the official docs. The logic is not exactly dissimilar to _iamlive_, you send requests and a policy is generated for you. Personally though I find it a pretty clunky system to interact with and apparently so do other people [citation needed].

- **Localstack** - I've talked about localstack [a couple of times in the past]({{ site.url }}/search.html?q=localstack) and whilst it's a very useful tool it's imperfect for this job. Running in console mode it does catch permission calls to each endpoint, but since moving to a freemium model more and more endpoints are impossible to test for free, there is also the drawback of having to construct your own endpoints. It is viable in certain situations however.

- **AWS CloudWatch** - While I was looking in to this topic a common suggestion was to catch every IAM interaction in CloudWatch, scrape them and construct your own policies. That sounds like unbelievably tedious work and would at the very least mean writing your own solution. Not my favourite but I suppose it could be done.

## A Final Consideration About Testing

A couple of final thoughts on this topic, mutation testing is pretty essential if you're going to run with a least-privilege policy system in the real world. It's not enough to just configure creation and deletion permissions. Ongoing maintenance of a system will mean regular changes and that means different permissions will need to be applied so implementing a robust set of tests before you blindly apply will save you time.
