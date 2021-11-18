---
title: "Use Terraform to Deploy the Remark42 Commenting System to Kubernetes and Integrate it with a Hugo Website"
date: 2021-05-09T09:31:13+02:00
cover:
  src: "img/cover.svg"
draft: false
comments: true
tags:
  - hugo
  - k8s
  - kubernetes
  - remark42
  - terraform
---

Building upon our previous work, we will deploy [Remark42](https://remark42.com/) on Kubernetes with [Terraform](https://www.terraform.io/) and integrate it with your existing [Hugo](https://gohugo.io/) website. Make sure to check out my previous posts about [creating a Hugo Website](../create-a-hugo-website-with-github-pages-github-actions-and-cloudflare) and [deploying an Azure Kubernetes Service cluster](../use-terraform-to-deploy-an-azure-kubernetes-service-aks-cluster-traefik-2-cert-manager-and-lets-encrypt) if you haven't already.

<!--more-->

## About Remark42

> Remark42 is a self-hosted, lightweight, and simple (yet functional) commenting system, which doesn't spy on users.

I like simplicity, I am a privacy enthusiast, and I build this blog with this in mind. More popular, hands-off solutions like [Disqus](https://disqus.com/) offer easier integration and more sophisticated features, like automated spam moderation and advertising. But for my intents, it's too bloated and invasive. For low-traffic websites like mine, Remark42 is just the better fit.

## Preparation

Besides a Hugo website and a Kubernetes cluster, you will have to install the following software on your workstation:

- [Terraform](https://www.terraform.io/downloads.html)
- [kompose](https://kompose.io/installation/) (optional)

## Converting the original `docker-compose.yml`

The [official Remark42 repository](https://github.com/umputun/remark42) includes a [`docker-compose.yml`](https://github.com/umputun/remark42/blob/master/docker-compose.yml) that we can download:

```shell
curl -O https://raw.githubusercontent.com/umputun/remark42/master/docker-compose.yml
```

We then run `kompose convert` to generate regular Kubernetes YAML manifests from the `docker-compose.yml` file:

```text
Kubernetes file "remark-deployment.yaml" created
Kubernetes file "remark-claim0-persistentvolumeclaim.yaml" created
```

To figure out what resources we need to create, we use the generated manifests as a starting point.

## Create a Namespace

First, we create the [`Namespace`](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) where our Remark42 resources will reside. We add the following Terraform code to a new file named `remark42.yml`:

```hcl
resource "kubernetes_namespace" "remark42" {
  metadata {
    name = "remark42"
  }
}
```

Deploy the changes by running:

```shell
terraform plan -out infrastructure.tfplan
terraform apply infrastructure.tfplan
```

## Create the Persistent Volume Claim

To enable persistence for Remark42, we create a [`PersistentVolumeClaim`](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) (PVC) resource. Using the generate `remark-claim0-persistentvolumeclaim.yaml` file as a blueprint, we can easily derive the Terraform equivalent from it and add it to the `remark42.tf` file:

```hcl
resource "kubernetes_persistent_volume_claim" "remark42" {
  metadata {
    name      = "remark42-pvc"
    namespace = kubernetes_namespace.remark42.metadata.0.name
    labels = {
      "app" = "remark42-pvc"
    }
  }

  spec {
    access_modes       = ["ReadWriteOnce"]
    storage_class_name = "azurefile"

    resources {
      requests = {
        "storage" = "1Gi"
      }
    }
  }
}
```

AKS comes pre-configured with multiple [`StorageClass`es](https://kubernetes.io/docs/concepts/storage/storage-classes/). Here, I use the `azurefile` storage class to dynamically provision a persistent volume with [Azure Files](https://docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv). At the time of this writing, I use a `B2ms`-sized VM as a node which is limited to four data disks. Using Azure Files whenever possible helps me circumventing this limitation.

## Create ConfigMap and Secret

To store configuration parameters for our Remark42 deployment, we make use of Kubernetes [`ConfigMap`](https://kubernetes.io/docs/concepts/configuration/configmap/) and [`Secret`](https://kubernetes.io/docs/concepts/configuration/secret/) resources.

To store sensitive values, we use `Secret`:

```hcl
resource "kubernetes_secret" "remark42" {
  metadata {
    name      = "remark42-secret"
    namespace = kubernetes_namespace.remark42.metadata.0.name
  }

  data = {
    "SECRET" = random_password.remark42_secret.result
  }
}
```

For non-sensitive stuff, we use `ConfigMap`:

```hcl
resource "kubernetes_config_map" "remark42" {
  metadata {
    name      = "remark42-cm"
    namespace = kubernetes_namespace.remark42.metadata.0.name
  }

  data = {
    "REMARK_URL" = "https://${cloudflare_record.remark42.hostname}"
    "SITE"       = "schnerring.net"
  }
}
```

For this post, I have kept the configuration to a minimum:

- `REMARK_URL`: URL to Remark42 server
- `SITE`: site name(s)
- `SECRET`: secret key

Check the [official documentation](https://remark42.com/docs) or [my code on GitHub](https://github.com/schnerring/infrastructure/blob/v0.2.0/remark42.tf) for more configuration options.

You might have noticed that I reference a `cloudflare_record` in the `REMARK_URL` part. That's because I also manage my DNS records with Terraform. The DNS record for `remark42.schnerring.net` pointing to the DNS record of my cluster looks like this:

```hcl
resource "cloudflare_record" "remark42" {
  zone_id = cloudflare_zone.schnerring_net.id
  name    = "remark42"
  type    = "CNAME"
  value   = cloudflare_record.traefik.hostname
  proxied = true
}
```

## Create the Deployment

Next, we add the [`Deployment`](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) to the `remark42.tf` file, using the `remark-deployment.yaml` file as a model. We map the previously defined configuration parameters to environment variables and mount the PVC to `/srv/var`.

```hcl
resource "kubernetes_deployment" "remark42" {
  metadata {
    name      = "remark42-deploy"
    namespace = kubernetes_namespace.remark42.metadata.0.name
    labels = {
      "app" = "remark42"
    }
  }
  spec {
    replicas = 1

    selector {
      match_labels = {
        "app" = "remark42"
      }
    }

    strategy {
      type = "Recreate"
    }

    template {
      metadata {
        labels = {
          "app" = "remark42"
        }
      }

      spec {
        hostname       = "remark42"
        restart_policy = "Always"

        container {
          name  = "remark42"
          image = "umputun/remark42:${var.remark42_image_version}"

          port {
            container_port = 8080
          }

          env {
            name = "REMARK_URL"

            value_from {
              config_map_key_ref {
                key  = "REMARK_URL"
                name = "remark42-cm"
              }
            }
          }

          env {
            name = "SECRET"

            value_from {
              secret_key_ref {
                key  = "SECRET"
                name = "remark42-secret"
              }
            }
          }

          env {
            name = "SITE"

            value_from {
              config_map_key_ref {
                key  = "SITE"
                name = "remark42-cm"
              }
            }
          }

          volume_mount {
            mount_path = "/srv/var"
            name       = "remark42-vol"
          }
        }

        volume {
          name = "remark42-vol"

          persistent_volume_claim {
            claim_name = "remark42-pvc"
          }
        }
      }
    }
  }
}
```

## Create the Service

To expose our deployment as a network service, we create a [`Service`](https://kubernetes.io/docs/concepts/services-networking/service/) resource by adding the following to the `remark42.tf` file:

```hcl
resource "kubernetes_service" "remark42" {
  metadata {
    name      = "remark42-svc"
    namespace = kubernetes_namespace.remark42.metadata.0.name
  }

  spec {
    selector = {
      "app" = "remark42"
    }

    port {
      name        = "http"
      port        = 80
      target_port = 8080
    }
  }
}
```

## Create the Ingress

I use [Traefik 2](https://doc.traefik.io/traefik/) as [`Ingress Controller`](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) in combination with the [Traefik Kubernetes Ingress provider](https://doc.traefik.io/traefik/providers/kubernetes-ingress/). To manage Let's Encrypt certificates, I use [cert-manager](https://cert-manager.io/docs/). So depending on your cluster configuration, the following steps might differ.

Let us add an [`Ingress`](https://kubernetes.io/docs/concepts/services-networking/ingress/) to the `remark42.tf` file, to expose our service to the world:

```hcl
resource "kubernetes_ingress" "remark42" {
  metadata {
    name      = "remark42-ing"
    namespace = kubernetes_namespace.remark42.metadata.0.name
    annotations = {
      "cert-manager.io/cluster-issuer"           = "letsencrypt-production"
      "traefik.ingress.kubernetes.io/router.tls" = "true"
    }
  }

  spec {
    rule {
      host = cloudflare_record.remark42.hostname

      http {
        path {
          path = "/"

          backend {
            service_name = "remark42-svc"
            service_port = 80
          }
        }
      }
    }

    tls {
      hosts       = [cloudflare_record.remark42.hostname]
      secret_name = "remark42-tls-secret"
    }
  }
}
```

Now run `terraform apply` to deploy everything. Note that we are using `letsencrypt-staging` as cluster issuer. We will have to change this to `letsencrypt-production` once we finished testing.

Browse to the demo site at [https://remark42.schnerring.net/web](https://remark42.schnerring.net/web) and check whether Remark42 works. If you want to post a comment on the demo site, make sure to add the `remark` site ID to the `SITE` environment variable, separated by a `,` (i.e., `schnerring.net,remark`).

## Integrate Remark42 with Hugo

To add the Remark42 comment widget to our Hugo site, we have to integrate it with our theme. At the time of this writing, I use the [Hello Friend](https://github.com/panr/hugo-theme-hello-friend/) theme, which includes a [partial template](https://gohugo.io/templates/partials/) for the comment section. We add the Remark42 widget to the `layouts/partials/comments.html` file:

```html
<div id="remark42"></div>

<script>
  var remark_config = {
    host: "https://remark42.schnerring.net",
    site_id: "schnerring.net",
    theme: getHelloFriendTheme(),
    show_email_subscription: false,
  };
</script>

<script>
  !(function (e, n) {
    for (var o = 0; o < e.length; o++) {
      var r = n.createElement("script"),
        c = ".js",
        d = n.head || n.body;
      "noModule" in r ? ((r.type = "module"), (c = ".mjs")) : (r.async = !0),
        (r.defer = !0),
        (r.src = remark_config.host + "/web/" + e[o] + c),
        d.appendChild(r);
    }
  })(remark_config.components || ["embed"], document);
</script>
```

You can find more configuration options for the widget in the [Remark42 GitHub README](https://github.com/umputun/remark42#comments).

Next, we implement the `getHelloFriendTheme()` function, so Remark42 loads the correct theme. The Hello Friend theme [stores the current theme in the local storage of the browser](https://github.com/panr/hugo-theme-hello-friend/blob/master/assets/js/theme.js). Knowing that, implementing the function is pretty straight forward:

```html
<script>
  const defaultTheme = "light"; // same as defaultTheme in config.toml
  function getHelloFriendTheme() {
    const theme = localStorage && localStorage.getItem("theme");
    if (!theme) {
      return defaultTheme;
    } else {
      return theme;
    }
  }
</script>
```

The last thing we have to take care of is to also toggle the Remark42 theme, when clicking the theme toggle button of the Hello Friend theme. To do so, we register an additional `click` event handler to the `.theme-toggle` button and call the [`window.REMARK42.changeTheme()` function](https://github.com/umputun/remark42#themes):

```html
<script>
  const themeToggle = document.querySelector(".theme-toggle");
  themeToggle.addEventListener("click", () => {
    setTimeout(() => window.REMARK42.changeTheme(getHelloFriendTheme()), 10);
  });
</script>
```

Note that we wait 10 ms before reading from local storage to avoid race conditions. All we have to do now is to enable comments by setting `comments: true` via [Hugo Front Matter](https://gohugo.io/content-management/front-matter/).

## What Do You Think?

You can find all the code on my GitHub. I also tagged the commits to make it easier to find the code for future reference:

- [schnerring/infrastructure](https://github.com/schnerring/infrastructure/blob/v0.2.0/remark42.tf) (Terraform, tag `v0.2.0`)
- [schnerring/schnerring.github.io](https://github.com/schnerring/schnerring.github.io/blob/v1.1.0/layouts/partials/comments.html) (Hugo, tag `v1.1.0`)

Any feedback in the comments below is appreciated.
