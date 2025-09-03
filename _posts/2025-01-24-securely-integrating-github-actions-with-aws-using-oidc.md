---
title: "Securely Integrating Github Actions with AWS using OIDC"
layout: post
date: 2025-01-24
categories: 
  - "aws"
  - "devops"
  - "integrations"
  - "security"
tags: 
  - "aws"
  - "cloud"
  - "devops"
  - "github"
  - "integration"
  - "oidc"
  - "secops"
  - "security"
---

If you're trying your hand at implementing a CI pipeline with **[Github Actions](https://github.com/features/actions)** and AWS you would be forgiven for finding yourself going in circles with some pretty confusing documentation when it comes to trying to get authentication working. Most of the examples and a lot of the blog posts out there tell you to just create an account, export the **[Access Key ID and Secret Access Key](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html)** to environment variables and then reference them in your pipeline, job done. In a lot of ways you can understand this, trying to explain anything more advanced to an absolute newcomer is pretty daunting.

In this post we'll look at a much more secure method for CI authentication using **OIDC** (Open ID Connect), that can do away with the pain of having to worry about secret management and the endless tedium of credential cycling that comes along with it.

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-2.png" class="scaled-img-75">

## What's So Wrong With Just Using An _Access Key ID_ and _Secret Access Key_?

What is left unsaid in examples that use an _Access Key ID_ and _Secret Access Key_ is just how insecure an implementations like this really are. It's pretty common in the wild to find examples where an _Access Key ID_ and _Secret Access Key_ have been generated and then haven't been rotated in years and are sitting around like a landmine waiting to be discovered, like a root password written on a post-it note.

Even if there **IS** a plan to rotate them, they are still pretty vulnerable. If a would be attacker gains access to your repository and is able to view or export them they may very well be able to use these same credentials to authenticate with your AWS account from anywhere (depending on what other security mechanisms you have in place) and an intrusion like this can easily go unnoticed. It's my experience that people tend to really know that there is something inherently wrong about about a setup like this but end up kludging it together anyway because it was in the official docs and OIDC sounds a bit too confusing.

## OIDC To The Rescue

A decent _OIDC_ implementation can help us out here. It works by performing _Authorization_ based on the source the _Authentication_ request (in this case, where our _Action_ is running).

In our example, we will be throwing away user account and _Access Key_ and replacing it with a couple of key components:

- An **[OIDC Provider](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html)**, which will mediate our authentication requests. AWS do a lot of the heavy lifting for us here, we just need to do some basic configuration.
- An _IAM role_, which our _Github Actions_ _steps_ will use to send _Authentication_ requests to the _OIDC Provider_
- A _Trust Relationship_ between the two, which allows access only from a specific Github Organisation or Repositories with a strict _TTL_ (Time To Live)

The below diagram shows our workflow at the high level:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-1.png)

As we can see, this arrangement removes the need for messing around with static credentials and instead gives us a tightly controlled system where short lived credentials are generated on demand and then revoked at the end of a session.

I'm not going to get bogged down in the details of how OIDC itself works, you can find that in a million places and this article will go on forever if we talk about it here too, if you are interested you can get a good breakdown [**here**](https://openid.net/developers/how-connect-works/).

So, that's a nice idea on paper, how do we actually set it up?

## Creating an AWS OIDC Provider

In the AWS console browse to **IAM** > **Identity Providers** > **Add Provider**. Configure:

- **Provider URL**: https:token.actions.githubusercontents.com - This will ensure that OIDC Authentication requests are accepted **ONLY** from _Github Actions Jobs_ and not from rogue sources
- **Audience**: This will ensure that STS tokens are issued only to an _IAM Role_ via the **[AWS Security Token Service](https://docs.aws.amazon.com/STS/latest/APIReference/welcome.html)** (I.E. a role configured via an appropriate _Trust Relationship_, which we will configure shortly).

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/image-1024x444.png)

Click **Add Provider**.

Our OIDC Provider is now configured and ready for use.

## Creating An IAM Role and Trust Relationship

Next, browse to **IAM** > **Roles** > **Add Role**. Configure:

- **Identity Provider**: This is the provider we just created which can be selected from a dropdown and will inform the rest of the fields we can choose.

- **Audience**: The audience(s) associated with our provider

- **Github organization**: Your _Github_ _Organisation_, this will ensure that OIDC _Authentication_ requests will only be accepted from this _Organisation_

- **Github repository**: This is optional, but can be used to further filter down the source of OIDC _Authentication_ requests. If specified then requests will only be accepted from a specific _Github Repository_

- **Github branch**: Also optional, if specified OIDC _Authentication_ requests will only be accepted from a specific branch on a specific repository

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/image-4-1024x645.png)

Click **Next** and attach a suitable set of _Policies_ to your _Role_:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/image-5-1024x298.png)

Click **Next** again and give the _Role_ a name and description. In this final page we can see how the **Trust Relationship** has been templated in JSON. We can manually edit the JSON document after creation via the _Role's_ _Trust Relationship_ tab if a more advanced set of filters is needed.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/07-1024x647.png)

Finally, click **Create** to create the new _Role_.

Our new role is now ready for consumption via _Github Actions_. By default it has a _TTL_ of 1 Hour (meaning that any sessions we initiate via _Github Actions_ have a 1 hour limit). Really this should be more than enough time, but it can be raised or lowered as needed under _Maximum Session Duration_:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/05-1024x216.png)

## Configuring IAM Credentials in Github Actions

Inside any _Github Actions_ manifest we can now use the official **[configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials)** action to perform our OIDC authentication as a _Step_:

```yaml{% raw %}
#--.github/workflows/example.yml

on:
  push: #--Execute on commit
  workflow_dispatch: #--Execute on manual request from web console

permissions:
      id-token: write #--This permission is neeeded to request a token from the OIDC provider

jobs:
  example_job:    
    name: Example Job
    runs-on: ubuntu-latest
    steps:

      - name: 'Configure Credentials'
        uses: aws-actions/configure-aws-credentials@v4.0.2
        with:
          role-to-assume: "your-role-arn" #--ARN of your role
          role-session-name: "gha_session_${{ github.run_id }}" #--Assign a unique ID to each session ID
          aws-region: "eu-west-2" #--Configure as appropriate
          
      - name: 'List All S3 Buckets'
        run: aws s3 ls
```
{% endraw %}

A quick note on the configuration of **role-session-name** above. When working in a complicated system or performing a lot of CI operations, it can get pretty painful trying to work out exactly which job was responsible for which change to your infrastructure. To that end it is a good idea to give each session a unique ID so it can be correlated back to the exact run that made a change. A lot of examples seem to suggest just setting this to "Github Actions" or something similar, but that can cause you some serious trouble when something gets deleted or broken and you need to try and pin down exactly how it happened!

If we try and execute this example, we can see that our _Role_ gets assumed fine and we can run _Jobs_ against AWS:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-1024x684.png">
  <figcaption>Bucket names hidden for privacy!</figcaption>
</figure>

As we can see in the image above, this all works just fine and cleans up the session cleans up after itself. With this nice arrangement we can get rid of the pain of pasting static secrets and limit ourselves to just thinking about the _Role ARN_. Simple!
