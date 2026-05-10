# Cluster Architecture

## Hardware

| Role | Machine | Cluster IP | Home Network IP |
|------|---------|------------|-----------------|
| Master / Control Plane | HP ProBook 4520s | `10.0.0.1` (`ens5.10`) | `192.168.87.10` (`ens5.1`) |
| Worker Node | Asus X555L | `10.0.0.2` (`enp2s0.10`) | `192.168.87.11` (`enp2s0.1`) |

K3s was installed with `--disable=traefik` and `--disable=servicelb` to remove the defaults and replace them with MetalLB and the NGINX Inc ingress controller managed via GitOps.

K3s uses the `10.0.0.x` cluster network for inter-node traffic (flannel VXLAN, kubelet, control plane communication). The `192.168.87.x` home network is used only for external-facing services (MetalLB LoadBalancer IPs, ingress traffic from Cloudflare Tunnel).

---

## Physical Network

A TP-Link TL-SG605E managed switch connects both nodes and the home router, with 802.1Q VLAN tagging to isolate cluster traffic from home network traffic.

```
                    ┌─────────────────────────┐
                    │   TL-SG605E Switch      │
                    │                         │
  Master ───────────┤ Port 1  (tagged 1, 10)  │
  Worker ───────────┤ Port 2  (tagged 1, 10)  │
  (future nodes) ───┤ Port 3  (tagged 1, 10)  │
  (future nodes) ───┤ Port 4  (tagged 1, 10)  │
  Home Router ──────┤ Port 5  (untagged 1)    │
                    └─────────────────────────┘
```

**VLAN 1 (default)** — home network, internet access, MetalLB LoadBalancer traffic. Untagged on the router uplink, tagged on node ports.

**VLAN 10 (cluster)** — private cluster network for K3s inter-node traffic. Tagged on node ports only, no router access. Subnet `10.0.0.0/24`.

Each node uses 802.1Q sub-interfaces configured in netplan: `ens5.1`/`ens5.10` on the master, `enp2s0.1`/`enp2s0.10` on the worker. The Linux kernel handles VLAN tagging; the switch enforces traffic isolation.

This setup means cluster operations are independent of the home network — changing routers, switching ISPs, or moving the cluster to a different network does not require reconfiguring K3s.

---

## Networking

```
Internet
    │
    ▼
Cloudflare Tunnel (cloudflared)     ← no open ports on home router
    │
    ▼
192.168.87.100  (MetalLB assigned IP)
    │
    ▼
NGINX Ingress Controller  (nginx-ingress namespace)
    │
    ├── danielmorgsilva.dev/                  → danielmorgsilva-dev service
    ├── linkding.danielmorgsilva.dev/         → linkding service
    └── grafana.danielmorgsilva.dev/          → prometheus-grafana service
```

MetalLB operates in L2 mode and assigns IPs from the pool `192.168.87.100–192.168.87.140` on the home network (VLAN 1). MetalLB speakers run on both nodes and communicate over the cluster network (VLAN 10).

The cluster is publicly accessible via Cloudflare Tunnel with no open ports on the home router and no home IP exposure. Cloudflare forwards traffic to `https://192.168.87.100`, using **Match SNI to Host** to ensure correct TLS hostname matching when connecting via IP.

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

TLS must be configured on the master ingress only — the NGINX Inc controller rejects TLS blocks on minion ingresses. The `nginx.org/redirect-to-https` annotation must also be set only on the master; setting it on a minion causes nginx to reject the configuration.

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

> Important: `reflection-auto-enabled` requires `reflection-allowed` to also be `true`.