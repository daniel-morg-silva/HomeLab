# Infrastructure

Cluster-level controllers and configuration, all managed via Flux CD HelmReleases.

## Controllers

### MetalLB
- **Namespace:** `metallb-system`
- **Purpose:** Provides LoadBalancer-type services on bare metal by assigning IPs from a local network pool
- **Chart version:** `0.15.3`

### NGINX Ingress Controller (NGINX Inc)
- **Namespace:** `nginx-ingress`
- **Purpose:** Routes external HTTP traffic to internal services based on host/path rules
- **Chart version:** `2.4.x`
- **Note:** This is the official NGINX Inc controller (`nginx/nginx-ingress`), not the community `ingress-nginx` controller. They behave differently — see [architecture.md](../docs/architecture.md) for the Mergeable Ingresses pattern required for multi-namespace routing.

### CloudNativePG
- **Namespace:** `cnpg-system`
- **Purpose:** Kubernetes operator that manages PostgreSQL clusters declaratively
- **Handles:** Automated failover, backups, scaling, and secret generation for database credentials

### cert-manager
- **Namespace:** `cert-manager`
- **Purpose:** Automates TLS certificate issuance and renewal from Let's Encrypt
- **Chart source:** `oci://quay.io/jetstack/charts` (OCI registry)
- **Challenge type:** DNS-01 via Cloudflare API token
- **Certificate:** Wildcard for `*.danielmorgsilva.dev` and `danielmorgsilva.dev`
- **CRDs:** Installed via `values.crds.enabled: true`

### reflector
- **Namespace:** `reflector` (emberstack/reflector)
- **Purpose:** Mirrors Kubernetes secrets across namespaces
- **Use case:** Syncs the wildcard TLS secret from `cert-manager` into app namespaces (e.g. `linkding`) so the master Ingress can reference it

### cloudflared
- **Namespace:** `cloudflared`
- **Purpose:** Cloudflare Tunnel agent — exposes cluster services publicly with no open ports on the home router
- **Tunnel token:** SOPS-encrypted secret
- **Routes:** `danielmorgsilva.dev` and `linkding.danielmorgsilva.dev` → `https://192.168.8.100`

---

## Configuration

### MetalLB IP Pool
Defined in `config/homelab/metallb/metallb-config.yaml`:
- **Pool:** `192.168.8.100 – 192.168.8.140`
- **Mode:** L2 Advertisement

### NGINX Master Ingress
Defined in `config/homelab/nginx/`:
- Owns the hostname `danielmorgsilva.dev` with no paths (master Ingress pattern)
- Holds the TLS configuration and redirect-to-HTTPS annotation
- App namespaces attach via minion Ingresses

---

## Dependency Chain

Four independent chains run in parallel:

```
metallb-controllers → metallb-config → nginx-ingress-controllers → nginx-config

infrastructure-controllers → infrastructure-config

cloudflared-config → cloudflared-controllers

apps  (independent)
```

cloudflared is split into two kustomizations: `cloudflared-config` creates the SOPS-decrypted tunnel token secret first, then `cloudflared-controllers` deploys the cloudflared Deployment that depends on it.

Enforced via `dependsOn` in `clusters/homelab/infrastructure.yaml`.
