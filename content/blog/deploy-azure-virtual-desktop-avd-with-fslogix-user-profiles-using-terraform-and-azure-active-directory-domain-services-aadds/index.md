---
title: "Deploy Azure Virtual Desktop (AVD) with FSLogix User Profiles Using Terraform and Azure Active Directory Domain Services (AADDS)"
date: "2022-02-22T17:30:52+01:00"
draft: true
comments: true
socialShare: true
toc: false
cover:
  src: cover.jpg
tags:
  - AADDS
  - AVD
  - Azure
  - Azure Active Directory Domain Services
  - Azure Virtual Desktop
  - FSLogix
  - IaC
  - Infrastructure as Code
  - Terraform
---

With [Azure Virtual Desktop (AVD)](https://azure.microsoft.com/en-us/services/virtual-desktop/), you can deliver secure Windows 11 desktops and environments anywhere. It's pretty easy to deploy and scale. You can provide a coherent user experience from any end-user device and reduce costs by leveraging Windows 11 multi-session licensing. In this post, I'll walk you through setting up AVD with FSLogix profiles using Terraform.

<!--more-->

## Prerequisites

Besides an active Azure subscription and [Terraform](https://www.terraform.io/) configured on your workstation, [Azure Active Directory Domain Services (AADDS)](https://azure.microsoft.com/en-us/services/active-directory-ds/) are required. [Check out my previous post on setting up AADDS with Terraform if you haven't already!](/blog/set-up-azure-active-directory-domain-services-aadds-with-terraform-updated)

## Do You Know What's Exciting?

It's possible to just [Azure AD-join (AAD-join) AVD sessions hosts](https://docs.microsoft.com/en-us/azure/architecture/example-scenario/wvd/azure-virtual-desktop-azure-active-directory-join), eliminating the requirement to use AADDS or on-premise AD DS and reduce the costs and complexity of AVD deployments even more. Unfortunately, it's not yet fully production-ready because [FSLogix profile support for AAD-joined AVD VMs is only in public preview](https://azure.microsoft.com/en-us/updates/public-preview-fslogix-profiles-support-for-azure-adjoined-vms-for-azure-virtual-desktop/). Currently, using AAD authentication with Azure Files still requires [hybrid identities](https://docs.microsoft.com/en-us/azure/active-directory/hybrid/whatis-hybrid-identity). But AVD is now one step closer to being a cloud-only solution. I can't wait to terraformify all of it! Stay tuned because I'll post about it as soon as things are generally available.

## Overview

We'll deploy AADDS and AVD resources to separate virtual networks and resource groups. It's a [hub-spoke network topology](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/hub-spoke), a typical approach to organize large-scale networks. The hub (`aadds-vnet`), the central connectivity point, typically contains other services besides AADDS. E.g. a [VPN gateway](https://azure.microsoft.com/en-us/services/vpn-gateway/) connecting your on-premises network to the Azure cloud, [Azure Bastion](https://docs.microsoft.com/en-us/azure/bastion/bastion-overview), or [Azure Firewall](https://docs.microsoft.com/en-us/azure/firewall/overview). Spoke networks (`avd-vnet`) contain isolated workloads using [network peerings](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview) to connect to the hub.

<!--![Hub-spoke network diagram](hub-spoke-network.png)-->

## Network Resources

Create the `avd-rg` resource group and add the `avd-vnet` spoke network to it. The network uses the AADDS domain controllers (DCs) as `dns_servers`:

```hcl
resource "azurerm_resource_group" "avd" {
  name     = "avd-rg"
  location = "Switzerland North"
}

# Network Resources

resource "azurerm_virtual_network" "avd" {
  name                = "avd-vnet"
  location            = azurerm_resource_group.avd.location
  resource_group_name = azurerm_resource_group.avd.name
  address_space       = ["10.10.0.0/16"]
  dns_servers         = azurerm_active_directory_domain_service.aadds.initial_replica_set.0.domain_controller_ip_addresses
}

resource "azurerm_subnet" "avd" {
  name                 = "avd-snet"
  resource_group_name  = azurerm_resource_group.avd.name
  virtual_network_name = azurerm_virtual_network.avd.name
  address_prefixes     = ["10.10.0.0/24"]
}
```

To give AVD VMs line of sight of AADDS, we need to add the following network peerings:

```hcl
resource "azurerm_virtual_network_peering" "aadds_to_avd" {
  name                      = "hub-to-avd-peer"
  resource_group_name       = azurerm_resource_group.aadds.name
  virtual_network_name      = azurerm_virtual_network.aadds.name
  remote_virtual_network_id = azurerm_virtual_network.avd.id
}

resource "azurerm_virtual_network_peering" "avd_to_aadds" {
  name                      = "avd-to-aadds-peer"
  resource_group_name       = azurerm_resource_group.avd.name
  virtual_network_name      = azurerm_virtual_network.avd.name
  remote_virtual_network_id = azurerm_virtual_network.aadds.id
}
```
