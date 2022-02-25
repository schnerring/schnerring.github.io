---
title: "Packer Images"
date: "2022-02-25T04:04:20+01:00"
draft: true
comments: true
socialShare: true
toc: false
#cover:
#src: cover.png
---

```powershell
Get-AzVMImage `
  -Location "Switzerland North" `
  -Publisher "MicrosoftWindowsDesktop" `
  -Offer windows-11 `
  -Sku win11-21h2-avd
```
