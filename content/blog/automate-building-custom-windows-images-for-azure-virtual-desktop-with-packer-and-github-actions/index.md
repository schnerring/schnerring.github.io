---
title: Automate Building Custom Windows Images For Azure Virtual Desktop With Packer And GitHub Actions
date: "2022-02-25T04:04:20+01:00"
draft: true
comments: true
socialShare: true
toc: true
#cover:
#  src: cover.png
tags:
  - AVD
  - Azure
  - Azure CLI
  - Azure Virtual Desktop
  - DevOps
  - GitHub Actions
  - IaC
  - Infrastructure as Code
  - Packer
  - Terraform
---

One aspect of managing [Azure Virtual Desktop (AVD)](https://azure.microsoft.com/en-us/services/virtual-desktop/) is keeping it up-to-date. One strategy is periodically building a "golden" image and re-deploying AVD session host VMs using the updated image. In this post, I'll demonstrate using [Packer](https://www.packer.io/) and [GitHub Actions](https://github.com/features/actions) to build images and push them to Azure.

<!--more-->

As usual, [all the code is available on GitHub](https://github.com/schnerring/packer-windows-avd).

## What We'll Need

- Active Azure subscription
- [Packer](https://www.packer.io/)
- [Terraform](https://www.terraform.io/) (optional)

## Overview

## Prepare Packer Resources with Terraform

Before being able to use Packer, we have to create some resources. To do so, we could use the Azure CLI or the Azure Portal. I like using Terraform.

### Resource Groups

First we create two resource groups:

```hcl,
resource "azurerm_resource_group" "packer_artifacts" {
  name     = "packer-artifacts-rg"
  location = "Switzerland North"
}

resource "azurerm_resource_group" "packer_build" {
  name     = "packer-build-rg"
  location = "Switzerland North"
}
```

[Packer's Azure ARM builder](https://www.packer.io/plugins/builders/azure/arm) uses the `packer-build-rg` resource group to provision the required build resources and should only contain resources during build time.
Packer will publish the resulting [managed images](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/capture-image-resource) to the `packer-artifacts-rg` resource group.

### Authentication and Authorization

We'll use [service principal (SP) authentication](https://www.packer.io/plugins/builders/azure/arm#service-principal) with Packer because it integrates well with GitHub Actions. We create an SP like this:

```hcl
resource "azuread_application" "packer" {
  display_name = "packer-sp-app"
}

resource "azuread_service_principal" "packer" {
  application_id = azuread_application.packer.application_id
}

resource "azuread_service_principal_password" "packer" {
  service_principal_id = azuread_service_principal.packer.id
}
```

To authorize the SP to manage resources inside the resource groups we created, we use [role-based access control (RBAC)](https://docs.microsoft.com/en-us/azure/role-based-access-control/overview) and assign the SP the `Contributor` role, scoped to the resource groups:

```hcl
resource "azurerm_role_assignment" "packer_build_contributor" {
  scope                = azurerm_resource_group.packer_build.id
  role_definition_name = "Contributor"
  principal_id         = azuread_service_principal.packer.id
}

resource "azurerm_role_assignment" "packer_artifacts_contributor" {
  scope                = azurerm_resource_group.packer_artifacts.id
  role_definition_name = "Contributor"
  principal_id         = azuread_service_principal.packer.id
}
```

### Export GitHub Actions Secrets

To make the credentials accessible to GitHub Actions, we export them as secrets like this:

```hcl
resource "github_actions_secret" "packer_client_id" {
  repository      = data.github_repository.packer_windows_11_avd.name
  secret_name     = "PACKER_CLIENT_ID"
  plaintext_value = azuread_application.packer.application_id
}

resource "github_actions_secret" "packer_client_secret" {
  repository      = data.github_repository.packer_windows_11_avd.name
  secret_name     = "PACKER_CLIENT_SECRET"
  plaintext_value = azuread_service_principal_password.packer.value
}

resource "github_actions_secret" "packer_subscription_id" {
  repository      = data.github_repository.packer_windows_11_avd.name
  secret_name     = "PACKER_SUBSCRIPTION_ID"
  plaintext_value = data.azurerm_subscription.subscription.subscription_id
}

resource "github_actions_secret" "packer_tenant_id" {
  repository      = data.github_repository.packer_windows_11_avd.name
  secret_name     = "PACKER_TENANT_ID"
  plaintext_value = data.azurerm_subscription.subscription.tenant_id
}
```

We'll also make use of the [Azure Login Action](https://github.com/Azure/login#configure-a-service-principal-with-a-secret) to dynamically query the latest Windows version available on Azure with the Azure CLI. You'll later see why. It [expects credentials as JSON](https://github.com/Azure/login#configure-a-service-principal-with-a-secret):

```json
{
  "clientId": "<GUID>",
  "clientSecret": "<GUID>",
  "subscriptionId": "<GUID>",
  "tenantId": "<GUID>"
}
```

```hcl
resource "github_actions_secret" "github_actions_azure_credentials" {
  repository  = data.github_repository.packer_windows_11_avd.name
  secret_name = "AZURE_CREDENTIALS"

  plaintext_value = jsonencode(
    {
      clientId       = azuread_application.packer.application_id
      clientSecret   = azuread_service_principal_password.packer.value
      subscriptionId = data.azurerm_subscription.subscription.subscription_id
      tenantId       = data.azurerm_subscription.subscription.tenant_id
    }
  )
}
```

```powershell
Get-AzVMImage `
  -Location "Switzerland North" `
  -Publisher "MicrosoftWindowsDesktop" `
  -Offer windows-11 `
  -Sku win11-21h2-avd
```
