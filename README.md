# AAP GitOps

Deploy Ansible Automation Platform on OpenShift using ArgoCD GitOps with an external CloudNativePG PostgreSQL database and automated secret management.

**Fork this repo, edit one file, deploy.**

## Quick Start

### 1. Fork and configure

Fork this repo, then edit `site/site-config.yaml` with your values:

```yaml
data:
  NAMESPACE: aap-cc                              # your namespace
  INSTANCE_NAME: aap-cc                          # AAP CR name
  PG_CLUSTER_NAME: aap-cc-pg                     # Postgres cluster name
  AAP_CHANNEL: stable-2.7                        # operator channel
  PG_INSTANCES: "2"                              # Postgres replicas
  PG_STORAGE_SIZE: "50Gi"                        # Postgres storage
  HUB_STORAGE_CLASS: ocs-storagecluster-cephfs   # RWX storage class for Hub
  PG_HOST: aap-cc-pg-rw.aap-cc.svc              # {PG_CLUSTER_NAME}-rw.{NAMESPACE}.svc
  PG_APP_SECRET: aap-cc-pg-app                   # {PG_CLUSTER_NAME}-app
```

Update `repoURL` in `argocd/app.yaml` to point to your fork.

### 2. Deploy

```bash
oc apply -f argocd/app.yaml
```

That's it. ArgoCD handles the rest using sync waves:

| Wave | What happens |
|---|---|
| 0 | Namespace, OperatorGroup, Subscription (AAP operator) |
| 14 | CNPG Postgres cluster |
| 15 | Postgres databases (controller, hub, eda, gateway) |
| 16 | External Secrets RBAC + SecretStore |
| 17 | ExternalSecrets — auto-creates DB connection secrets from CNPG password |
| 20 | AnsibleAutomationPlatform CR (Controller, Hub, EDA, Gateway) |

### 3. Verify

```bash
oc get pods -n <NAMESPACE>
oc get route -n <NAMESPACE> | grep gateway
oc get secret <INSTANCE_NAME>-admin-password -n <NAMESPACE> \
  -o jsonpath='{.data.password}' | base64 -d
```

Login with username `admin` and the password above.

## Prerequisites

- OpenShift cluster with:
  - **OpenShift GitOps (ArgoCD)** installed
  - **CloudNativePG operator** installed
  - **External Secrets Operator** installed
  - A default StorageClass (or ODF)
- `oc` CLI authenticated to the cluster

## Repo Structure

```
aap-gitops/
├── base/                              # Generic manifests — don't edit
│   ├── operator/                      #   Namespace, OperatorGroup, Subscription
│   ├── cnpg/                          #   CNPG Cluster + Databases
│   ├── secrets/                       #   RBAC, SecretStore, ExternalSecrets
│   └── platform/                      #   AnsibleAutomationPlatform CR
├── site/                              # Your configuration
│   ├── site-config.yaml               #   ← THE VARS FILE (edit this)
│   └── kustomization.yaml             #   Wiring (replacements + resources)
└── argocd/
    └── app.yaml                       # Bootstrap ArgoCD Application
```

## How It Works

### Configuration flow

All values in `site/site-config.yaml` are injected into the base manifests via
Kustomize `replacements` defined in `site/kustomization.yaml`. The base files
contain `__replaced_by_site_config__` placeholders that are never manually edited.

### Secret management

Database credentials are handled automatically — no manual secret creation needed:

```
CNPG Cluster (wave 14)
  └── auto-creates Secret: {PG_CLUSTER_NAME}-app (username + password)
        │
SecretStore (wave 16)
  └── reads secrets from the target namespace via a ServiceAccount
        │
ExternalSecrets ×4 (wave 17)
  ├── pulls username + password from CNPG-generated secret
  └── templates full connection secrets (host, port, database, sslmode, type)
        │
AAP Platform CR (wave 20)
  └── references the 4 connection secrets — fully wired
```

The External Secrets Operator (already installed on OpenShift) syncs the
CNPG-generated password into properly formatted connection secrets for each
AAP component (gateway, controller, hub, eda).

## Site Config Variables

| Variable | Description | Example |
|---|---|---|
| `NAMESPACE` | Target namespace for all resources | `aap-cc` |
| `INSTANCE_NAME` | AAP custom resource name | `aap-cc` |
| `PG_CLUSTER_NAME` | CNPG Postgres cluster name | `aap-cc-pg` |
| `AAP_CHANNEL` | AAP operator subscription channel | `stable-2.7` |
| `PG_INSTANCES` | Number of Postgres replicas | `"2"` |
| `PG_STORAGE_SIZE` | Postgres PVC storage size | `"50Gi"` |
| `HUB_STORAGE_CLASS` | Storage class for Hub files (must be RWX) | `ocs-storagecluster-cephfs` |
| `PG_HOST` | Postgres service hostname (`{PG_CLUSTER_NAME}-rw.{NAMESPACE}.svc`) | `aap-cc-pg-rw.aap-cc.svc` |
| `PG_APP_SECRET` | CNPG app user secret (`{PG_CLUSTER_NAME}-app`) | `aap-cc-pg-app` |

## What Gets Deployed

| Component | Description |
|---|---|
| AAP Operator | OLM subscription on your chosen channel |
| CNPG Postgres | Dedicated PostgreSQL cluster with 4 databases |
| External Secrets | RBAC + SecretStore + 4 ExternalSecrets for DB credentials |
| Automation Controller | Playbook execution engine |
| Automation Hub | Private collection/EE registry |
| Event-Driven Ansible | Event-based automation triggers |
| AAP Gateway | Unified entry point for all components |

## Notes

- **No secrets in Git.** Database passwords are auto-generated by CNPG and synced by the External Secrets Operator.
- **Coexistence.** Multiple AAP instances can run on the same cluster in different namespaces. The OperatorGroup scopes each AAP operator to its own namespace.
- **Base files are generic.** Anyone can fork this repo, edit `site/site-config.yaml` and `argocd/app.yaml`, and deploy their own AAP instance.
