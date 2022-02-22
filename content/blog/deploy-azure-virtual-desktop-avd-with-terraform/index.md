---
title: "Deploy Azure Virtual Desktop (AVD) with Terraform"
date: "2022-02-22T17:30:52+01:00"
draft: true
comments: true
socialShare: true
toc: false
cover:
  src: cover.jpg
tags:
  - AVD
  - Azure
  - Azure Virtual Desktop
  - IaC
  - Infrastructure as Code
  - Terraform
---

With [Azure Virtual Desktop (AVD)](https://azure.microsoft.com/en-us/services/virtual-desktop/) you can deliver secure Windows 11 desktops and environments anywhere. It's pretty easy to deploy and scale, enables you to deliver a coherent user experience from any client, and reduces costs by leveraging Windows 11 multi-session licensing. In this post I'll walk you through setting up AVD with Terraform.

<!--more-->

## Prerequisites

Besides an active Azure subscription and [Terraform](https://www.terraform.io/) configured on your workstation, [Azure Active Directory Domain Services (AADDS)](https://azure.microsoft.com/en-us/services/active-directory-ds/) are required. [Check out my previous post on setting up AADDS with Terraform if you haven't already!](/blog/set-up-azure-active-directory-domain-services-aadds-with-terraform-updated)

## Overview

AADDS and AVD resources are deployed to separate virtual networks and resource groups. It's a [hub-spoke network topology](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/hub-spoke), a typical approach to organize large-scale networks. The hub (`aadds-vnet`), the central connectivity point, typically contains other services besides AADDS. E.g. a [VPN gateway](https://azure.microsoft.com/en-us/services/vpn-gateway/) connecting your on-premises network to the Azure cloud, [Azure Bastion](https://docs.microsoft.com/en-us/azure/bastion/bastion-overview), or [Azure Firewall](https://docs.microsoft.com/en-us/azure/firewall/overview). Spoke networks (`avd-vnet`) contain isolated workloads with [network peering](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview).

<!--![Hub-spoke network diagram](hub-spoke-network.png)-->

## Network Resources

Create the `avd-rg` resource group and add the `avd-vnet` spoke network to it:

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

  # Use AADDS DCs as DNS servers
  dns_servers = azurerm_active_directory_domain_service.aadds.initial_replica_set.0.domain_controller_ip_addresses
}

resource "azurerm_subnet" "avd" {
  name                 = "avd-snet"
  resource_group_name  = azurerm_resource_group.avd.name
  virtual_network_name = azurerm_virtual_network.avd.name
  address_prefixes     = ["10.10.0.0/24"]
}
```
