# Previder Secure Vault and External Secrets Operator in Kubernetes

## About Previder Secure Vault

Previder Secure Vault is a secrets management service available through the [Previder Portal](https://portal.previder.nl) self-service portal. No installation within the Kubernetes cluster is required. The Vault itself, along with its tokens and secrets, is fully provisioned and managed through the **Vault Dashboard** in the portal.

## About External Secrets Operator

External Secrets Operator (ESO) is a Kubernetes operator that reads secrets from an external system in this case Previder Secure Vault and creates a native Kubernetes `Secret` from them, which applications consume as usual. ESO has built-in, native support for Previder Secure Vault, so no custom integration is needed.

## Overview

This guide creates a Secure Vault environment and its tokens and secrets through the Previder Portal's Vault Dashboard, and uses External Secrets Operator's native Previder provider to deliver a secret into the cluster as a Kubernetes `Secret`:

1. Create a Secure Vault environment in the Previder Portal.
2. Create a ReadWrite and a ReadOnly access token for it.
3. Store a secret (e.g. a password) in the vault using the ReadWrite token.
4. Install External Secrets Operator in the cluster.
5. Configure a `SecretStore` for the vault and an `ExternalSecret` for the secret to retrieve, using the ReadOnly token.
6. The application reads the result as a normal Kubernetes `Secret`.

### Token Types

Previder Secure Vault uses three token types, each with a different scope:

| | EnvironmentAdmin | ReadWrite | ReadOnly |
|---|---|---|---|
| Manage tokens | ✅ | | |
| Manage secrets | | ✅ | |
| Get decrypted secret | | ✅ | ✅ |

The token received when a Secure Vault environment is created is always an **EnvironmentAdmin** token. In the Vault Dashboard, that token is used once to create:

- A **ReadWrite** token, used to manage secrets (create, delete), from the dashboard.
- A **ReadOnly** token, used by the cluster to read secrets. This is the only token that ends up inside Kubernetes.

**Important:** Only a ReadOnly token is stored in the cluster. The token can decrypt any secret in its environment, if the secret's ID or name is known.

Access is not restricted to specific secrets, this means that if the cluster is compromised, an attacker could use the token to read any secret in that environment. However, the token cannot be used to create, modify, or delete secrets, or to create additional tokens.

If secrets need to be isolated for example, secrets belonging to different applications or teams. Use separate Secure Vault environments rather than separate tokens within the same environment.

---

# Prerequisites

This guide assumes:

- A running Kubernetes cluster, as described in the [Installatiehandleiding](https://github.com/previder/kubernetes-examples/tree/main/docs)
- `helm` and `kubectl` configured against the cluster
- Access to the [Previder Portal](https://portal.previder.nl) to create a Secure Vault environment

---

# 1. Create a Secure Vault Environment

Create a Secure Vault environment via the [Previder Portal](https://portal.previder.nl). This generates the initial **EnvironmentAdmin** token for that environment.

---

# 2. Create a ReadWrite and a ReadOnly Token

Open the **[Vault Dashboard](https://vault.previder.io/ui/#/login)** for the environment and create two tokens: a **ReadWrite** token and a **ReadOnly** token.

**Note:** Give each application its own ReadOnly token with a clear description (e.g. `hello-app - production`) rather than sharing a single token across the entire cluster. This makes revocation and auditing easier on a per-application basis. However, separate tokens do **not** restrict access to specific secrets: each token can still read any secret in the same environment (see the note above). For actual isolation between applications or teams, use separate **Secure Vault environments** instead.


---

# 3. Store a Secret in the Vault

In the **[Vault Dashboard](https://vault.previder.io/ui/#/login)**, using the **ReadWrite** token, create an example secret for an application called `hello-app`. Give it a clear name or description, such as `hello-app-api-key` this exact name is needed again in step 7.

**Note:** Instead of the Vault Dashboard, secrets and keys can also be managed from the command line using [`vault-cli`](https://github.com/previder/vault-cli). See that repository for installation and usage instructions this guide only covers the dashboard, since it requires nothing to install.

---

# 4. Install External Secrets Operator

```
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace
```

Verify:

```
kubectl -n external-secrets get pods
```

Expected output:

```
NAME                                    READY   STATUS    RESTARTS   AGE
external-secrets-...                    1/1     Running   0          30s
external-secrets-cert-controller-...    1/1     Running   0          30s
external-secrets-webhook-...            1/1     Running   0          30s
```

---

# 5. Store the ReadOnly Token as a Kubernetes Secret

Create the namespace the application will run in:

```
kubectl create namespace hello-app
```

Store the ReadOnly token from step 2 as a Kubernetes `Secret`, directly from the command line so it never touches a file on disk:

```
kubectl -n hello-app create secret generic previder-vault-token \
  --from-literal=previder-vault-token="<readonly_token>"
```

---

# 6. Create a SecretStore

A `SecretStore` tells External Secrets Operator how to reach Previder Secure Vault. This one is scoped to the `hello-app` namespace, matching the Secret created in step 5.

`hello-app-secretstore.yaml`

```
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: previder-backend
  namespace: hello-app
spec:
  provider:
    previder:
      auth:
        secretRef:
          accessToken:
            name: previder-vault-token
            key: previder-vault-token
```

Apply:

```
kubectl apply -f hello-app-secretstore.yaml
```

Verify:

```
kubectl -n hello-app get secretstore previder-backend
```

Expected output:

```
NAME               AGE   STATUS   CAPABILITIES   READY
previder-backend   10s   Valid    ReadOnly       True
```

---

# 7. Create an ExternalSecret

`hello-app-externalsecret.yaml`

```
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: hello-app-api
  namespace: hello-app
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: previder-backend
    kind: SecretStore
  target:
    name: hello-app-api
    creationPolicy: Owner
  data:
  - secretKey: API_KEY
    remoteRef:
      key: hello-app-api-key
```

**Note:** `remoteRef.key` is the id/description used in step 3. `refreshInterval` controls how often ESO checks the vault for changes, if the secret's value is updated later, the Kubernetes `Secret` is updated automatically within that interval, no `kubectl apply` needed.

Apply:

```
kubectl apply -f hello-app-externalsecret.yaml
```

---

# 8. Verify

```
kubectl -n hello-app get externalsecret hello-app-api
```

Expected output:

```
NAME            STORE              REFRESH INTERVAL   STATUS         READY
hello-app-api   previder-backend   1h                  SecretSynced   True
```

Confirm the value ended up correctly in the Kubernetes Secret:

```
kubectl -n hello-app get secret hello-app-api -o jsonpath='{.data.API_KEY}' | base64 -d
```

This should show the value you chose in step 3, for example:

```
s3cr3t-api-key-value
```

---

# 9. Update the Secret and Observe the Sync

To demonstrate that ESO keeps the Kubernetes `Secret` in sync with the vault, delete the `hello-app-api-key` secret in the Vault Dashboard (using the ReadWrite token) and recreate it under the **same** name/id, but with a different value for example `s3cr3t-api-key-value-v2`.

By default, this reaches Kubernetes within the `refreshInterval` configured in step 7 (`1h`). To confirm the sync without waiting, force it immediately:

```
kubectl -n hello-app annotate externalsecret hello-app-api force-sync=$(date +%s) --overwrite
```

Check that the Kubernetes `Secret` now holds the new value:

```
kubectl -n hello-app get secret hello-app-api -o jsonpath='{.data.API_KEY}' | base64 -d
```

Expected output:

```
s3cr3t-api-key-value-v2
```

**Note:** `envFrom` reads the Kubernetes `Secret` only when the Pod starts. When a value in the vault is updated, the change is automatically propagated to the Kubernetes `Secret`, but running Pods only pick up the new value after they are restarted.

---

# 10. Add More Secrets

The token, the Secret holding it and the `SecretStore` (steps 2, 5 and 6) are set up once and are reused for every additional secret. For a second (or third) password, only two things are needed:

1. **Create the secret in the Vault Dashboard** using the ReadWrite token, as in step 3 for example `hello-app-db-password`.
2. **Reference it in an `ExternalSecret`.** There are two options:

**Option A: add it to the existing `ExternalSecret`**, so both values end up in the same Kubernetes `Secret` as separate keys. Add another entry under `data` in `hello-app-externalsecret.yaml`:

```
  data:
  - secretKey: API_KEY
    remoteRef:
      key: hello-app-api-key
  - secretKey: DB_PASSWORD
    remoteRef:
      key: hello-app-db-password
```

```
kubectl apply -f hello-app-externalsecret.yaml
```

**Option B: create a separate `ExternalSecret`**, resulting in its own Kubernetes `Secret`. For example, `hello-app-db-externalsecret.yaml`:

```
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: hello-app-db
  namespace: hello-app
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: previder-backend
    kind: SecretStore
  target:
    name: hello-app-db
    creationPolicy: Owner
  data:
  - secretKey: DB_PASSWORD
    remoteRef:
      key: hello-app-db-password
```

**Note:** This is `hello-app-externalsecret.yaml` from step 7, with `metadata.name`, `target.name`, `secretKey` and `remoteRef.key` changed — it reuses the same `secretStoreRef`, so no extra `SecretStore` is needed.

```
kubectl apply -f hello-app-db-externalsecret.yaml
```

Verify the new value:

```
kubectl -n hello-app get secret hello-app-db -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

---

**Important:**
- Use a separate Secure Vault **environment** per application or team that needs real isolation, tokens within the same environment are not restricted to specific secrets, so separate environments are what actually keeps secrets apart.
- Within one environment, still give each application its own ReadOnly token and `SecretStore`, so access can be revoked and audited per application, just don't rely on this for isolation.
- Manage secrets from the Vault Dashboard (or `vault-cli`) using the ReadWrite token, never from inside the cluster.
