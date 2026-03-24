# Cluster Architecture

## Hardware

| Role | Machine | Notes |
|------|---------|-------|
| Master / Control Plane | HP ProBook 4520s | Static IP `192.168.8.10` |
| Worker Node | HP Pavilion dm1 | Static IP `192.168.8.11` |

K3s was installed with `--disable=traefik` and `--disable=servicelb` to remove the defaults and replace them with MetalLB and the NGINX Inc ingress controller managed via GitOps.

---

## Networking

```
Internet / Home Network
        │
        ▼
  192.168.8.100  (MetalLB assigned IP)
        │
        ▼
NGINX Ingress Controller  (nginx-ingress namespace)
        │
        ├── danielmorgsilva.net/          → nginx-service (default namespace)
        └── danielmorgsilva.net/linkding  → linkding service (linkding namespace)
```

MetalLB operates in L2 mode and assigns IPs from the pool `192.168.8.100–192.168.8.140`.

### Ingress: Mergeable Ingresses

This cluster uses the **NGINX Inc** ingress controller (not the community `ingress-nginx`). Unlike the community controller, the NGINX Inc controller does not merge Ingress rules across namespaces for the same host — it rejects duplicates with `host is taken by another resource`.

The solution is **Mergeable Ingresses**:
- One `master` Ingress in `default` owns the hostname with no paths defined
- Each app in its own namespace defines a `minion` Ingress that adds paths

```yaml
# Master (default namespace)
annotations:
  nginx.org/mergeable-ingress-type: "master"

# Minion (app namespace)
annotations:
  nginx.org/mergeable-ingress-type: "minion"
```

---

## Flux CD Dependency Chain

Flux reconciles resources in a specific order to avoid race conditions (e.g. NGINX waiting for a LoadBalancer IP while MetalLB is still deploying):

```
flux-system  (watches clusters/homelab)
      │
      ├── infrastructure-controllers  (MetalLB Helm chart)
      │         │
      │         └── metallb-controllers
      │                   │
      │                   └── infrastructure-config  (IPAddressPool, L2Advertisement)
      │                             │
      │                             └── nginx-ingress-controllers
      │
      └── apps  (watches apps/homelab — all app deployments)
```

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
