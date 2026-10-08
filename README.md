# AAP GitOps

Deploy Ansible Automation Platform on OpenShift using ArgoCD GitOps with an external CloudNativePG PostgreSQL database.

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
  HUB_STORAGE_CLASS: ocs-storagecluster-cephfs   # RWX storage class for Hub
```

Also update `namespace:` in `site/kustomization.yaml` to match your `NAMESPACE`.

Update `repoURL:` in `argocd/app.yaml` to point to your fork.

### 2. Deploy the ArgoCD Application

```bash
oc apply -f argocd/app.yaml
```

ArgoCD uses sync waves to deploy in order:
- **Wave 0** — Namespace, OperatorGroup, Subscription (AAP operator)
- **Wave 14** — CNPG Postgres cluster
- **Wave 15** — Postgres databases
- **Wave 20** — AnsibleAutomationPlatform CR

### 3. Create the database secrets

Once the Postgres cluster is healthy, get the password:

```bash
oc get secret <PG_CLUSTER_NAME>-superuser -n <NAMESPACE> \
  -o jsonpath='{.data.password}' | base64 -d
```

Copy and edit the template:

```bash
cp site/postgres-secrets.yaml.template /tmp/postgres-secrets.yaml
# Replace CHANGEME_NAMESPACE, CHANGEME_PG_CLUSTER, CHANGEME_PASSWORD
oc apply -f /tmp/postgres-secrets.yaml
rm /tmp/postgres-secrets.yaml
```

### 4. Verify

```bash
oc get pods -n <NAMESPACE>
oc get route -n <NAMESPACE> | grep gateway
oc get secret <INSTANCE_NAME>-admin-password -n <NAMESPACE> \
  -o jsonpath='{.data.password}' | base64 -d
```

## Prerequisites

- OpenShift cluster with:
  - **OpenShift GitOps (ArgoCD)** installed
  - **CloudNativePG operator** installed
  - A default StorageClass (or ODF)
- `oc` CLI authenticated to the cluster

## Repo Structure

```
aap-gitops/
├── base/                              # Generic manifests — don't edit
│   ├── operator/                      #   Namespace, OperatorGroup, Subscription
│   ├── cnpg/                          #   CNPG Cluster + Databases
│   └── platform/                      #   AnsibleAutomationPlatform CR
├── site/                              # Your configuration
│   ├── site-config.yaml               #   ← THE VARS FILE (edit this)
│   ├── kustomization.yaml             #   Wiring + numeric values
│   └── postgres-secrets.yaml.template #   DB secrets (apply manually)
└── argocd/
    └── app.yaml                       # Bootstrap ArgoCD Application
```

## What gets deployed

| Component | Description |
|---|---|
| AAP Operator | OLM subscription on your chosen channel |
| CNPG Postgres | Dedicated PostgreSQL cluster with 4 databases |
| Automation Controller | Playbook engine |
| Automation Hub | Collection/EE registry |
| Event-Driven Ansible | Event-based automation |
| AAP Gateway | Unified entry point |

## Notes

- **Secrets are NOT stored in Git.** Use the template to create them manually, or adopt [SOPS](https://github.com/getsops/sops) / [SealedSecrets](https://github.com/bitnami-labs/sealed-secrets).
- **Postgres sizing** (instances, storage) is configured in `site/kustomization.yaml` under `patches`.
- **Coexistence** — multiple AAP instances can run on the same cluster in different namespaces.
