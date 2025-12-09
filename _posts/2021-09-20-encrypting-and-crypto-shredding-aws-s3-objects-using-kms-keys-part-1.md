---
title: "Encrypting and Crypto-Shredding AWS S3 Objects Using KMS Keys"
layout: post
date: 2021-09-20
categories: 
  - "aws"
  - "devops"
  - "security"
tags: 
  - "aws"
  - "cloud"
  - "devops"
  - "encryption"
  - "kms"
  - "s3"
  - "secops"
  - "security"
---

S3 seems to really rule the roost for cloud-based **[Object Storage](https://en.wikipedia.org/wiki/Object_storage)** and it's not really a surprise given how flexible it is; often seeing use as hosting for static websites, storing bulk analytics or logs or providing the storage backend for applications amongst many other uses.

As S3 content often needs to be presented to the public for anonymous access; the contents of a _Bucket_ are **not encrypted by default** and this can and does lead to all kinds of security and compliance disasters. In these posts we're going to take a look at some options for encrypting data within S3 using the AWS **KMS** service, how to _Crypto-shred_ that data by revoking or destroying our _KMS_ keys and how to give ourselves a grace period to back out in an emergency.

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01.png" class="scaled-img-75">

## Crypto-Shredding? What?

The term sounds cool right? Well I think so anyway, but what does it mean exactly? It's a phrase that seems to have popped up within InfoSec circles in recent years as a magic bullet to the right to be forgotten and has been thrown around like too much snake oil (and now I'm making it worse).

What's being implied by this colourful term is roughly that any data which is **[Encrypted at Rest](https://en.wikipedia.org/wiki/Data_at_rest#Encryption)** should be destroyable by destroying the specific _[**Encryption Key**](https://en.wikipedia.org/wiki/Key_\(cryptography\))_ with which it was originally encrypted, that sounds simple right.

That logic has existed for as long as encryption itself so personally I'm not convinced that definition goes far enough. If I'm going to call something _Crypto-Shredding_ I want to see the process of key destruction "shred" all of the backups of this same data within our system and have some means of targeting specific items within a wider dataset (such as lines of text or in our case, targeted objects in an _S3 Bucket_). As a PaaS service that interlocks with the other AWS PaaS services, KMS ticks these boxes nicely.

As an aside, there seems to be no wide consensus among InfoSec pros that I can find on the definition of "_Crypto-Shredding_", if anyone would like to correct me on that then please provide me with an authoritative source :).

## What Are We Working With?

So let's take a quick look at what we wan to be working with and the rough workflow:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-3.png)

1. Our _IAM Role_ **tinfoil-s3-role** will need a suitable _IAM Policy_ attached before it can make any requests to anything (we're using an existing _Role_ here but a _User_ will work just as well, if less scalable and secure).

2. Our _IAM Role_ makes a request for an object to our _S3 Bucket_ **tinfoil-bucket**

3. The request is evaluated against the _Bucket's_ own policy

4. The _Symmetric KMS Customer Managed Key_ **tinfoil-key**, used to encrypt the S3 objects being requested is called

5. The request is evaluated against the _KMS Key's_ own policy

6. Finally, the object is decrypted and returned to the requesting role

There's a lot that can go wrong in writing the various policies there and it's easily done so let's break it down. We'll create each component using the _AWS CLI_ and it's policy as we go along.

## Creating and Configuring a KMS Key and Policy

First, we'll define our _KMS CMK_ policy ahead of time in a file called **kmspolicy.json**. Since we know the ARN of our _IAM Role_ (which will be accessing the key) we can include it's name in the policy:

```json
#--kmspolicy.json

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "RootEmergencyAccess",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::123456789012:root"
            },
            "Action": "kms:*",
            "Resource": "*"
        },
        {
            "Sid": "GrantRootAccess",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::123456789012:role/tinfoil-s3-role"
            },
            "Action": [
                "kms:Get*",
                "kms:ReEncrypt*",
                "kms:List*",
                "kms:GenerateDataKey*",
                "kms:Encrypt",
                "kms:DescribeKey",
                "kms:Decrypt",
                "kms:Create*"
            ],
            "Resource": "*"
        }
    ]
}
```

A couple of things to take note of here.

1. The first statement is granting access to the account's **root user.** This is a good idea to explicitly define in-case you get something wrong and lock yourself out of your own key. Your root account should always let you back in but it's my experience that isn't always totally failsafe if you don't declare it!

2. The second statement is granting the **minimum possible permissions** to perform read/write permissions to S3 whilst using KMS encryption. For a full list of KMS policy _actions_ see the documentation **[here](https://docs.aws.amazon.com/kms/latest/developerguide/key-policies.html)**.

With this in place we can now create our key:

```
#--Create new KMS Key
aws kms create-key --policy file://kmspolicy.json
```

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-1.png">
  <figcaption>The key is created correctly, make a note of the KeyID and ARN, we'll need them later. Note that a Symmetric Key has been created with Encrypt/Decrypt Key Usage, as is the default. These are both necessary for our use as we need to both encrypt and decrypt data with the same key.</figcaption>
</figure>

```
#--Assign a human-readable alias to the KMS Key
aws kms create-alias \
    --alias-name alias/tinfoil-key \
    --target-key-id db37a7b8-e1f2-4b6f-99c9-100cc8b79017

```

## Creating and Configuring an S3 Bucket and Policy

With the KMS Key in place, we can get on to S3. As with KMS, let's define our _Policy_ first and save it as **bucketpolicy.json**. We're not going to go too wild here and the only thing we're going to do is enforce is HTTPS transport with a single statement and grant emergency access for our **root user** just as with KMS:

```json
#--bucketpolicy.json

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "RootEmergencyAccess",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::123456789012:root"
            },
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::tinfoil-bucket-example",
                "arn:aws:s3:::tinfoil-bucket-example/*"
            ]
        },
        {
            "Sid": "DenyNonHTTPSAccess",
            "Effect": "Deny",
            "Principal": {
                "AWS": "*"
            },
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::tinfoil-bucket-example",
                "arn:aws:s3:::tinfoil-bucket-example/*"
            ],
            "Condition": {
                "Bool": {
                    "aws:SecureTransport": "false"
                }
            }
        }
    ]
}
```

We will also create a second JSON file named **encryptionrules.json** which will define how KMS will be used to encrypt out _Bucket_:

```json
{
  "Rules": [
    {
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "db37a7b8-e1f2-4b6f-99c9-100cc8b79017"
      }
    }
  ]
}
```

Now to create and configure the Bucket:

```bash
#--Create Bucket. Set your region as appropriate. LocationConstraint config needed outside us-east-1 only!
aws s3api create-bucket --bucket tinfoil-bucket-example --region eu-west-2 --create-bucket-configuration LocationConstraint=eu-west-2
#{
#    "Location": "http://tinfoil-bucket-example.s3.amazonaws.com/"
#}

#--Block Public Access
aws s3api put-public-access-block \
    --bucket tinfoil-bucket-example \
    --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

#--Attach Bucket Policy
aws s3api put-bucket-policy --bucket tinfoil-bucket-example --policy file://bucketpolicy.json

#--Apply KMS Encryption
aws s3api put-bucket-encryption \
    --bucket tinfoil-bucket-example \
    --server-side-encryption-configuration file://encryptionrules.json
```

As we can see, everything is now in place with S3:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/04-2-1.png)

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/05-1.png)

## Creating and Configuring an IAM Role and Policy

Finally, with everything in place we can define our _IAM Policy_ as below in a file called **iampolicy.json**:

```json
#--iampolicy.json

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ReadWriteEncryptedS3",
            "Effect": "Allow",
            "Action": [
                "s3:*Object*",
                "s3:List*",
                "s3:DescribeJob",
                "kms:Encrypt",
                "kms:Decrypt",
                "kms:ReEncrypt*",
                "kms:GenerateDataKey*",
                "kms:DescribeKey"
            ],
            "Resource": [
                "arn:aws:kms:eu-west-1:123456789012:key/db37a7b8-e1f2-4b6f-99c9-100cc8b79017",
                "arn:aws:s3:::tinfoil-bucket-example",
                "arn:aws:s3:::tinfoil-bucket-example/*"
            ]
        }
    ]
}
```

There's a few things going on here:

1. The _actions_ grant the **minimum permissions** for Read/Write S3 access

2. Additional _actions_ are also granted for _KMS_, again allowing the **minimum permissions** for Read/Write S3 access for KMS encrypted objects

3. The _resources_ list is allowing access to both our _S3 Bucket_ and _KMS Key_

So let's create the policy and attach it to our role:

```bash
#--Create Policy
aws iam create-policy --policy-name tinfoil-s3-policy --policy-document file://iampolicy.json

#--Attach Policy to Role
aws iam attach-role-policy --policy-arn arn:aws:iam::123456789012:policy/tinfoil-s3-policy --role-name tinfoil-s3-role
```

## Temporarily Disabling File Access

So we have everything in place, let's upload a file to our _Bucket_ and confirm we can view it:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/06-1.png)

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/07-1.png">
  <figcaption>It's working!</figcaption>
</figure>

Now we'll disable our key, this acts as an effective revocation and should make the objects in the bucket inaccessible as we have no way to decrypt them:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/11.png)

A warning will be shown advising that this will block **any and all operations** using the key and you will have to confirm the action:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/12.png)

Now if we try to open our test image again, we aren't so fortunate. We're informed that our key is disabled:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/13-1.png)

This method offers us a very satisfying solution to key revocation, we can simply disable a key and then enable it again to restore access. In a situation of a potential security breach that turns out to be nothing this can save you an administration nightmare for no good reason.

This is all well and good if we're reacting to a possible security incident, but it won't get us very far if we're dealing with some day to day admin, say destroying some customer data that we're no longer entitled to hold, in these scenarios we'll need a proper method to _Crypto-Shred_ our data.

## Crypto-Shredding

To truly _Crypto-Shred_ our data, I.E. render it in such a state that it so it can never be accessed by anyone ever again, you can delete your keys outright.

Our key will need to be disabled first, once it is, we will be presented with an additional option to schedule your key for deletion:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/05.png)

In an effort to save you from yourself, you have to leave your key in a scheduled for deletion state for a **minimum** of 7 days (and a maximum of 30 days):

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/06.png)

Set your schedule period, tick the box to confirm that you want to delete the key automatically once the wait period is over and click **Schedule Deletion**. With this set the key will be destroyed when the wait is up.

This will render any data that has been encrypted with it inaccessible forever, whilst the objects will remain in S3 they will be functionally worthless and cannot be read by anyone, including by Amazon...so don't go doing this lightly. You wouldn't feed your documents in to a shredder unless you were sure!
