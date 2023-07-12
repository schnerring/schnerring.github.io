---
title: "Use PowerShell to Request Quotes for Azure Reservations"
date: "2022-04-26T04:14:21+02:00"
draft: false
comments: true
socialShare: true
toc: true
cover:
  src: cover.png
tags:
  - Azure
  - Azure PowerShell
  - Azure Reservations
  - Azure Virtual Machines
  - PowerShell
---

Over the weekend, I was on a quest to find the cheapest available Azure Virtual
Machine to house my Azure Kubernetes Service cluster. I previously wrote about
[optimizing storage cost with Azure Kubernetes Service (AKS)](/blog/reduce-storage-costs-when-deploying-azure-kubernetes-service-clusters-with-terraform),
where I talk about the importance of selecting VMs that support
[ephemeral OS disks](https://docs.microsoft.com/en-us/azure/virtual-machines/ephemeral-os-disks).
But what VM type and region should I choose? Searching through reservation
offers in the Azure Portal turned out to be tedious. This time, I'll demonstrate
how to find the best offers using PowerShell and export the results to a CSV
file.

<!--more-->

[As always, you can find the source code on my GitHub.](https://github.com/schnerring/infrastructure-core/blob/main/Get-VmReservationQuotes.ps1)

## So Many Choices

My AKS cluster hosts a handful of services and doesn't need a lot of resources.
Two vCPUs, eight gigs of RAM, and
[Premium Storage](https://docs.microsoft.com/en-us/azure/virtual-machines/premium-storage-performance)
are sufficient. The [analytics data](/stats/) suggests that most of my readers
are from the US and Europe, so any region between the eastern US and western
Europe will do:

```text
canadaeast
canadacentral
eastus
eastus2
northeurope
westeurope
ukwest
uksouth
switzerlandnorth
germanywestcentral
francecentral
norwayeast
swedencentral
```

We can browse offers in the Azure Portal under
{{< breadcrumb "Reservations" "Add" "Virtual machine" >}}. However, we can only
request one quote for a reservation at a time.

{{< video src="./browse-azure-reservations" autoplay="true" controls="false" loop="true" >}}

Clicking through the VMs for each region would take a long time. 😴 It certainly
would have been faster than writing this blog article. But hacking together a
PowerShell script was way more fun! I used a bunch of heuristics that "just
work", so it's not perfect and causes some **400 Bad Request** responses. But it
gets the job done.

## Requirements

We'll need the following things to get started:

- [PowerShell version 7.2.2 or later](https://docs.microsoft.com/en-us/powershell/scripting/install/installing-powershell?view=powershell-7.2)
- [Azure Az PowerShell module](https://docs.microsoft.com/en-us/powershell/azure/what-is-azure-powershell?view=azps-7.4.0)
- Active Azure subscription

## Parameters and Helper Functions

Let's create a file named `Get-VmReservationQuotes.ps1` and add the following
parameters to it:

```powershell
param (
  [Parameter(Mandatory, HelpMessage="Azure subscription ID")]
  [string]
  $SubscriptionId,

  [Parameter(HelpMessage="Reservation term duration in years (1 or 3)")]
  [int]
  $TermYears = 3,

  [Parameter(HelpMessage="Azure regions")]
  [string[]]
  $Locations = @(
    "canadaeast",
    "canadacentral",
    "eastus",
    "eastus2",
    "northeurope",
    "westeurope",
    "ukwest",
    "uksouth",
    "switzerlandnorth",
    "germanywestcentral",
    "francecentral",
    "norwayeast",
    "swedencentral"
  ),

  [Parameter(HelpMessage="CSV output filename")]
  [string]
  $OutFile = "VmReservationQuotes.csv"
)
```

This allows us to call the script like this:

```powershell
.\Get-VmReservationQuotes.ps1 -SubscriptionId "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

We also add some helper functions that we'll re-use throughout the script:

```powershell
function Get-VmLocation ($vmSize) {
  return $vmSize.Locations[0].ToLower()
}

function Test-VmCapability ($vmSize, $capabilityName, $capabilityValue) {
  foreach($capability in $vmSize.Capabilities)
  {
    if ($capability.Name -eq $capabilityName -and $capability.Value -eq $capabilityValue)
    {
      return $true
    }
  }
  return $false
}
```

`Get-VmLocation` extracts the location from a VM object returned by the
[`Get-AzComputeResourceSku`](https://docs.microsoft.com/en-us/powershell/module/az.compute/get-azcomputeresourcesku?view=azps-7.4.0)
cmdlet. With `Test-VmCapability`, we can check whether the VM supports a
specific feature like ephemeral OS disks:

```powershell
Test-VmCapability $vmSize "EphemeralOSDiskSupported" "True"
```

## Query and Filter Available VM Types

Using `Get-AzComputeResourceSku` and our helper functions, we query Azure for
all the available VM types and apply our filter heuristics:

```powershell
$vmSizes = @()

foreach ($vmSize in Get-AzComputeResourceSku) {
  # Skip `availibilitySets`, `disks`, etc.
  if (-not ($vmSize.ResourceType -eq 'virtualMachines')) {
    continue
  }

  # Skip unavailable offers
  if ($vmSize.Restrictions.Count -gt 0) {
    continue
  }

  # Filter locations
  if (-not $Locations.Contains((Get-VmLocation $vmSize))) {
    continue
  }

  # Select D-Series VMs
  if (-not $vmSize.Name.StartsWith("Standard_D")) {
    continue
  }

  # Exclude confidential VMs
  if ($vmSize.Name.StartsWith("Standard_DC")) {
    continue
  }

  # Exclude memory-optimized VMs
  if ($vmSize.Name.StartsWith("Standard_D11") -or $vmSize.Name.StartsWith("Standard_DS11")) {
    continue
  }

  if (-not (Test-VmCapability $vmSize "EphemeralOSDiskSupported" "True")) {
    continue
  }

  if (-not (Test-VmCapability $vmSize "PremiumIO" "True")) {
    continue
  }

  if (-not (Test-VmCapability $vmSize "vCPUs" "2")) {
    continue
  }

  $vmSizes += $vmSize
}
```

Pretty straightforward, right?

## Request Quotes

Now we put the filtered `$vmSizes` through another loop and query Azure for
quotes using
[`Get-AzReservationQuote`](https://docs.microsoft.com/en-us/powershell/module/az.reservations/get-azreservationquote?view=azps-7.4.0).
After, we calculate the monthly price and export the results as CSV.

```powershell
$i = 0
foreach ($vmSize in $vmSizes) {
  $location = Get-VmLocation $vmSize
  $displayName = "$($vmSize.Name)-$location"

  # Progress bar
  $i++
  $percent = [Math]::Floor(($i / $vmSizes.Count) * 100)
  Write-Progress -Activity "Requesting VM quotes" -Status "$percent% $displayName" -PercentComplete $percent

  $quote = Get-AzReservationQuote `
    -ReservedResourceType "VirtualMachines" `
    -Sku $vmSize.Name `
    -Location $location `
    -Term "P${TermYears}Y" `
    -BillingScopeId $SubscriptionId `
    -Quantity 1 `
    -AppliedScopeType Shared `
    -DisplayName "$displayName"

  # BillingCurrencyTotal is a JSON string, e.g.:
  # {
  #   "currencyCode": "CHF",
  #   "amount": 1130
  # }

  # Extract amount value `1130` from JSON
  $billingCurrencyTotal = $quote.BillingCurrencyTotal | ConvertFrom-Json
  $termPrice = $billingCurrencyTotal.amount

  @{
    Name = $vmSize.Name;
    Location = $location;
    PricePerMonth = $termPrice / ($TermYears * 12)
  } | Export-Csv -Path "${OutFile}" -NoTypeInformation -Delimiter ";" -Append
}
```

Besides the progress bar, the trickiest part is extracting the price from the
`BillingCurrencyTotal` multi-string field via regular expression.

## And The Winner Is 🥁

<!-- markdownlint-disable MD033 -->

|              |                            |
| ------------ | -------------------------- |
| VM           | `Standard_D2as_v4`         |
| Location     | `eastus`                   |
| Monthly Cost | `CHF 25.94`<br>`USD 27.10` |

<!-- markdownlint-enable MD033 -->

Pretty cool, eh? Do you have any thoughts or questions? Let me know in the
comments or on Twitter. Feedback is always welcome!
