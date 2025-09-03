---
title: "Terraform and Azure - Automated Deployment of Site To Site VPNs"
layout: post
date: 2020-05-28
categories: 
  - "automation"
  - "azure"
  - "devops"
tags: 
  - "automation"
  - "azure"
  - "cloud"
  - "devops"
  - "networking"
  - "terraform"
---

The creation of an Azure Site to Site VPN is (even by Software Defined Networking standards)...involved. This isn't a problem unique to Azure and isn't aided by the desire by vendors to call all of their components something unusual rather than the terminology that already exists. Setup is a **very** manual and time consuming process, however Terraform can completely automate and codify the process.

Example code for this post can be found in my GitHub at [here](/terraform-and-azure-automated-deployment-of-s2s-vpns/).

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-8.png" class="scaled-img-75">

## Before Jumping In...

We need to define the usual settings, the **local** **gateway** (usually an on-premise firewall), the **VPN Gateway** (Azure's VPN Gateway) and the **Connection** (the VPN connection between the two), however all three of these need to be defined in Azure, this can lead to some confusion as on the surface you might assume that the **Local Gateway** has no business being defined in Azure since it's not a Cloud item (not to mention the various SKU oddities that crop up along the way).

Despite the **Local Gateway** being defined in Azure, this isn't some kind of magic self configuring and self routing VPN, you will still need to configure your actual local device(s) to do their part, Microsoft have tried to lay out a good chunk of a assistance in providing configuration guides for _supported_ devices in their [documentation](https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-about-vpn-devices) (though I know from experience that "unsupported" devices will work with varying degrees of success as long as you can make the protocols and proposals match).

It is also critical to know that Azure has a **mandatory** requirement for an entire /24 **Transport Subnet** inside the _Address Space_ your VNet has been created in named **GatewaySubnet**, if this isn't in place when you attempt to create your first VPN you'll get nowhere.

Finally, I'm assuming that authentication is going to be done with Pre-Shared Keys of a good length, since the key needs to be pre-shared, I'm going to have it entered at run time rather than randomly generated using Terraform's _pseudorandom_ generation utilities.

## How Does The VPN Look?

According to Microsoft, the VPN should look something like this:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/02-8.png">
  <figcaption>Simple right?</figcaption>
</figure>

...except that simplistic view of things isn't exactly how anything works, how could it? The **Local Network Gateway** isn't a real device, it's just a digital representation of a real network appliance. We're also not seeing any mention of our transport subnet. It's more reasonable to say that the real setup looks like:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-7.png">
  <figcaption>Not as pretty, but accurate</figcaption>
</figure>

## Let's Try and Make Something

With all of this in mind, let's try and make something.

The code can get a little long to read for a simple blog entry so let's just look at automating the creation of a single VPN entry, adding loops and counts is simple enough but is only going to confuse the matter right now.

Below is the standard **providers.tf**, simple enough, just a single **Provider** for AzureRM:

```terraform
#--provider.tf

provider "azurerm" {
    version = "=2.1.0"
    features {}
    subscription_id = var.subscription_id
    tenant_id       = var.tenant_id
    client_id       = var.client_id
    client_secret   = var.client_secret
}
```

As usual, we want to define as **much as possible** in variables, this will aid with parameterisation and allow us to scale the routine if we want to add loops and counts later:

```terraform
#--variables.tf

#--Primary Location
variable "location" {
    type        = string
    description = "Primary Location"
    default      = "uksouth"
}

#--Subscription
variable "subscription_id" {
    type        = string
    description = "Subscription id"
}

#--Tenant
variable "tenant_id" {
    type        = string
    description = "Tenant id"
}

##############################
#---Auth and Secret Params---#
##############################

#--Service Principle AppID
variable "client_id" {
    type        = string
    description = "Client id"
}

#--Service Principle Secret
variable "client_secret" {
    type        = string
    description = "Client secret"
}

#--Service Principle Secret
variable "vpn_psk" {
    type        = string
    description = "VPN PSK"
}

#####################
#---Deploy Params---#
#####################

#--Resource Groups
variable "resource_group" {
    description = "Resource Group"
    type        = string
    default     = "tinfoil_network_rg"
}

#--Base VNet
variable "vnet" {
    description = "Base vnet"
    type        = string
    default     = "tinfoil_vnet"
}

#--Subnet Address Spaces
variable "peer_subnet_address_spaces" {
    description = "All peer subnets"
    type        = list(string)
    default     = ["172.16.1.0/24",]
}

#--Transport Subnet Address Space
variable "transport_subnet_address_space" {
    description = "All subnets"
    type        = list(string)
    default     = ["10.0.3.0/24"]
}

#--VPN Gateway
variable "vpn_gateway" {
    description = "VPN Gateway"
    type        = string
    default     = "tinfoil_vpn_gateway"
}

#--Peer VPN Gateway
variable "peer_vpn_gateway" {
    description = "Peer VPN Gateway"
    type        = string
    default     = "madcaplaughs_vpn_gateway"
}

#--VPN Connection
variable "vpn_connection" {
    description = "VPN Connection"
    type        = string
    default     = "tinfoil_vpn_connection"
}

#--VPN Connection
variable "vpn_public_ip" {
    description = "VPN Public IP"
    type        = string
    default     = "tinfoil_vpn_ip"
}
```

With everything in place, we can now use our **main.tf** for the deployment of the Azure VPN components, there's a few things to be aware of so I've added commends in-line:

```terraform
data "azurerm_subnet" "tinfoilvpn" { #--We need to look this up as as list as we need to get the ID of the Subnet
    name                 = var.transport_subnet_address_space[count.index]
    count                = length(var.transport_subnet_address_space)
    resource_group_name  = var.resource_group
    virtual_network_name = var.vnet
}

resource "azurerm_local_network_gateway" "madcaplaughs" {
    name                = var.peer_vpn_gateway
    location            = var.location
    resource_group_name = var.resource_group
    gateway_address     = "xx.xx.xx.xx" #--Your local device public IP here
    address_space       = var.peer_subnet_address_spaces
}

resource "azurerm_public_ip" "tinfoilvpn" {
    name                = var.vpn_public_ip
    location            = var.location
    resource_group_name = var.resource_group
    allocation_method   = "Dynamic" #--Dynamic set means Azure will generate an IP for your Azure VPN Gateway
}

resource "azurerm_virtual_network_gateway" "tinfoilvpn" {
    name                    = var.vpn_gateway
    location                = var.location
    resource_group_name     = var.resource_group
    type                    = "Vpn" #--Other option is ExpressRoute, predictably for ExpressRoute VPNs
    vpn_type                = "RouteBased" #--Policy based is also acceptable here, depending on your use case
    active_active           = false
    enable_bgp              = false
    sku                     = "Basic" #--A whole load of oddities occur around SKUs, see MS Docs for details
    ip_configuration {
        public_ip_address_id          = azurerm_public_ip.tinfoilvpn.id
        private_ip_address_allocation = "Dynamic"
        subnet_id                     = data.azurerm_subnet.tinfoilvpn.0.id #--There's that ID we needed, for the Transport Subnet
    }
}

resource "azurerm_virtual_network_gateway_connection" "tinfoilvpn" {
    name                       = var.vpn_connection
    location                   = var.location
    resource_group_name        = var.resource_group
    type                       = "IPsec"
    virtual_network_gateway_id = azurerm_virtual_network_gateway.tinfoilvpn.id
    local_network_gateway_id   = azurerm_local_network_gateway.madcaplaughs.id
    shared_key                 = var.vpn_psk #-Provided at run time
}
```

Now when we **terraform init** we will load the AzureRM backend, and when we **terraform apply** get ready for a very long wait as the provisioning of these resources takes a good long time (seriously expect it to be up to 30 minutes for the provisioning of the **Azure Virtual Network Gateway** and then around 15-30 minutes further before the Azure RM starts to show any traffic in or out. **This isn't a Terraform limitation, this is the speed of Azure**:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/10-4.png">
  <figcaption>Running all the way...</figcaption>
</figure>

If we look in to the AzureRM now at our active VPN connections, we can see that the connection has been created, and our Remote and Local gateways are on either end of it (IP addresses redacted for privacy):

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/11-5-1024x224.png)

## Future Considerations

I would also add that it's ill advised to link the creation of VNets, address spaces and subnets to the creation of the VPNs themselves as when you modify the configurations and reapply the entire state will be modified and you will end up **reprovisioning any and all VPNs defined by the configuration**, and at around an hour **per VPN** that's a tedious waste of time you could well do without.

After all, you don't want to interrupt services or waste your time watching progress counters tick along forever!
