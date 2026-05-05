# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A bare-metal Kubernetes homelab managed entirely via GitOps (Flux CD v2). All cluster state lives in this repo as YAML — there are no build steps, no scripts to run, and no test suite. Changes are deployed by pushing to `main`, after which Flux reconciles automatically.

## Deployment

The only "build" command is:
```bash
git push
```
Flux watches GitHub and applies changes within its reconciliation interval (10 minutes default). To force immediate reconciliation from the cluster:
```bash
flux reconcile kustomization <name> --with-source
```

Validate YAML before pushing:
```bash
kubectl apply --dry-run=client -f <file>
```

## Repository Structure

```
clusters/homelab/     # Flux entry points — infrastructure.yaml and apps.yaml
infrastructure/
  controllers/        # HelmRelease resources for cluster-level tools
  config/             # Cluster-level configuration (IP pools, Ingress master, ClusterIssuers)
apps/
  base/               # Shared namespace + kustomization definitions
  homelab/            # Per-app directories (whoop, linkding, nginx)
docs/                 # architecture.md + journal.md (learning narrative)
```

## Core Architecture

**Flux reconciles two independent chains:**

1. **Infrastructure** (`clusters/homelab/infrastructure.yaml`): `metallb` → `nginx-ingress` → `cert-manager` → `reflector`. Each step has explicit `dependsOn` to enforce ordering. Breaking this chain is the most common source of cluster issues.

2. **Apps** (`clusters/homelab/apps.yaml`): Watches `apps/homelab/`, deploys all apps with SOPS decryption enabled.

## Key Patterns

### Mergeable Ingresses
This cluster runs the **NGINX Inc official controller** (not community `ingress-nginx`). It does not merge Ingress rules across namespaces automatically. The solution used here is the Mergeable Ingresses pattern:
- One **master** Ingress in `infrastructure/config/homelab/nginx/` owns the hostname with no paths
- Each app has a **minion** Ingress in its own namespace that adds paths
- Master and minion are linked via annotations: `nginx.org/mergeable-ingress-type: "master"` / `"minion"`

### SOPS Secrets
Secrets are encrypted with `age` and committed to Git. Any Kustomization that uses secrets must include:
```yaml
decryption:
  provider: sops
  secretRef:
    name: sops-age
```
The `sops-age` secret lives in `flux-system`. SOPS rules are in `clusters/homelab/.sops.yaml`.

### Secret Reflection
`reflector` syncs secrets across namespaces. TLS secrets created by cert-manager in one namespace are reflected to others via annotations on the Secret resource.

## Tech Stack

| Component | Tool | Version |
|-----------|------|---------|
| Kubernetes | K3s | — |
| GitOps | Flux CD v2 | — |
| Load Balancer | MetalLB | 0.15.3 (L2 mode, pool: 192.168.8.100–140) |
| Ingress | NGINX Inc controller | 2.4.x |
| PostgreSQL | CloudNativePG operator | 1.28+ |
| TLS | cert-manager + Let's Encrypt | 1.20.x (Cloudflare DNS01) |
| Secrets | SOPS + age | — |
| Secret sync | reflector | — |

## Applications

- **whoop**: Health data pipeline. CronJob scrapes Whoop API daily at 6 AM, stores to CloudNativePG cluster (3-node HA). Secrets (API tokens) are SOPS-encrypted.
- **linkding**: Bookmark manager at `danielmorgsilva.net/linkding`. Single-pod deployment with PVC for SQLite persistence. Requires `LD_CONTEXT_PATH=/linkding` env var.
