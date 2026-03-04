## Error mapping

I created this page to document and map the error I encounter during this Kubernetes journey and how I managed to handle them. 

---

### Kluster up but not responding after engine restart
After restarting my Engine with a new configuration for a test I was doing. I started my cluster and it seems I was up and running but the when I did a `kubectl cluster-info` I got the error: 

```bash
Fernandos-MacBook-Air:homelab-k3s fernandoalvear$ k3d cluster start faf-homelab
INFO[0000] Using the k3d-tools node to gather environment information 
INFO[0000] Starting existing tools node k3d-faf-homelab-tools... 
INFO[0000] Starting node 'k3d-faf-homelab-tools'        
INFO[0000] Starting new tools node...                   
INFO[0000] Starting node 'k3d-faf-homelab-tools'        
INFO[0001] Starting cluster 'faf-homelab'               
INFO[0001] All servers already running.                 
INFO[0001] All agents already running.                  
INFO[0001] Starting helpers...                          
INFO[0001] Starting node 'k3d-faf-homelab-tools'        
INFO[0001] Started cluster 'faf-homelab'                
Fernandos-MacBook-Air:homelab-k3s fernandoalvear$ kubectl cluster-info

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
error: Get "https://0.0.0.0:56924/api/v1/namespaces/kube-system/services?labelSelector=kubernetes.io%2Fcluster-service%3Dtrue": EOF - error from a previous attempt: read tcp 127.0.0.1:58064->127.0.0.1:56924: read: connection reset by peer
```

#### Troubleshooting 
After doing some troubleshooting I found out that the error `EOF` or `connection reset by peer` in `0.0.0.0` occurs because after restarting my engine the networks tunnels that expose the Kubernetes API broke so my cluster even tho is running is like it was `deaf`. 

#### Quick-fix
To fix this I had to merge the new context using the commad: 

```bash 
k3d kubeconfig merge faf-homelab --kubeconfig-switch-context
```
After that I restart my cluster and it was ready to go. 

---

## Service ingress error 
I was creating an nginx service just to test how they work and interact, I created the service in a custom namespace, but the ingress controller for that service was created on the default namespace mistakenly. 
So, I learned that in Kubernetes the **Namespace** isolation works like a logic barrier. An ingress on Namespace `A` can't see by default the services on Namespace `B` or any other rather than `A`. 
In other to fix the error I just changed the namespace in my ingress file config and deleted the created ingress to apply the file with the correct config. After that it worked like a charm!

---

## Persistant volumen cluster creation error 
Trying to create my cluster with a persistant volume to avoid losing my data if I delete or migrate my cluster I got the error creating the cluster:

``` bash
ERRO[0001] failed Cluster Creation: failed setup of server/agent node k3d-faf-homelab-server-0: failed to create node: runtime failed to create node 'k3d-faf-homelab-server-0': failed to create container for node 'k3d-faf-homelab-server-0': docker failed to create container 'k3d-faf-homelab-server-0': Error response from daemon: Duplicate mount point: /var/lib/rancher/k3s/storage 
ERRO[0001] Failed to create cluster >>> Rolling Back    
```
and this was beacuse I'm using env variables in my project root and the cluster configuration file is in other directory, so in order to fix this I had to use this commando to keep the .env variables in memory while creating the cluster: 

```bash
export $(grep -v '^#' .env | xargs) && k3d cluster create --config cluster/k3d-config.yaml
```

---

## Issue: ERR_TOO_MANY_REDIRECTS / SSL Handshake Failed

### Symptoms
*   **Browser:** Displays `ERR_TOO_MANY_REDIRECTS` (Infinite Loop).
*   **Browser:** Displays `404 Not Found` or `Mixed Content` errors.
*   **Pod Logs:** `SSL_do_handshake() failed (SSL: error:0A000412:SSL routines::ssl/tls alert bad certificate:SSL alert number 42)`.

---

### 1. Root Cause Analysis
This typically occurs when there is a mismatch between the **Ingress Controller (Traefik)** and the **Passbolt Pod** regarding SSL termination:
1.  **Redirection Loop:** The Ingress handles SSL (Port 443) but talks to the Pod via HTTP (Port 80). If `PASSBOLT_SSL_FORCE` is `true`, the Pod rejects the HTTP request and tells the browser to redirect to HTTPS, creating a loop.
2.  **Handshake Error:** If trying to use HTTPS between the Ingress and the Pod, the Ingress rejects the Pod's self-signed certificate (Alert 42).

---

### 2. Troubleshooting & Resolution

#### A. Environment Variables (Deployment)
To break the redirection loop while maintaining an HTTPS external URL, configure the following in the `Deployment`:


| Variable | Value | Description |
| :--- | :--- | :--- |
| `APP_FULL_BASE_URL` | `https://passbolt.local:2101` | Must match the EXACT URL and PORT used in the browser. |
| `PASSBOLT_SSL_FORCE` | `false` | Disables internal Nginx redirection (Ingress handles this). |
| `PASSBOLT_SSL_CONTENT_REWRITE` | `true` | **Crucial:** Forces Passbolt to generate HTTPS links even if internal traffic is HTTP. |

#### B. Ingress Configuration
Ensure the Ingress is set to talk to the Service via **Port 80 (HTTP)** to avoid certificate validation issues inside the cluster.

```yaml
spec:
  tls:
  - hosts:
    - passbolt.local
    secretName: passbolt-tls-secret
  rules:
  - host: passbolt.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: passbolt-service
            port:
              number: 80 # Use HTTP internally
```