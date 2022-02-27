---
title: "Deploy Azure Virtual Desktop (AVD) with FSLogix User Profiles Using Terraform and Azure Active Directory Domain Services (AADDS)"
date: "2022-02-27T21:32:52+01:00"
draft: true
comments: true
socialShare: true
toc: true
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

Besides an active Azure subscription and [Terraform](https://www.terraform.io/) configured on your workstation, [Azure Active Directory Domain Services (AADDS)](https://azure.microsoft.com/en-us/services/active-directory-ds/) are required. [Check out my previous post on setting up AADDS with Terraform if you haven't already!](/blog/set-up-azure-active-directory-domain-services-aadds-with-terraform-updated) Some Terraform resources in this guide, e.g., the network peerings and domain-join VM extension, depend on the AADDS resources in that post.

## Do You Know What's Exciting?

It's possible to just [Azure AD-join (AAD-join) AVD session hosts](https://docs.microsoft.com/en-us/azure/architecture/example-scenario/wvd/azure-virtual-desktop-azure-active-directory-join), eliminating the requirement to use AADDS or on-premise AD DS and reduce the costs and complexity of AVD deployments even more. Unfortunately, it's not yet fully production-ready because [FSLogix profile support for AAD-joined AVD VMs is only in public preview](https://azure.microsoft.com/en-us/updates/public-preview-fslogix-profiles-support-for-azure-adjoined-vms-for-azure-virtual-desktop/). Currently, using AAD authentication with Azure Files still requires [hybrid identities](https://docs.microsoft.com/en-us/azure/active-directory/hybrid/whatis-hybrid-identity). But it's nice that AVD is one step closer to being a cloud-only solution. I can't wait to terraformify all of it! Stay tuned because I'll post about it as soon as things are generally available.

## Overview

We'll deploy AADDS and AVD resources to separate virtual networks and resource groups. It is called a [hub-spoke network topology](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/hub-spoke), a typical approach to organize large-scale networks. The hub (`aadds-vnet`), the central connectivity point, typically contains other services besides AADDS. E.g., a [VPN gateway](https://azure.microsoft.com/en-us/services/vpn-gateway/) connecting your on-premises network to the Azure cloud. [Azure Bastion](https://docs.microsoft.com/en-us/azure/bastion/bastion-overview) or [Azure Firewall](https://docs.microsoft.com/en-us/azure/firewall/overview) are also services that might reside in the hub network. Spoke networks (`avd-vnet`) contain isolated workloads using [network peerings](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview) to connect to the hub.

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

## Host Pool

> A [host pool](https://docs.microsoft.com/en-us/azure/virtual-desktop/environment-setup#host-pools) is a collection of Azure virtual machines that register to Azure Virtual Desktop as session hosts when you run the Azure Virtual Desktop agent. All session host virtual machines in a host pool should be sourced from the same image for a consistent user experience.

We add the AVD host pool and the registration info. We'll later register the session hosts via VM extension to the host pool using a token from the registration info:

```hcl
locals {
  # Switzerland North is not supported
  avd_location = "West Europe"
}

resource "azurerm_virtual_desktop_host_pool" "avd" {
  name                = "avd-hp"
  location            = local.avd_location
  resource_group_name = azurerm_resource_group.avd.name

  type                = "Pooled"
  load_balancer_type  = "BreadthFirst"
  friendly_name       = "AVD Host Pool using AADDS"
  start_vm_on_connect = true
}

resource "time_rotating" "avd_registration_expiration" {
  # Must be between 1 hour and 30 days
  rotation_days = 29
}

resource "azurerm_virtual_desktop_host_pool_registration_info" "avd" {
  hostpool_id     = azurerm_virtual_desktop_host_pool.avd.id
  expiration_date = time_rotating.avd_registration_expiration.rotation_rfc3339
}
```

I deploy my AADDS and AVD resources to the `Switzerland North` region. However, I have to deploy AVD _service_ resources to `West Europe` because [the AVD service isn't available in all regions](https://docs.microsoft.com/en-us/azure/virtual-desktop/data-locations).

To get the latest supported regions, [re-register the AVD resource provider](https://docs.microsoft.com/en-us/azure/virtual-desktop/troubleshoot-set-up-issues#i-only-see-us-when-setting-the-location-for-my-service-objects) like this:

1. Select your subscription under **Subscriptions** in the Azure Portal.
2. Select the **Resource Provider** menu.
3. **Re-register** `Microsoft.DesktopVirtualization`.

We also enable `start_vm_on_connect`. I like this option for small-scale deployments that don't require daily access. The trade-off for reducing costs this way is that the first person connecting to the pool will have to wait for the session host to boot.

## Workspace and App Group

Next, we create a [workspace](https://docs.microsoft.com/en-us/azure/virtual-desktop/environment-setup#workspaces) and add an [app group](https://docs.microsoft.com/en-us/azure/virtual-desktop/environment-setup#app-groups) to it. Two types of app groups exist:

- `Desktop`: full desktop
- `RemoteApp`: individual apps

Adding the following gives AVD users the full desktop experience:

```hcl
resource "azurerm_virtual_desktop_workspace" "avd" {
  name                = "avd-ws"
  location            = local.avd_location
  resource_group_name = azurerm_resource_group.avd.name
}

resource "azurerm_virtual_desktop_application_group" "avd" {
  name                = "desktop-ag"
  location            = local.avd_location
  resource_group_name = azurerm_resource_group.avd.name

  type          = "Desktop"
  host_pool_id  = azurerm_virtual_desktop_host_pool.avd.id
  friendly_name = "Full Desktop"
}

resource "azurerm_virtual_desktop_workspace_application_group_association" "avd" {
  workspace_id         = azurerm_virtual_desktop_workspace.avd.id
  application_group_id = azurerm_virtual_desktop_application_group.avd.id
}
```

## Session Hosts

Let's add two session hosts to the AVD host pool. To be able to adjust the amount of VMs inside the host pool later, we define a variable like this:

```hcl
variable "avd_host_pool_size" {
  type        = number
  description = "Number of session hosts to add to the AVD host pool."
}
```

Next, we add the VM NICs for the session hosts:

```hcl
resource "azurerm_network_interface" "avd" {
  count               = var.avd_host_pool_size
  name                = "avd-nic-${count.index}"
  location            = azurerm_resource_group.avd.location
  resource_group_name = azurerm_resource_group.avd.name

  ip_configuration {
    name                          = "avd-ipconf"
    subnet_id                     = azurerm_subnet.avd.id
    private_ip_address_allocation = "Dynamic"
  }
}
```

After, we add the session hosts like this:

```hcl
resource "random_password" "avd_local_admin" {
  length = 64
}

resource "azurerm_windows_virtual_machine" "avd" {
  count               = length(azurerm_network_interface.avd)
  name                = "avd-vm-${count.index}"
  location            = azurerm_resource_group.avd.location
  resource_group_name = azurerm_resource_group.avd.name

  size                  = "Standard_D4s_v5"
  license_type          = "Windows_Client"
  admin_username        = "avd-local-admin"
  admin_password        = random_password.avd_local_admin.result
  network_interface_ids = [azurerm_network_interface.avd[count.index].id]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
  }

  source_image_reference {
    publisher = "MicrosoftWindowsDesktop"
    offer     = "windows-11"
    sku       = "win11-21h2-avd"
    version   = "latest"
  }
}
```

[To ensure the session hosts utilize the licensing benefits available with AVD](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/windows-desktop-multitenant-hosting-deployment#verify-your-vm-is-utilizing-the-licensing-benefit), we select `Windows_Client` as `license_type` value.

## Understanding the VM Extensions

To figure out the required VM extension, I used the AVD wizard of the Azure Portal. During the _review + create_ step, I downloaded the ARM template and reverse-engineered them.

Sometimes, creating complex deployments via Azure Portal look like black magic. Backtracking the generated ARM templates is something I like to do to get a deeper understanding of what's happening under the hood. It's usually my initial step when trying to terraformify something for the first time that I can't find good examples of elsewhere.

You can find the AVD ARM templates on official Azure GitHub [github.com/Azure/RDS-Templates/ARM-wvd-templates](https://github.com/Azure/RDS-Templates/tree/master/ARM-wvd-templates). The VM templates reside in the `nestedtemplates` directory [containing the VM extension resources that we replicated with Terraform](https://github.com/Azure/RDS-Templates/blob/9a443a5d4e29304247ae8d9b1bcddd504f8bd72e/ARM-wvd-templates/nestedtemplates/managedDisks-galleryvm.json#L460-L533):

```json
    {
      "apiVersion": "2018-10-01",
      "type": "Microsoft.Compute/virtualMachines/extensions",
      "name": "[concat(parameters('rdshPrefix'), add(copyindex(), parameters('vmInitialNumber')), '/', 'Microsoft.PowerShell.DSC')]",
      "location": "[parameters('location')]",
      "dependsOn": [ "rdsh-vm-loop" ],
      "copy": {
        "name": "rdsh-dsc-loop",
        "count": "[parameters('rdshNumberOfInstances')]"
      },
      "properties": {
        "publisher": "Microsoft.Powershell",
        "type": "DSC",
        "typeHandlerVersion": "2.73",
        "autoUpgradeMinorVersion": true,
        "settings": {
          "modulesUrl": "[parameters('artifactsLocation')]",
          "configurationFunction": "Configuration.ps1\\AddSessionHost",
          "properties": {
            "hostPoolName": "[parameters('hostpoolName')]",
            "registrationInfoToken": "[parameters('hostpoolToken')]",
            "aadJoin": "[parameters('aadJoin')]",
            "sessionHostConfigurationLastUpdateTime": "[parameters('SessionHostConfigurationVersion')]"
          }
        }
      }
    },
    {
      "condition": "[not(parameters('aadJoin'))]",
      "apiVersion": "2018-10-01",
      "type": "Microsoft.Compute/virtualMachines/extensions",
      "name": "[concat(parameters('rdshPrefix'), add(copyindex(), parameters('vmInitialNumber')), '/', 'joindomain')]",
      "location": "[parameters('location')]",
      "dependsOn": [ "rdsh-dsc-loop" ],
      "copy": {
        "name": "rdsh-domain-join-loop",
        "count": "[parameters('rdshNumberOfInstances')]"
      },
      "properties": {
        "publisher": "Microsoft.Compute",
        "type": "JsonADDomainExtension",
        "typeHandlerVersion": "1.3",
        "autoUpgradeMinorVersion": true,
        "settings": {
          "name": "[variables('domain')]",
          "ouPath": "[parameters('ouPath')]",
          "user": "[parameters('administratorAccountUsername')]",
          "restart": "true",
          "options": "3"
        },
        "protectedSettings": {
          "password": "[parameters('administratorAccountPassword')]"
        }
      }
    },
```

However, the default parameters of the ARM templates downloaded from the Azure Portal differ from the values found on GitHub, e.g., `modulesParameter`:

- GitHub: `https://raw.githubusercontent.com/Azure/RDS-Templates/master/ARM-wvd-templates/DSC/Configuration.zip`
- Azure: `https://wvdportalstorageblob.blob.core.windows.net/galleryartifacts/Configuration_01-20-2022.zip`

It seems that Microsoft periodically releases the `Configuration.zip` to the `galleryartifacts` container of the `wvdportalstorageblob` storage account. [Have a look at all releases here.](https://wvdportalstorageblob.blob.core.windows.net/galleryartifacts?restype=container&comp=list&prefix=Configuration)

The URLs that the Azure Portal sometimes change. At the time of writing, it lags a bit behind. It uses the `Configuration_01-20-2022.zip` file despite `Configuration_02-23-2022.zip` being available.

## AADDS Domain-join the VMs

We domain-join the session hosts with the `JsonADDomainExtension` VM extension:

```hcl
resource "azurerm_virtual_machine_extension" "avd_aadds_domain_join" {
  count                      = length(azurerm_windows_virtual_machine.avd)
  name                       = "aadds-domain-join-vmext"
  virtual_machine_id         = azurerm_windows_virtual_machine.avd[count.index].id
  publisher                  = "Microsoft.Compute"
  type                       = "JsonADDomainExtension"
  type_handler_version       = "1.3"
  auto_upgrade_minor_version = true

  settings = <<-SETTINGS
    {
      "Name": "${azurerm_active_directory_domain_service.aadds.domain_name}",
      "OUPath": "${join(",", formatlist("DC=%s", split(".", azurerm_active_directory_domain_service.aadds.domain_name)))}",
      "User": "${azuread_user.dc_admin.user_principal_name}",
      "Restart": "true",
      "Options": "3"
    }
    SETTINGS

  protected_settings = <<-PROTECTED_SETTINGS
    {
      "Password": "${random_password.dc_admin.result}"
    }
    PROTECTED_SETTINGS

  lifecycle {
    ignore_changes = [settings, protected_settings]
  }

  depends_on = [
    azurerm_virtual_network_peering.aadds_to_avd,
    azurerm_virtual_network_peering.avd_to_aadds
  ]
}
```

We have to ensure that the session hosts have line of sight to the AADDS DCs. To do that, we add the network peering resources to the `depends_on` list of the VM extension.

The `join(",", formatlist("DC=%s", split(".", azurerm_active_directory_domain_service.aadds.domain_name)))` expression transforms a string like `aadds.example.com` to `DC=aadds,DC=example,DC=com`.

After a VM has been domain-joined, it doesn't make sense to domain-join it again when the `settings` or `protected_settings` of the VM extension change, so we `ignore_changes` of these properties.

## Register VMs to the Host Pool

First, let's add a variable containing the URL to the zip file containing the DSC configuration:

```hcl
variable "avd_add_session_host_dsc_modules_url" {
  type    = string
  default = "https://wvdportalstorageblob.blob.core.windows.net/galleryartifacts/Configuration_02-23-2022.zip"
}
```

Then, we register the session hosts to the host pool with the `DSC` VM extension like this:

```hcl
resource "azurerm_virtual_machine_extension" "avd_add_session_host" {
  count                = length(azurerm_windows_virtual_machine.avd)
  name                 = "add-session-host-vmext"
  virtual_machine_id   = azurerm_windows_virtual_machine.avd[count.index].id
  publisher            = "Microsoft.Powershell"
  type                 = "DSC"
  type_handler_version = "2.73"

  settings = <<-SETTINGS
    {
      "modulesUrl": "${var.avd_add_session_host_dsc_modules_url}",
      "configurationFunction": "Configuration.ps1\\AddSessionHost",
      "properties": {
        "hostPoolName": "${azurerm_virtual_desktop_host_pool.avd.name}",
        "aadJoin": false
      }
    }
    SETTINGS

  protected_settings = <<-PROTECTED_SETTINGS
    {
      "properties": {
        "registrationInfoToken": "${azurerm_virtual_desktop_host_pool_registration_info.avd.token}"
      }
    }
    PROTECTED_SETTINGS

  lifecycle {
    ignore_changes = [settings, protected_settings]
  }

  depends_on = [azurerm_virtual_machine_extension.avd_aadds_domain_join]
}
```

We `ignore_changes` to the `settings` and `protected_settings` properties, analog to the AADDS domain-join VM extension.
