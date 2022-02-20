---
title: "Set Up Azure Active Directory Domain Services (AADDS) with Terraform"
date: 2022-02-20T06:05:36+01:00
cover:
  src: cover.jpg
comments: true
socialShare: true
tags:
  - AADDS
  - ARM
  - Azure
  - Azure Active Directory Domain Services
  - Cloud
  - IaC
  - Infrastructure As Code
  - Terraform
---

I wanted to revisit this topic for a while because the [previous guide I wrote about setting up Azure Active Directory Domain Services (AADDS) with Terraform](/blog/set-up-azure-active-directory-domain-services-aadds-with-terraform) is outdated. However, the article still attracts around 100 visitors per month. People also keep downloading the deprecated Terraform module I created. Time to set things right!

<!--more-->

With [v2.69.0 of the official Terraform azurerm provider](https://github.com/terraform-providers/terraform-provider-azurerm/releases/tag/v2.69.0) released, the [`active_directory_domain_service`](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/active_directory_domain_service) and [`active_directory_domain_service_replica_set`](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/active_directory_domain_service_replica_set) resources are now available. In this post, I'll briefly walk you through the required steps of setting up AADDS. [See also the official Microsoft documentation for more details.](https://docs.microsoft.com/en-us/azure/active-directory-domain-services/powershell-create-instance#create-required-azure-ad-resources)

[I also published the code to a sample GitHub repo.](https://github.com/schnerring/terraform-azurerm-aadds-avd)

## What Are Azure Active Directory Domain Services?

Bringing traditional Active Directory Domain Services (AD DS) to the cloud, typically required to set up, secure, and maintain domain controllers (DCs). Azure Active Directory Domain Services (AADDS or Azure AD DS) is a _Microsoft-managed_ solution, providing a subset of [traditional AD DS features](https://docs.microsoft.com/en-us/azure/active-directory-domain-services/compare-identity-solutions) without the need to _self-manage_ DCs. One such service that requires AD DS features is [Azure Virtual Desktop (AVD)](https://docs.microsoft.com/en-us/azure/virtual-desktop/overview).

## Prerequisites

Before getting started, you need the following things:

- Active Azure subscription
- [Azure Active Directory (Azure AD / AAD) tenant](https://docs.microsoft.com/en-us/azure/active-directory/develop/quickstart-create-new-tenant)

## Service Principal

First, create the service principal for the _Domain Controller Services_ published application. In public Azure, the ID is `2565bd9d-da50-47d4-8b85-4c97f669dc36`. For other clouds the value is `6ba9a5d4-8456-4118-b521-9c5ca10cdf84`.

```hcl
resource "azuread_service_principal" "aadds" {
  application_id = "2565bd9d-da50-47d4-8b85-4c97f669dc36"
}
```

If it already exists, you can import it into Terraform like this:

```shell

```

## `Microsoft.AAD` Resource Provider Registration

To use AADDS, register the `Microsoft.AAD` resource provider:

```hcl
resource "azurerm_resource_provider_registration" "aadds" {
  name = "Microsoft.AAD"
}
```

If the provider is already registered, you can import it into Terraform with the following command:

```shell
terraform import azurerm_resource_provider_registration.aadds /subscriptions/00000000-0000-0000-0000-000000000000/providers/Microsoft.AAD
```

## DC Admin Group and User

Next, create an Azure AD group for users administering the AADDS domain and add an admin.

```hcl
resource "azuread_group" "dc_admins" {
  display_name     = "AAD DC Administrators"
  description      = "AADDS Administrators"
  members          = [azuread_user.dc_admin.object_id]
  security_enabled = true
}

resource "random_password" "dc_admin" {
  length = 64
}

resource "azuread_user" "dc_admin" {
  user_principal_name = "dc-admin@example.com"
  display_name        = "AADDS DC Administrator"
  password            = random_password.dc_admin.result
}
```

## Resource Group

Add the resource group for AADDS resources:

```hcl
resource "azurerm_resource_group" "aadds" {
  name     = "aadds-rg"
  location = "Switzerland North"
}
```

## Network Resources

Add the virtual network next. Make sure to set `dns_servers` to the IP addresses of the DCs. To verify, you can find them on the **Overview** page of the managed domain after the deployment succeeded.

```hcl
resource "azurerm_virtual_network" "aadds" {
  name                = "aadds-vnet"
  location            = azurerm_resource_group.aadds.location
  resource_group_name = azurerm_resource_group.aadds.name
  address_space       = ["10.0.0.0/16"]

  # AADDS DCs
  #dns_servers = ["10.0.0.4", "10.0.0.5"] TODO?
}

resource "azurerm_subnet" "aadds" {
  name                 = "aadds-snet"
  resource_group_name  = azurerm_resource_group.aadds.name
  virtual_network_name = azurerm_virtual_network.aadds.name
  address_prefixes     = ["10.0.0.0/24"]
}
```

To lock down access to the managed domain, add the following _network security group_:

```hcl
resource "azurerm_network_security_group" "aadds" {
  name                = "aadds-nsg"
  location            = azurerm_resource_group.aadds.location
  resource_group_name = azurerm_resource_group.aadds.name

  security_rule {
    name                       = "AllowSyncWithAzureAD"
    priority                   = 101
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "443"
    source_address_prefix      = "AzureActiveDirectoryDomainServices"
    destination_address_prefix = "*"
  }

  # See https://docs.microsoft.com/en-us/azure/active-directory-domain-services/alert-nsg#inbound-security-rules
  security_rule {
    name                       = "AllowRD"
    priority                   = 201
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "3389"
    source_address_prefix      = "CorpNetSaw"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "AllowPSRemoting"
    priority                   = 301
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "5986"
    source_address_prefix      = "AzureActiveDirectoryDomainServices"
    destination_address_prefix = "*"
  }

  # See https://docs.microsoft.com/en-us/azure/active-directory-domain-services/alert-ldaps#resolution
  # See https://docs.microsoft.com/en-us/azure/active-directory-domain-services/tutorial-configure-ldaps#lock-down-secure-ldap-access-over-the-internet
  security_rule {
    name                       = "AllowLDAPS"
    priority                   = 401
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "636"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}

resource azurerm_subnet_network_security_group_association "aadds" {
  subnet_id                 = azurerm_subnet.aadds.id
  network_security_group_id = azurerm_network_security_group.aadds.id
}
```

## AADDS Managed Domain

The final step is to deploy the AADDS managed domain.

```hcl
resource "azurerm_active_directory_domain_service" "aadds" {
  name                = "aadds"
  location            = azurerm_resource_group.aadds.location
  resource_group_name = azurerm_resource_group.aadds.name

  domain_name           = "aadds.example.com"
  sku                   = "Standard"

  initial_replica_set {
    subnet_id = azurerm_subnet.aadds.id
  }

  notifications {
    additional_recipients = ["alice@example.com", "bob@example.com"]
    notify_dc_admins      = true
    notify_global_admins  = true
  }

  security {
    sync_kerberos_passwords = true
    sync_ntlm_passwords     = true
    sync_on_prem_passwords  = true
  }

  depends_on = [
    azuread_service_principal.aadds,
    azurerm_resource_provider_registration.aadds,
    azurerm_subnet_network_security_group_association.aadds,
  ]
}
```

Run `terraform apply` to deploy everything. It takes around 45 minutes to complete. Let me know what you think in the comments or on Twitter!
