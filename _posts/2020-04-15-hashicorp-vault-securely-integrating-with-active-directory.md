---
title: "Hashicorp Vault - Securely Integrating with Active Directory"
layout: post
date: 2020-04-15
categories: 
  - "devops"
  - "integrations"
  - "security"
  - "windows"
tags: 
  - "activedirectory"
  - "devops"
  - "integration"
  - "secops"
  - "secrets"
  - "security"
  - "vault"
  - "windows"
---

Even in the age of Linux dominance on public clouds, there's no denying that Windows still rules the roost in on-premise deployments and Active Directory still lies at the heart of authentication schemes. AD is everywhere to the point where it's a surprise for some admins to learn that LDAP and Kerberos aren't native to Microsoft. Knowing that, it is often essential for a good product to provide LDAP authentication with a view to being compatible with Active Directory, as it turns out, Vault has just that facility and have gone so far as to [documenting some specific use cases](https://www.vaultproject.io/docs/auth/ldap).

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/0-3-1024x248.png)

## Active Directory - Considerations

When leafing though the specs of the Vault LDAP implementation, it specifies that they've conformed to [RFC4514](https://tools.ietf.org/html/rfc4514) which defines LDAP namespaces, it is therefore down to the **end user** to handle any escape characters that they have inadvertently introduced to Active Directory (incidentally, Microsoft don't conform to the RFC, thanks Microsoft), [this TechNet post](https://social.technet.microsoft.com/wiki/contents/articles/5312.active-directory-characters-to-escape.aspx) provides a good breakdown of escape characters in Active Directory and handling them.

As I rule I tend to avoid strange characters and even spaces where possible in Active Directory designs as they only introduce potential errors later in the day. Below is the structure of AD that we'll be working with:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/3-2.png)

The key objects in Active Directory that we'll be working with are:

- **madcaplaughs.co.uk/_MCL/Group_Objects/Admin_Groups/VaultAdmins** - A Global Security Group which will contain users who can access Vault
- **madcaplaughs.co.uk/_MCL/User_Objects/Service_Accounts/SVC_VaultLDAP** - An LDAP Binding Service Account which Vault will use to create a bind with Active Directory
- **madcaplaughs.co.uk/_MCL/User_Objects/Administrators/Andy Welsh** - My own domain account to be used for authentication with Vault with the samAccountName **welsh**

We'll also be using the **apiaccess** policy created in the [REST API](/hashicorp-vault-tokens-and-the-rest-api/) post.

## Active Directory - Security First

I have Active Directory configured using LDAPS (LDAP over SSL, actually TLS, bad name for a protocol), external integration with Active Directory is not very secure out of the box and LDAP when not secured sends all of it's traffic in the clear, so if you're serious about doing this right, you don't want to be working with normal LDAP.

This setup of LDAPS has been covered ad nauseam in better articles than mine and is pretty well understood now so I see no reason to recount it, a guide I've used myself in the past however (and which is very good) can be found [here](https://techcommunity.microsoft.com/t5/sql-server/step-by-step-guide-to-setup-ldaps-on-windows-server/ba-p/385362). The only thing to really know here is that I've imported the CA Certificate from my own CA and that the Domain Controller is part of the same PKI.

**Any steps relating to certificates and TLS CAN be skipped and Vault will work, it just won't be communicating in it's most secure manner, as ever I wouldn't recommend it, security is everything when working with this kind of data**.

## Configuring Vault

The LDAP configuration can be done via the UI, API or CLI, the fastest of these is the CLI, allowing only a few commands to be issued.

First, we will need to export some **Environment Variables** in order that the Vault CLI will read values correctly on the shell:

```bash
# Export Vault Environment Variables:
export VAULT_TOKEN='ENTER_A_SUITABLE_TOKEN'
export VAULT_ADDR='https://mc-vault.madcaplaughs.co.uk:8200'
export VAULT_CLIENT_CER='/etc/ssl/certs/mc-vault.cer'
export VAULT_CLIENT_KEY='/etc/ssl/private/mc-vault.key'
```

With these defined, we can now use the **vault** command which will make automatic calls to the API. First we will enable the LDAP authentication method:

```bash
# Enable LDAP Authentication method in Vault
vault auth enable ldap
```

If we were to look at the UI now, we can see that the LDAP auth method has indeed appeared:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/2-4.png)

Now we need to push a configuration to the newly created authentication method, this can be delivered using the **vault** command again:

```bash{% raw %}
vault write auth/ldap/config \
    url="ldap://mc-dc01.madcaplaughs.co.uk" \
    userattr=sAMAccountName \
    userdn="ou=User_Objects,ou=_MCL,dc=madcaplaughs,dc=co,dc=uk" \
    groupdn="ou=Admin_Groups,ou=Group_Objects,ou=_MCL,dc=madcaplaughs,dc=co,dc=uk" \
    groupfilter="(&(objectClass=person)(uid={{.Username}}))" \
    groupattr="memberOf" \
    binddn="cn=SVC_VaultLDAP,ou=Service_Accounts,ou=User_Objects,OU=_MCL,dc=madcaplaughs,dc=co,dc=uk" \
    bindpass='Sup3rS3cr3tB1nd1ngPa55!' \
    certificate=@/etc/ssl/certs/mc-strongbox-ca.cer \
    insecure_tls=false \
    starttls=true
```
{% endraw %}

Broken down, these individual values are defining:

- **url**: URL of your LDAP server, formatted correctly, multiple options are provided for this
- **userattr**: The attribute for a user to lookup when authenticating
- **userdn**: The root OU to lookup **users** from when searching during authentication
- **groupdn**: The root OU to lookup **groups** for user membership during authentication
- **groupfilter**: The method for filtering group membership, in this case look for _user_ objects only
- **groupattr**: Which attribute to use to determine group membership, in this case _memberOf_
- **binddn**: The **user** object to use for Active Directory binding
- **bindpass**: The **password** for the bind account
- **certificate**: The path to the PEM formatted CA certificate shared between this server and the Domain Controller, stored somewhere on the local machine (Optional, but only if you're not using LDAPS, not recommended)
- **insecure_tls**: Use insecure TLS (Optional, but only if you're not using LDAPS, not recommended)
- **starttls**: Initiate STARTTLS (Optional, but only if you're not using LDAPS, not recommended)

Once this configuration is submitted you will receive a confirmation message that it was applied, there is still one more step.

## LDAP Mapping

LDAP objects need to be mapped between Vault and Active Directory, so we will need to first represent the Group(s) and User(s) we wish to authenticate within Vault, we will do this using the **vault** command again:

```bash
# Create a new group within the LDAP config named VaultAdmins, linked to the apiaccess policy
vault write auth/ldap/groups/VaultAdmins policies=apiaccess

# Create a new user within the LDAP config named welsh, within the VaultAdmins group, linked to
# the apiaccess policy
vault write auth/ldap/users/welsh groups=VaultAdmins policies=apiaccess
```

These objects will now be matches to the objects existing in Active Directory, based on the rules defined in the earlier configuration.

## Testing It Out

Now that the configuration is in place, we can test first from the CLI, again with the **vault** command:

```bash
vault login -method=ldap username=welsh
Password (will be hidden): ******************
# Success! You are now authenticated. The token information displayed below
# is already stored in the token helper. You do NOT need to run "vault login"
# again. Future Vault requests will automatically use this token.
Key                    Value
--- -----
token                  <YOUR_ACCOUNT_TOKEN>
token_accessor         <TOKEN_ACCESSOR>
token_duration         10h
token_renewable        true
token_policies         ["apiaccess" "default"]
identity_policies      []
policies               ["apiaccess" "default"]
token_meta_username    welsh
```

Finally, we can now confirm access at the UI. At the logon screen we first need to set the login method to LDAP instead of token:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/4-3.png)

Now we should access using an account which has been correctly mapped:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/5-2.png)

Once logged in, we can see that we have access to the KV Secrets Engine allowed by the **apiaccess** policy and that we have indeed logged in as an individual user:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/6-4-1024x277.png">
  <figcaption>Shared and Private Secret Storage - LDAPS Integrated</figcaption>
</figure>

This is just a simple example of course and the Vault documentation offers great options for creating complex mappings and structures to tailor to your environment.
