# HomeLab - Evolution from Virtual to Physical

Welcome to my homelab journal, chronicling the process of building a kubernetes cluster from scratch as a total beginner.

## 📅 Phase 1: Virtual Foundations (January 2025)

This initial phase was focused on learning basic networking and kubernetes concepts. This phase I was heavily guided with AI, hence the somewhat fast progress.

**Key Technology Stack:**
*   **Hypervisor:** Virtual Machine Manager (QEMU/KVM)
*   **Guest OS:** Ubuntu Server
*   **Orchestration:** K3s (Lightweight Kubernetes)
*   **Database:** PostgreSQL
*   **Management:** PgAdmin4

---

### 🗓️ 13 January 2025
**Accomplishment:** Established the virtual network infrastructure.
*   Created a dedicated virtual network bridge to enable communication between virtual machines.
*   Provisioned two Ubuntu Server virtual machines, designated for the Kubernetes control plane and worker node roles.

**Resources Utilszed:**
*   [KVM Host Network Configurations using Virt-Manager](https://cloudspinx.com/kvm-host-network-configurations-using-virt-manager/)
*   [Virtual Machine Manager Documentation](https://virt-manager.org/screenshots.html)

### 🗓️ 15 January 2025
**Accomplishment:** Successfully deployed a functional K3s Kubernetes cluster.
*   Initialized a master node.
*   Joined a worker node to the cluster, enabling distributed workload execution.
*   Verified cluster health using `kubectl get nodes`, confirming both nodes were in a `Ready` state.

**Resources Utilised:**
*   [K3s Quick-Start Guide](https://docs.k3s.io/quick-start)
*   [Kubernetes `kubectl` Command Reference](https://kubernetes.io/docs/reference/kubectl/)

### 🗓️ 16 January 2025
**Accomplishment:** Initiated the "Whoop Project" for health data integration.
*   Registered an application with the Whoop API to obtain OAuth 2.0 client credentials.
*   Developed the initial Python script to handle authentication and establish a connection to the Whoop API, retrieving an access token.

**Resources Utilised:**
*   [Whoop API Documentation](https://developer.whoop.com/api)
*   [Python `whoopy` Library](https://github.com/felixnext/whoopy)

### 🗓️ 17 January 2025
**Accomplishment:** Built the persistent data layer and management interface for the project.
*   Deployed a PostgreSQL database as a StatefulSet within the K3s cluster to ensure data persistence.
*   Created a dedicated Python virtual environment (`venv`) to manage project dependencies cleanly.
*   Designed and executed SQL schema to create necessary tables for storing Whoop activity, sleep, and recovery data.
*   Deployed PgAdmin4 as a web-based database administration tool and connected it to the cluster's PostgreSQL instance.

**Resources Utilised:**
*   [PostgreSQL Kubernetes Deployment](https://www.digitalocean.com/community/tutorials/how-to-deploy-postgres-to-kubernetes-cluster)
*   [PgAdmin4 Official Site](https://www.pgadmin.org/)

### 🗓️ 18 January 2025
**Accomplishment:** Advanced the data pipeline towards containerization.
*   Enhanced the Python script to retrieve specific data sets (e.g., sleep cycles, workout summaries) from the Whoop API.
*   Created a `Dockerfile` to package the script, its virtual environment, and all dependencies into a portable container image.
*   Successfully built the Docker image and tested it locally.

**Resources Utilised:**
*   [Docker Get Started](https://docs.docker.com/get-started/)

---

## 🖥️ Phase 2: Bare-Metal Transition (Current)

In this phase my goal is to take what I have learned in phase 1, and apply it in a bare metal repurposed laptop. While I'll still use AI, my focus will be much more on learning all the fundamental knowledge and tools having a much richer learning environment in detriment of fast progress.

**Hardware & Stack:**
*   **Master Node:** HP ProBook 4520s
*   **Worker Node:** HP Pavilion dm1
*   **Base OS:** Ubuntu 24.04.3 LTS (Server)
*   **Orchestration:** K3s (Bare-metal installation)
*   **Ingress Controller:** Traefik (default with K3s)
*   **Service Exposure:** Kubernetes Ingress API

---

### 🗓️ [5 February 2026]
**Accomplishment:** Provisioned and configured the new bare-metal K3s cluster.
*   Performed a clean installation of Ubuntu Server 24.04.3 LTS on both the HP ProBook and HP Pavilion laptops.
*   Assigned static IP addresses (`192.168.8.10` for master, `192.168.8.11` for worker) to ensure stable network identity for the cluster nodes.
*   Installed K3s on the ProBook, designating it as the master/control plane node.
*   Joined the Pavilion laptop to the cluster as a worker node, expanding the cluster's compute capacity.

**Resources Utilised:**
*   [Turning an Old Laptop into a Home Server! (2026)](https://www.youtube.com/watch?v=46T4cDQBkDs&t=255s)
*   [K3s Installation Guide](https://docs.k3s.io/installation)

### 🗓️ [6 February 2026 - Ingress Configuration]
**Accomplishment:** Deployed NGINX and exposed it externally using Ingress.
*   Created a Kubernetes `Service` of type `ClusterIP` for the NGINX deployment.
*   Defined an `Ingress` resource with appropriate routing rules, specifying the desired hostname (`mynginx123.com`).
*   Successfully applied the configuration, allowing the Traefik ingress controller (pre-installed with K3s) to route external HTTP traffic to the internal NGINX service.
*   Resolved local network DNS by adding a host entry, enabling access to the service via the defined hostname from within the home network.

**Key Learning:** Understood the role of the Ingress controller as a traffic router versus the Service as an internal load balancer.

**Resources Utilised:**
*   [Kubernetes Ingress Tutorial for Beginners | simply explained | Kubernetes Tutorial 22](https://www.youtube.com/watch?v=80Ew_fsV4rM)
*   [Kubernetes Networking Explained](https://www.youtube.com/watch?v=J8jAzqbXxjE)

---

## 🚀 Next Objectives

1.  Migrate the Whoop Project (PostgreSQL, CronJob, PgAdmin) from the old virtual cluster to the new bare-metal cluster.
2.  Configure persistent storage for the database using a local path provisioner or Longhorn.
3.  Explore implementing TLS certificates for secure (HTTPS) access to services via Ingress.
4.  Set up centralized cluster monitoring with Prometheus and Grafana.
