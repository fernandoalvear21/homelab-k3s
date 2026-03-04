# 🛸 My Homelab: Kubernetes with k3s and k3d
This project documents my journey building a personal Homelab from scratch, optimized to run on a laptop (Mac) with minimal resource impact.

## Homelab Purpose
This project leads me to implement and understand application management using Kubernetes as the main orchestrator and also aims to demonstrate that to have a homelab it's not necessary to wait for "the perfect infrastructure"

## 🏗️ Selected Architecture
For this lab, I have chosen the following technology stack based on efficiency:

1. k3s (Kubernetes distribution): A lightweight version of Kubernetes certified by the CNCF. It removes unnecessary components and external dependencies to reduce RAM consumption to less than 512MB per node.
2. k3d (Orchestrator): Instead of using Virtual Machines (VMs) that reserve fixed RAM, k3d runs k3s nodes as Docker containers. This allows the host operating system to share resources dynamically.
3. Multi-node Topology:
    1 Control-plane (Server): Manages cluster state.
    2 Agents (Workers): Where workloads reside (like Passbolt). This allows me to practice high availability and node affinity concepts.

## ⚙️ Cluster Configuration (k3d-config.yaml)
I have defined the cluster as code to ensure it's reproducible and easy to migrate:

| Configuration | Reason |
|---------------|--------|
| servers: 1    | Sufficient for a local development environment. |
| agents: 2     | Allows testing load balancing and node failures. |
| port: 8080:80 | Maps web traffic from my Mac to the cluster to access apps. |
| image: k3s-version | Fixes a specific version to avoid unexpected changes in automatic updates. |

## 🛠️ Prerequisites
Before starting, I installed these tools on my Mac using Homebrew:

```bash
brew install k3d kubectl helm
```
> Note: Make sure you have Docker Desktop or OrbStack running.

### 🚀 How to Set Up the Environment
Clone the repository:

```bash
git clone https://github.com/fernandoalvear21/homelab-k3s
cd homelab-k3s
```

### Create the cluster:

```bash
k3d cluster create --config cluster/k3d-config.yaml
```

### Verify status:
```bash
kubectl get nodes
```

## 📂 Repository Structure
- `/cluster`: k3d configuration files and provisioning.
- `/apps`: Kubernetes manifests (YAML) or Helm Charts for each application.
- `/apps/passbolt`: My main password manager.
- `/docs`: Detailed guides and lessons learned.

## 📓 Learning Log
Day 1: Initial setup, choosing k3d over Minikube for its lightweight nature on macOS. Defining GitOps structure.

### Synchronization Process (Manual GitOps)
Every time I modify a configuration file, I apply changes with:
```bash
kubectl apply -f apps/app-name/
```
Day 2: Created an Nginx service just to test and learn how to services and ingress work together. Created and error logbook in docks/errors.md to map errors and troubleshooting steps in this journey!

Day 3: Found out that in order to 'put down' a service I need to remove the service deployment and the ingress associated with that service. I can do it with this command i.e taking down my test nginx service. 
```bash
kubectl delete -f deployment-file.yaml
kubectl delete ingress <ingress-name> -n <namespace> 
```
Also learned about persistent volumes, in order to keep my secrets and critical data from my password manager I needed to make the cluster configuration to be persistent so I define a volume on the cluster configuration to recreate it. 

Day 4: Passbolt is fully operational. Now I'll be installing and testing an observability stack to get controll of everything that could happen in my cluster. 