---
title: "Terraform and AWS - Provisioning HA NFS Shares with Elastic File System"
layout: post
date: 2021-03-17
categories: 
  - "automation"
  - "aws"
  - "devops"
tags: 
  - "automation"
  - "aws"
  - "cloud"
  - "devops"
  - "ec2"
  - "efs"
  - "integration"
  - "linux"
  - "nfs"
  - "terraform"
  - "ubuntu"
---

Recently I've been looking AWS' Elastic File Service platform, which allows for the provisioning of highly available PaaS storage which can accessed via NFS by multiple services at at very low cost. Whilst this is good, what's even better is templating and automating the provisioning. In this post we'll look at how to provision HA EFS storage using Terraform.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01.png)

## What Do We Want?

We have the option to create EFS volumes within a single AWS _Availability Zone_. This might do for a lab environment but isn't suitable for most real word scenarios where high availability is king so we won't even consider it. NFS is a versatile system but we'll just be looking at a simple connection from Ubuntu Linux on EC2.

What we need from our share is for it to be:

1. Accessible from all _VPC Subnets_ where we have _EC2 Instances_ running (we'll be using two, each in a different _Availability Zone_; **eu-west-1-az1** and **eu-west-1-az2**).
2. Accessible using a single DNS name, regardless of node location.
3. Remain available in the event of an outage of either _AWS Availability Zone_.

## Provisioning with Terraform

We'll be working an existing network roughly built in an earlier post [here](/infrastructure-as-code-multi-environment-continuous-deployment-with-terraform-and-bitbucket/), so I won't go in to the process of configuring the Terraform _backend_, we'll simply need a single Terraform _provider_ for AWS. Below are the few variables we'll use:

```bash
#--environment.auto.tfvars
environment_name    = "dev"
private_cidr        = "10.1.20.0/24"
private_cidr_2      = "10.1.21.0/24"
```

EFS is pretty easy to stand up, first we'll need to configure the EFS volume:

```terraform
#--Provision EFS Volume
resource "aws_efs_file_system" "tinfoil" {
    creation_token      = "tinfoilefs"
    encrypted           = true
    performance_mode    = "generalPurpose"
    throughput_mode     = "bursting"
    lifecycle_policy {
        transition_to_ia = "AFTER_90_DAYS"
    }
    tags = {
        Name        = "efs-${var.environment_name}"
        Environment = var.environment_name
    }
}
```

We're configuring a simple volume with a few advanced options configured:

- Native Encryption at Rest has been enabled, this is **not active by default**. Options for _KMS_ also exist if this is required.
- The volume has been configured **General Purpose** performance mode, with **Bursting** throughput. Other options are available for High IO or rate limited connections, however this configuration should suffice for most setups.
- A lifecycle policy to transfer objects to _Infrequent Access_ after 90 days.

Now that the _Volume_ is configured, we can create our _Mount Points_. A couple of points here:

- The _Availability Zone_ placement is inferred from the _ID_of the VPC Subnet that we pass to **subnet\_ID**.
- Our _EC2 Instances_ and _EFS Mounts_ must share **at least one of the same _Security Groups_.**

```terraform
#--Provision EFS Mounts
resource "aws_efs_mount_target" "tinfoil_az1" {
    availability_zones
    file_system_id      = aws_efs_file_system.tinfoil.id
    security_groups     = [aws_security_group.tinfoil_private.id]
    subnet_id           = aws_subnet.tinfoil_private.id
}

resource "aws_efs_mount_target" "tinfoil_az2" {
    file_system_id      = aws_efs_file_system.tinfoil.id
    security_groups     = [aws_security_group.tinfoil_private.id]
    subnet_id           = aws_subnet.tinfoil_private2.id
}
```

Applying this configuration we can now see that our EFS Volume is regionally available and encrypted:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02.png)

and has two available mount points:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03.png)

## Connecting From EC2

This is great, but how do we actually connect? Clicking the **Attach** button will present us with a few connection strings that we can use; in both DNS and IP flavours:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/04-1.png)

Using IP is ill-advised as it will lead to both configuration messes if it ever changes and is ill suited for use in other HA-applications. Your NFS connections will not survive an _AZ_ outage if connecting via IP and DNS should always be the preferred connection method as it will lookup the nearest IP for your _Availability Zone_ and fail over in the event of an outage:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/05-1024x334.png)

On our EC2 instances, we'll be installing the **nfs-common** package and using standard mounting as we're using Ubuntu Linux (the mount helper requires the **amazon-efs-utils** package which is available but needs to be built for Ubuntu), we'll also be creating a mount point at **/etc/efs**. From here we can finally mount the share:

```bash
sudo apt-get install nfs-common
sudo mkdir /mnt/efs
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timo=600,retrans=2,noresvport fs-a82ecb9c.efs.eu-west.1amazonaws.com:/ /mnt/efs
```

## Conclusion

That's the basics, but there's still a few points to cover. EFS has a few security holes out of the box which is uncommon for AWS; such as _anonymous_ and _root_ access being permitted by default. In the future I'll come back to hardening the configuration with proper security policies and ACLs, as with all systems the better the security the less trouble it's likely to cause later.
