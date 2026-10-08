# AAP GitOps — aap-cc

Deploy Ansible Automation Platform 2.7 on OpenShift using ArgoCD GitOps with an external CloudNativePG (CNPG) PostgreSQL database.

## Overview

This repo deploys a fully GitOps-managed AAP 2.7 instance named **aap-cc** with:

- **Automation Controller** — job engine for running playbooks
- **Automation Hub** — private collection/EE registry
- **Event-Driven Ansible (EDA)** — event-based automation triggers
- **AAP Gateway** — unified entry point for all components
- **External PostgreSQL** — dedicated CNPG cluster (`aap-cc-pg`) with separate databases per component

## Prerequisites

- OpenShift cluster with:
  - **OpenShift GitOps (ArgoCD)** installed
  - **CloudNativePG operator** installed (in `cnpg-system`)
  - **OpenShift Data Foundation (ODF)** or a default StorageClass available
- `oc` CLI authenticated to the cluster

## Repo Structure

```
aap-gitops/
├── argocd/                          # ArgoCD Application manifests
│   ├── aap-cc-operator.yaml        #   AAP operator subscription
│   ├── aap-cc-platform.yaml        #   AAP platform CR (Controller, Hub, EDA, Gateway)
│   └── cnpg-cluster.yaml           #   CNPG Postgres cluster + databases
├── operators/
│   └── aap-cc/                      # AAP operator installation
│       ├── namespace.yaml           #   aap-cc namespace
│       ├── operatorgroup.yaml       #   Scoped to aap-cc only
│       ├── subscription.yaml        #   stable-2.7 channel
│       └── kustomization.yaml
└── operands/
    ├── aap-cc/                      # AAP platform
    │   ├── aap.yaml                 #   AnsibleAutomationPlatform CR
    │   ├── postgres-secrets.yaml.template  # DB connection secrets (template)
    │   └── kustomization.yaml
    └── cnpg/                        # PostgreSQL
        ├── cluster.yaml             #   2-instance PG16 cluster (aap-cc-pg)
        ├── databases.yaml           #   4 databases: controller, hub, eda, gateway
        └── kustomization.yaml
```

## Deployment Steps

### Step 1 — Install the AAP Operator

```bash
oc apply -f argocd/aap-cc-operator.yaml
```

Wait for the operator to install (~2–3 minutes):

```bash
oc get csv -n aap-cc -w
```

Look for `Succeeded` in the PHASE column for the AAP operator.

### Step 2 — Deploy the PostgreSQL Cluster

```bash
oc apply -f argocd/cnpg-cluster.yaml
```

Wait for the Postgres cluster to become healthy (~1–2 minutes):

```bash
oc get cluster.postgresql.cnpg.io -n aap-cc -w
```

Look for `Cluster in healthy state` in the STATUS column.

### Step 3 — Create the Database Connection Secrets

Get the CNPG-generated password:

```bash
oc get secret aap-cc-pg-superuser -n aap-cc \
  -o jsonpath='{.data.password}' | base64 -d
```

Copy the template and fill in the real password:

```bash
cp operands/aap-cc/postgres-secrets.yaml.template /tmp/postgres-secrets.yaml
```

Edit `/tmp/postgres-secrets.yaml` and replace all `CHANGEME` values with the password from above.

Apply the secrets:

```bash
oc apply -f /tmp/postgres-secrets.yaml
```

Clean up:

```bash
rm /tmp/postgres-secrets.yaml
```

### Step 4 — Deploy the AAP Platform

```bash
oc apply -f argocd/aap-cc-platform.yaml
```

This deploys the `AnsibleAutomationPlatform` CR which creates Controller, Hub, EDA, and Gateway. The operator will take ~5–10 minutes to fully reconcile.

Monitor progress:

```bash
oc get aap -n aap-cc -w
```

### Step 5 — Verify

Check all pods are running:

```bash
oc get pods -n aap-cc
```

Get the AAP URL:

```bash
oc get route -n aap-cc | grep gateway
```

Get the admin password:

```bash
oc get secret aap-cc-admin-password -n aap-cc \
  -o jsonpath='{.data.password}' | base64 -d
```

Login with username `admin` and the password above.

## Namespaces

| Namespace | Contents |
|---|---|
| `aap-cc` | AAP operator, all AAP components, CNPG Postgres cluster |
| `cnpg-system` | CNPG operator (shared, already exists on cluster) |
| `openshift-gitops` | ArgoCD Applications |

## Notes

- **Secrets are NOT stored in Git.** The `postgres-secrets.yaml.template` file is a template only. For production, consider using [SOPS](https://github.com/getsops/sops) or [SealedSecrets](https://github.com/bitnami-labs/sealed-secrets).
- **Storage:** Postgres uses the cluster default StorageClass (`ocs-storagecluster-ceph-rbd`). Hub file storage uses `ocs-storagecluster-cephfs`.
- **Coexistence:** This deployment is fully independent from the existing AAP in the `aap` namespace. The OperatorGroup scopes the AAP operator to `aap-cc` only.
