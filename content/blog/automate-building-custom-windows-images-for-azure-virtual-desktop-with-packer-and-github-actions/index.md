---
title: Automate Building Custom Windows Images For Azure Virtual Desktop With Packer And GitHub Actions
date: "2022-02-25T04:04:20+01:00"
draft: true
comments: true
socialShare: true
toc: false
#cover:
#src: cover.png
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

One important task of managing [Azure Virtual Desktop (AVD)](https://azure.microsoft.com/en-us/services/virtual-desktop/) is keeping it up-to-date. One of many strategies is to periodically build a "golden" image and re-redeploy AVD session host VMs using the new image. In this post, I'll demonstrate using [Packer](https://www.packer.io/) and [GitHub Actions](https://github.com/features/actions) to automatically build up-to-date images and push them to Azure.

```powershell
Get-AzVMImage `
  -Location "Switzerland North" `
  -Publisher "MicrosoftWindowsDesktop" `
  -Offer windows-11 `
  -Sku win11-21h2-avd
```
