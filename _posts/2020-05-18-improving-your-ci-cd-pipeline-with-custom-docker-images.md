---
title: "Improving Your CI/CD Pipeline with Custom Docker Images"
layout: post
date: 2020-05-18
categories: 
  - "aws"
  - "containers"
  - "devops"
tags: 
  - "ansible"
  - "automation"
  - "aws"
  - "bitbucket"
  - "ci-cd"
  - "cloud"
  - "containers"
  - "devops"
  - "docker"
---

Previously we looked at [**implementing a CI/CD pipeline using both Terraform and Ansible for provisioning and Configuration Management**](/2020/05/13/bitbucket-terraform-and-ansible-extending-infrastructure-ci-cd-in-to-configuration-management/). In this deployment we relied on an official Python Docker image to build our Ansible environment, however this required a few steps that add a few top-heavy steps that could be solved by creating our own Docker image instead.

The sample code for this post is in my GitHub **[here](https://github.com/tinfoilcipher/ansible-aws-docker-image)**.

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-3.png" class="scaled-img-50">

## Speeding up Delivery

When we're talking about _Continuous Delivery_ with containers, there are a couple of factors that can be overlooked:

- **Size Matters**: Containers should be kept as lightweight as possible, including only the essential binaries, modules and libraries that are actually needed.
- **Speed Matters**: This comes down to seconds, if you have a choice of using a container that delivers your build in 60 seconds and one that delivers in 30 seconds, you should use the one that delivers in 30 seconds, if you're doing dozens of deliveries **those seconds are going to add up**.

Our initial delivery has a few flabby issues that we can improve on:

- We're using a full Python container, which is **262MB** in size.
- We load in additional Python modules using **pip** after the container is started, increasing the size of the container further and slowing the build down by a few more seconds (this is improved in future runs by the use of a cache).

We'll overcome these with our newly created image, but first we'll need to design it, build it and get it implemented in our pipeline.

## Docker - Defining a New Image

So how do we get a new image in the first place? We have the source code for our **Dockerfile** in GitHub, we can close this repo and use the **Dockerfile** to mint our new image, first lets take a look at the file contents:

```docker
FROM alpine:latest

WORKDIR /opt/app
COPY requirements.txt /opt/app/requirements.txt

RUN apk add --update \
		git \
		gcc \
		bash \
		openssh \
		python2 \
		py-setuptools \
		py-pip \
		python2-dev \
		musl-dev \
		libffi-dev \
		openssl-dev

RUN pip install -U setuptools
RUN pip install -r requirements.txt

ENTRYPOINT ["ansible"]
```

The **Dockerfile** contains a list of _declarative_ statements that **docker** will use to build the image. Let's take a look at them.

- **FROM**: This states the _base image_ that our image will be based on. In our case we're using the [Alpine Linux](https://alpinelinux.org/) image which is a wonderful 2.5MB Linux Distro popular for containers.
- **WORKDIR**: This defines our working directory where executions will take place
- **COPY**: We're using this to copy in our **requirements.txt** file which we're using to define the Python packages we want to install with **pip**
- **RUN**: These are commands to execute on the shell, we're using two commands here, **apk** which is the Alpine package manager and **pip** which is the Python package manager
- **ENTRYPOINT**: This defines the container will run as the **ansible** package executable and allow us to interact with Ansible packages

For clarity, below is the **requirements.txt**, it hasn't changed from the one we've used previously:

```bash
ansible>=2.9.7
boto>=2.49.0
boto3>=1.13.6
botocore>=1.16.6
```

## Docker - Building a New Image

Before we do anything we first need to authenticate with our **Docker Hub** account on the shell:

```bash
sudo docker login --username=tinfoilcipher
Password: ***********************

Login Succeeded
```

Now that we understand how the image is actually defined in code, we need to build it in to a functional image. To do this we'll clone the repository containing the **Dockerfile** and **requirements.txt** and build the image using **docker build**:

```bash
#--Clone the git Repository
mkdir ~/docker
cd ~/docker
git clone git@github.com:tinfoilcipher/ansible-aws-docker-image.git
# Cloning into 'ansible-aws-docker-image'...
# remote: Enumerating objects: 15, done.
# remote: Counting objects: 100% (15/15), done.
# remote: Compressing objects: 100% (12/12), done.
# remote: Total 15 (delta 1), reused 0 (delta 0), pack-reused 0
# Receiving objects: 100% (15/15), done.
# Resolving deltas: 100% (1/1), done.

#--Build the Docker Image
cd ~/docker/ansible-aws-docker-image
sudo docker build --tag tinfoilcipher/ansible-aws:release .
```

The **docker build** command has been executed using the **tag** **tinfoilcipher/ansible-aws:release**. This corresponds to a **Public Docker Hub Repository** that has already been created and we'll be appending the image with the tag _release_, the "." defines that we'll be using the contents of the current directory to build the image.

Once we execute this last command we will see the build process begin:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-3.png">
  <figcaption>docker build initiated, Alpine is download as the base (Step 1), apk is used to install dependencies for our Python packages (Step 4)</figcaption>
</figure>

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/04-2.png">
  <figcaption>docker build continues, pip is used to install Python packages (Step 6)</figcaption>
</figure>

## Docker - Pushing the Image to Docker Hub

Now the image is built, but it doesn't exist anywhere outside of our local system, we need it to be available on Docker Hub in order for our pipeline to be able to gain access (technically any container repository would work, but I'm using Docker Hub).

Let's take a look at getting the image up to Docker Hub:

```bash
#--Verify that image is present
sudo docker images
# REPOSITORY                  TAG                 IMAGE ID            CREATED              SIZE
# tinfoilcipher/ansible-aws   release             c4e4376668b3        About a minute ago   418MB
# alpine                      latest              f70734b6a266        2 weeks ago          5.61MB

sudo docker push tinfoilcipher/ansible-aws:release
# The push refers to repository [docker.io/tinfoilcipher/ansible-aws]
# f58517ef3644: Pushed 
# a3f7f2db445e: Pushed 
# ea48c900d1d0: Pushing [==>                                                ]  8.881MB/205MB
# ea48c900d1d0: Pushing [=========>                                         ]  38.77MB/205MB
# ea48c900d1d0: Pushed 
# 8c05aeb2cd2b: Pushing [======================>                            ]  87.64MB/198.4MB
# 8c05aeb2cd2b: Pushed 
# 3e207b409db3: Pushed 
# release: digest: sha256:d6b540a90588058ad072d573ac2b3032082c237ec41ee495579b9de2a4cb1e63 size: 1577
```

Note that the image that we **push** must match the **user** (tinfoilcipher), **image name** (ansible-aws) and **tag** (release) in the exact format conforming to a Docker _namespace_ otherwise the push will fail.

If we now look at Docker Hub, we can see that our image is present:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/10-1-1024x323.png">
  <figcaption>ansible-aws image: Now available for public use</figcaption>
</figure>

Heavy compression is applied during the **docker push** process, and if we compare this against the official Python image we were using earlier, we can see a significant reduction in size:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/12-4.png)

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/11-3.png)

## Implementing in the Pipeline

Finally, now that the image is present, we can modify our deployment pipeline, since we are no longer using **pip** in the deployment pipeline we can remove the cache and remove the requirement for the in-line **pip** installations, below is an extract showing the new _step_ in our BitBucket deployment pipeline:

```yaml
#--Extract from bitbucket-pipelines.yml
...
- step:
    name: Ansible Configuration
    image: tinfoilcipher/ansible-aws:release
    script:
      - ansible-playbook configure-ec2.yml
...
```

We are now using our custom image and this change to the **bitbucket-pipelines.yml** file will cause a new pipeline execution, as we see below, we see a time gain of just under 50% over the previous execution for the **Configure Ansible** step:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/13-1.png">
  <figcaption>Left: Build using Python Official<br/>Right: Build using custom ansible-aws</figcaption>
</figure>
