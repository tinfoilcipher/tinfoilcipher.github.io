---
title: "Hashicorp Vault - Tokens and the REST API"
layout: post
date: 2020-04-13
categories: 
  - "devops"
  - "linux"
  - "security"
tags: 
  - "api"
  - "devops"
  - "linux"
  - "rest"
  - "secops"
  - "secrets"
  - "security"
  - "vault"
---

In my [last post](/hashicorp-vault-secure-installation-and-setup/) I covered the setup and hardening of Hashicorp's Vault platform, in this post I'll be looking at getting to grips with REST API and the Token authentication method. Tokens are core to the Vault authentication system, the platform is at it's heart designed to be interacted with programmatically by external systems over the API and the UI exists only to make the platform less bewildering for day to day administration.

## One Token So Far?

At the time of the initial setup, Vault delivered two things:

1. The master key for unsealing the Vault instance, split in to several segments
2. The root API token, at the time the only means of authenticating with the the platform in any way

When returning to the Vault UI to log in, you will be prompted for this token:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/1-1.png)

In a system designed for use by possibly thousands of external systems, we probably don't want a single token being used for all of this, let alone one that grants root access (though I'm sure plenty of you have seen such horror stories in the field, I know I have). Hashicorp clearly are aware of this, driving the point home with a warning as soon as you log in with a root token:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/2-1.png">
  <figcaption>Hashicorp: Security First, Saving you from yourself</figcaption>
</figure>

## ACL Policies - Granular Control

The language used in the warning is important, that we're using **A** root token, not the only one that can or will ever exist, this is a token linked to the **root** access policy, any such tokens will have unlimited access to the system. Root accounts can generate tokens, as can any child token which is correctly permissioned.

Logging in to the UI and browsing to the **Policies** tab, we can now see the two policies which exist by default, **root** and **default**. It is **CRITICAL** to understand that when a token is requested it will be added to the **default** policy **as well as** any defined policy if the unless the **no\_default\_policy** argument is set.

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/0-1-1-1024x318.png)

ACL Policies themselves are defined as HCL, to demonstrate the point I have cloned the content of the **default** policy a new one named **apiaccess** but made a single critical change regarding access to our new **kv** Secret Engine, so it can be used in another example, the content as represented in HCL is below and is fairly self explanatory:

```terraform
# Allow tokens to look up their own properties
path "auth/token/lookup-self" {
    capabilities = ["read"]
}

# Allow tokens to renew themselves
path "auth/token/renew-self" {
    capabilities = ["update"]
}

# Allow tokens to revoke themselves
path "auth/token/revoke-self" {
    capabilities = ["update"]
}

# Allow a token to look up its own capabilities on a path
path "sys/capabilities-self" {
    capabilities = ["update"]
}

# Allow a token to look up its own entity by id or name
path "identity/entity/id/{{identity.entity.id}}" {
  capabilities = ["read"]
}
path "identity/entity/name/{{identity.entity.name}}" {
  capabilities = ["read"]
}

# Allow a token to look up its resultant ACL
path "sys/internal/ui/resultant-acl" {
    capabilities = ["read"]
}

# Allow Lease renewal
path "sys/renew" {
    capabilities = ["update"]
}
path "sys/leases/renew" {
    capabilities = ["update"]
}

# Allow looking up lease properties.
path "sys/leases/lookup" {
    capabilities = ["update"]
}

# Allow a token to manage its own cubbyhole
path "cubbyhole/*" {
    capabilities = ["create", "read", "update", "delete", "list"]
}

# Grant Read Only access to the KV Secret Engine and all Secrets
path "kv/*" {
  capabilities = ["read", "list"]
}

# Allow a token to wrap arbitrary values in a response-wrapping token
path "sys/wrapping/wrap" {
    capabilities = ["update"]
}
path "sys/wrapping/lookup" {
    capabilities = ["update"]
}

# Grant access to sys functions
path "sys/wrapping/unwrap" {
    capabilities = ["update"]
}
path "sys/tools/hash" {
    capabilities = ["update"]
}
path "sys/tools/hash/*" {
    capabilities = ["update"]
}
path "sys/control-group/request" {
    capabilities = ["update"]
}
```

The critical addition can be seen on lines 53-55 where we are granting explicit **Read** and **List** permissions to the path **kv/\*** this allows read only and list permissions for all Secrets within the Secrets Engine "kv" for any tokens linked to this policy. NOTE, without adding the **List** permission we can query a single secret but we first need to know it's name, List allows us to see the Secrets both over the API and the UI.

A full breakdown of the ACL syntax is broken down in detail within the Vault [documentation](https://www.vaultproject.io/docs/concepts/policies).

## API Basic Structure

The Vault REST API is well [documented](https://www.vaultproject.io/api-docs), API routes are prefixed with http(s)://hostname:port/v1/ and followed by the endpoint as defined in the documentation.

In our **vault.hcl** we have explicitly defined the API URL via the **api\_addr** variable being set to **https://192.168.1.43:8200**, however were this not set it would fallback to the default TCP listener address of the Vault instance.

## Requesting New Tokens

Now that an appropriately policy exists (which isn't root) we can set about issuing tokens.

Commands exist within the Vault CLI to generate new tokens, but we're here to use the API so let's use the API to generate the new token over the API. I'm going to be using CURL, but Postman works just as well. Authentication is sent in a header Key Vault Pair with the token being set as the Key **X-Vault-Token**.

```json
// Payload example, saved as newtoken.json
{
  "policies": ["apiaccess"],
  "ttl": "10h",
  "renewable": true,
  "no_default_policy": "true"
}
```

```bash
# CURL POST to request new token from the /auth/token/create endpoint
# Using existing root token in the X-Vault-Token header
curl \
> --header "X-Vault-Token: YOUR_ROOT_TOKEN_HERE" \
> --request POST \
> --data @newtoken.json \
> https://mc-vault.madcaplaughs.co.uk:8200/v1/auth/token/create
```

```json
// API Return Data - Newly Minted Token is returned as client_token
{  
    "request_id":"a65ee8f9-c0e8-5c7c-6f44-6afa93facda4",
    "lease_id":"",
    "renewable":false,
    "lease_duration":0,
    "data":null,
    "wrap_info":null,
    "warnings":null,
    "auth":{
        "client_token":"---YOUR_NEW_TOKEN---",
        "accessor":"---ACCESSOR-ID---",
        "policies":[
            "apiaccess",
        ],
    "token_policies":[
        "apiaccess",
        ],
    "metadata":null,
    "lease_duration":36000,
    "renewable":true,
    "entity_id":"",
    "token_type":"service",
    "orphan":false
    }
}
```

## Something to Query

Now that we have a token to authenticate with, I've created a secret in the KV (Key/Value) Secrets Engine. KV is a Key Value database which is used to store secrets as lists containing any number of Key Value Pairs, as of the Version 2 release of KV, Secrets now support native version control, so the API structure differs significantly to handle filtering secret versions.

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/4-1-1024x226.png">
  <figcaption>The Secret secret1 is stored in a KV Secret Engine, it contains two Key Value Pairs named username and password.</figcaption>
</figure>

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/5-1024x148.png">
  <figcaption>If we view the Secret as raw JSON, we can see both the structure and the plaintext in UI, as we have already unsealed the Vault.</figcaption>
</figure>

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/6-1.png">
  <figcaption>Looking at the Version drop down, we can also see that the Secret has only a single version and that we're looking at it.</figcaption>
</figure>

## Querying Secrets

Having all of this information, we can construct our query, again I'm going to use CURL. The full documentation for KV (Version 2) can be found in the Vault [documentation](https://www.vaultproject.io/api-docs/secret/kv/kv-v2). For our purposes of viewing secrets, the endpoint is **/<SECRET\_ENGINE>/config** (where our Secret Engine is named **kv** this will be **/kv/config** but some additional filtering will need to be applied to select both the secret and the version:

```bash
# CURL GET to request secret data from /kv/config endpoint
# specifying the secret as secret1 and version number 1
curl \
>  --header "X-Vault-Token: YOUR_TOKEN_HERE" \
>  https://mc-vault.madcaplaughs.co.uk:8200/v1/kv/data/secret1?version=1
```

```json
// API Return Data - Specified Secret Data
{
    "request_id":"f1fc5533-ac6a-6368-3a25-c9f697973441",
    "lease_id":"",
    "renewable":false,
    "lease_duration":0,
    "data":{
        "data":{
        "password":"S3cr3tC0de",
        "username":"welsh"
},
"metadata":{
    "created_time":"2020-04-10T16:33:22.624292607Z",
    "deletion_time":"",
    "destroyed":false,
    "version":1
}
},
    "wrap_info":null,
    "warnings":null,
    "auth":null
}
```

Next I'll be looking at some integrations with other systems :)
