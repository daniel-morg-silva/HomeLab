# Learning Journal

Chronological record of building this homelab from scratch, including every major problem encountered and how it was solved.

---

## Phase 1: Virtual Foundations (January 2026)

Focused on learning basic networking and Kubernetes concepts in a safe virtual environment before committing to bare metal. Heavily guided by AI during this phase.

**Stack:**
- Hypervisor: Virtual Machine Manager (QEMU/KVM)
- Guest OS: Ubuntu Server
- Orchestration: K3s
- Database: PostgreSQL
- Management: PgAdmin4

### 13 January 2026 — Virtual Network Infrastructure

Created a dedicated virtual network bridge to enable VM-to-VM communication. Provisioned two Ubuntu Server VMs for the Kubernetes control plane and worker node roles.

**Resources:**
- [KVM Host Network Configurations using Virt-Manager](https://cloudspinx.com/kvm-host-network-configurations-using-virt-manager/)
- [Virtual Machine Manager Documentation](https://virt-manager.org/screenshots.html)

### 15 January 2026 — K3s Cluster

Initialized a master node and joined a worker node. Verified cluster health with `kubectl get nodes`.

**Resources:**
- [K3s Quick-Start Guide](https://docs.k3s.io/quick-start)
- [Kubernetes kubectl Command Reference](https://kubernetes.io/docs/reference/kubectl/)

### 16 January 2026 — Whoop API Integration

Registered an OAuth 2.0 application with the Whoop API and wrote a Python script to authenticate and retrieve an access token.

**Resources:**
- [Whoop API Documentation](https://developer.whoop.com/api)
- [Python whoopy Library](https://github.com/felixnext/whoopy)

### 17 January 2026 — PostgreSQL + PgAdmin4

Deployed PostgreSQL as a StatefulSet inside K3s. Created the database schema for Whoop data (sleep, workouts, recovery). Deployed PgAdmin4 as a web-based management UI.

**Resources:**
- [PostgreSQL Kubernetes Deployment](https://www.digitalocean.com/community/tutorials/how-to-deploy-postgres-to-kubernetes-cluster)
- [PgAdmin4 Official Site](https://www.pgadmin.org/)

### 18 January 2026 — Containerisation

Enhanced the Python script to retrieve sleep cycles and workout summaries. Created a Dockerfile to package the script and its dependencies. Built and tested the image locally.

**Resources:**
- [Docker Get Started](https://docs.docker.com/get-started/)

---

## Phase 2: Bare-Metal (February – March 2026)

Taking everything learned in Phase 1 and applying it to real hardware. Less AI guidance, more deliberate learning. Focus on understanding fundamentals rather than fast progress.

**Hardware:**
- Master Node: HP ProBook 4520s
- Worker Node: HP Pavilion dm1
- OS: Ubuntu 24.04.3 LTS

---

### 5 February 2026 — Bare-Metal K3s Cluster

Performed a clean Ubuntu Server installation on both laptops. Assigned static IPs (`192.168.8.10` master, `192.168.8.11` worker). Installed K3s on the ProBook as master, joined the Pavilion as a worker.

**Resources:**
- [Turning an Old Laptop into a Home Server (2026)](https://www.youtube.com/watch?v=46T4cDQBkDs&t=255s)
- [K3s Installation Guide](https://docs.k3s.io/installation)

---

### 6 February 2026 — First Ingress

Deployed NGINX and exposed it via the Traefik ingress controller (K3s default). Created a ClusterIP Service and an Ingress resource with a hostname rule. Added a local `/etc/hosts` entry to test from the home network.

**Key Learning:** The Ingress controller is a traffic router (application layer). The Service is an internal load balancer. They serve different purposes and work together.

**Resources:**
- [Kubernetes Ingress Tutorial for Beginners](https://www.youtube.com/watch?v=80Ew_fsV4rM)
- [Kubernetes Networking Explained](https://www.youtube.com/watch?v=J8jAzqbXxjE)

---

### 10 February 2026 — MetalLB + NGINX Ingress Controller

Replaced Traefik and K3s's default service load balancer with MetalLB and the NGINX Inc ingress controller.

**What was done:**
- Deployed MetalLB via Helm into `metallb-system`, configured an `IPAddressPool` for range `192.168.8.100–192.168.8.220`
- Deployed NGINX Ingress Controller via Helm, configured as `LoadBalancer` type — MetalLB assigned it `192.168.8.100`
- Validated the full traffic path: Client → MetalLB IP → NGINX Ingress → Service → Pod

**Key Learning:** MetalLB handles Layer 3/4 (IP assignment). NGINX handles Layer 7 (HTTP routing by host/path). These are complementary, not competing.

**Resources:**
- [MetalLB Documentation](https://metallb.io/configuration/)
- [NGINX Ingress Controller Documentation](https://docs.nginx.com/nginx-ingress-controller/configuration/ingress-resources/basic-configuration/)
- [Step 4: Install MetalLB on K3s Cluster](https://www.youtube.com/watch?v=uDBpPI_tz4Q)

---

### 17 February 2026 — GitOps with Flux CD

Rebuilt the entire cluster from scratch using GitOps. This is the most significant architectural change in the project.

**What was done:**
- Reset both nodes with the K3s uninstall script
- Reinstalled K3s with `--disable=traefik --disable=servicelb`
- Bootstrapped Flux CD, pointing it at this Git repository
- Re-deployed MetalLB, NGINX Ingress, and the NGINX test app — all through Git

**Problem: NGINX hanging on startup**
NGINX's LoadBalancer service was hanging waiting for an IP from MetalLB, but MetalLB wasn't fully deployed yet. Flux was trying to deploy everything simultaneously.

**Solution:** Structured Flux Kustomizations with explicit `dependsOn` chains:
```
metallb-controllers → infrastructure-config → nginx-ingress-controllers
```
Each step uses `wait: true` so Flux blocks until CRDs and pods are fully ready before proceeding.

**Key Learning:** GitOps is not just "put YAML in Git". Dependency ordering is a real operational concern. Infrastructure has to come up in the right sequence.

**Resources:**
- [The GitOps Way to Run Your Kubernetes Homelab in 2025](https://www.youtube.com/watch?v=FcBs2iwXC-U)
- [Advanced K8S with FluxCD](https://www.youtube.com/playlist?list=PLINxtbqxrBCu9Y-r_ZBMqYW8uHeQhLW9P)
- [Flux Documentation](https://fluxcd.io/flux/)
- [flux2-kustomize-helm-example](https://github.com/fluxcd/flux2-kustomize-helm-example)

---

### 24 February 2026 — CloudNativePG

Set up the CloudNativePG operator for declarative PostgreSQL management.

**Why CloudNativePG over a manual StatefulSet:** A StatefulSet requires manually writing backup scripts, failover logic, and secret rotation. CloudNativePG handles all of this through CRDs. One `Cluster` manifest replaces a significant amount of manual YAML and scripting.

**What was done:**
- Deployed CloudNativePG via HelmRelease into `cnpg-system`
- Generated an age key pair and stored the private key as a Kubernetes secret (`sops-age` in `flux-system`)
- Configured `.sops.yaml` for future encrypted secrets

**Key Learning:** Kubernetes operators are a major force multiplier. The operator pattern turns complex stateful applications into manageable custom resources.

**Resources:**
- [CloudNativePG: Kubernetes Databases Made Simple (Full Course)](https://www.youtube.com/watch?v=g59ki9z2SO8&t=201s)
- [CloudNativePG Documentation](https://cloudnative-pg.io/docs/1.28/installation_upgrade)

---

### 8–17 March 2026 — Whoop Data Pipeline

Deployed the full Whoop health data pipeline end-to-end and backfilled 2 years of historical data.

**What was built:**
- CloudNativePG cluster (3 nodes) for PostgreSQL
- SOPS-encrypted secret for Whoop OAuth credentials
- Kubernetes CronJob for daily scraping (6 AM)
- One-off batch Jobs for the 2-year historical backfill
- ~3,500 records across sleep, workouts, cycles, and recovery

**Problem: 401 Unauthorized errors**
The Whoop access token expires. The first run worked, subsequent runs failed.

**Solution:** The scraper handles token refresh — it reads the current tokens from the database, exchanges them for new ones via the Whoop OAuth endpoint, and writes the updated tokens back to the database before proceeding.

**Problem: 429 Rate Limiting**
Requesting 2 years of data in bulk triggered Whoop's API rate limiter.

**Solution:** Batched requests by month, running a separate Job per month. Each Job scraped one month's worth of data, staying within rate limits.

**Problem: Job immutability**
Tried to update a running Job to fix a bug. Kubernetes rejected it — Jobs are immutable once created.

**Solution:** Delete the Job first (`kubectl delete job <name> -n whoop`), then recreate it with the corrected manifest.

**Problem: Wrong executable path**
The container's entrypoint wasn't at an obvious location. Attempts to call `whoop-scraper` directly failed.

**Solution:** Inspected the container and found the correct path at `/app/.venv/bin/whoop-scraper`.

**Resources:**
- [CloudNativePG Documentation](https://cloudnative-pg.io/docs/1.28/)
- [Whoop API Developer Portal](https://developer.whoop.com/api)
- [Kubernetes Jobs Documentation](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
- [Whoop Scraper Application](https://github.com/mischavandenburg/whoop-scraper)

---

### 24 March 2026 — Linkding Bookmark Manager

Deployed [Linkding](https://github.com/sissbruecker/linkding), a self-hosted bookmark manager, at `danielmorgsilva.net/linkding`.

This deployment hit the most bugs of any single app so far. Every problem and fix is documented here as a reference.

**Problem: Namespace not appearing in cluster**

The `linkding` namespace was defined in Git but never appeared in the cluster.

**Root cause:** The commit existed locally but had not been pushed to GitHub. Flux pulls from the remote repository — local commits are invisible to it.

**Fix:** `git push`


**Problem: Secret reference wrong type**

The deployment was pulling `LD_SUPERUSER_PASSWORD` via `configMapKeyRef` — but the source was a `Secret`, not a `ConfigMap`.

**Fix:** Changed to `secretKeyRef`. ConfigMaps and Secrets use different ref types even though the syntax looks identical.


**Problem: Secret in wrong namespace**

`secrets.yaml` had `namespace: whoop` left over from copying a template. The deployment was in the `linkding` namespace.

**Root cause:** Kubernetes Secrets are namespace-scoped. A pod in `linkding` cannot mount a secret from `whoop`.

**Fix:** Changed `namespace: whoop` → `namespace: linkding`.


**Problem: Ingress rejected — host already taken**

Created a separate Ingress in the `linkding` namespace for `danielmorgsilva.net/linkding`. The NGINX Inc controller rejected it with:
```
host danielmorgsilva.net is taken by another resource
```

**Root cause:** The NGINX Inc controller (`nginx/nginx-ingress`) treats a hostname as owned by the first Ingress that claims it. This is different from the community `ingress-nginx` controller, which merges rules from multiple Ingress resources across namespaces.

**Fix:** Switched to **Mergeable Ingresses** — the NGINX Inc pattern for multi-namespace routing:
- The existing Ingress in `default` becomes the `master` (owns the host, no paths)
- The linkding Ingress in `linkding` becomes the `minion` (adds paths)

```yaml
annotations:
  nginx.org/mergeable-ingress-type: "master"   # on nginx-ingress in default
  nginx.org/mergeable-ingress-type: "minion"   # on linkding-ingress in linkding
```


**Problem: Master Ingress rejected after adding annotation**

After adding the `master` annotation, the Ingress was still rejected.

**Root cause:** A `master` Ingress cannot define any `http.paths`. The master only defines the host. All paths must live in minion Ingresses.

**Fix:** Removed the `http.paths` block from the master Ingress entirely.


**Problem: 404 on `/linkding` — double slash in paths**

After fixing the ingress, requests reached the linkding pod but returned 404. Logs showed:
```
[uwsgi-static] added mapping for //linkdingstatic => static
WARNING Not Found: /linkding
```

**Root cause:** `LD_CONTEXT_PATH` was set to `/linkding` (with a leading slash). Linkding prepends its own `/` internally, producing `//linkding`.

**Fix:** Changed `LD_CONTEXT_PATH` value from `/linkding` to `linkding` (no leading slash).


**Problem: Login failed — passwords didn't match**

Linkding loaded but login failed for any password entered.

**Root cause:** The `apps` Flux Kustomization was missing the `decryption` block. Flux applied the SOPS-encrypted `secrets.yaml` without decrypting it, so the Kubernetes Secret contained the literal `ENC[AES256_GCM,...]` string as the password. Linkding created the superuser with that encrypted string as the password on first startup.

**Fix:** Added the decryption block to `clusters/homelab/apps.yaml`:
```yaml
spec:
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```
Then forced a pod restart so the superuser was recreated with the correct decrypted password:
```bash
kubectl rollout restart deployment/linkding -n linkding
```
**Resources:**
- [Mergeable Ingress Types Support](https://github.com/nginx/kubernetes-ingress/tree/v5.4.0/examples/ingress-resources/mergeable-ingress-types)
- [Cross-namespace configuration](https://docs.nginx.com/nginx-ingress-controller/configuration/ingress-resources/cross-namespace-configuration/)

---

### 9 April 2026 — New Worker Node (Asus X555L)

Added an Asus X555L as a worker node, bringing the cluster to two machines total. Installed Ubuntu Server 24.04.3 LTS, assigned static IP `192.168.8.11`, and joined it to the K3s cluster. Configured `HandleLidSwitch=ignore` in `/etc/systemd/logind.conf` to keep the node running with the lid closed.

**Problem: Secure Boot blocking USB boot**

The ASUS UEFI firmware failed to initialize the MOK (Machine Owner Key) list on boot, displaying `import_mok_state() failed: Out of Resources` and halting before the installer loaded.

**Root cause:** The laptop's NVRAM didn't have enough space to create the MOK entries required by Secure Boot.

**Fix:** Disabled Secure Boot in the BIOS (F2 on boot → Security tab). Not needed for a homelab node.


**Problem: SSH key authentication failing**

Initial SSH attempts were rejected with `Permission denied (publickey)`. Editing `/etc/ssh/sshd_config` and setting `PasswordAuthentication yes` had no effect.

**Root cause:** Ubuntu Server 24.04 ships with a cloud-init override file at `/etc/ssh/sshd_config.d/50-cloud-init.conf` that takes precedence over the main config. The setting there was still `no`.

**Fix:** Edited `50-cloud-init.conf` instead, restarted SSH, ran `ssh-copy-id`, then reverted the change.


**Problem: Known hosts conflict on `192.168.8.11`**
After arriving in Denmark and connecting the Asus to the homelab network, SSH refused to connect with WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED.

**Root cause:** During setup in Portugal, the Asus was temporarily configured with a different IP (192.168.1.50). That initial connection was recorded in ~/.ssh/known_hosts on the main machine. When the IP was changed to 192.168.8.11 in Denmark, SSH detected a key mismatch for that address and rejected the connection as a potential security issue.

**Fix:**
```bash
ssh-keygen -R 192.168.8.11
```
Then reconnected and accepted the new key.

**Key Learning:** Ubuntu Server 24.04's cloud-init creates SSH config overrides in `sshd_config.d/` that silently win over changes to the main `sshd_config`. Always check that directory when SSH config changes don't take effect.

**Resources:**
- [K3s Installation Guide](https://docs.k3s.io/installation)


---

### April 12–13 2026 - TLS, cert-manager, and Public Exposure via Cloudflare Tunnel

This update focused on securing all cluster services with HTTPS and exposing the homelab publicly — with no open ports on the home router.

#### cert-manager

Deployed cert-manager via Flux following the standard controller pattern (`infrastructure/controllers/cert-manager/`). Used the OCI registry as the chart source (`oci://quay.io/jetstack/charts`) per the official cert-manager recommendation. CRDs were installed via `values.crds.enabled: true`.

A `ClusterIssuer` was configured to use Let's Encrypt with the DNS-01 challenge via Cloudflare API token (stored as a SOPS-encrypted secret in `infrastructure/config/homelab/cert-manager/`). A wildcard `Certificate` resource was created for `*.danielmorgsilva.dev` and `danielmorgsilva.dev`, stored in the `cert-manager` namespace.

**Problem: Let's Encrypt rate limit hit**

During debugging, the certificate secret was deleted multiple times, causing Let's Encrypt to issue 5 certificates against the same domain set within 168 hours — hitting the production rate limit. Resolution window: 2026-04-13 00:29:55 UTC.

**Fix:** Waited for the rate limit window to expire.

**Key Learning:** Never delete the certificate secret during debugging — cert-manager manages renewal automatically. The secret deletion triggers a new issuance attempt, and repeated attempts burn through the rate limit quickly.


#### reflector

Deployed `emberstack/reflector` to mirror the wildcard TLS secret from the `cert-manager` namespace to other namespaces (`linkding`, etc.). The `Certificate` resource uses `secretTemplate.annotations` to configure auto-reflection.

**Problem: Silent skip — reflection not happening**

The TLS secret was not being mirrored to the target namespace despite the reflector pod running.

**Root cause:** `reflection-auto-enabled` requires `reflection-allowed` to also be `true`. Without both annotations, reflector silently skips mirroring.

**Fix:** All four annotations are required on the Certificate's `secretTemplate`:

```yaml
reflector.v1.k8s.emberstack.com/reflection-allowed: "true"
reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces: "linkding"
reflector.v1.k8s.emberstack.com/reflection-auto-enabled: "true"
reflector.v1.k8s.emberstack.com/reflection-auto-namespaces: "linkding"
```


#### TLS on Ingresses

Updated ingresses for `danielmorgsilva.dev` and `linkding.danielmorgsilva.dev` to use the wildcard certificate.

**Problem: TLS rejected on minion Ingress**

Placing the `tls:` block on the minion Ingress caused the NGINX Inc controller to reject it.

**Root cause:** The NGINX Inc controller requires TLS to be configured exclusively on the master Ingress. Minion Ingresses inherit TLS from the master.

**Fix:** Moved the `tls:` block and `nginx.org/redirect-to-https: "true"` annotation to the master Ingress only.

**Key Learning:** The `tls.secretName` must reference a secret in the same namespace as the master Ingress — hence the need for reflector to mirror the cert-manager secret into that namespace.


#### Cloudflare Tunnel

Registered `danielmorgsilva.dev` via Cloudflare Registrar. Deployed `cloudflared` as a Kubernetes Deployment in the `cloudflared` namespace using the official Cloudflare container image, with the tunnel token stored as a SOPS-encrypted secret.

Configured two public application routes in the Cloudflare Zero Trust dashboard:
- `danielmorgsilva.dev` → `https://192.168.8.100`
- `linkding.danielmorgsilva.dev` → `https://192.168.8.100`

Both routes use **Match SNI to Host** to ensure correct TLS hostname matching when connecting via IP.

Both services are now publicly accessible from the internet with valid HTTPS certificates, with no open ports on the home router and no home IP exposure.


#### Flux Dependency Chain Refactor

Refactored the Flux Kustomization structure to remove circular dependencies and add granular ordering. Four independent chains now run in parallel:

```
metallb-controllers → metallb-config → nginx-ingress-controllers → nginx-config

infrastructure-controllers → infrastructure-config

cloudflared-config → cloudflared-controllers

apps  (independent)
```

cloudflared is deliberately split into two kustomizations: `cloudflared-config` runs first (no deps) and creates the SOPS-decrypted tunnel token secret; `cloudflared-controllers` depends on it and deploys the cloudflared Deployment that mounts that secret.

**Resources:**
- [ What Is HTTPS? How Does It Work? ](https://www.youtube.com/watch?v=D7ijCjE31GA)
- [Certificate resource](https://cert-manager.io/docs/usage/certificate/)
- [ Configure cert-manager ClusterIssuer for Cluster-Wide Certificate Authority ](https://oneuptime.com/blog/post/2026-02-09-cert-manager-clusterissuer/view)
- [ Let's Encrypt - Getting Started ](https://letsencrypt.org/getting-started/)
- [ Advanced configuration with Annotations ](https://docs.nginx.com/nginx-ingress-controller/configuration/ingress-resources/advanced-configuration-with-annotations/)
- [ Kubernetes - Ingress ](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [ Reflector - Github ](https://github.com/emberstack/kubernetes-reflector)
- [ Cert-Manager - Syncing Secrets Across Namespaces ](https://cert-manager.io/docs/devops-tips/syncing-secrets-across-namespaces/)
- [ Cert-Manager - Troubleshooting Problems with ACME / Let's Encrypt Certificates ](https://cert-manager.io/docs/troubleshooting/acme/)
- [ Cloudflared - Kubernetes Setup ](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/deployment-guides/kubernetes/)
---

### 16 April 2026 — Zero Trust SSH Access

Configured remote SSH access to the cluster nodes using Cloudflare Access and short-lived certificates - no open ports on the home router, identity-verified on every new connection.

**What was done:**
- Created a Cloudflare Access self-hosted application for `ssh.danielmorgsilva.dev` with an email-based allow policy
- Added a public hostname route in the Cloudflare Tunnel dashboard: `ssh.danielmorgsilva.dev` → `ssh://192.168.8.10:22`
- Fetched Cloudflare's SSH CA public key from `https://danielmorgsilva.cloudflareaccess.com/cdn-cgi/access/certs` and added it as `TrustedUserCAKeys` in `/etc/ssh/sshd_config` on the master node
- Installed `cloudflared` on the client machine (Fedora) and configured `~/.ssh/config` to use it as a ProxyCommand

**How it works:**
Running `ssh ssh.danielmorgsilva.dev` triggers a browser-based authentication flow via Cloudflare Access. On success, Cloudflare issues a short-lived certificate to the client. The SSH session is then proxied through the existing `cloudflared` tunnel in the cluster — no static SSH keys on the client, no open ports on the router.

**Key Learning:** This is the zero-trust SSH pattern used in production environments. Authentication is identity-based and time-limited, rather than relying on long-lived keys sitting on client machines.

**Resources:**
- [Cloudflare Access SSH Documentation](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/use-cases/ssh/ssh-cloudflared-authentication/)

---

### 5–7 May 2026 — Network Migration and VLAN Setup

A network change cascaded into a multi-day cluster recovery and ended in a complete network architecture overhaul. The original setup (both nodes on WiFi, single subnet, IPs assigned by router DHCP) turned out to be the root cause of nearly every problem encountered during this period.

**Trigger:** A roommate left and took the router with them. The new router used a different subnet (`192.168.87.x` instead of `192.168.8.x`), forcing a network change.

**Problem: K3s crash-looping after network change**
After the IP change, K3s crashed continuously with `failed to find interface with specified node ip`. The node IP stored in K3s's SQLite database was the old `192.168.8.10`, and K3s could not find an interface with that IP on the new network. Setting `--node-ip` in `config.yaml` was ignored due to a duplicate flag conflict with the systemd service file. K3s restarted 500+ times over several hours, accumulating database bloat and triggering an IO storm that made the master unresponsive.

**Root cause:** Node IP detection in K3s reads from multiple sources, and conflicting configuration causes silent failures. The `node-ip` flag must be set in exactly one place (the systemd service file), and existing node records in the database must be deleted when the IP changes.

**Fix:**
1. Updated `--node-ip` in `/etc/systemd/system/k3s.service` (master) and `/etc/systemd/system/k3s-agent.service` (worker)
2. Removed `node-ip` from `/etc/rancher/k3s/config.yaml` to avoid duplicate flag
3. Updated `K3S_URL` in `/etc/systemd/system/k3s-agent.service.env` on the worker
4. Deleted node records from kine: `DELETE FROM kine WHERE name LIKE '%/registry/nodes/%'`

**Problem: Worker WiFi unreliable**
With both nodes on WiFi, inter-node traffic suffered up to 66% packet loss. CoreDNS, MetalLB controller, cert-manager webhook, and other components running on the worker became unreachable from the master, causing cascading failures. Pods stuck in `Terminating`, endpoints empty despite running pods, kubelet timeouts.

**Fix (temporary):** Moved critical components to the master via imperative `kubectl patch` with nodeSelector, suspended Prometheus stack to free RAM. This kept the cluster functional but introduced GitOps drift.

**Fix (permanent):** Connected the worker via wired ethernet (`enp2s0`) and configured a static IP in netplan. Inter-node packet loss dropped to 0%.

#### VLAN Setup with TL-SG605E Switch

Bought a TP-Link TL-SG605E managed switch to set up a dedicated private network between the nodes, isolating cluster traffic from the home network entirely.

**Why a dedicated cluster network:** With both nodes on the home network, all inter-node traffic (flannel VXLAN, kubelet, control plane communication) had to traverse the router. This added latency, depended on router stability, and meant any home network change could break the cluster.

**Design:**
- **VLAN 1 (default)** — home network access, ports 1-4 tagged, port 5 untagged (router uplink)
- **VLAN 10 (cluster)** — private cluster network, ports 1-4 tagged, no router access
- Each node uses 802.1Q sub-interfaces: `ens5.1`/`ens5.10` on master, `enp2s0.1`/`enp2s0.10` on worker
- Cluster subnet: `10.0.0.0/24` (master `10.0.0.1`, worker `10.0.0.2`)
- Home subnet preserved: master `192.168.87.10`, worker `192.168.87.11`

**What was done:**
1. Configured 802.1Q VLANs on the switch (VLAN 1 and VLAN 10)
2. Set tagged membership for ports 1-4 on both VLANs, untagged port 5 on VLAN 1
3. Updated netplan on both nodes with VLAN sub-interfaces
4. Updated K3s `--node-ip` to use `10.0.0.x` addresses
5. Added `tls-san` entries in `/etc/rancher/k3s/config.yaml` to include both old and new IPs in the API server certificate
6. Updated `K3S_URL` on the worker to `https://10.0.0.1:6443`
7. Deleted node records from kine to force re-registration with new IPs

**Result:** Inter-node latency dropped from ~8ms (via router) to ~0.3ms (direct switch connection). The cluster network is now completely independent of the home network — future router changes will not affect cluster operations.

**Key Learning:** Network architecture matters as much as Kubernetes architecture. A homelab built on shared, unstable network infrastructure will inherit all the instability of that infrastructure. Separating cluster traffic onto a dedicated VLAN is a small change with disproportionate benefits: it isolates failure domains, reduces latency, and eliminates a class of cascading failures caused by external network changes.

**Resources:**
- [ Tagged vs Untagged VLAN: What's the Difference? ](https://youtu.be/_BfLSd6C-X8?si=v_vw-d5-sR5ZHqTb)
- [TP-Link TL-SG605E User Guide](https://www.tp-link.com/en/support/download/tl-sg605e/)
- [Netplan VLAN Configuration](https://netplan.readthedocs.io/en/stable/examples/#using-vlans)
- [K3s Networking Documentation](https://docs.k3s.io/installation/network-options)

## Next Objectives

1. Set up centralized cluster monitoring with Prometheus and Grafana.
