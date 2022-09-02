---
title: Automate Building Custom Windows Images For Azure Virtual Desktop With Packer And GitHub Actions
date: "2022-03-27T08:12:20+02:00"
draft: false
comments: true
socialShare: true
toc: true
cover:
  src: cover.jpg
tags:
  - AVD
  - Azure
  - Azure CLI
  - Azure Virtual Desktop
  - CI
  - Continuous Integration
  - DevOps
  - GitHub Actions
  - IaC
  - Infrastructure as Code
  - Packer
  - Terraform
---

One aspect of managing [Azure Virtual Desktop (AVD)](https://azure.microsoft.com/en-us/services/virtual-desktop/) is keeping it up-to-date. One strategy is periodically building a "golden" image and re-deploying AVD session host VMs using the updated image. In this post, we'll use [Packer](https://www.packer.io/) and [GitHub Actions](https://github.com/features/actions) to build a Windows 11 image and push it to Azure.

<!--more-->

First, we'll use [Terraform](https://www.terraform.io/) to prepare some resources for Packer: a resource group for build artifacts and a [service principal (SP)](https://www.packer.io/plugins/builders/azure/arm#service-principal) for authentication. We'll also export the SP credentials as GitHub Actions secrets, making them available to our CI workflow.

Then we'll build a customized Windows 11 image with Packer suitable for software development workstations. We'll use [Chocolatey](https://chocolatey.org/) to install some apps like Visual Studio 2022 for .NET development, 7zip, the Kubernetes CLI and more. We'll also use a custom PowerShell script to install [Azure PowerShell](https://github.com/Azure/azure-powershell).

Finally, we'll [schedule a GitHub Actions workflow](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#schedule) that runs the Packer build. We'll query Azure daily to check for new Windows releases and run Packer as soon as a new version is made available by Microsoft.

As usual, [all the code is available on GitHub](https://github.com/schnerring/packer-windows-avd/tree/v0.3.0).

## Get the Latest Windows 11 Version Available on Azure

[Microsoft releases monthly quality updates for Windows](https://docs.microsoft.com/en-us/windows/deployment/update/quality-updates#quality-updates), on _Patch Tuesday_, the second Tuesday of each month. Microsoft can provide a release outside of the monthly schedule in exceptional circumstances, e.g., to fix a critical security vulnerability.

We can use the Azure CLI to query the available Windows versions on Azure like this:

```bash
az vm image list \
  --publisher MicrosoftWindowsDesktop \
  --offer office-365 \
  --sku win11-21h2-avd-m365 \
  --all
```

If you prefer using a Windows 11 base image without Office 365, use `--offer windows-11` and `--sku win11-21h2-avd` instead. You can discover more images using the Azure CLI commands `az vm image list-publishers`, `az vm image list-offers`, and `az vm image list-skus`.

So, to specify an image, a value for _publisher_, _offer_, _SKU_, and _version_ is required. To get the _latest version number_ available, we use the follwing snippet:

```bash
az vm image list \
  --publisher "${IMAGE_PUBLISHER}" \
  --offer "${IMAGE_OFFER}" \
  --sku "${IMAGE_SKU}" \
  --all \
  --query "[*].version | sort(@)[-1:]" \
  --out tsv
```

The magic lies within the `--query` part. It contains a [JMESPath](https://jmespath.org/) expression and does the following:

- `[*].version` flattens the command result to only contain a list of version numbers
- `sort(@)[-1:]` selects the last element of the version number list to get the latest version

The result of the command above looks like this:

```text
22000.556.220308
```

## Prepare Packer Resources with Terraform

Before being able to use Packer, we have to create some resources. I like using Terraform, but you could also use the Azure CLI or the Azure Portal.

### Resource Groups

First we create two resource groups:

```hcl
resource "azurerm_resource_group" "packer_artifacts" {
  name     = "packer-artifacts-rg"
  location = "Switzerland North"
}

resource "azurerm_resource_group" "packer_build" {
  name     = "packer-build-rg"
  location = "Switzerland North"
}
```

Packer puts resources required during build time into the `packer-build-rg` resource group, which means it should only contain resources during build time.
Packer publishes the resulting [managed images](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/capture-image-resource) to the `packer-artifacts-rg` resource group. We could also let Packer manage the creation of resource groups, but in my opinion, it's easier to scope permissions if we pre-provision them.

### Authentication and Authorization

We'll use [SP authentication](https://www.packer.io/plugins/builders/azure/arm#service-principal=) with Packer because it integrates well with GitHub Actions:

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

To authorize the SP to manage resources inside the resource groups, we use [role-based access control (RBAC)](https://docs.microsoft.com/en-us/azure/role-based-access-control/overview) and assign the SP the `Contributor` role. To allow the SP to check for existing images via the Azure CLI [`az image show` command](https://docs.microsoft.com/en-us/cli/azure/image?view=azure-cli-latest#az-image-show), we also assign it the `Reader` role on the subscription level.

```hcl
data "azurerm_subscription" "subscription" {}

resource "azurerm_role_assignment" "subscription_reader" {
  scope                = data.azurerm_subscription.subscription.id
  role_definition_name = "Reader"
  principal_id         = azuread_service_principal.packer.id
}

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

To make the credentials accessible to GitHub Actions, we export them to GitHub as [encrypted secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets) like this:

```hcl
data "github_repository" "packer_windows_avd" {
  full_name = "schnerring/packer-windows-avd"
}

resource "github_actions_secret" "packer_client_id" {
  repository      = data.github_repository.packer_windows_avd.name
  secret_name     = "PACKER_CLIENT_ID"
  plaintext_value = azuread_application.packer.application_id
}

resource "github_actions_secret" "packer_client_secret" {
  repository      = data.github_repository.packer_windows_avd.name
  secret_name     = "PACKER_CLIENT_SECRET"
  plaintext_value = azuread_service_principal_password.packer.value
}

resource "github_actions_secret" "packer_subscription_id" {
  repository      = data.github_repository.packer_windows_avd.name
  secret_name     = "PACKER_SUBSCRIPTION_ID"
  plaintext_value = data.azurerm_subscription.subscription.subscription_id
}

resource "github_actions_secret" "packer_tenant_id" {
  repository      = data.github_repository.packer_windows_avd.name
  secret_name     = "PACKER_TENANT_ID"
  plaintext_value = data.azurerm_subscription.subscription.tenant_id
}
```

We'll later also make use of the [Azure Login Action](https://github.com/Azure/login#configure-a-service-principal-with-a-secret) to dynamically query the latest Windows version available on Azure with the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/). [It expects the credentials in the following JSON format](https://github.com/Azure/login#configure-a-service-principal-with-a-secret):

```json
{
  "clientId": "<GUID>",
  "clientSecret": "<GUID>",
  "subscriptionId": "<GUID>",
  "tenantId": "<GUID>"
}
```

Let's export another secret named `AZURE_CREDENTIALS` containing the credentials in the JSON format above using Terraform's `jsonencode` function:

```hcl
resource "github_actions_secret" "github_actions_azure_credentials" {
  repository  = data.github_repository.packer_windows_avd.name
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

Finally, we export the names of the resource groups we created like this:

```hcl
resource "github_actions_secret" "packer_artifacts_resource_group" {
  repository      = data.github_repository.packer_windows_avd.name
  secret_name     = "PACKER_ARTIFACTS_RESOURCE_GROUP"
  plaintext_value = azurerm_resource_group.packer_artifacts.name
}

resource "github_actions_secret" "packer_build_resource_group" {
  repository      = data.github_repository.packer_windows_avd.name
  secret_name     = "PACKER_BUILD_RESOURCE_GROUP"
  plaintext_value = azurerm_resource_group.packer_build.name
}
```

### Add Terraform Outputs

To run Packer locally, we also need to access the credentials locally. We can use [Terraform outputs](https://www.terraform.io/language/values/outputs) for this:

```hcl
output "packer_artifacts_resource_group" {
  value     = azurerm_resource_group.packer_artifacts.name
}

output "packer_build_resource_group" {
  value     = azurerm_resource_group.packer_build.name
}

output "packer_client_id" {
  value     = azuread_application.packer.application_id
  sensitive = true
}

output "packer_client_secret" {
  value     = azuread_service_principal_password.packer.value
  sensitive = true
}

output "packer_subscription_id" {
  value     = data.azurerm_subscription.subscription.subscription_id
  sensitive = true
}

output "packer_tenant_id" {
  value     = data.azurerm_subscription.subscription.tenant_id
  sensitive = true
}
```

After running `terraform apply` to deploy everything, we can access output values like this: `terraform output packer_client_secret`.

## Create the Packer Template

Let's add a Packer template file named `windows.pkr.hcl`.

### Input Variables

[Input variables](https://www.packer.io/guides/hcl/variables) allow us to parameterize the Packer build. We can later set their values from default values, environment, file, or CLI arguments.

We need to add variables allowing us to pass the SP credentials and resource group names, as well as the image publisher, offer, SKU, and version:

```hcl
variable "client_id" {
  type        = string
  description = "Azure Service Principal App ID."
  sensitive   = true
}

variable "client_secret" {
  type        = string
  description = "Azure Service Principal Secret."
  sensitive   = true
}

variable "subscription_id" {
  type        = string
  description = "Azure Subscription ID."
  sensitive   = true
}

variable "tenant_id" {
  type        = string
  description = "Azure Tenant ID."
  sensitive   = true
}

variable "artifacts_resource_group" {
  type        = string
  description = "Packer Artifacts Resource Group."
}

variable "build_resource_group" {
  type        = string
  description = "Packer Build Resource Group."
}

variable "source_image_publisher" {
  type        = string
  description = "Windows Image Publisher."
}

variable "source_image_offer" {
  type        = string
  description = "Windows Image Offer."
}

variable "source_image_sku" {
  type        = string
  description = "Windows Image SKU."
}

variable "source_image_version" {
  type        = string
  description = "Windows Image Version."
}
```

### Configure Azure ARM Builder

Next, we configure [Packer's Azure Resource Manager (ARM) Builder](https://www.packer.io/plugins/builders/azure/arm). We start with the [`source`](https://www.packer.io/docs/templates/hcl_templates/blocks/source) configuration block:

```hcl
source "azure-arm" "avd" {
  # WinRM Communicator

  communicator   = "winrm"
  winrm_use_ssl  = true
  winrm_insecure = true
  winrm_timeout  = "5m"
  winrm_username = "packer"

  # Service Principal Authentication

  client_id       = var.client_id
  client_secret   = var.client_secret
  subscription_id = var.subscription_id
  tenant_id       = var.tenant_id

  # Source Image

  os_type         = "Windows"
  image_publisher = var.source_image_publisher
  image_offer     = var.source_image_offer
  image_sku       = var.source_image_sku
  image_version   = var.source_image_version

  # Destination Image

  managed_image_resource_group_name = var.artifacts_resource_group
  managed_image_name                = "${var.source_image_sku}-${var.source_image_version}"

  # Packer Computing Resources

  build_resource_group_name = var.build_resource_group
  vm_size                   = "Standard_D4ds_v4"
}
```

The [WinRM communicator](https://www.packer.io/docs/communicators/winrm) is Packer's way of talking to Azure Windows VMs. We give the resulting image a unique `managed_image_name` by concatenating the values of the SKU and version. The remaining options are self-explanatory.

Next, we define a `builder` block that contains our provisioning steps:

```hcl
build {
  source "azure-arm.avd" {}

  # Install Chocolatey: https://chocolatey.org/install#individual
  provisioner "powershell" {
    inline = ["Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))"]
  }

  # Install Chocolatey packages
  provisioner "file" {
    source      = "./packages.config"
    destination = "D:/packages.config"
  }

  provisioner "powershell" {
    inline = ["choco install --confirm D:/packages.config"]
    # See https://docs.chocolatey.org/en-us/choco/commands/install#exit-codes
    valid_exit_codes = [0, 3010]
  }

  provisioner "windows-restart" {}

  # Azure PowerShell Modules
  provisioner "powershell" {
    script = "./install-azure-powershell.ps1"
  }

  # Generalize image using Sysprep
  # See https://www.packer.io/docs/builders/azure/arm#windows
  # See https://docs.microsoft.com/en-us/azure/virtual-machines/windows/build-image-with-packer#define-packer-template
  provisioner "powershell" {
    inline = [
      "while ((Get-Service RdAgent).Status -ne 'Running') { Start-Sleep -s 5 }",
      "while ((Get-Service WindowsAzureGuestAgent).Status -ne 'Running') { Start-Sleep -s 5 }",
      "& $env:SystemRoot\\System32\\Sysprep\\Sysprep.exe /oobe /generalize /quiet /quit /mode:vm",
      "while ($true) { $imageState = Get-ItemProperty HKLM:\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Setup\\State | Select ImageState; if($imageState.ImageState -ne 'IMAGE_STATE_GENERALIZE_RESEAL_TO_OOBE') { Write-Output $imageState.ImageState; Start-Sleep -s 10  } else { break } }"
    ]
  }
}
```

Let's look at what's happening here:

- First, we reference the `source` block that we defined before.
- We install Chocolatey using the [one-liner PowerShell command from the official docs](https://chocolatey.org/install#individual).
- We copy the `packages.config` file, an XML manifest containing a list of apps for Chocolatey to install, to the [temporary disk `D:` of the VM](https://docs.microsoft.com/en-us/azure/virtual-machines/managed-disks-overview#temporary-disk). Then we pass the manifest to the [`choco install`](https://docs.chocolatey.org/en-us/choco/commands/install) command. [When the command exits with the code `3010`, a reboot is required](https://docs.chocolatey.org/en-us/choco/commands/install#exit-codes). We make Packer aware of that by passing `3010` to the list of `valid_exit_codes`.
- For good measure, we reboot the VM.
- We run a custom PowerShell script to install the [Azure PowerShell modules](https://docs.microsoft.com/en-us/powershell/azure/install-az-ps?view=azps-7.3.2).
- [Finally, we generalize the image using Sysprep](https://www.packer.io/plugins/builders/azure/arm#windows)

The `packages.config` manifest looks like this:

```xml
<?xml version="1.0" encoding="utf-8"?>
<packages>
  <!-- PowerShell -->
  <!-- See https://docs.microsoft.com/en-us/powershell/scripting/install/installing-powershell-on-windows?view=powershell-7.2#install-the-msi-package-from-the-command-line -->
  <package id="powershell-core" installArguments="ADD_FILE_CONTEXT_MENU_RUNPOWERSHELL=1 ADD_EXPLORER_CONTEXT_MENU_OPENPOWERSHELL=1" />
  <package id="openssh" />
  <package id="curl" />
  <package id="wget" />

  <!-- Visual Studio 2022 Community -->
  <!-- See https://docs.microsoft.com/en-us/visualstudio/install/use-command-line-parameters-to-install-visual-studio?view=vs-2022#layout-command-and-command-line-parameters -->
  <!-- See https://docs.microsoft.com/en-us/visualstudio/install/workload-component-id-vs-community?view=vs-2022 -->
  <package id="visualstudio2022community" packageParameters="--add Microsoft.VisualStudio.Workload.Azure;includeRecommended --add Microsoft.VisualStudio.Workload.ManagedDesktop;includeRecommended --add Microsoft.VisualStudio.Workload.NetCrossPlat;includeRecommended --add Microsoft.VisualStudio.Workload.NetWeb;includeRecommended --add Microsoft.VisualStudio.Workload.Office;includeRecommended" />

  <!-- Editors -->
  <package id="notepadplusplus" />
  <package id="vscode" />

  <!-- System Administration -->
  <package id="sysinternals" />
  <package id="rsat" />

  <!-- Developer Tools -->
  <package id="azure-cli" />
  <package id="filezilla" />
  <package id="git" />
  <package id="microsoftazurestorageexplorer" />
  <package id="sql-server-management-studio" />
  <package id="postman" />

  <!-- Kubernetes -->
  <package id="kubernetes-cli" />
  <package id="kubernetes-helm" />

  <!-- Hashicorp -->
  <package id="packer" />
  <package id="terraform" />
  <package id="graphviz" />

  <!-- Browsers -->
  <package id="firefox" />
  <package id="googlechrome" />

  <!-- Common Apps -->
  <package id="7zip" />
  <package id="drawio" />
  <package id="foxitreader" />
  <package id="keepassxc" />
</packages>
```

Note that installing Chocolatey packages like that is a pretty naive approach that I wouldn't recommend for production-level scenarios. [Using the Chocolatey community repository has limits in terms of reliability, control, and trust.](https://docs.chocolatey.org/en-us/community-repository/community-packages-disclaimer). Also, any failing package installation breaks the entire build, so pinning versions is a good idea.

The `install-azure-powershell.ps1` provisioning script to install the Azure PowerShell modules looks like this:

```powershell
# See https://docs.microsoft.com/en-us/powershell/azure/install-az-ps-msi?view=azps-7.3.2#install-or-update-on-windows-using-the-msi-package

$ErrorActionPreference = "Stop"

$downloadUrl = "https://github.com/Azure/azure-powershell/releases/download/v7.3.2-March2022/Az-Cmdlets-7.3.2.35305-x64.msi"
$outFile = "D:\az_pwsh.msi" # temporary disk

Write-Host "Downloading $downloadUrl ..."
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
Invoke-WebRequest -Uri $downloadUrl -OutFile $outFile

Write-Host "Installing ..."
Start-Process "msiexec.exe" -Wait -ArgumentList "/package $outFile"

Write-Host "Done."
```

We only covered two methods for installing software into the golden image. Often it doesn't even make sense to bake apps into your golden image, e.g., when app deployments frequently change. For these scenarios, other solutions like [Microsoft Endpoint Manager](https://docs.microsoft.com/en-us/mem/endpoint-manager-overview) or [MSIX App Attach](https://docs.microsoft.com/en-us/azure/virtual-desktop/what-is-app-attach) exist.

### Run Packer Locally

To run Packer locally, we use the following command (note the `.` at the very end):

```bash
packer build \
  -var "artifacts_resource_group=$(terraform output -raw packer_artifacts_resource_group)" \
  -var "build_resource_group=$(terraform output -raw packer_build_resource_group)" \
  -var "client_id=$(terraform output -raw packer_client_id)" \
  -var "client_secret=$(terraform output -raw packer_client_secret)" \
  -var "subscription_id=$(terraform output -raw packer_subscription_id)" \
  -var "tenant_id=$(terraform output -raw packer_tenant_id)" \
  -var "source_image_publisher=MicrosoftWindowsDesktop" \
  -var "source_image_offer=office-365" \
  -var "source_image_sku=win11-21h2-avd-m365" \
  -var "source_image_version=22000.556.220308" \
  .
```

With the `-var` option, we can set Packer variables. We pass the Terraform outputs that we configured earlier to Packer using command substitution.

## GitHub Actions

Next, we tie everything together using GitHub Actions. Add a workflow by adding a file named `.github/workflows/packer.yml`. We name the workflow and specify the events that trigger it:

```yml
name: Packer Windows 11

on:
  push:
    branches:
      - main
  schedule:
    - cron: 0 0 * * *
```

It's triggered whenever we push code to the `main` branch. We schedule to run the workflow daily at 0:00 UTC using the [POSIX cron syntax](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/crontab.html#tag_20_25_07).

Using workflow-level environment variables, we specify the desired source image publisher, offer, and SKU :

```yml
env:
  IMAGE_PUBLISHER: MicrosoftWindowsDesktop

  # With Office 365
  IMAGE_OFFER: office-365
  IMAGE_SKU: win11-21h2-avd-m365

  # Without Office 365
  #IMAGE_OFFER: windows-11
  #IMAGE_SKU: win11-21h2-avd
```

Here is a high-level overview of the workflow `jobs` where all the magic happens:

```yml
jobs:
  latest_windows_version:
    # Get latest Windows version from Azure

  check_image_exists:
    # Check if latest version has already been built

  packer:
    # Run Packer
```

Let's look into each job in detail next.

### Job: `latest_windows_version`

We use the `AZURE_CREDENTIALS` secret we defined earlier with Terraform to authenticate to Azure with the [Azure Login Action](https://github.com/Azure/login).

After, we use the [Azure CLI Action](https://github.com/Azure/cli) to run [the snippet to calculate the latest available image version](#get-the-latest-windows-11-version-available-on-azure).

To allow using the result from the other jobs, we define the `version` job output. It's set using the `echo "::set-output name=version::${latest_version}"` command.

```yml
latest_windows_version:
  name: Get latest Windows version from Azure
  runs-on: ubuntu-latest
  outputs:
    version: ${{ steps.get_latest_version.outputs.version }}
  steps:
    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: Get Latest Version
      id: get_latest_version
      uses: azure/CLI@v1
      with:
        azcliversion: 2.34.1
        inlineScript: |
          latest_version=$(
            az vm image list \
              --publisher "${IMAGE_PUBLISHER}" \
              --offer "${IMAGE_OFFER}" \
              --sku "${IMAGE_SKU}" \
              --all \
              --query "[*].version | sort(@)[-1:]" \
              --out tsv
          )
          echo "Publisher: ${IMAGE_PUBLISHER}"
          echo "Offer:     ${IMAGE_OFFER}"
          echo "SKU:       ${IMAGE_SKU}"
          echo "Version:   ${latest_version}"
          echo "::set-output name=version::${latest_version}"
```

### Job: `check_image_exists`

This job `needs` the `latest_windows_version` job to finish first because it depends on its `version` output.

Remember that we name our Packer images (`managed_image_name`) like `${sku}-${version}`. Using the `IMAGE_SKU` workflow environment variable and the `version` output from the `latest_windows_version` job, we can use the `az image show` command to check for existing images inside the Packer artifacts resource group.

Depending on whether or not an image exists, we set the job output `exists` to `true` or `false`.

```yml
check_image_exists:
  name: Check if latest version has already been built
  runs-on: ubuntu-latest
  needs: latest_windows_version
  outputs:
    exists: ${{ steps.get_image.outputs.exists }}
  steps:
    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: Check If Image Exists
      id: get_image
      uses: azure/CLI@v1
      with:
        azcliversion: 2.34.1
        inlineScript: |
          if az image show \
            --resource-group "${{ secrets.PACKER_ARTIFACTS_RESOURCE_GROUP }}" \
            --name "${IMAGE_SKU}-${{ needs.latest_windows_version.outputs.version }}"; then
            image_exists=true
          else
            image_exists=false
          fi
          echo "Image Exists: ${image_exists}"
          echo "::set-output name=exists::${image_exists}"
```

### Job: `packer`

This job `needs` the outputs from both previous jobs. It only runs `if` the `exists` output of the `check_image_exists` job equals `false`. Otherwise, it's skipped.

We use the [Checkout Action](https://github.com/actions/checkout) to make the code available to the workflow and validate the syntax of the Packer template by running `packer validate`.

Finally, we run `packer build`. This time we use environment variables in the form of `PKR_VAR_*` to set the Packer inputs opposed to the `-var` CLI option we used earlier when running Packer locally.

```yml
packer:
  name: Run Packer
  runs-on: ubuntu-latest
  needs: [latest_windows_version, check_image_exists]
  if: needs.check_image_exists.outputs.exists == 'false'
  steps:
    - name: Checkout Repository
      uses: actions/checkout@v2

    - name: Validate Packer Template
      uses: hashicorp/packer-github-actions@master
      with:
        command: validate
        arguments: -syntax-only

    - name: Build Packer Image
      uses: hashicorp/packer-github-actions@master
      with:
        command: build
        arguments: -color=false -on-error=abort
      env:
        PKR_VAR_client_id: ${{ secrets.PACKER_CLIENT_ID }}
        PKR_VAR_client_secret: ${{ secrets.PACKER_CLIENT_SECRET }}
        PKR_VAR_subscription_id: ${{ secrets.PACKER_SUBSCRIPTION_ID }}
        PKR_VAR_tenant_id: ${{ secrets.PACKER_TENANT_ID }}
        PKR_VAR_artifacts_resource_group: ${{ secrets.PACKER_ARTIFACTS_RESOURCE_GROUP }}
        PKR_VAR_build_resource_group: ${{ secrets.PACKER_BUILD_RESOURCE_GROUP }}
        PKR_VAR_source_image_publisher: ${{ env.IMAGE_PUBLISHER }}
        PKR_VAR_source_image_offer: ${{ env.IMAGE_OFFER }}
        PKR_VAR_source_image_sku: ${{ env.IMAGE_SKU }}
        PKR_VAR_source_image_version: ${{ needs.latest_windows_version.outputs.version }}
```

### Job: `cleanup`

If the pipeline crashes somewhere along the way, resources inside the `packer-build-rg` resource group may linger and incur costs. We want to define a job that purges everything from the resource group.

One way to achieve this is by starting a deployment to the resource group using an empty ARM template. First, we add an empty [Bicep](https://docs.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview?tabs=bicep) named `cleanup-resource-group.bicep`. We can then deploy the template using [`az deployment group create`](https://docs.microsoft.com/en-us/cli/azure/deployment/group?view=azure-cli-latest#az-deployment-group-create):

```yml
cleanup:
  name: Cleanup Packer Resources
  runs-on: ubuntu-latest
  needs: packer
  steps:
    - name: Checkout Repository
      uses: actions/checkout@v2

    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: Cleanup Resource Group
      uses: azure/CLI@v1
      with:
        azcliversion: 2.34.1
        inlineScript: |
          az deployment group create \
            --mode Complete \
            --resource-group "${{ secrets.PACKER_BUILD_RESOURCE_GROUP }}" \
            --template-file cleanup-resource-group.bicep
```

## What Do You Think?

Phew! That was quite a bit of work. [Here's one of the workflow runs](https://github.com/schnerring/packer-windows-avd/runs/5711503619) that took around one hour to complete:

![Sample workflow summary](./workflow-summary.png)

I especially like how the GitHub Actions part turned out. Leave a comment or at me on Twitter to let me know what you think about it!
