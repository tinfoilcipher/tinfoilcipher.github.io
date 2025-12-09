---
title: "Creating a Private Helm Repo with Chartmuseum using AWS S3 and EC2"
layout: post
date: 2021-04-26
categories: 
  - "aws"
  - "containers"
  - "devops"
  - "linux"
tags: 
  - "aws"
  - "chartmuseum"
  - "cloud"
  - "devops"
  - "ec2"
  - "helm"
  - "kubernetes"
  - "linux"
  - "microservices"
  - "s3"
  - "ubuntu"
---

**[Helm](https://helm.sh)** is an incredibly popular package manager for Kubernetes, however despite it's incredibly widespread use there isn't a huge amount of information or options out there for creating private repositories using Open Source platforms. **[Chartmuseum](https://chartmuseum.com)** seeks to solve this problem by offering us just that. In this post I'm looking at how to deploy and bootstrap Chartmuseum on Ubuntu Linux 18.04, using a secure _AWS S3_ backend.

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-3.png">
  <figcaption>A Private, Open Source option</figcaption>
</figure>

## Getting Started

Chartmuseum is shipped as a single binary, getting this running on traditional compute is going to mean bootstrapping by means of creating a service, the documentation is great for Chartmuseum but doesn't really cover how to go about getting this running.

For the sake of brevity, we will assume that we already have the following set up:

- An Ubuntu 18.04 Server named **mc-chartmuseum** which will host our Chartmuseum deployment
- An AWS _S3 Bucket_ named **tinfoil-chartmuseum** which is:
    - Not publicly accessible
    - Encrypted at rest
    - Has versioning enabled
- An _AWS IAM Service Account_ created, with a single policy attached (see below), and an **Access Key ID** and **Secret Access Key** exported

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowListObjects",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::tinfoil-chartmuseum"
    },
    {
      "Sid": "AllowObjectsCRUD",
      "Effect": "Allow",
      "Action": [
        "s3:DeleteObject",
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::tinfoil-chartmuseum/*"
    }
  ]
}
```

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-6.png)

## Installation

Since we're working with a single binary we can just pull down the latest version from the Chartmuseum releases and configure it:

```bash
#--Download latest Chartmuseum binary
curl -LO https://s3.amazonaws.com/chartmuseum/release/latest/bin/linux/amd64/chartmuseum

#--Make the binary executable
chmod +x chartmuseum

#--Move binary to executable location
sudo mv chartmuseum /usr/bin/
```

Running **chartmuseum** from the shell should now return auto complete commands, but there is still much more to do, however we can now verify that Chartmuseum is being detected in the system $PATH by running **chartmuseum –version**

```bash
chartmuseum --version
# ChartMuseum version 0.12.0 (build 101e26a)
```

For authentication we'll also need the AWS CLI, so let's get that installed too:

```bash
#--Update apt cache and install awscli
apt-get update
apt-get install awscli
```

## Service Accounts - Security First

Now that we have Chartmuseum installed, we should make a secure base to work from. We'll be creating a Service Account named **chartmuseum** that our new service will run as. To secure the process we’re going to make this user non-accessible by assigning the _nologin_ shell (removing the ability to use the STDIN shell and hardening further):

```bash
# Create user "chartmuseum", -r defines this as a system account,
# -s defines the shell as /bin/nologon
sudo useradd -r -s /bin/nologin chartmuseum
```

## Configuring Chartmuseum

Now that a service account is in place we can begin the process of bootstrapping of Chartmuseum. First we will need to create an _EnvironmentFile_ which we will use to pass our parameters to _systemd_ when creating a _service_.

First, let's create the _EnvironmentFile_:

```bash
# Create a new EnvironmentFile for Chartmuseum
sudo nano /etc/chartmuseum.config
```

...and paste in the configuration below:

```bash
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
ARGS=\
--port=8080 \
--storage="amazon" \
--storage-amazon-bucket="tinfoil-chartmuseum" \
--storage-amazon-prefix="" \
--storage-amazon-region="eu-west-2" \
--basic-auth-user=admin \
--basic-auth-pass=mypassword \
--auth-anonymous-get
```

Be sure to substitute the **AWS\_ACCESS\_KEY\_ID**, **AWS\_SECRET\_ACCESS\_KEY** values as relevant, as well as the other configuration values and save your file with **CTRL + O** and exit with **CTRL+X**.

HTTP basic authentication is enabled using the credentials defined in both **basic-auth-user** and **basic-auth-password**. The other settings are fairly self explanatory, we're running Chartmuseum on HTTP port 8080, and connecting to our previously mentioned S3 bucket as a backend.

**NOTE**: This configuration will allow **anonymous GET requests** from your repository, to disabled this remove the **\--auth-anonymous-get** parameter from the **chartmuseum.config** file.

With this file created, we should also set it's permissions on this file to ensure it is only accessible by our _Service Account_:

```bash
#--Set User Ownership
sudo chown chartmuseum /etc/chartmuseum.config

# Set file system permissions, mode 640
sudo chmod 700 /etc/chartmuseum.config
```

## Daemonising With systemd

Now that we have a running system and a valid configuration, we should create a **systemd** Unit Service in order that Chartmuseum can be stopped and started as a service, critically, by our service account.

First, let's create the _Unit File_:

```bash
# Create a new unit file for the new chartmuseum.service Unit Service
sudo nano /etc/systemd/system/chartmuseum.service
```

...and paste in the configuration below:

```ini
[Unit]
Description=Chartmuseum Service
Documentation=https://chartmuseum.com/
Requires=network-online.target
After=network-online.target

[Service]
User=root
EnvironmentFile=/etc/chartmuseum.config
ExecStart=/usr/sbin/chartmuseum $ARGS
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

The **EnvironmentFile** argument is looking up our _EnvironmentFile_ created in the previous step and the **ExecStart** argument is defining the binary that our service will use as well as which parameters it should execute with ( again, using the values defined in our _EnvironmentFile_. Save this file **CTRL+O** and exit with **CTRL+X**.

We can now verify if the service runs as expected:

```bash
# Reload systemd daemon
sudo systemctl daemon-reload

# Start chartmuseum service
sudo systemctl start chartmuseum.service

# Verify status
sudo systemctl status chartmuseum.service

# chartmuseum.service - Chartmuseum Service
#  Loaded: loaded (/etc/systemd/system/chartmuseum.service; disabled; vendor preset: enabled)
#  Active: active (running) since Mon 2021-04-19 19:38:31 UTC; 1s ago
#    Docs: https://chartmuseum.com/
# Main PID: 3267 (chartmuseum)
#   Tasks: 4 (limit: 4659)
#  CGroup: /system.slice/chartmuseum.service
```

## Accessing and Using Chartmuseum

Now that the service is up and running we should see the service up and running if we try and access it in a web browser:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03.png">
  <figcaption>Up and running, but that port and certificate aren't great...we'll get to that</figcaption>
</figure>

We can now add the repository using the Helm:

```bash
#--Add Repository, substitute username and password as defined in your earlier configuration
helm repo add mc-chartmuseum --username admin --password mypassword http://mc-chartmuseum.madcaplaughs.co.uk:8080
# "mc-chartmuseum" has been added to your repositories

#--List Repositories
helm repo list
# NAME          	URL                                          
# mc-chartmuseum	http://mc-chartmuseum.madcaplaughs.co.uk:8080
```

Chartmuseum comes with [**a documented API for interaction**](https://chartmuseum.com/docs/), but I'm going to push a chart using the **[Helm Push plugin](https://github.com/chartmuseum/helm-push)** to add a test chart from my system:

```basha
#--Install Helm Push Plugin
helm plugin install https://github.com/chartmuseum/helm-push.git

#--Push Test Chart
helm push testchart mc-chartmuseum
# Pushing testchart-0.1.0.tgz to mc-chartmuseum...
# Done.
```

## Verification and Further Steps

If we look at the contents of our _S3_ backend we can see that the package has indeed been uploaded:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/04-1.png">
  <figcaption>description</figcaption>
</figure>

If we verify on the shell, we will also see that our Chart is present in the repository:

```bash
#--Search the contents of our repository
helm serach repo mc-chartmuseum

# NAME                                 	CHART VERSION	APP VERSION            	DESCRIPTION  
# testchart                             0.1.0                                   Test chart that serves to purpose
```

Our next step is going to be to do something about that lack of TLS and serving on an insecure port which we'll address by using an [NGINX Reverse Proxy in the next post](h/chartmuseum-helm-repository-reverse-proxy-with-nginx/).
