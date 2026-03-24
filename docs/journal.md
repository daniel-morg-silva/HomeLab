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

---

**Problem: Secret reference wrong type**
The deployment was pulling `LD_SUPERUSER_PASSWORD` via `configMapKeyRef` — but the source was a `Secret`, not a `ConfigMap`.

**Fix:** Changed to `secretKeyRef`. ConfigMaps and Secrets use different ref types even though the syntax looks identical.

---

**Problem: Secret in wrong namespace**
`secrets.yaml` had `namespace: whoop` left over from copying a template. The deployment was in the `linkding` namespace.

**Root cause:** Kubernetes Secrets are namespace-scoped. A pod in `linkding` cannot mount a secret from `whoop`.

**Fix:** Changed `namespace: whoop` → `namespace: linkding`.

---

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

---

**Problem: Master Ingress rejected after adding annotation**
After adding the `master` annotation, the Ingress was still rejected.

**Root cause:** A `master` Ingress cannot define any `http.paths`. The master only defines the host. All paths must live in minion Ingresses.

**Fix:** Removed the `http.paths` block from the master Ingress entirely.

---

**Problem: 404 on `/linkding` — double slash in paths**
After fixing the ingress, requests reached the linkding pod but returned 404. Logs showed:
```
[uwsgi-static] added mapping for //linkdingstatic => static
WARNING Not Found: /linkding
```

**Root cause:** `LD_CONTEXT_PATH` was set to `/linkding` (with a leading slash). Linkding prepends its own `/` internally, producing `//linkding`.

**Fix:** Changed `LD_CONTEXT_PATH` value from `/linkding` to `linkding` (no leading slash).

---

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

---

## Next Objectives

1. Set up centralized cluster monitoring with Prometheus and Grafana.
2. Implement TLS certificates for HTTPS access via cert-manager.
