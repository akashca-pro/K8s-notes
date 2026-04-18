# Kubernetes Quick Revision Notes

## 1) Kubernetes in one line
Kubernetes is a **container orchestration platform** that gives you a **unified API** to run and manage workloads across many nodes (VMs or bare metal), instead of manually operating each machine.

---

## 2) Core concepts (quick recall)

### Pod
- Smallest deployable unit in Kubernetes.
- Usually contains one main container, optionally sidecars (logging, proxy, metrics, etc.).
- Containers in a Pod share:
  - Network namespace (same Pod IP/ports)
  - Volumes/storage
  - Lifecycle
- A Pod is always scheduled on **one node only** (never split across nodes).

### Service
- Stable networking abstraction for Pods.
- Gives stable virtual IP + DNS name.
- Selects Pods using labels/selectors and load-balances traffic.
- Common types:
  - `ClusterIP`: internal-only access (default)
  - `NodePort`: exposes on each node IP:port
  - `LoadBalancer`: provisions external LB (typically cloud)

### Deployment
- Declarative controller for **stateless** applications.
- Manages ReplicaSets, rollout strategy, and desired replicas.
- Supports rolling updates and rollbacks.

### ReplicaSet
- Ensures a desired number of identical Pod replicas are running.
- Usually managed by Deployments (rarely used directly).

---

## 3) Control plane and node components

### API Server (`kube-apiserver`)
- Front door of Kubernetes control plane.
- `kubectl` and controllers talk to it over HTTPS.
- Stores cluster state in `etcd` (via control plane).

### Scheduler (`kube-scheduler`)
- Watches for Pods without a node assignment.
- Picks a node based on resources, constraints, affinities, taints/tolerations, and policies.

### Controller Manager (`kube-controller-manager`)
- Runs reconciliation loops (Deployment, ReplicaSet, Node, Job, etc.).
- Continuously drives actual state toward desired state.

### Kubelet (on each node)
- Node agent that talks to API server.
- Ensures containers for scheduled Pods are running.

### Kube-proxy (on each node)
- Implements Service traffic routing/load balancing (iptables/ipvs/eBPF stack depending on setup).

---

## 4) Typical workflow (mental model)
1. You `kubectl apply -f deployment.yaml`.
2. API server validates + stores desired state.
3. Deployment controller creates/updates ReplicaSet.
4. Scheduler assigns pending Pods to nodes.
5. Kubelet pulls images and starts containers.
6. Service selects matching Pods and provides stable access.

---

## 5) Networking and external access

### Internal traffic
- Pod to Pod: via cluster networking (CNI).
- Service to Pod: via Service VIP + kube-proxy implementation.

### External traffic (common paths)
- Internet -> Cloud Load Balancer -> `Service(type=LoadBalancer)` -> Pods
- Internet -> Ingress Controller -> Service -> Pods (HTTP/HTTPS routing, host/path rules, TLS)

---

## 6) Updates and scaling

### Rolling updates (Deployment)
- Gradual replacement of old Pods with new Pods.
- Controlled by strategy settings:
  - `maxUnavailable`
  - `maxSurge`

### Horizontal Pod Autoscaler (HPA)
- Automatically scales replica count based on metrics (CPU/memory/custom/external).

---

## 7) Other important building blocks

| Concept | Purpose | Example |
|---|---|---|
| ConfigMap | Non-sensitive configuration | App settings, feature flags |
| Secret | Sensitive data | API keys, DB passwords, TLS cert refs |
| PersistentVolume (PV) / PersistentVolumeClaim (PVC) | Durable storage | Databases like MySQL/PostgreSQL |
| Namespace | Logical isolation inside a cluster | `dev`, `staging`, `prod` separation |

---

## 8) Corrections and clarifications from original notes
- "Kubernetes is abstraction over VMs" -> More accurate: Kubernetes orchestrates **containers across nodes**; nodes can be VMs or physical machines.
- Pods are **not** spread across nodes; each Pod runs on a single node.
- Services do not "discover all Pods automatically" without labels; they route only to Pods matching their selector.
- Deployments are for stateless workloads; stateful apps usually use StatefulSets + persistent storage.
- `kubectl` sends REST calls to API server; manifests are typically YAML (or JSON).

---

## 9) Fast revision checklist (30-second recall)
- Pod = smallest unit, one node, shared net/storage.
- Deployment manages stateless app lifecycle.
- ReplicaSet keeps N Pods alive.
- Service gives stable networking to dynamic Pods.
- Scheduler chooses node; Kubelet runs workload.
- ConfigMap/Secret for config; PV/PVC for storage; Namespace for isolation; HPA for autoscaling.

---

## 10) Handy practice command
```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml
```
Use this to generate and inspect a basic Pod manifest quickly.
