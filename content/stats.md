---
title: Plausible Analytics
draft: false
---

I self-host [Plausible Analytics](https://plausible.io/), an open-source, privacy-respecting, and lightweight website analytics tool. I use [Terraform](https://www.terraform.io/) and [Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/) to host it. Check my [Terraform configuration for Plausible on GitHub](https://github.com/schnerring/infrastructure/blob/main/plausible.tf) if you want to learn how.

Plausible doesn't collect personal data. It doesn't require cookies and is fully compliant with the GDPR, CCPA, and PECR.

[A standalone public dashboard is also available.](https://plausible.schnerring.net/schnerring.net)

{{< plausible-stats >}}
