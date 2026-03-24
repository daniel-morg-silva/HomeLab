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

## Configuration

### MetalLB IP Pool
Defined in `config/homelab/metallb/metallb-config.yaml`:
- **Pool:** `192.168.8.100 – 192.168.8.140`
- **Mode:** L2 Advertisement

## Dependency Chain

MetalLB must be fully ready before its IP pool is configured, and both must be ready before NGINX can obtain a LoadBalancer IP:

```
metallb-controllers → infrastructure-config → nginx-ingress-controllers
```

This is enforced via `dependsOn` in `clusters/homelab/infrastructure.yaml`.
