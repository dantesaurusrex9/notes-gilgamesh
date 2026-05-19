---
title: "8 - ConfigMaps, Secrets, and Configuration"
created: 2026-05-19
updated: 2026-05-19
tags: []
aliases: []
---

# 8 - ConfigMaps, Secrets, and Configuration

[toc]

> **TL;DR:** ConfigMaps and Secrets are the Kubernetes primitives for injecting non-secret configuration (ConfigMap) and sensitive data (Secret) into Pods, decoupled from the container image. The injection mechanisms are environment variables or volume mounts. Secrets are base64-encoded in etcd by default — not encrypted — so production clusters must either enable etcd encryption at rest or use an external secrets manager (Vault, AWS Secrets Manager) via the External Secrets Operator. Immutable ConfigMaps and Secrets (since Kubernetes 1.21) improve performance and prevent accidental live edits.

## Vocabulary

**ConfigMap**: A key-value store for non-secret configuration data. Keys are strings; values can be arbitrary text (config files, entire scripts, JSON blobs). Limited to 1MiB total size.

---

**Secret**: A key-value store for sensitive data. Values are base64-encoded (NOT encrypted) by default. Kubernetes prevents Secrets from being stored in the API server's response cache and marks them with `kubectl get` truncation, but they are stored in plain base64 in etcd without additional configuration.

---

**Environment variable injection**: Injecting a ConfigMap key or Secret key directly as an environment variable in a container. The value is baked in at Pod start time — subsequent ConfigMap updates do NOT update running Pods.

---

**Volume mount injection**: Mounting a ConfigMap or Secret as a directory of files inside the container. Each key becomes a filename; each value becomes file contents. Volume-mounted ConfigMaps and Secrets DO update automatically (with a kubelet sync period, typically 1–2 minutes).

---

**Projected volume**: A volume that combines multiple sources — ServiceAccount token, ConfigMap, Secret, and downward API — into a single directory. Used for fine-grained control over file paths and permissions.

---

**Immutable ConfigMap/Secret**: Setting `immutable: true` prevents updates to the object's data. Tells the kubelet to stop watching the API server for changes to this object — reduces API server load in clusters with many ConfigMaps.

---

**etcd encryption at rest**: A Kubernetes feature that encrypts Secret data before writing to etcd using a provider key (AES-CBC, AES-GCM, or a KMS provider). Configured in the kube-apiserver's `EncryptionConfiguration` resource.

---

**Sealed Secrets**: An open-source project (Bitnami) that allows Secrets to be encrypted using a cluster-specific keypair and committed to git safely. The `SealedSecret` CRD is decrypted by a controller running in the cluster.

---

**External Secrets Operator (ESO)**: A controller that syncs secrets from external stores (AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager, Azure Key Vault) into Kubernetes Secrets. The canonical cloud-native secrets management pattern.

---

**Downward API**: A mechanism to inject Pod metadata (name, namespace, labels, annotations, resource limits, node name) into containers as environment variables or files. The Pod introspects itself without calling the API server.

---

**stringData**: A write-only convenience field on Secrets that accepts plaintext strings. The API server base64-encodes them on write. On read, the encoded values appear in `.data`, not `.stringData`.

---

## Intuition

The motivation for ConfigMaps and Secrets is the twelve-factor app principle: configuration is everything that varies between deployments (dev, staging, prod). Baking configuration into container images means rebuilding images for every config change. Externalizing configuration into Kubernetes objects means you can redeploy just by updating a ConfigMap and rolling the Deployment.

Secrets are ConfigMaps with extra caution: base64 encoding is NOT security. It is an encoding that makes binary data safe to embed in YAML. A Secret in etcd is as readable as a ConfigMap to anyone with etcd access or with Kubernetes API access to Secrets. The security model has two distinct layers: (1) who can read the Secret via the Kubernetes API (RBAC controls this), and (2) whether the data is protected if etcd is compromised (etcd encryption at rest or external secrets manager handles this).

## How it Works

### ConfigMap Anatomy

A ConfigMap has a flat map of string keys to string values. Values can be:
- **Single-line values**: environment variable style (`key: value`).
- **Multi-line values**: entire config files embedded with YAML block scalar syntax (`key: |`).
- **Binary data**: stored in `.binaryData` field as base64-encoded bytes.

The two injection modes have different update semantics:

| Injection mode | Created when | Updated when ConfigMap changes |
| :--- | :--- | :--- |
| Environment variable | Pod starts | Never — requires Pod restart |
| Volume mount | Pod starts | Yes — kubelet sync period (~1–2 min) |
| Projected volume | Pod starts | Yes — kubelet sync period |

> [!IMPORTANT]
> Environment variables from ConfigMaps are resolved at Pod creation time and do NOT update when the ConfigMap changes. If your application reads config at startup from environment variables, you must restart the Pod to pick up new values. Volume-mounted ConfigMaps update live — ideal for applications that watch config files (e.g., `nginx -s reload`, Prometheus config reloader).

### Secret Types

Kubernetes defines several built-in Secret types that are treated specially by the system:

| Type | Purpose |
| :--- | :--- |
| `Opaque` | Generic key-value secret (default) |
| `kubernetes.io/tls` | TLS certificate + key (used by Ingress) |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials (`imagePullSecrets`) |
| `kubernetes.io/service-account-token` | Legacy ServiceAccount token (deprecated in favor of projected tokens) |
| `kubernetes.io/basic-auth` | Username + password |
| `kubernetes.io/ssh-auth` | SSH private key |

### etcd Encryption at Rest

By default, all Secrets are stored base64-encoded in etcd — readable by anyone with etcd access. Enabling encryption at rest requires configuring the kube-apiserver with an `EncryptionConfiguration` manifest specifying which resources to encrypt and with which provider.

The recommended provider is `kms` (a KMS envelope encryption provider) — it uses a locally generated Data Encryption Key (DEK) to encrypt each Secret, and the DEK itself is encrypted by a Key Encryption Key (KEK) stored in the cloud KMS. This means: even if etcd is fully compromised, the Secrets are unreadable without the KMS key.

```yaml
---
# EncryptionConfiguration (passed to kube-apiserver via --encryption-provider-config)
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - kms:
          name: aws-kms
          endpoint: unix:///var/run/kmsplugin/socket.sock
          apiVersion: v2
      - identity: {}   # fallback for existing unencrypted secrets during migration
```

### External Secrets Operator Pattern

The ESO pattern is the modern best practice for production secrets management. The operator watches `ExternalSecret` CRDs, pulls secret values from an external store on a configurable sync interval, and creates/updates Kubernetes `Secret` objects in the cluster. Applications consume the resulting Kubernetes Secret normally — no SDK changes needed.

```mermaid
flowchart LR
    ESO["External Secrets Operator"] -- "GET /secrets/prod/db-password" --> SM["AWS Secrets Manager\n/ Vault / GCP SM"]
    SM -- "secret value" --> ESO
    ESO -- "create/update Secret" --> KS["Kubernetes Secret\n(in cluster)"]
    KS -- "env or volume mount" --> POD["Application Pod"]
```

The advantage over manual Secret creation: the external store is the source of truth. Rotation in the external store propagates to the cluster automatically on the next sync cycle.

## Real-world Example

A web application that reads its database password from a Secret (volume mount for live rotation) and its non-sensitive config from a ConfigMap (env vars), with an ExternalSecret for production.

```yaml
---
# ConfigMap: non-sensitive application config
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  LOG_LEVEL: info
  MAX_CONNECTIONS: "100"
  FEATURE_FLAGS: |
    new_ui: true
    dark_mode: false
    beta_api: false
---
# Secret: database credentials (in development; use ExternalSecret in prod)
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque
stringData:                           # plaintext for human readability; API server encodes
  DB_PASSWORD: "s3cr3t-pa55word"
  DB_CONNECTION_STRING: "postgres://app:s3cr3t-pa55word@postgres:5432/appdb"
---
# ExternalSecret: production — pulls from AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h                 # re-sync from AWS SM every hour
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials              # the Kubernetes Secret to create/update
    creationPolicy: Owner
  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: prod/app/db
        property: password
    - secretKey: DB_CONNECTION_STRING
      remoteRef:
        key: prod/app/db
        property: connection_string
---
# Deployment: inject config and secrets
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
        - name: app
          image: myregistry/app:1.5.0
          envFrom:
            - configMapRef:
                name: app-config     # inject all keys as env vars
          env:
            - name: POD_NAME         # downward API: inject pod metadata
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          volumeMounts:
            - name: secrets
              mountPath: /etc/secrets
              readOnly: true         # volume mount for live rotation
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              memory: 256Mi
      volumes:
        - name: secrets
          secret:
            secretName: db-credentials
            defaultMode: 0400        # owner read-only — restrictive permissions
```

```bash
#!/usr/bin/env bash
set -euo pipefail

# View ConfigMap contents
kubectl get configmap app-config -n production -o yaml

# Decode a Secret value (base64)
kubectl get secret db-credentials -n production -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
# s3cr3t-pa55word

# Verify Secret is mounted in Pod
POD=$(kubectl get pods -l app=app -n production -o jsonpath='{.items[0].metadata.name}')
kubectl exec "${POD}" -n production -- ls /etc/secrets/
# DB_CONNECTION_STRING  DB_PASSWORD

kubectl exec "${POD}" -n production -- cat /etc/secrets/DB_PASSWORD
# s3cr3t-pa55word (live — updates within ~1m if Secret is rotated)
```

> [!WARNING]
> `kubectl get secret <name> -o yaml` prints the base64-encoded Secret values in plaintext YAML to your terminal. In shared environments or when recording terminal sessions, this leaks credentials. Use `-o jsonpath='{.data.<key>}'` to extract specific fields, and consider piping through `base64 -d` without logging the result.

## In Practice

**Secret rotation without Pod restarts:** Volume-mounted Secrets update automatically within one kubelet sync period (default 60 seconds, configurable with `--sync-frequency`). Applications that watch the file (e.g., reload on inotify event) can pick up rotated credentials without a Pod restart. Applications that only read credentials at startup must be restarted via a rolling update after rotation.

**ConfigMap size limits:** ConfigMaps are stored in etcd, which has a default 1.5MiB per object size limit (configurable). Do not store large binary assets, ML model weights, or large datasets in ConfigMaps — use external storage (S3, GCS) for that. The 1MiB soft limit on ConfigMap data is a deliberate guardrail.

**Immutable Secrets at scale:** In clusters with thousands of Secrets (common when using cert-manager which creates one Secret per certificate), setting `immutable: true` on Secrets that will not change (e.g., registry pull secrets) eliminates the kubelet's watch overhead. The kubelet watches every Secret mounted in a Pod by default; with immutable Secrets, it stops watching after the first successful sync.

> [!NOTE]
> Kubernetes `Secrets` are accessible to any Pod in the same namespace that has the appropriate ServiceAccount RBAC permissions. The default ServiceAccount in a namespace has no Secret read permissions, but misconfigured RBAC (e.g., wildcard rules like `resources: ["*"]`) can inadvertently expose Secrets to all Pods. Audit RBAC regularly with `kubectl auth can-i get secrets --as=system:serviceaccount:<namespace>:<sa-name>`.

## Pitfalls

- **"Secrets are encrypted."** — Base64 encoding is NOT encryption. Without etcd encryption at rest or an external secrets manager, Secrets are stored in plaintext in etcd and readable via the API by anyone with Secrets read permission.
- **"Updating a ConfigMap updates running Pods."** — Only volume-mounted ConfigMaps update live. Environment variables baked in at Pod creation time are static — the Pod must be restarted.
- **"CommitSecrets to git for GitOps."** — Never commit unencrypted Secrets to git. The options: Sealed Secrets (encrypted blob in git), External Secrets Operator (reference in git, value in external store), or SOPS (Mozilla's encryption tool for YAML files).
- **"Secret RBAC is namespace-scoped automatically."** — Secrets in namespace A are accessible to anyone with `get secrets` permission in namespace A, but not from namespace B. However, ClusterRoles that bind to `secrets` across all namespaces can read all Secrets cluster-wide. Review ClusterRoleBindings that grant secrets access.
- **"`defaultMode: 0777` for volumes is harmless."** — File permissions on Secret mounts matter. Secrets mounted with `defaultMode: 0644` (the default) are world-readable inside the container — any process in the container can read them. Use `0400` (owner read-only) for credentials. When multiple containers share a Secret volume, use `subPath` mounts to limit each container's access to only the keys it needs.

## Exercises

### Exercise 1 — Conceptual: When to Use env vs Volume Mount

You have a database password that the application reads at startup and a TLS certificate that a web server (nginx) reloads every 30 seconds. How should you inject each?

#### Solution

**Database password → `env` from Secret** is acceptable if you can tolerate pod restarts on rotation. However, the better choice is a **volume mount** if your connection pool supports credential refresh without restart. The deciding factor: does the application re-read the credential on each new database connection (volume mount works), or only at startup (env works but requires Pod restart on rotation)?

For the TLS certificate → **volume mount, always**. nginx uses `nginx -s reload` to reload certificates without dropping connections. A volume-mounted Secret updates within ~60 seconds of being updated. The application (or a sidecar reload controller) watches the file via inotify and triggers the reload. Injecting a TLS cert as an env var is wrong on two counts: env vars cannot be hot-updated without Pod restart, and a cert needs to be a file on disk for the TLS library to load it.

Rule of thumb: If the application supports live-reload of the value (file watch + application signal), use volume mount. If the application only reads the value once at startup and you can tolerate rolling restarts on rotation, env is simpler.

### Exercise 2 — YAML: ConfigMap as a Config File

Write a ConfigMap that contains an nginx.conf, and a Pod that mounts it at `/etc/nginx/nginx.conf`.

#### Solution

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: default
data:
  nginx.conf: |
    user  nginx;
    worker_processes  auto;
    error_log  /var/log/nginx/error.log warn;
    pid        /var/run/nginx.pid;

    events {
      worker_connections  1024;
    }

    http {
      include       /etc/nginx/mime.types;
      default_type  application/octet-stream;
      sendfile      on;
      keepalive_timeout  65;

      server {
        listen       80;
        server_name  localhost;

        location / {
          root   /usr/share/nginx/html;
          index  index.html;
        }

        location /health {
          return 200 'ok';
          add_header Content-Type text/plain;
        }
      }
    }
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-custom-config
  namespace: default
spec:
  containers:
    - name: nginx
      image: nginx:1.27-alpine
      ports:
        - containerPort: 80
      volumeMounts:
        - name: nginx-conf
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf          # mount only this key, not the entire ConfigMap
          readOnly: true
      resources:
        requests:
          cpu: 50m
          memory: 64Mi
        limits:
          cpu: 200m
          memory: 128Mi
  volumes:
    - name: nginx-conf
      configMap:
        name: nginx-config
```

The `subPath: nginx.conf` is important here — without it, the volume mount would replace the entire `/etc/nginx/` directory with the ConfigMap contents, deleting `mime.types` and other required files. `subPath` mounts only the specified key as a file at the specified path, leaving the rest of the directory intact. The trade-off: `subPath` mounts do NOT receive live updates when the ConfigMap changes — you need a Pod restart. For live updates, mount the ConfigMap as a directory at a custom path and symlink or configure nginx to include from that path.

### Exercise 3 — Design: GitOps-Compatible Secrets Strategy

Your team uses ArgoCD to deploy all Kubernetes manifests from git. How do you handle Secrets in a GitOps workflow without committing plaintext credentials?

#### Solution

Three viable patterns, in order of operational simplicity:

**Option 1 — External Secrets Operator (recommended for cloud-hosted clusters):**
- Store all secrets in AWS Secrets Manager / GCP Secret Manager / Vault.
- Commit `ExternalSecret` CRDs to git — these reference the external secret by path/key, not by value.
- The ESO controller in the cluster syncs values from the external store into Kubernetes Secrets automatically.
- ArgoCD sees the ExternalSecret as any other CRD and applies it. The actual secret value never touches git.

**Option 2 — Sealed Secrets (recommended for on-premises or simple setups):**
- Install the Bitnami Sealed Secrets controller in the cluster. It generates an asymmetric keypair.
- Encrypt secrets with `kubeseal` (uses the cluster's public key): `kubeseal -f secret.yaml -w sealed-secret.yaml`.
- Commit the `SealedSecret` object (encrypted) to git.
- ArgoCD applies the SealedSecret; the controller decrypts it using the private key (stored only in the cluster) and creates the Kubernetes Secret.
- Rotation: re-encrypt with `kubeseal` and commit the new SealedSecret.

**Option 3 — SOPS (for teams already using age or GPG):**
- Encrypt entire YAML files with SOPS using age/GPG keys shared with your team and the cluster's CD agent.
- ArgoCD's SOPS plugin decrypts manifests during sync.
- More complex key management than ESO but works without an external store dependency.

For a cloud-hosted cluster: choose ESO — the external store is the authoritative source of truth, rotation is automatic, and the ArgoCD workflow is clean (no secret values in git, no encryption ceremony).

## Sources

- Kubernetes docs — ConfigMaps. https://kubernetes.io/docs/concepts/configuration/configmap/
- Kubernetes docs — Secrets. https://kubernetes.io/docs/concepts/configuration/secret/
- Kubernetes docs — Encrypting Secret Data at Rest. https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/
- External Secrets Operator. https://external-secrets.io/latest/
- Bitnami Sealed Secrets. https://github.com/bitnami-labs/sealed-secrets
- Lukša, M. *Kubernetes in Action*, 2nd ed. Chapter 7 (ConfigMaps and Secrets).

## Related

- [4 - Pods and Workload Resources](./4-pods-and-workload-resources.md)
- [7 - Storage — Volumes, PV, PVC, CSI](./7-storage-volumes-pv-pvc-csi.md)
- [9 - RBAC, Service Accounts, and Security](./9-rbac-service-accounts-and-security.md)
- [12 - Observability and Production Operations](./12-observability-and-production-operations.md)
