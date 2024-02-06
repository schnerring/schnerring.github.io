---
title:
  "Use Terraform to Deploy an Azure Kubernetes Service (AKS) Cluster, Traefik 2,
  cert-manager, and Let's Encrypt Certificates"
date: 2021-04-25T01:00:27+02:00
cover:
  src: cover.png
draft: false
comments: true
socialShare: true
tags:
  - AKS
  - Azure
  - Azure Kubernetes Service
  - cert-manager
  - Cloud
  - DevOps
  - Helm
  - k8s
  - Kubernetes
  - Let's Encrypt
  - Network
  - Reverse Proxy
  - Self-host
  - Terraform
  - Traefik
aliases:
  - /posts/use-terraform-to-deploy-an-azure-kubernetes-service-aks-cluster-traefik-2-cert-manager-and-lets-encrypt
  - /posts/use-terraform-to-deploy-an-azure-kubernetes-service-aks-cluster-traefik-2-cert-manager-and-lets-encrypt-certificates
---

In this post, we will deploy a simple
[Azure Kubernetes Service (AKS)](https://azure.microsoft.com/en-us/services/kubernetes-service/)
cluster from scratch. To expose our web services securely, we will install
Traefik 2 and configure [cert-manager](https://cert-manager.io/) to manage Let's
Encrypt certificates. The best part about it: we will do everything with
[Terraform](https://www.terraform.io/).

<!--more-->

## Prerequisites

Keep in mind that I have tested the steps that follow on Windows. So if you are
using another OS, you might have to modify them slightly. I also omit some
non-essential steps in between, so it helps if you are already familiar with
Azure, Kubernetes, and Terraform.

To follow along, you will need the following things:

- [An active Azure subscription](https://azure.microsoft.com/en-us/free/)
- [Install Terraform](https://www.terraform.io/downloads.html)
- [Install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/)
- A registered domain
- A DNS provider:
  - that
    [supports cert-manager DNS01 challenge validation](https://cert-manager.io/docs/configuration/acme/dns01/#supported-dns01-providers)
  - with API access, so we can manage DNS records with Terraform — I use
    Cloudflare

## Overview

Here is an outline of the steps required to build our solution:

1. [Setup Terraform](#step-1-setup-terraform)
2. [Create the AKS cluster](#step-2-create-the-aks-cluster)
3. [Deploy cert-manager](#step-3-deploy-cert-manager)
4. [Configure Let's Encrypt certificates](#step-4-configure-lets-encrypt-certificates)
5. [Deploy Traefik](#step-5-deploy-traefik)
6. [Deploy a demo application](#step-6-deploy-a-demo-application)

## Step 1: Setup Terraform

We will make use of Terraform providers to put everything together:

- [`azurerm`](https://registry.terraform.io/providers/hashicorp/azurerm/latest)
  to manage our AKS cluster
- [`helm`](https://registry.terraform.io/providers/hashicorp/helm/latest) to
  deploy cert-manager and Traefik
- [`kubernetes`](https://registry.terraform.io/providers/hashicorp/kubernetes/latest)
  to manage namespaces and deploy our demo app
- [`cloudflare`](https://registry.terraform.io/providers/cloudflare/cloudflare/latest)
  to manage DNS records

We add a `provider.tf` file with the following content:

```hcl
terraform {
  required_version = ">= 1.4.0"

  required_providers {
    azurerm = {
      source  = "azurerm"
      version = ">= 3.47.0"
    }

    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = ">= 4.1.0"
    }

    helm = {
      source  = "helm"
      version = ">= 2.9.0"
    }

    kubernetes = {
      source  = "kubernetes"
      version = ">= 2.18.1"
    }
  }
}

provider "azurerm" {
  features {}
}

provider "cloudflare" {}
```

For now, we only configured the `azurerm` and `cloudflare` providers. After
setting up the AKS cluster, we will configure the `helm` and `kubernetes`
providers.

I have opted to
[configure the `azurerm` provider with environment variables](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/guides/service_principal_client_secret#configuring-the-service-principal-in-terraform).
You might want to
[choose a different approach](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs#authenticating-to-azure)
depending on your needs.

To authenticate the `cloudflare` provider, I use a
[Cloudflare API Token](https://developers.cloudflare.com/api/tokens/create) with
`Edit Zone` permissions.

After, we make sure to run `terraform init` to get started.

## Step 2: Create the AKS cluster

Creating a production-ready AKS cluster is out of scope for this post, which
means that we will not delve too deep into AKS configuration. There are many
things we are skipping over, like
[backups](https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-storage)
and
[monitoring](https://docs.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-overview).
For a little more elaborate example, check out the
[official Terraform on Azure documentation](https://docs.microsoft.com/en-us/azure/developer/terraform/create-k8s-cluster-with-tf-and-aks).

We create a new file `k8s.tf` with the following content:

```hcl
resource "azurerm_resource_group" "k8s" {
  name     = "k8s-rg"
  location = var.location
}

resource "random_id" "random" {
  byte_length = 2
}

resource "azurerm_kubernetes_cluster" "k8s" {
  name                = "k8s-aks"
  resource_group_name = azurerm_resource_group.k8s.name
  location            = var.location
  tags                = var.tags

  dns_prefix = "k8saks${random_id.random.dec}"

  default_node_pool {
    name       = "default"
    node_count = 1
    vm_size    = "Standard_B2ms"
  }

  identity {
    type = "SystemAssigned"
  }
}
```

Note that I have defined the `var.location` and `var.tags` variables in a
separate
[variables.tf](https://github.com/schnerring/infrastructure-core/blob/v0.6.0/variables.tf)
file.

To be able to access the AKS cluster locally with `kubectl`, we define a
Terraform [`output`](https://www.terraform.io/docs/language/values/outputs.html)
in the `outputs.tf` file:

```hcl
output "kube_config" {
  value       = azurerm_kubernetes_cluster.k8s.kube_config_raw
  description = "kubeconfig for kubectl access."
  sensitive   = true
}
```

We have to set `sensitive = true` so our credentials will not get leaked, which
could happen if we later decide to run Terraform with GitHub Actions. We apply
our configuration by running Terraform:

```shell
terraform plan -out infrastructure.tfplan
terraform apply infrastructure.tfplan
```

After the deployment completes, we set up `kubectl` to be able to access our
cluster:

```shell
terraform output -raw kube_config > ~/.kube/config
```

We check the deployment by running `kubectl get all`:

```text
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.0.0.1     <none>        443/TCP   3m54s
```

## Step 3: Deploy cert-manager

To issue free [Let's Encrypt](https://letsencrypt.org/) certificates for the web
services we provide, the first thing we have to deploy is
[cert-manager](https://cert-manager.io/docs/installation/kubernetes/#installing-with-helm).
We need to configure the
[`helm`](https://registry.terraform.io/providers/hashicorp/helm/latest/docs#credentials-config)
provider first. While we are on it, we also configure the
[`kubernetes`](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs#authentication)
provider:

```hcl
provider "helm" {
  kubernetes {
    host = azurerm_kubernetes_cluster.k8s.kube_config.0.host

    client_certificate     = base64decode(azurerm_kubernetes_cluster.k8s.kube_config.0.client_certificate)
    client_key             = base64decode(azurerm_kubernetes_cluster.k8s.kube_config.0.client_key)
    cluster_ca_certificate = base64decode(azurerm_kubernetes_cluster.k8s.kube_config.0.cluster_ca_certificate)
  }
}

provider "kubernetes" {
  host = azurerm_kubernetes_cluster.k8s.kube_config.0.host

  client_certificate     = base64decode(azurerm_kubernetes_cluster.k8s.kube_config.0.client_certificate)
  client_key             = base64decode(azurerm_kubernetes_cluster.k8s.kube_config.0.client_key)
  cluster_ca_certificate = base64decode(azurerm_kubernetes_cluster.k8s.kube_config.0.cluster_ca_certificate)
}
```

Stacking the providers above with our managed Kubernetes cluster resources can
lead to errors and
[should be avoided](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs#stacking-with-managed-kubernetes-cluster-resources).
I will mention this once more at the [end of the post](#one-last-thing).

Next, we deploy cert-manager with Helm by adding the following Terraform code to
the `k8s.tf` file:

```hcl
resource "kubernetes_namespace" "cert_manager" {
  metadata {
    name = "cert-manager"
  }
}

resource "helm_release" "cert_manager" {
  name       = "cert-manager"
  repository = "https://charts.jetstack.io"
  chart      = "cert-manager"
  version    = "v1.14.1"
  namespace  = kubernetes_namespace.cert_manager.metadata.0.name

  set {
    name  = "installCRDs"
    value = "true"
  }
}
```

We then run `terraform apply` to deploy cert-manager. We then check our work by
running `kubectl get all --namespace cert-manager`, which should display
something like this:

```text
NAME                                           READY   STATUS    RESTARTS   AGE
pod/cert-manager-7998c69865-vrvrd              1/1     Running   0          75s
pod/cert-manager-cainjector-7b744d56fb-qb54k   1/1     Running   0          75s
pod/cert-manager-webhook-7d6d4c78bc-svzq5      1/1     Running   0          75s

NAME                           TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
service/cert-manager           ClusterIP   10.0.141.59   <none>        9402/TCP   75s
service/cert-manager-webhook   ClusterIP   10.0.18.192   <none>        443/TCP    75s

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/cert-manager              1/1     1            1           75s
deployment.apps/cert-manager-cainjector   1/1     1            1           75s
deployment.apps/cert-manager-webhook      1/1     1            1           75s

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/cert-manager-7998c69865              1         1         1       75s
replicaset.apps/cert-manager-cainjector-7b744d56fb   1         1         1       75s
replicaset.apps/cert-manager-webhook-7d6d4c78bc      1         1         1       75s
```

## Step 4: Configure Let's Encrypt Certificates

In Kubernetes, `Issuer`s are Kubernetes resources representing certificate
authorities able to generate certificates. We have to create a single
`ClusterIssuer`, a cluster-wide `Issuer`, using
[DNS01 challenge validation](https://cert-manager.io/docs/configuration/acme/dns01/)
with Let's Encrypt servers. As mentioned earlier, we will use Cloudflare, but
[many other DNS providers are supported](https://cert-manager.io/docs/configuration/acme/dns01/#supported-dns01-providers).

First, we need to create a Cloudflare API Token on the Cloudflare website, at
`User Profile` &rarr; `API Tokens`.
[The following permissions are required](https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/#api-tokens):

- `Zone` - `DNS` - `Edit`
- `Zone` - `Zone` - `Read`

To securely pass the token to Terraform, we create a `sensitive` variable. We
also add a variable containing the email address where Let's Encrypt can notify
us about expiring certificates:

```hcl
variable "letsencrypt_email" {
  type        = string
  description = "Email address that Let's Encrypt will use to send notifications about expiring certificates and account-related issues to."
  sensitive   = true
}

variable "letsencrypt_cloudflare_api_token" {
  type        = string
  description = "Cloudflare API token with Zone-DNS-Edit and Zone-Zone-Read permissions, which is required for DNS01 challenge validation."
  sensitive   = true
}
```

With Terraform, we then add the secret containing the API token to our cluster.
Since `ClusterIssuer` is a cluster-scoped resource, we need to make sure the
secret is globally available by putting it in the `cert-manager` namespace:

```hcl
resource "kubernetes_secret" "letsencrypt_cloudflare_api_token_secret" {
  metadata {
    name      = "letsencrypt-cloudflare-api-token-secret"
    namespace = kubernetes_namespace.cert_manager.metadata.0.name
  }

  data = {
    "api-token" = var.letsencrypt_cloudflare_api_token
  }
}
```

Next, we add the staging and production `ClusterIssuer` cert-manager CRD
resources that use Let's Encrypt servers. We will have to use regular Kubernetes
YAML manifests since we cannot deploy CRDs with the `kubernetes` provider. Here,
the
[`kubernetes_manifest`](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs/resources/manifest)
resource comes in. Together with the Terraform
[`yamldecode()`](https://www.terraform.io/docs/language/functions/yamldecode.html)
and
[`templatefile()`](https://www.terraform.io/docs/language/functions/templatefile.html)
functions, we get a pretty nice solution.

Let's start by defining the `letsencrypt-issuer.tpl.yaml` template file:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: ${name}
spec:
  acme:
    email: ${email}
    server: ${server}
    privateKeySecretRef:
      name: issuer-account-key-${name}
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: ${api_token_secret_name}
              key: ${api_token_secret_data_key}
```

We then create the staging and production `ClusterIssuer`s like so:

```hcl
resource "kubernetes_manifest" "letsencrypt_issuer_staging" {
  manifest = yamldecode(templatefile(
    "${path.module}/letsencrypt-issuer.tpl.yaml",
    {
      "name"                      = "letsencrypt-staging"
      "email"                     = var.letsencrypt_email
      "server"                    = "https://acme-staging-v02.api.letsencrypt.org/directory"
      "api_token_secret_name"     = kubernetes_secret.letsencrypt_cloudflare_api_token_secret.metadata.0.name
      "api_token_secret_data_key" = keys(kubernetes_secret.letsencrypt_cloudflare_api_token_secret.data).0
    }
  ))

  depends_on = [helm_release.cert_manager]
}

resource "kubernetes_manifest" "letsencrypt_issuer_production" {
  manifest = yamldecode(templatefile(
    "${path.module}/letsencrypt-issuer.tpl.yaml",
    {
      "name"                      = "letsencrypt-production"
      "email"                     = var.letsencrypt_email
      "server"                    = "https://acme-v02.api.letsencrypt.org/directory"
      "api_token_secret_name"     = kubernetes_secret.letsencrypt_cloudflare_api_token_secret.metadata.0.name
      "api_token_secret_data_key" = keys(kubernetes_secret.letsencrypt_cloudflare_api_token_secret.data).0
    }
  ))

  depends_on = [helm_release.cert_manager]
}
```

Now we `terraform apply` the changes.

## Step 5: Deploy Traefik

To manage external access to our Kubernetes cluster, we need to configure
Kubernetes
[`Ingress` resources](https://kubernetes.io/docs/concepts/services-networking/ingress/).
To satisfy an `Ingress`, we first need to configure an
[Ingress Controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/).
We will use Traefik for this.

To manage ingress, we could also use the Traefik `IngressRoute` CRD. At the time
of this writing,
[cert-manager cannot directly interface with Traefik CRDs](https://doc.traefik.io/traefik/providers/kubernetes-crd/#letsencrypt-support-with-the-custom-resource-definition-provider),
so we would have to manage `Certificate` and `Secret` resources manually, which
is cumbersome.

We add the following to the `k8s.tf` file:

```hcl
resource "kubernetes_namespace" "traefik" {
  metadata {
    name = "traefik"
  }
}

resource "helm_release" "traefik" {
  name       = "traefik"
  repository = "https://helm.traefik.io/traefik"
  chart      = "traefik"
  version    = "26.0.0"
  namespace  = kubernetes_namespace.traefik.metadata.0.name

  set {
    name  = "ports.web.redirectTo.port"
    value = "websecure"
  }

  # Trust private AKS IP range
  set {
    name  = "additionalArguments"
    value = "{--entryPoints.websecure.forwardedHeaders.trustedIPs=10.0.0.0/8}"
  }
}
```

Setting `ports.web.redirectTo.port` to `websecure` forces all HTTP traffic to be
redirected to HTTPS.

To
[configure Traefik to trust forwarded headers](https://doc.traefik.io/traefik/v2.3/routing/entrypoints/#forwarded-headers)
from Azure, we set
`entryPoints.websecure.forwardedHeaders.trustedIPs=10.0.0.0/8`.

After running `terraform apply`, we check the deployment by running
`kubectl get all --namespace traefik`:

```text
NAME                           READY   STATUS    RESTARTS   AGE
pod/traefik-6b6767d778-hxzzw   1/1     Running   0          68s

NAME              TYPE           CLUSTER-IP   EXTERNAL-IP      PORT(S)                      AGE
service/traefik   LoadBalancer   10.0.247.4   51.103.157.225   80:32468/TCP,443:31284/TCP   69s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/traefik   1/1     1            1           69s

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/traefik-6b6767d778   1         1         1       69s
```

Next, we add a DNS record with the IP of our Traefik service. To obtain the
external IP address of the service, we leverage the `kubernetes_service`
[Data Source](https://www.terraform.io/docs/language/data-sources/index.html) of
the `kubernetes` provider. We then add the DNS record `k8s.schnerring.net`
pointing to the external IP of Traefik.

Let us update the `k8s.tf` file accordingly and `terraform apply` the changes:

```hcl
data "kubernetes_service" "traefik" {
  metadata {
    name      = helm_release.traefik.name
    namespace = helm_release.traefik.namespace
  }
}

resource "cloudflare_record" "traefik" {
  zone_id = cloudflare_zone.schnerring_net.id
  name    = "k8s"
  type    = "A"
  value   = data.kubernetes_service.traefik.status.0.load_balancer.0.ingress.0.ip
  proxied = true
}
```

How awesome is that?

## Step 6: Deploy a Demo Application

We are almost at the finish line. All that is missing is reaping the fruit of
our hard labor. To create a simple demo, we will use the
[nginxdemos/hello](https://hub.docker.com/r/nginxdemos/hello/) image and make it
available at `https://hello.k8s.schnerring.net/`.

> I moved the demo to
> [https://hello.schnerring.net](https://hello.schnerring.net). I put my AKS
> cluster behind Cloudflare and the free universal SSL certificate only supports
> subdomains (`sub.schnerring.net`) but not subsubdomains
> (`sub.sub.schnerring.net`).

To make it happen, we add a `kubernetes_namespace`, `kubernetes_deployment`,
`kubernetes_service`, and `kubernetes_ingress` resource to a new `hello.tf`
file:

```hcl
resource "kubernetes_namespace" "hello" {
  metadata {
    name = "hello"
  }
}

resource "kubernetes_deployment" "hello" {
  metadata {
    name      = "hello-deploy"
    namespace = kubernetes_namespace.hello.metadata.0.name

    labels = {
      app = "hello"
    }
  }

  spec {
    replicas = 2

    selector {
      match_labels = {
        app = "hello"
      }
    }

    template {
      metadata {
        labels = {
          app = "hello"
        }
      }

      spec {
        container {
          image = "nginxdemos/hello"
          name  = "hello"

          port {
            container_port = 80
          }
        }
      }
    }
  }
}

resource "kubernetes_service" "hello" {
  metadata {
    name      = "hello-svc"
    namespace = kubernetes_namespace.hello.metadata.0.name
  }

  spec {
    selector = {
      app = kubernetes_deployment.hello.metadata.0.labels.app
    }

    port {
      port        = 80
      target_port = 80
    }
  }
}

resource "kubernetes_ingress_v1" "hello" {
  metadata {
    name      = "hello-ing"
    namespace = kubernetes_namespace.hello.metadata.0.name
    annotations = {
      "cert-manager.io/cluster-issuer"           = "letsencrypt-staging"
      "traefik.ingress.kubernetes.io/router.tls" = "true"
    }
  }

  spec {
    rule {
      host = "hello.k8s.schnerring.net"

      http {
        path {
          path = "/"

          backend {
            service {
              name = "hello-svc"

              port {
                number = 80
              }
            }
          }
        }
      }
    }

    tls {
      hosts       = ["hello.k8s.schnerring.net"]
      secret_name = "hello-tls-secret"
    }
  }
}
```

After running `terraform apply` again, we should be able to visit the demo site
`https://hello.k8s.schnerring.net/`:

![nginx Demo](nginx-demo.png)

To verify the HTTPS redirect works, we run
`curl -svDL http://hello.k8s.schnerring.net` (PowerShell), or
`curl -sLD - http://hello.k8s.schnerring.net` (Bash):

```text
...
< HTTP/1.1 301 Moved Permanently
< Location: https://hello.k8s.schnerring.net/
...
```

To get rid of the certificate warning, set
`"cert-manager.io/cluster-issuer" = "letsencrypt-production"`. But be aware of
[rate limits that apply to the Let's Encrypt production API](https://letsencrypt.org/docs/rate-limits/)!

## One Last Thing

If we want to tear down the cluster and rebuild, we cannot achieve this in _one_
`terraform apply` operation. The reason is that the `kubernetes` provider
requires an operational cluster during the `terraform plan` phase. On top of
that, any CRDs we deploy have to be available during `terraform plan`, too.

So when rebuilding, we would first create the AKS cluster and deploy
cert-manager and then apply the rest:

```shell
terraform destroy -target "azurerm_resource_group.k8s"

terraform plan -out infrastructure.tfplan -target "helm_release.cert_manager"
terraform apply infrastructure.tfplan

terraform plan -out infrastructure.tfplan
terraform apply infrastructure.tfplan
```

I already mentioned this earlier. The need for the workaround above originates
from stacking Kubernetes cluster infrastructure with Kubernetes resources which
the
[official Kubernetes provider documentation discourages](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs#stacking-with-managed-kubernetes-cluster-resources).
Adhering to the docs and separating cluster and Kubernetes resources into
different modules will probably save you a headache!

Other than that, we created a pretty cool solution, fully managed by Terraform,
did we not?

You can find all the code on GitHub in my
[schnerring/infrastructure-core repository](https://github.com/schnerring/infrastructure-core/),
which is evolving continuously.

I have updated and refactored my Terraform code several times since I published
this post.
[The original, all-in-one but outdated Terraform code can be found here.](https://github.com/schnerring/infrastructure-core/blob/v0.5.0/k8s.tf)

The up-to-date code looks slightly different and can be found here:

- [aks.tf](https://github.com/schnerring/infrastructure-core/blob/v0.6.0/core/aks.tf)
- [cert-manager.tf](https://github.com/schnerring/infrastructure-core/blob/v0.6.0/kubernetes/cert-manager.tf)
- [traefik-v2.tf](https://github.com/schnerring/infrastructure-core/blob/v0.6.0/kubernetes/traefik-v2.tf)
- [letsencrypt.tf](https://github.com/schnerring/infrastructure-core/blob/v0.6.0/kubernetes/letsencrypt.tf)
- [hello.tf](https://github.com/schnerring/infrastructure-core/blob/v0.6.0/kubernetes/hello.tf)
