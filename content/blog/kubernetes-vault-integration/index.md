
---
title: Get secrets from Vault into Kubernetes pods
date: "2020-01-27T14:20:00Z"
description: Get secrets from Vault into Kubernetes pods
---

# Get secrets from Vault into Kubernetes pods

1. Motivation:

We would like to use Vault to manage the application secrets like API token, DB credentials. Also we don't want to change the application to tight couple with Vault API. This article will discuss a solution that get secrets from Vault in Kubernetes without changing a line of application code.

2. Run Vault in Kubernetes

  * The first thing we need to do is to create a Vault instance. The easiest way to run Vault in Kubernetes is to use the [official vault helm chart][5] from Hashicorp.

  * If you plan to run this for a production environment, it's better to use counsul as the backend for HA. Hashicorp also has an [official consul helm chart][6]

  * After start the Vault on kubernetes, you need run `vault operator init` to initialize the vault and `vault operator unseal` to unseal the vault instance.

  * Then we can enable the Vault KV secret engine at `/secret` path and create some secrets.

```
vault secrets enable -version=2 -path=secret kv
vault kv put secret/my-secret name="Max Cai" email=zhiyuan.cai@gmail.com
```

3. Authorize Kubernetes to access Vault

Vault supports multiple [auth methods](https://www.vaultproject.io/docs/auth/index.html) including GitHub, LDAP, JWT/OIDC. Here we will use [kubernetes auth method](https://www.vaultproject.io/docs/auth/kubernetes/) as all clients will be run inside the kubernetes cluster.

To make things simple, I'm using Terraform Vault provider and Kubernetes provider to create the policy and service account in Vault and Kubernetes.

  *  use Kubernetes in Terraform
```
    provider "kubernetes" {
      config_context = "vault-test.kubernetes"
    }
```
  
  * create service account and role binding
```
resource "kubernetes_service_account" "vault_auth" {
  metadata {
    name      = "vault-auth"
    namespace = "default"
  }
}

resource "kubernetes_cluster_role_binding" "vault_auth" {
  metadata {
    name = "vault-auth"
  }
  role_ref {
    api_group = "rbac.authorization.k8s.io"
    kind      = "ClusterRole"
    name      = "system:auth-delegator"
  }
  subject {
    kind      = "ServiceAccount"
    name      = "vault-auth"
    namespace = "default"
    api_group = ""
  }
}
```

  * use Vault in Terraform, make sure you have `VAULT_ADDR` and `VAULT_TOKEN` in your environment variable.
```
provider "vault" {
}
```

  * Create policy for Kubernetes to read secrets
``` 
# Read all secret
# filename: data/secret-all-read.policy
path "secret/*"
{
  capabilities = ["read", "list"]
}
```
```
resource "vault_policy" "secret-all-read" {
  name   = "secret-all-read"
  policy = "${file("${path.module}/data/secret-all-read.policy")}"
}
```

  * get token of Kubernetes service account and config vault auth backend
```
data "kubernetes_secret" "vault_auth" {
  metadata {
    name      = "${kubernetes_service_account.vault_auth.default_secret_name}"
    namespace = "default"
  }
}
resource "vault_auth_backend" "kubernetes" {
  type        = "kubernetes"
  path        = "kubernetes"
  description = "auth for kubernetes"
}
resource "vault_kubernetes_auth_backend_config" "kubernetes" {
  backend            = "${vault_auth_backend.kubernetes.path}"
  kubernetes_host    = "https://api.vault-test.k8s.bonesoul.com"
  kubernetes_ca_cert = "${data.kubernetes_secret.vault_auth.data["ca.crt"]}"
  token_reviewer_jwt = "${data.kubernetes_secret.vault_auth.data["token"]}"
}
resource "vault_kubernetes_auth_backend_role" "readonly" {
  backend                          = vault_auth_backend.kubernetes.path
  role_name                        = "readonly"
  bound_service_account_names      = ["vault-user"]
  bound_service_account_namespaces = ["*"]
  token_ttl                        = 3600
  token_policies                   = ["secret-all-read"]
}
```

4. Get secrets from Vault

  * To get secrets from Vault, we need use authorize with Vault from Kubernetes. First we need create a service account to match the name in vault policy

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault-user

```

  * Then we need use the token of the service account to authorize with Vault, there are 2 ways to do this.

    * the hard way
    in [https://medium.com/@jackalus/vault-kubernetes-auth-and-database-secrets-engine-6551d686a12], it descible a method to get the JWT token from `/var/run/secrets/kubernetes.io/serviceaccount/token` and login to Vault to get the vault token.

    * the easy way
    [Vault Agent][1] can manage the authrization with Vault for Kubernetes with a simple configuation file. Also we can use Vault Agent template to render the secrets from Vault into a text file for application to read.

```
auto_auth {
  method "kubernetes" {
    mount_path = "auth/kubernetes"
    config={
      role = "readonly"
    }
  }
  sink "file" {
    config = {
      path = "/var/vault/out/.token"
    }
  }
}

template {
  source      = "/var/vault/config/vault.ctmpl"
  destination = "/var/vault/out/vault.txt"
}

exit_after_auth = true
pid_file = "./pidfile"

vault {
  address = "http://vault.default"
}

```
```
# vault.ctmpl
{{ with secret "secret/stg/demo" }}{{ range $k, $v := .Data.data }}
{{ $k }}={{ $v }}
{{ end }}{{ end }}
```


5. Update Kubernetes secrets based on the value from Vault

 * The last step is to run an init container before the application start to retrieve secrets from Vault. However, without changing the application code, we cannot covert the secrets from a text file to environment variable for application to use.

  * one solution is to add a shell script in the docker image to `export` the value to environment variable and start the application.
  * another solution is to create a new secrets object in Kubernetes to hold the secrets from Vault and add them to environment with `envFrom.secretRef`. I wrote a small tool with [client-go][2] to [access the API within a pod][4] covert the text file get from Vault agent to a secrets object in Kubernetes [https://github.com/c4po/ftks](https://github.com/c4po/ftks).

6. put all togetger

  * The full code can be found at
[https://github.com/c4po/kubernetes-vault](https://github.com/c4po/kubernetes-vault)


7. reference:

[1]:  https://www.vaultproject.io/docs/agent/
[2]: https://github.com/kubernetes/client-go/tree/master/examples
[3]: https://github.com/sethvargo/vault-kubernetes-authenticator
[4]: https://kubernetes.io/docs/tasks/administer-cluster/access-cluster-api/
[5]: https://github.com/hashicorp/vault-helm
[6]: https://github.com/hashicorp/consul-helm
