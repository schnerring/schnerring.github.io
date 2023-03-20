---
title: Plausible Analytics
draft: false
---

I self-host [Plausible Analytics](https://plausible.io/), an open-source, privacy-respecting, and lightweight web analytics tool. I use [Terraform](https://www.terraform.io/) and [Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/) to host it. Check my [Terraform configuration for Plausible on GitHub](https://github.com/schnerring/infrastructure-core/blob/main/kubernetes/plausible.tf) if you want to learn how.

Plausible doesn't collect personal data. It doesn't require cookies and is fully compliant with the GDPR, CCPA, and PECR.

[Here you can find a standalone public dashboard.](https://plausible.schnerring.net/schnerring.net)

[Thanks to Marko Saric for inspiring me to create this stats page.](https://markosaric.com/stats/)

{{< plausible-stats >}}
