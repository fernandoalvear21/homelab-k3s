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


