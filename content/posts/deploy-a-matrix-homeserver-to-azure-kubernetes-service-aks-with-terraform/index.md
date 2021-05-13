---
title: "Deploy a Matrix Homeserver to Azure Kubernetes Service (AKS) with Terraform"
date: 2021-05-14T01:13:33+02:00
cover: "img/cover.svg"
useRelativeCover: true
hideReadMore: true
comments: true
tags:
  - aks
  - homeserver
  - k8s
  - kubernetes
  - matrix
  - synapse
  - synapse admin ui
  - terraform
---

Did you ever think about running a Matrix homeserver? In this post, we will set one up on the [Azure Kubernetes Service](https://azure.microsoft.com/en-us/services/kubernetes-service/) (AKS). We will use the reference homeserver implementation, which is [Synapse](https://github.com/matrix-org/synapse/) from the folks at [matrix.org](https://matrix.org/). This post focuses on the Kubernetes stuff, so we will keep the Synapse configuration to a minimum. Things like [federation](https://github.com/matrix-org/synapse/blob/master/docs/federate.md), [delegation](https://github.com/matrix-org/synapse/blob/master/docs/delegate.md) and [PostgreSQL](https://github.com/matrix-org/synapse/blob/master/docs/postgres.md#set-up-database) are out of scope, because plenty of excellent guides and the official documentation exist covering that. The icing on the cake will be the [Synapse Admin UI](https://github.com/Awesome-Technologies/synapse-admin) deployment with secure access to the [administration endpoints](https://github.com/matrix-org/synapse/blob/develop/docs/reverse_proxy.md#synapse-administration-endpoints) to make management of our homeserver easier.

<!--more-->

## Preface

Besides some basic knowledge about AKS, Kubernetes, and Terraform, we have to set up a couple of other things before getting started.

We need an AKS cluster with a configured [Ingress Controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) to be able to expose our homeserver to the world. I use [Traefik 2](https://doc.traefik.io/traefik/) in combination with its [Kubernetes Ingress](https://doc.traefik.io/traefik/providers/kubernetes-ingress/) implementation.

Synapse requires valid TLS certificates to work and [ships with functionality](https://github.com/matrix-org/synapse/blob/master/docs/ACME.md) to automatically provision certificates through [Let's Encrypt](https://letsencrypt.org/). However, I use [cert-manager](https://cert-manager.io/) as a certificate management solution for all my services. So we skip over that part of the configuration, too.

The last thing we have to set up is [Terraform](https://www.terraform.io/) with a [properly configured `kubernetes` provider](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs). If you do not want to use Terraform, transforming the code to regular YAML manifests is trivial.

Depending on your needs, reverse proxy (ingress) functionality and certificate management is the part where your setup differs the most. If you are starting out from scratch, check out [my previous post](../use-terraform-to-deploy-an-azure-kubernetes-service-aks-cluster-traefik-2-cert-manager-and-lets-encrypt), covering much of the steps required to set up the AKS cluster.

## Ingress

We work from the outside to the inside. The `Ingress` is what exposes our Kubernetes `Service` to the public internet. We also add a `Namespace` for all our Matrix resources to a new Terraform file `matrix.tf`:

```hcl
resource "kubernetes_namespace" "matrix" {
  metadata {
    name = "matrix"
  }
}

resource "kubernetes_ingress" "matrix" {
  metadata {
    name      = "matrix-ing"
    namespace = kubernetes_namespace.matrix.metadata.0.name

    annotations = {
      "cert-manager.io/cluster-issuer"           = "letsencrypt-staging"
      "traefik.ingress.kubernetes.io/router.tls" = "true"
    }
  }

  spec {
    rule {
      host = var.synapse_server_name

      http {
        path {
          path = "/_matrix"

          backend {
            service_name = "matrix-svc"
            service_port = 8008
          }
        }

        path {
          path = "/_synapse/client"

          backend {
            service_name = "matrix-svc"
            service_port = 8008
          }
        }

        path {
          path = "/"

          backend {
            service_name = "matrix-admin-svc"
            service_port = 8080
          }
        }
      }
    }

    tls {
      secret_name = "matrix-tls-secret"
      hosts       = [var.synapse_server_name]
    }
  }
}
```

To route external traffic to the Synapse `Service` (`matrix-svc`), we set up the `/_matrix` and `/_synapse/client` endpoints according to the [official reverse proxy documentation](https://github.com/matrix-org/synapse/blob/develop/docs/reverse_proxy.md). The `/` endpoint routes traffic to the Synapse Admin UI (`matrix-admin-svc`). We do **NOT** expose the `/_synapse/admin` endpoint. We will look at how to access this endpoint at the end of this post.

After we finish testing, we must change the certificate issuer from `letsencrypt-staging` to `letsencrypt-production`. Later on, we will also define the `var.synapse_server_name` Terraform input variable.

## Services

`Service`s expose our Synapse and Synapse Admin UI deployments as network services inside our Kubernetes cluster:

```hcl
resource "kubernetes_service" "matrix" {
  metadata {
    name      = "matrix-svc"
    namespace = kubernetes_namespace.matrix.metadata.0.name
  }

  spec {
    selector = {
      "app" = "matrix"
    }

    port {
      port        = 8008
      target_port = 8008
    }
  }
}

resource "kubernetes_service" "matrix_admin" {
  metadata {
    name      = "matrix-admin-svc"
    namespace = kubernetes_namespace.matrix.metadata.0.name
  }

  spec {
    selector = {
      "app" = "matrix-admin"
    }

    port {
      port        = 8080
      target_port = 80
    }
  }
}
```

Pretty self-explanatory. We map the service ports to container ports of the deployments that we add in a later step.

## Configuration Files

Before adding the deployment to our cluster, we generate the initial configuration files for Synapse:

- `homeserver.yaml`: Synapse configuration
- `matrix.schnerring.net.log.config`: logging configuration
- `matrix.schnerring.net.signature.key`: the key, Synapse signs messages with

We can do that with the `generate` command which is part of Synapse:

```shell
kubectl run matrix-generate-pod \
  --stdin \
  --rm \
  --restart=Never \
  --command=true \
  --image matrixdotorg/synapse:latest \
  --env="SYNAPSE_REPORT_STATS=yes" \
  --env="SYNAPSE_SERVER_NAME=matrix.schnerring.net" \
  -- bash -c "/start.py generate && sleep 300"
```

This command creates a pod, runs `generate`, `sleep`s for 300 seconds, and then cleans itself up. Five minutes should be enough time to copy the generated files from the container before it terminates. The following command copies the entire `/data` directory from the container to a local `synapse-config` directory:

```shell
kubectl cp matrix-generate-pod:/data synapse-config
```

We then store the three configuration files as `Secret` on the cluster. But before, make sure to change the value of `handlers.file.filename` from `/homeserver.log` to `/data/homeserver.log` inside the `*.log.config` file. Otherwise, you will run into [a permission issue caused by the default logger configuration](https://github.com/matrix-org/synapse/issues/9970) that I discovered.

```hcl
locals {
  synapse_log_config       = "/data/${var.synapse_server_name}.log.config"
  synapse_signing_key_path = "/data/${var.synapse_server_name}.signing.key"
}

resource "kubernetes_secret" "matrix" {
  metadata {
    name      = "matrix-secret"
    namespace = kubernetes_namespace.matrix.metadata.0.name
  }

  # See also: https://github.com/matrix-org/synapse/blob/master/docker/README.md#generating-a-configuration-file
  data = {
    "homeserver.yaml" = templatefile(
      "${path.module}/synapse-config/homeserver.tpl.yaml",
      {
        "server_name"                = var.synapse_server_name
        "report_stats"               = var.synapse_report_stats
        "log_config"                 = local.synapse_log_config
        "signing_key_path"           = local.synapse_signing_key_path
        "registration_shared_secret" = var.synapse_registration_shared_secret
        "macaroon_secret_key"        = var.synapse_macaroon_secret_key
        "form_secret"                = var.synapse_form_secret
      }
    )

    "log.config" = templatefile(
      "${path.module}/synapse-config/log.tpl.config",
      {
        "log_filename" = "/data/homeserver.log"
        "log_level"    = "INFO"
      }
    )

    "signing.key" = var.synapse_signing_key
  }
}
```

To protect the signing key and the secrets contained inside the `homeserver.yaml` file, we use the Terraform [`templatefile()`](https://www.terraform.io/docs/language/functions/templatefile.html) function, which allows us to put variable placeholders into the configuration files that are interpolated during `terraform apply`. This way, we can commit the configuration files to source control securely. To denote the files as template files, we change the file names to `homeserver.tpl.yaml` and `log.tpl.config`. Here is an example of how to define interpolation sequences inside `homeserver.yaml`:

```yaml
server_name: "${server_name}"

# ...

# a secret which is used to sign access tokens. If none is specified,
# the registration_shared_secret is used, if one is given; otherwise,
# a secret key is derived from the signing key.
#
macaroon_secret_key: "${macaroon_secret_key}"

# a secret which is used to calculate HMACs for form values, to stop
# falsification of values. Must be specified for the User Consent
# forms to work.
#
form_secret: "${form_secret}"
```

Of course, you can also check out the [config file templates on my GitHub](https://github.com/schnerring/infrastructure/tree/0.3.0/synapse-config).

To find the values inside the huge `homeserver.yaml` file, we use the following regular expression. It matches any line that is not a comment or whitespace:

```regexp
^(?!^\s*#)^(?!\n).*
```

All we have to do now is define the [Terraform input variables](https://www.terraform.io/docs/language/values/variables.html) and set them to the values we originally generated:

```hcl
variable "synapse_image_version" {
  type        = string
  description = "Synapse image version."
  default     = "v1.33.2"
}

variable "synapse_server_name" {
  type        = string
  description = "Public Synapse hostname."
}

variable "synapse_report_stats" {
  type        = bool
  description = "Enable anonymous statistics reporting."
}

variable "synapse_signing_key" {
  type        = string
  description = "Signing key Synapse signs messages with."
  sensitive   = true
}

variable "synapse_registration_shared_secret" {
  type        = string
  description = "Allows registration of standard or admin accounts by anyone who has the shared secret."
  sensitive   = true
}

variable "synapse_macaroon_secret_key" {
  type        = string
  description = "Secret which is used to sign access tokens."
  sensitive   = true
}

variable "synapse_form_secret" {
  type        = string
  description = "Secret which is used to calculate HMACs for form values."
  sensitive   = true
}
```

## Persistent Volume Claim

To be able to persist media, we create a [`PersistentVolumeClaim`](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) (PVC):

```hcl
resource "kubernetes_persistent_volume_claim" "matrix" {
  metadata {
    name      = "matrix-pvc"
    namespace = kubernetes_namespace.matrix.metadata.0.name
  }

  spec {
    access_modes = ["ReadWriteOnce"]

    resources {
      requests = {
        "storage" = "4Gi"
      }
    }
  }
}
```

Note that the `default` AKS [`StorageClass`](https://kubernetes.io/docs/concepts/storage/storage-classes/) has the `ReclaimPolicy` set to `Delete` . I recommend defining a custom storage class for production. Setting its reclaim policy to `Retain` makes accidentally purging the media PVC much harder.

## Synapse

Let us finally deploy Synapse. We just need to mount the configuration files and the PVC we just defined:

```hcl
resource "kubernetes_deployment" "matrix" {
  metadata {
    name      = "matrix-deploy"
    namespace = kubernetes_namespace.matrix.metadata.0.name
    labels = {
      "app" = "matrix"
    }
  }
  spec {
    replicas = 1

    selector {
      match_labels = {
        "app" = "matrix"
      }
    }

    strategy {
      type = "Recreate"
    }

    template {
      metadata {
        labels = {
          "app" = "matrix"
        }
      }

      spec {
        hostname       = "matrix"
        restart_policy = "Always"

        container {
          name  = "synapse"
          image = "matrixdotorg/synapse:${var.synapse_image_version}"

          port {
            container_port = 8008
          }

          volume_mount {
            mount_path = "/data"
            name       = "data-vol"
          }

          volume_mount {
            name       = "secret-vol"
            mount_path = "/data/homeserver.yaml"
            sub_path   = "homeserver.yaml"
            read_only  = true
          }

          volume_mount {
            name       = "secret-vol"
            mount_path = local.synapse_log_config
            sub_path   = "log.config"
            read_only  = true
          }

          volume_mount {
            name       = "secret-vol"
            mount_path = local.synapse_signing_key_path
            sub_path   = "signing.key"
            read_only  = true
          }
        }

        volume {
          name = "data-vol"

          persistent_volume_claim {
            claim_name = "matrix-pvc"
          }
        }

        volume {
          name = "secret-vol"

          secret {
            secret_name = "matrix-secret"
          }
        }
      }
    }
  }
}
```

To deploy everything, we run the following commands:

```shell
terraform plan -out infrastructure.tfplan
terraform apply infrastructure.tfplan
```

After the deployment succeeds, do not forget to change the cluster issuer to `letsencrypt-production` and `terraform apply` again.

Next, we [register a new user](https://github.com/matrix-org/synapse/blob/5688a74cf3042c0064e3600cb03f6edaa7f7c838/INSTALL.md#registering-a-user). Synapse includes the `register_new_matrix_user` command. First, we query the name of the pod:

```shell
kubectl get pods --namespace matrix
```

We then run the following to connect to the pod:

```shell
kubectl exec --stdin --tty --namespace matrix matrix-deploy-xxxxxxxxxx-xxxxx -- bash
```

Now we register the user:

```shell
register_new_matrix_user \
  --admin \
  --user michael \
  --config /data/homeserver.yaml \
  http://localhost:8008
```

We use [Element](https://element.io/) or any other Matrix client and sign in. Great!

## Synapse Admin UI

The Synapse API includes administration endpoints but lacks a UI for administration tasks. There is an [open GitHub issue](https://github.com/matrix-org/synapse/issues/2032) which seems to be inactive. However, [awesometechnologies/synapse-admin](https://github.com/Awesome-Technologies/synapse-admin) is being actively developed and works well.

We already configured the ingress to route traffic to the `matrix-admin-svc`, so the only thing missing is the deployment:

```hcl
resource "kubernetes_deployment" "matrix_admin" {
  metadata {
    name      = "matrix-admin-deploy"
    namespace = kubernetes_namespace.matrix.metadata.0.name
    labels = {
      "app" = "matrix-admin"
    }
  }
  spec {
    replicas = 1

    selector {
      match_labels = {
        "app" = "matrix-admin"
      }
    }

    strategy {
      type = "Recreate"
    }

    template {
      metadata {
        labels = {
          "app" = "matrix-admin"
        }
      }

      spec {
        hostname       = "matrix-admin"
        restart_policy = "Always"

        container {
          name  = "synapse-admin"
          image = "awesometechnologies/synapse-admin:latest"

          port {
            container_port = 80
          }
        }
      }
    }
  }
}
```

No configuration is required because the Admin UI is a client-side application. After running `terraform apply`, we can browse to `matrix.schnerring.net` to access the app:

![Synapse Admin UI Login](./img/admin-ui-login.png)

Synapse Admin UI requires access to the `_synapse/admin` endpoint. But we do not want to expose that endpoint to the public internet, so we have to connect to it by other means. `kubectl port-forward` allows us to securely forward a local port to a port on a Kubernetes service:

```shell
kubectl port-forward service/matrix-svc --namespace matrix 8008:8008
```

We can now enter `http://localhost:8008` as homeserver URL and login to the admin UI with the user we created earlier:

![Synapse Admin UI Users](./img/admin-ui-users.png)

## What's Next?

Regardless of what you plan on doing with your homeserver, replacing SQLite with PostgreSQL should be at the top of the priority list. Other than that, there is tons of other stuff to configure depending on your needs.

I am super happy with what we have built and curious about what you think. I would appreciate it if you shared your opinion in the comments below.
