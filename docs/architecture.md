# Cluster Architecture

## Hardware

| Role | Machine | Notes |
|------|---------|-------|
| Master / Control Plane | HP ProBook 4520s | Static IP `192.168.8.10` |
| Worker Node | Asus X555L | Static IP `192.168.8.11` — added April 2026 |

K3s was installed with `--disable=traefik` and `--disable=servicelb` to remove the defaults and replace them with MetalLB and the NGINX Inc ingress controller managed via GitOps.

---

## Networking

```
Internet
    │
    ▼
Cloudflare Tunnel (cloudflared)     ← no open ports on home router
    │
    ▼
192.168.8.100  (MetalLB assigned IP)
    │
    ▼
NGINX Ingress Controller  (nginx-ingress namespace)
    │
    ├── danielmorgsilva.dev/          → nginx-service (default namespace)
    └── danielmorgsilva.dev/linkding  → linkding service (linkding namespace)
```

MetalLB operates in L2 mode and assigns IPs from the pool `192.168.8.100–192.168.8.140`.

The cluster is publicly accessible via Cloudflare Tunnel with no open ports on the home router and no home IP exposure. Cloudflare forwards `danielmorgsilva.dev` and `linkding.danielmorgsilva.dev` to `https://192.168.8.100`, using Match SNI to Host for correct TLS hostname matching.

---

## Ingress: Mergeable Ingresses

This cluster uses the **NGINX Inc** ingress controller (not the community `ingress-nginx`). Unlike the community controller, the NGINX Inc controller does not merge Ingress rules across namespaces for the same host — it rejects duplicates with `host is taken by another resource`.

The solution is **Mergeable Ingresses**:
- One `master` Ingress in `infrastructure/config/homelab/nginx/` owns the hostname with no paths defined
- Each app in its own namespace defines a `minion` Ingress that adds paths

```yaml
# Master (infrastructure/config namespace)
annotations:
  nginx.org/mergeable-ingress-type: "master"
  nginx.org/redirect-to-https: "true"

# Minion (app namespace)
annotations:
  nginx.org/mergeable-ingress-type: "minion"
```

TLS must be configured on the master ingress only — the NGINX Inc controller rejects TLS blocks on minion ingresses.

---

## TLS

Wildcard TLS certificates for `*.danielmorgsilva.dev` and `danielmorgsilva.dev` are issued by Let's Encrypt via cert-manager using the DNS-01 challenge (Cloudflare API token). The certificate lives in the `cert-manager` namespace and is reflected to other namespaces by reflector.

### reflector

`reflector` (emberstack/reflector) mirrors secrets across namespaces. The Certificate resource uses `secretTemplate.annotations` to configure automatic reflection. All four annotations are required:

```yaml
reflector.v1.k8s.emberstack.com/reflection-allowed: "true"
reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces: "linkding"
reflector.v1.k8s.emberstack.com/reflection-auto-enabled: "true"
reflector.v1.k8s.emberstack.com/reflection-auto-namespaces: "linkding"
```

> Important: `reflection-auto-enabled` requires `reflection-allowed` to also be `true`. Without both, reflector silently skips mirroring.

The TLS secret is then referenced by the master Ingress in `infrastructure/config/homelab/nginx/`.

---

## Flux CD Dependency Chain

Flux reconciles resources in a specific order to avoid race conditions. There are four independent chains running in parallel:

```
metallb-controllers → metallb-config → nginx-ingress-controllers → nginx-config

infrastructure-controllers → infrastructure-config

cloudflared-config → cloudflared-controllers

apps  (independent, watches apps/homelab)
```

cloudflared is split into two kustomizations deliberately: `cloudflared-config` creates the SOPS-decrypted tunnel token secret first, then `cloudflared-controllers` deploys the cloudflared Deployment that mounts it.

Each step uses `dependsOn` and `wait: true` so Flux blocks until CRDs and controllers are fully ready before proceeding.

---

## Secret Management

Secrets are encrypted at rest in Git using [SOPS](https://github.com/getsops/sops) with an age key pair.

- The age private key is stored as a Kubernetes secret (`sops-age`) in the `flux-system` namespace
- The `.sops.yaml` at `clusters/homelab/.sops.yaml` defines the encryption rules
- Flux decrypts secrets at apply time via the `decryption` block in each Kustomization:

```yaml
spec:
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

> Important: every Flux Kustomization that includes SOPS-encrypted files must have this decryption block, otherwise secrets are applied with their encrypted values intact.
