---
title: "Set Up Azure Active Directory Domain Services (AADDS) with Terraform"
date: 2021-05-22T16:23:36+02:00
cover:
  src: "img/cover.svg"
  alt: "Test"
hideReadMore: true
comments: true
tags:
  - aadds
  - arm
  - azure active directory domain services
  - azure resource manager template
  - terraform
---

## Update 2021-08-03

With [v2.69.0 of the official Terraform azurerm provider](https://github.com/terraform-providers/terraform-provider-azurerm/releases/tag/v2.69.0) released two weeks ago, the [`active_directory_domain_service`](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/active_directory_domain_service) and [`active_directory_domain_service_replica_set`](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/active_directory_domain_service_replica_set) resources are now available. If you are newly adding AADDS, there is no point in reading any further — use the official resources.

I will switch in the coming weeks and write a short migration guide for the people using my [AADDS Terraform module](https://registry.terraform.io/modules/schnerring/aadds/azurerm/latest).

---

Bringing traditional Active Directory Domain Services (AD DS) to the cloud, typically required to set up, secure, and maintain domain controllers (DCs). Azure Active Directory Domain Services (AADDS or Azure AD DS) is a _Microsoft-managed_ solution, providing a subset of [traditional AD DS features](https://docs.microsoft.com/en-us/azure/active-directory-domain-services/compare-identity-solutions) without the need to _self-manage_ DCs.

One such service that requires AD DS features is [Windows Virtual Desktop (WVD)](https://docs.microsoft.com/en-us/azure/virtual-desktop/overview). I have successfully deployed WVD with Terraform, but until recently, I struggled to do the same with AADDS. Today, I show you how to deploy AADDS with Terraform.

<!--more-->

If you are lazy, you can [skip to the end](#wrapping-up) and use the custom Terraform module [I published to the Terraform Registry](https://registry.terraform.io/modules/schnerring/aadds/azurerm/latest).

## Prerequisites

Before getting started, we need the following things:

- Active Azure subscription
- [Azure Active Directory (Azure AD or AAD) tenant](https://docs.microsoft.com/en-us/azure/active-directory/develop/quickstart-create-new-tenant)
- [Install Terraform](https://www.terraform.io/downloads.html)

## Building Blocks

So how do we figure out what the required resources are to deploy AADDS? By reverse-engineering the AADDS configuration wizard in the Azure Portal! We launch it by adding a new managed domain after navigating to **Azure AD Domain Services**.

What we find is that the following resources are required:

- Resource group
- Virtual network and subnet
- `AAD DC Administrators` user group

Using default values for the remaining configuration options, we can download an [Azure Resource Manager (ARM) template](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/overview) in the final review step of the wizard.

Analyzing the ARM template reveals that besides the resource group, virtual network, and subnet, a [Network Security Group (NSG)](https://docs.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview) with the [security rules required for AADDS](https://docs.microsoft.com/en-us/azure/active-directory-domain-services/alert-nsg#inbound-security-rules) is added.

The ARM template also contains the `Microsoft.AAD/domainServices` resource, with its parameters set to the configuration options from the wizard. The following is version `2021-03-01` of the ARM template format [from the official Azure Template documentation](https://docs.microsoft.com/en-us/azure/templates/microsoft.aad/2021-03-01/domainservices?tabs=json):

```json
{
  "name": "string",
  "type": "Microsoft.AAD/domainServices",
  "apiVersion": "2021-03-01",
  "location": "string",
  "tags": {},
  "properties": {
    "domainName": "string",
    "replicaSets": [
      {
        "location": "string",
        "subnetId": "string"
      }
    ],
    "ldapsSettings": {
      "ldaps": "string",
      "pfxCertificate": "string",
      "pfxCertificatePassword": "string",
      "externalAccess": "string"
    },
    "resourceForestSettings": {
      "settings": [
        {
          "trustedDomainFqdn": "string",
          "trustDirection": "string",
          "friendlyName": "string",
          "remoteDnsIps": "string",
          "trustPassword": "string"
        }
      ],
      "resourceForest": "string"
    },
    "domainSecuritySettings": {
      "ntlmV1": "string",
      "tlsV1": "string",
      "syncNtlmPasswords": "string",
      "syncKerberosPasswords": "string",
      "syncOnPremPasswords": "string",
      "kerberosRc4Encryption": "string",
      "kerberosArmoring": "string"
    },
    "domainConfigurationType": "string",
    "sku": "string",
    "filteredSync": "string",
    "notificationSettings": {
      "notifyGlobalAdmins": "string",
      "notifyDcAdmins": "string",
      "additionalRecipients": ["string"]
    }
  }
}
```

The docs also reference the [Azure Resource Manager QuickStart Template on GitHub](https://github.com/Azure/azure-quickstart-templates/tree/master/101-AAD-DomainServices). Its README confirms our previous findings but shows that the configuration wizard also must perform the following steps under the hood:

- [Register the Azure Active Directory Application Service Principal `2565bd9d-da50-47d4-8b85-4c97f669dc36`](https://github.com/Azure/azure-quickstart-templates/tree/820b8b7c172d08d0966b0c6f4a80691ca1972410/101-AAD-DomainServices#3-register-the-azure-active-directory-application-service-principal)
- [Register the `Microsoft.AAD` Resource Provider](https://github.com/Azure/azure-quickstart-templates/tree/820b8b7c172d08d0966b0c6f4a80691ca1972410/101-AAD-DomainServices#5-register-resource-provider)

With everything figured out, we can continue with the fun part: Terraform!

## Putting Everything Together

Let's register the service principal and resource provider first:

```hcl
resource "azuread_service_principal" "aadds" {
  application_id = "2565bd9d-da50-47d4-8b85-4c97f669dc36"
}

resource "azurerm_resource_provider_registration" "aadds" {
  name = "Microsoft.AAD"
}
```

Next, we add the `AAD DC Administrators` user group:

```hcl
resource "azuread_group" "aadds" {
  display_name = "AAD DC Administrators"
  description  = "Delegated group to administer Azure AD Domain Services"
}
```

Adding the resource group, virtual network, subnet, and NSG is pretty straightforward:

```hcl
resource "azurerm_resource_group" "aadds" {
  name     = "aadds-rg"
  location = "Switzerland North"
}

resource "azurerm_virtual_network" "aadds" {
  name                = "aadds-vnet"
  resource_group_name = azurerm_resource_group.aadds.name
  location            = "Switzerland North"

  address_space = ["10.0.0.0/16"]

  # AADDS DCs
  dns_servers = ["10.0.0.4", "10.0.0.5"]
}

resource "azurerm_subnet" "aadds" {
  name                 = "aadds-snet"
  resource_group_name  = azurerm_resource_group.aadds.name
  virtual_network_name = azurerm_virtual_network.aadds.name

  address_prefixes = ["10.0.0.0/24"]
}

resource "azurerm_network_security_group" "aadds" {
  name                = "aadds-nsg"
  location            = "Switzerland North"
  resource_group_name = azurerm_resource_group.aadds.name

  security_rule {
    name                       = "AllowRD"
    access                     = "Allow"
    priority                   = 201
    direction                  = "Inbound"
    protocol                   = "Tcp"
    source_address_prefix      = "CorpNetSaw"
    source_port_range          = "*"
    destination_address_prefix = "*"
    destination_port_range     = "3389"
  }

  security_rule {
    name                       = "AllowPSRemoting"
    access                     = "Allow"
    priority                   = 301
    direction                  = "Inbound"
    protocol                   = "Tcp"
    source_address_prefix      = "AzureActiveDirectoryDomainServices"
    source_port_range          = "*"
    destination_address_prefix = "*"
    destination_port_range     = "5986"
  }
}

resource "azurerm_subnet_network_security_group_association" "aadds" {
  subnet_id                 = azurerm_subnet.aadds.id
  network_security_group_id = azurerm_network_security_group.aadds.id
}
```

Make sure to set `dns_servers` to the IP addresses of the DCs. You can find them on the **Overview** page of the managed domain after the deployment succeeded.

The final step is to add the AADDS deployment. Define the ARM template as Terraform [`templatefile`](https://www.terraform.io/docs/language/functions/templatefile.html) named `aadds-arm-template.tpl.json`:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "resources": [
    {
      "name": "${domainName}",
      "type": "Microsoft.AAD/domainServices",
      "apiVersion": "2021-03-01",
      "location": "${location}",
      "tags": ${tags},
      "properties": {
        "domainName": "${domainName}",
        "replicaSets": [
          {
            "location": "${location}",
            "subnetId": "${subnetId}"
          }
        ],
        "domainSecuritySettings": {
          "ntlmV1": "${ntlmV1}",
          "tlsV1": "${tlsV1}",
          "syncNtlmPasswords": "${syncNtlmPasswords}",
          "syncKerberosPasswords": "${syncKerberosPasswords}",
          "syncOnPremPasswords": "${syncOnPremPasswords}",
          "kerberosRc4Encryption": "${kerberosRc4Encryption}",
          "kerberosArmoring": "${kerberosArmoring}"
        },
        "domainConfigurationType": "${domainConfigurationType}",
        "sku": "${sku}",
        "filteredSync": "${filteredSync}",
        "notificationSettings": {
          "notifyGlobalAdmins": "${notifyGlobalAdmins}",
          "notifyDcAdmins": "${notifyDcAdmins}",
          "additionalRecipients": ${additionalRecipients}
        }
      }
    }
  ]
}
```

We then populate its values dynamically like so:

```hcl
resource "azurerm_resource_group_template_deployment" "aadds" {
  name                = "aadds-deploy"
  resource_group_name = azurerm_resource_group.aadds.name

  deployment_mode = "Incremental"

  template_content = templatefile(
    "${path.module}/aadds-arm-template.tpl.json",
    {
      # Basics
      "domainName"              = "aadds.schnerring.net"
      "location"                = "Switzerland North"
      "sku"                     = "Standard"
      "domainConfigurationType" = "FullySynced"

      # Networking
      "subnetId" = azurerm_subnet.aadds.id

      # Administration
      "notifyGlobalAdmins"   = "Enabled"
      "notifyDcAdmins"       = "Enabled"
      "additionalRecipients" = jsonencode([])

      # Synchronization
      "filteredSync" = "Enabled"

      # Security
      "tlsV1"                 = "Enabled"
      "ntlmV1"                = "Enabled"
      "syncNtlmPasswords"     = "Enabled"
      "syncOnPremPasswords"   = "Enabled"
      "kerberosRc4Encryption" = "Enabled"
      "syncKerberosPasswords" = "Enabled"
      "kerberosArmoring"      = "Disabled"

      # Tags
      "tags" = jsonencode({})
    }
  )

  depends_on = [azurerm_resource_provider_registration.aadds]
}
```

Run `terraform apply` to deploy everything. It takes around 45 minutes to complete.

## Wrapping Up

I created a custom module wrapping the above functionality and [published it to the Terraform Registry](https://registry.terraform.io/modules/schnerring/aadds/azurerm/latest). You can also find [the code on my GitHub](https://github.com/schnerring/terraform-azurerm-aadds) along with some examples. I decided that the creation of network resources is out of the module's scope. Depending on what network topology you prefer, pre-provisioning the virtual network and subnet gives you more flexibility.

The module provides the same options as the Azure Portal configuration wizard. More advanced configuration options like LDAP and forests are not yet supported. Feel free to comment below, or open an issue or pull request on GitHub if you find something to improve.

A minimal deployment with the custom module looks like this:

```hcl
resource "azurerm_resource_group" "aadds" {
  name     = "aadds-rg"
  location = "Switzerland North"
}

resource "azurerm_virtual_network" "aadds" {
  name                = "aadds-vnet"
  resource_group_name = azurerm_resource_group.aadds.name
  location            = "Switzerland North"

  address_space = ["10.0.0.0/16"]

  # AADDS DCs
  dns_servers = ["10.0.0.4", "10.0.0.5"]
}

resource "azurerm_subnet" "aadds" {
  name                 = "aadds-snet"
  resource_group_name  = azurerm_resource_group.aadds.name
  virtual_network_name = azurerm_virtual_network.aadds.name

  address_prefixes = ["10.0.0.0/24"]
}

module "aadds" {
  source  = "schnerring/aadds/azurerm"
  version = "0.1.1"

  resource_group_name = azurerm_resource_group.aadds.name
  location            = "Switzerland North"
  domain_name         = "aadds.schnerring.net"
  subnet_id           = azurerm_subnet.aadds.id
}
```
