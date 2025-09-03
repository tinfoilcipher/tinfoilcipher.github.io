---
title: "Hashicorp Vault - Enabling HA with AWS RDS and PostgreSQL"
layout: post
date: 2020-06-02
categories: 
  - "aws"
  - "devops"
  - "integrations"
  - "security"
tags: 
  - "aws"
  - "devops"
  - "integration"
  - "postgresql"
  - "rds"
  - "secops"
  - "secrets"
  - "security"
---

Vault offers an array of flexible storage backends with a view to providing a highly available storage location to store secrets, this is a great baked-in design choice as if you make Vault an integral part of your infrastructure you can ill afford a sudden outage, a perfect platform for storing structured data is, of course, a **RDBMS** (Relational Database Management System), as many of the mainstays are _scalable_ and support _clustering_ out of the box and in the age of Cloud providers **RDBMS**' are often provided as a PaaS or SaaS offering. In this example we'll be looking at using AWS **RDS** (Amazon Relational Database) running **PostgreSQL** as a viable HA backend for Vault.

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-10.png" class="scaled-img-50">

## Creating A PostgreSQL Instance on AWS RDS

First things first, we can't use a database that doesn't exist, so lets get one stood up, for the sake of understanding the process, lets use the AWS GUI. Within the RDS console we'll **Create** a new RDS _Instance_:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-9.png">
  <figcaption>We'll be using a Standard Create to get as many options as possible and using PostgreSQL at the default offered version of 11.6-R1, but there are no real differences between any versions when it comes to this part of deployment</figcaption>
</figure>

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-11.png">
  <figcaption>We'll now set up the Name and Master Password for the Instance</figcaption>
</figure>

Next we have the option to define **Storage** settings, these can be tailored to your requirements, for this setup I've stuck with the default settings, we can always come back and scale them later.

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/05-4.png">
  <figcaption>Availiability is critical, we always want our Instance to be available, so let's have a standby to ensure we don't go offline</figcaption>
</figure>

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/06-4.png">
  <figcaption>VPC ID obscured for security</figcaption>
</figure>

Next we define **Connectivity** to the _Instance_ and this has a couple of gotchas:

- First, unless you're sure the rest of your **VPC** is properly configured (and you're sure that you're 100% actually need to and are comfortable making the _Instance_ publicly available then don't do it). I would still err on the side of caution as this database is going to contain all of your secrets and do you really want that to be on the internet?
- Your VPC **must** have at least two **Subnets** created in at least two different **Availability Zones** ahead of time (your service wouldn't be very _highly available_ if it was all in one place after all)
- Even **if** you do make your _Instance_ publicly available, you will still need to permit TCP port **5432** (PostgreSQL) through your VPC's Security Group(s) or create a new one and **then** allow it
- The **VPC** that you select **MUST** be configured for both **DNS Resoltion** and **DNS Hostnames**. Failure to have these enabled will simply deny the creation of an _Instance_

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/07-3.png">
  <figcaption>Finally, we define Authentication methods, for this example we'll stick with basic Password authentication</figcaption>
</figure>

Once this is all completed we will see our _Instance_ is in a "creating" state, this takes a few minutes to complete:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/08-4-1024x130.png">
  <figcaption>Get comfortable...</figcaption>
</figure>

## Creating and Configuring the PostgreSQL Database

Now we have an _Instance_ but it doesn't actually contain a database that we can use, RDS isn't psychic after all and doesn't know that we intend to use this platform to host our Vault backend.

Before we can connect to the _Instance_, we need to know it's address, once the creation is complete we can click in to the _Instance_ and see the _endpoint_ details under **Connectivity & Security**, these are what we will use to allow any downstream system to connect:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/09-1.png">
  <figcaption>These are the details you've been looking for...</figcaption>
</figure>

The best way to reach in to the _Instance_ from here and do some damage is using **pgAdmin**, a free, cross-platform GUI for managing PostgreSQL which can be downloaded for free from [https://www.pgadmin.org/](https://www.pgadmin.org/).

Once downloaded, you will need to set up your connection back to your _Instance_:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/10-5.png)

All going well, you will be able to connect to your instance, if you can't Amazon provide a very helpful guide for troubleshooting connections [here](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ConnectToPostgreSQLInstance.html#USER_ConnectToPostgreSQLInstance.Troubleshooting), but there's a very high chance it comes down to security rules in your **VPC**.

Once connected, we'll need to complete a few tasks:

1. Create a new **Database** for Vault
2. Create the **KV Store**, **Schema** and **Indexes**
3. Create the **HAEnabled** backend
4. Create a **Service Account** so we aren't doing everything with the master password

All of these are done with SQL statements which we can execute directly against the _Instance_ within **pgAdmin**:

**Create New Database Named vault**:

```sql
CREATE DATABASE vault;
```

**Create KV Store, Schema and Indexes**:

```sql
CREATE TABLE vault_kv_store (
	parent_path TEXT COLLATE "C" NOT NULL,
	path        TEXT COLLATE "C",
	key         TEXT COLLATE "C",
	value       BYTEA,
	CONSTRAINT pkey PRIMARY KEY (path, key)
);

CREATE INDEX parent_path_idx ON vault_kv_store (parent_path);
```

**Create the HAEnabled backend**:

```sql
CREATE TABLE vault_ha_locks (
	ha_key                                      TEXT COLLATE "C" NOT NULL,
	ha_identity                                 TEXT COLLATE "C" NOT NULL,
	ha_value                                    TEXT COLLATE "C",
	valid_until                                 TIMESTAMP WITH TIME ZONE NOT NULL,
	CONSTRAINT ha_key PRIMARY KEY (ha_key)
);
```

**Create a Database Service Account and grant Full Access to vault Database** (substitute your own password as appropriate):

```sql
CREATE USER vaultadmin WITH PASSWORD 'supersecretpassword';
GRANT ALL PRIVILEGES ON DATABASE vault TO vaultadmin;
```

We can now see our new Database and Service Account are configured if we refresh **pgAdmin**:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/x.png)

## Configuring Vault

Now that our database is created and configured, we need to configured Vault to look at our new backend.

On the host where Vault is installed (in our case **mc-vault**) we will need to edit **/etc/vault.hcl** and change the **storage** _stanza_ to reflect that we wish to use **postgres** and provide a URL comprising of the **Service Account Credentials** and the **Database Endpoint**, a functional configuration is as below:

```terraform
storage "postgresql" {
        connection_url = "postgres://vaultadmin:supersecretpassword@vaultbackend.crfdefx7ec1z.eu-west-2.rds.amazonaws.com:5432/vault"
}

ui = true

listener "tcp" {
        address = "192.168.1.47:8200"
        tls_disable = 0
}

max_lease_ttl = "10h"
default_lease_ttl = "10h"
api_addr = "https://192.168.1.47:8200"
```

Finally we can simply restart the Vault service with **systemctl restart vault**. If we now browse to the Vault UI we can see a new backend has been loaded and is waiting for first time initialisation:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/11-6.png)
