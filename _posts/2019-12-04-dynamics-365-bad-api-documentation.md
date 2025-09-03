---
title: "Dynamics 365 - Using The REST API (Despite The Documentation)"
layout: post
date: 2019-12-04
categories: 
  - "devops"
tags: 
  - "api"
  - "azure"
  - "cloud"
  - "dynamics365"
  - "rest"
---

In working with the Dynamics 365 Finance and Operations APIs a couple of things became apparent quickly, the first is that the documentation is pretty dreadful, the second is that the documentation makes wild assumptions about other technologies and is geared directly towards developers. Having come from a sysadmin background this created a problem that a lot of us have had to deal with.

It seems unreasonable to complain that documentation is aimed at developers, so I suppose we'd best try and understand how that works, as for the bad documentation...well let's just try and get on with it.

# Sysadmins Reading Developers Documentation

You'd be forgiven for thinking that D365 Finance and Operations didn't even have any REST APIs given how hidden away any mentions of them are, but they exist.

For the most part the three functions to work with are [Data Management](https://docs.microsoft.com/en-us/dynamics365/fin-ops-core/dev-itpro/data-entities/data-management-api), [Recurring Integrations](https://docs.microsoft.com/en-us/dynamics365/fin-ops-core/dev-itpro/data-entities/recurring-integrations) and [**REST Metadata**](https://docs.microsoft.com/en-us/dynamics365/fin-ops-core/dev-itpro/data-entities/services-home-page) (which is read only). The documentation however provides no real guidance as you would expect to find for a well documented API (definitive return codes, endpoint structure, essential request data), it's pretty clear that this is an unloved component of the system that doesn't really want to be worked with (so if you've found yourself here I'd try and steer away from it if at all possible)!

The technical documentation does have a page on [Service Endpoints](https://docs.microsoft.com/en-us/dynamics365/fin-ops-core/dev-itpro/data-entities/services-home-page), however that page is limited to explaining the rough outline of what a Service Endpoint is, and if you don't know that then you probably wouldn't be working on an API in the first place. This page does however provide a critical bit of information in the process.

# Missing Link - App Registration

Dynamics 365 itself does offer and API, however in order to actually gain access an [Application Registration](https://docs.microsoft.com/en-us/graph/auth-register-app-v2) must first be created in the associated Azure Tenancy and **then** integrated with your Dynamics 365 deployment. The D365 docs link our to a guide [here](https://docs.microsoft.com/en-us/azure/active-directory/develop/articles/active-directory/develop/app-registrations-training-guide-for-app-registrations-legacy-users.md), but the link 404s, because it doesn't exist...

_Application Registrations_ are at the core of most Azure integrations and are well documented in the Azure docs, that said if you have no idea what the concept even is then this whole thing is going to be pretty alien and it does trip up a lot of people (myself included).

# I Can't Authenticate

So let's take a look at how to authenticate. We're forced in to using a strict method of authentication which is very secure...but not very straight forward:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/services-authentication-1024x476.png">
  <figcaption>Authorize access to web applications using OAuth 2.0 and Azure Active Directory<br/>Source: https://docs.microsoft.com/en-us/dynamics365/fin-ops-core/dev-itpro/data-entities/services-home-page</figcaption>
</figure>

So...that isn't exactly simple to follow if you don't already have a strong grasp of identity services...but it's not impossible, we can get there. This IS a strong system but implementing it is going to be difficult without any real advice on how. A straight forward guide for the uninitiated is always appreciated...and this isn't one.

# All Theory, No Guides

All the theory in the world is great, but what ever happened to a guide telling you how to just do something?

So, as we said, in order to actually gain access to a Dynamics 365 API endpoint an **App Registration** will first need to be created within your Azure Tenancy. This lives within **Azure Active Directory** > **App Registrations**. Creating your registration you have the ability to assign **API Permissions**, these permissions are the actual APIs which will be linked to your Dynamics deployment and you'll want to add **Dynamics ERP: Odata.FullAccess** (which is the Data Management API) and **Dynamics ERP: Connector.FullAccess** (which is the Recurring Integrations API). On top of these, you'll also need to add **Microsoft Graph: User.Read**.

When the App Registration gets created, 3 values are listed along with it, and in true Microsoft style, they have more than one name. Thanks Microsoft.

- **Application ID** (Also known as the **ClientID**): Unique registration ID for the App

- **Directory ID** (Also known as the **TenantID**): Unique ID for the AzureAD Instance

- **Object ID**: Unique ID for the object within AzureAD

You'll also need a secret, this will act as the passphrase to authenticate with Dynamics and request a session token. Within the **Certificates and Secrets** section of App Registration, generate a new secret. **MAKE A NOTE OF THIS, YOU WON'T BE ABLE TO SEE IT AGAIN.**

# Linking It All Together

Back in Dynamics 365, within **System Administration** > **Azure Active Directory Applications**, you will need to define the application that you just created, by defining the **Client ID** of the App Registration you just created, any name of your choosing, and a username within Dynamics that will ultimately execute tasks that you send to the API.

So you have the API connection all built now. Brilliant right? Except.....

# How Do You Actually Authenticate?

This is actually broken down reasonably well within the [**Authorization Grant Flow Document**](https://docs.microsoft.com/en-us/previous-versions/azure/dn645542\(v=azure.100\)?redirectedfrom=MSDN) but for the sake of completion:

The App Registration you created earlier also now contains your API endpoints, you can see what they are by browsing to **Overview** > **Endpoints**, so the endpoints live in Azure and the request is redirected to Dynamics.

Your bearer token request will need to contain the below params:

```yaml
Grant Type: Implicit
Callback URL: https://yourcompany.operations.dynamics.com
Auth URL: https://login.microsoftonline.com/TENANTID/oauth2/v2.0/authorize
Client ID: APPLICATION_ID
Scope: user.read
State: N/A
Client Auth: Send as Basic Auth Header
```

Now that you have a bearer token, it can be used to request an access (session) token. This request should be POSTed to the **oauth2/token** endpoint E.G. **https://login.microsoftonline.com/DIRECTORYID/oauth2/token**

Your request will need to contain the below params:

```yaml
#params
tenant_id:TENANTID

#body-form-data
tenant_id:TENANTID
client_id:CLIENTID
client_secret:APPSECRET
grant_type:client_credentials
resource:https://yourcompany.operations.dynamics.com
```

....and now you have authentication and you can interact with the API.

# What are the actual Dynamics Endpoints?

Because they're not very well documented either...  

- **api/connector/** - Recurring Integrations  

- **data/** - Data Management
