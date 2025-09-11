---
title: "OpenTofu – Configuring State Encryption in AWS with KMS"
layout: post
date: 2025-09-11
categories: 
  - "aws"
  - "devops"
  - "integrations"
tags: 
  - "devops"
  - "integration"
  - "s3"
  - "opentofu"
  - "tofu"
  - "encryption"
  - "security"
  - "secops"
---

One of the problems that's been present in [Terraform](https://developer.hashicorp.com/terraform) for a while is that any secrets that make there way in to your state remain visible in the clear. The state is, after all, just a big JSON document. There is an argument to be made that secrets shouldn't really be in the state, but this is at the mercy of both the engineers and developers using your Terraform implementation and the compromises that might need to be made for all kinds of reasons, usually stemming from legacy software and messy integrations. Here in the real world, secrets often to find their way in to the state file and we rely on the encryption at rest functionality of the storage volume holding our states as the only means of security. Meaning that the storage volume is a goldmine for a potential attacker.

Over the years I have seen solutions to this problem designed by security minded admins using PGP, but these are all quite brittle and PGP is a pretty unfriendly tool. Luckily [OpenTofu](https://opentofu.org) (the open source fork of Terraform) has solved this problem with native state-level encryption that plugs in to several of the major cloud players. The [official docs on this topic](https://opentofu.org/docs/language/state/encryption/) are good but pretty dense, so in this post I'll be looking at how to get set up using a standard AWS S3 backend.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01.png)

## What Are We Working With?

For the purposes of these examples, I'm not going to cover the creation of a KMS key, we'll just assume that one has already been created. You will need a **symmetric** KMS key. Whether or not your create the key material via KMS or some external source is up to you, but this does not affect how the key will interact with *tofu*.

## Creating a New tofu Configuration with State Encryption

A basic configuration should look like this:

```terraform
terraform {
  backend "s3" {
    bucket = "tfc-tofu-example-states-11-09-2025"
    key    = "example01.tfstate"
    region = "eu-west-2"
  }

  encryption {
    key_provider "aws_kms" "tfc" {
      kms_key_id = "3abf6123-6adf-4813-952b-faf3baac371f3" #--Replace with your KMS key ID
      region     = "eu-west-2"
      key_spec   = "AES_256" #--Standard for a KMS generated key, replace if you provided your own key material
    }

    method "aes_gcm" "tfc" {
      keys = key_provider.aws_kms.tfc #--From key provider above
    }

    state {
      method = method.aes_gcm.tfc #--From method above
    }
  }
}
```

This basic configuration will ensure that any resources we create are encrypted. So let's create a resource:

```terraform
resource "random_password" "example_password" {
  length  = 32
  special = true
}

output "example_password" {
  value     = random_password.example_password.result
  sensitive = true
}
```

```bash
tofu init
# ...
# OpenTofu has been successfully initialized!
# ...
tofu apply
#
# Plan: 1 to add, 0 to change, 0 to destroy.
#
# Do you want to perform these actions?
#  OpenTofu will perform the actions described above.
#  Only 'yes' will be accepted to approve.
#
#  Enter a value: yes
#
# random_password.example_password: Creating...
# random_password.example_password: Creation complete after 0s [id=none]
#
# Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
#
#Outputs:
#
#example_password = <sensitive>
```

So far so normal, that was painless. As we see, most things which are explicitly marked as secrets are hidden from the **console**, but they are still in the state. If we try and read the state we can see the value:

```bash
tofu output example_password  
")slWtpL:-eY%#]M1l}nRsDBS()B7I5wy"
```
However, if we investigate the actual raw JSON of our state file from the backend without authentication, we will see that it is encrypted, with the entire contents of the state file being represented as the **encrypted_data** key and then base64 encoded.

```json
{
  "serial": 2,
  "lineage": "1c1586af-d0be-8d12-d6af-e43826dbafd5",
  "meta": {"key_provider.aws_kms.tfc": "eyJjaXBoZXJ0ZXh0X2Jsb2IiOiJ4NkgxVDB4MXJHY0RUdlhmYWtJd1pwSmNvejBpampLTnVYdFRXVTlxUmJoQU10ZGJmWUR3UzZzS2hNV2E4TUhQM3B5aFhRWkRYTkRVRT..."}, //--Shortened for brevity
  "encrypted_data": "aYrN+3b2y6kqOcNegRAvpMmwOaLEqOM6pDG4m9YC...", //--Shortened for brevity
  "encryption_version": "v0"
}
```

## What About Existing Configurations?

This is all good and well if you're starting from scratch, but what about turning on encryption for an existing project? That requires another hoop to jump through, the existing state first needs to be encrypted and then handed off in to encryption mode. If you try to just turn on encryption you will see the error below:

```bash
tofu apply
#╷
#│ Error: error loading state: encountered unencrypted payload without unencrypted method configured
#│ 
```

Too migrate to an encrypted state, configure the main *terraform* config as below:

```terraform
terraform {
  backend "s3" {
    bucket = "tfc-tofu-example-states-11-09-2025"
    key    = "example01.tfstate"
    region = "eu-west-2"
  }

  encryption {
    key_provider "aws_kms" "tfc" {
      kms_key_id = "3abf6123-6adf-4813-952b-faf3baac371f3" #--Replace with your KMS key ID
      region     = "eu-west-2"
      key_spec   = "AES_256" #--Standard for a KMS generated key, replace if you provided your own key material
    }

    method "unencrypted" "migration" {} #--Additional unencrypted method for migration

    method "aes_gcm" "tfc" {
      keys = key_provider.aws_kms.tfc
    }

    state {
      method = method.aes_gcm.tfc
      fallback {
        method = method.unencrypted.migration #--Fallback method to allow unencrypted state operations
      }
    }
  }
}
```

With this in place, run an *apply* operation, you will see a warning that you should **ONLY** use this configuration for a migration. To avoid any possible state corruption, do not make any *resource* or *data source* changes at the same time:

```bash
tofu apply
# ╷
# │ Warning: Unencrypted method configured
# │ 
# │ Method unencrypted is present in configuration. This is a security risk and should only be enabled during migrations.
# ╵
# No changes. Your infrastructure matches the configuration.
# 
# OpenTofu has compared your real infrastructure against your configuration and found no differences, so no changes are needed.
# 
# Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
```

With this complete, remove the below stanzas from your *terraform* configuration and run another *apply*:

```terraform
method "unencrypted" "migration"
```

```terraform
  ...
    fallback {
      method = method.unencrypted.migration
    }
  ...
```

The state is now encrypted and can be used as normal.

## Turning Off Encryption

Finally, if you need to turn off state encryption, you will still need to have access to your key and need to roughly go through the process above in reverse using the **unencrypted** method as shown below:

```terraform
terraform {
  backend "s3" {
    bucket = "tfc-tofu-example-states-11-09-2025"
    key    = "example01.tfstate"
    region = "eu-west-2"
  }

  encryption {
    key_provider "aws_kms" "tfc" {
      kms_key_id = "3abf6123-6adf-4813-952b-faf3baac371f3" #--Replace with your KMS key ID
      region     = "eu-west-2"
      key_spec   = "AES_256" #--Standard for a KMS generated key, replace if you provided your own key material
    }

    method "unencrypted" "migrate" {} #--Unencrypted method, used to migrate away from an encrypted state

    method "aes_gcm" "tfc" {
      keys = key_provider.aws_kms.tfc #--The key should remain unchanged
    }

    state {
      method = method.unencrypted.migrate #--The PRIMARY method used for the state should now be unencrypted
      fallback {
        method = method.aes_gcm.tfc #--The fallback method should now be the encryption method that is being migrated away from
      }
    }
  }
}
```

With this configuration in place, run an **apply** to remove state encryption. Do not make any *resource* or *data source* changes at the same time to avoid state corruption:

```bash
tofu apply
...
# No changes. Your infrastructure matches the configuration.
#
# OpenTofu has compared your real infrastructure against your configuration and found no differences, so no changes are needed.
#
# Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
```

If you have no plans to return to an encrypted state, the encryption configuration can be fully removed from your configuration, I.E.:

```terraform
terraform {
  backend "s3" {
    bucket = "tfc-tofu-example-states-11-09-2025"
    key    = "example01.tfstate"
    region = "eu-west-2"
  }
}
```

## DR Considerations

Finally it should be pointed out that whilst this is all very secure, if you lose access to your keys you will be completely without recourse to access your states and will be forced to either re-import all of your resources from scratch (which is an **agonising** process for large estates) or fall back on some other kind of DR process. So before steaming in to turning this functionality on, be sure to consider exactly:

- How available and restorable your keys are
- How you will rotate them if one becomes compromised
- How you can restore one if one becomes disabled or deleted
