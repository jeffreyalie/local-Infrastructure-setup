
##                                          Kubernetes - Microk8s structure diagram
---

```
                                            Developer / GitHub Actions Runner
                                            +-----------------------------------+
                                            | Source Code + Helm Chart Folder   |
                                            | Chart.yaml, values.yaml, templates|
                                            | README.md                         |
                                            +-----------------------------------+
                                                 |                     |                                                                                      
                                                 |                     |                    
                                                 |                     | 
                                                 |                     |                                  
                                        helm install/upgrade       Docker build push -------------------+                                       
                                                 │                                                      |                      
                                                 |                                              +-----------------------+
                                                 |                                              |Local registry / Harbor|
                                                 |                                              +-----------------------+
                                                 ▼                                                      |         
                            +------------------------------------------------------------+              |
                            |  Kubernetes Controllers (inside MicroK8s) creates pod       |             |   
                            | - Deployment Controller ensures desired pods               |              |
                            |  - Service Controller manages networking                   |              |  
                            |  - Ingress Controller manages routing                      |              |  
                            |  - Reconciliation loop keeps actual state = desired state  |              |   
                            |- Kubelet on the node is the one that pulls the image       |              |
                            |  (first time or when missing) and starts                   |              |
                            +------------------------------------------------------------+              |
                                                        |                                               |                         
                                                        |                                               |  
                                            +----------------------+                                    |   
                                            | Helm Metadata        |                                    |
                                            | (Secrets/ConfigMaps) |                                    |
                                            | - Release name       |                                    |   
                                            | - Chart version      |                                    | 
                                            | - Values used        |                                    |
                                            | - History snapshot   |                                    |
                                            +----------------------+                                    |
                                                        │                                               |
                                                        ▼                                               |
                                            +----------------------+                                    |
                                            | Kubernetes Objects   |                                    |
                                            | (stored in etcd)     |                                    |    
                                            | - Deployment (spec   |                                    |    
                                            |   references registry|                                    |    
                                            |   image)             |                                    |    
                                            | - Service            |                                    |        
                                            | - Ingress            |                                    |           
                                            | - ConfigMaps/Secrets |                                    |                
                                            +----------------------+                                    |                    
                                                        │                                               |                
                                                        ▼                                               |            
                                            +--------------------------+                                |
                                            | Pods (runtime)           |                                |    
                                            |--------------------------|                                |
                                            |- Kubelet Pull images from|                                |
                                            |   MicroK8s registry      |   <----Kubelet on each node----+
                                            | - Ephemeral, auto-       |        pulls images and starts 
                                            |   recreated by K8s       |        containers
                                            +--------------------------+

```
---

## 🧩 What Happens Step by Step

## 1. Helm Client Action

* Helm reads your chart (Chart.yaml, values.yaml, templates/*.yaml).
* It renders the manifests into plain Kubernetes YAML.
* It sends those manifests to the Kubernetes API server inside MicroK8s.

## 2. Kubernetes API Server

* Accepts the manifests and stores them in **etcd** (the cluster database).
* At this point, Helm’s job is done — Kubernetes now owns the lifecycle.

## 3. Kubernetes Controllers

* **Deployment Controller:** Ensures the desired number of pods are running, recreates them if deleted.
* **ReplicaSet Controller:** Tracks pod replicas.
* **Service Controller:** Manages networking endpoints.
* **Ingress Controller:** Manages external routing.

👉 These controllers continuously reconcile the “desired state” (from your manifests) with the “actual state” of the cluster.

## 4. Pods

* Created by the Deployment spec.
* Each pod pulls its container image from the MicroK8s registry (if not cached locally).
* Pods are ephemeral — if they die, the Deployment controller recreates them automatically.

---

## 🔗 Where Registry Fits

### Kubelet (on each node)
The kubelet is the agent running on every node.
* When the scheduler assigns a pod to a node, the kubelet is responsible for creating the container runtime process.
* The kubelet checks if the image is already cached locally.
    * If cached → reuse.
    * If not cached → kubelet instructs the container runtime (containerd in MicroK8s) to pull the image from the registry.