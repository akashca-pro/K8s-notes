# How the Kubernetes Control Plane Works — Deep Dive

---

## 1) What problem does the Control Plane solve?

When you run Kubernetes, you are not directly telling containers what to do. You declare **what you want** — "3 replicas of this app, on this image, with these resources" — and Kubernetes makes that happen, keeps it running, and recovers from failures automatically.

The **control plane** is the brain that makes this work. Without it:
- Nobody would know which Pods to create when you apply a manifest.
- Nobody would notice when a Pod crashes and restart it.
- Nobody would know which node has free resources to place a new Pod on.
- Cluster state would have no authoritative source of truth.

The control plane is a set of components that together implement a **continuous reconciliation loop**:

```
You declare desired state (kubectl apply)
        │
        ▼
Control Plane observes actual state
        │
        ▼
Control Plane drives actual state → desired state
        │
        ▼
(loop repeats forever)
```

> One-liner: The control plane is the **cluster's nervous system** — it stores state, enforces desired configuration, assigns work, and heals failures, all without you intervening.

---

## 2) Control plane components at a glance

```
┌─────────────────────────────────────────────────────────┐
│                     CONTROL PLANE                       │
│                                                         │
│   ┌──────────────┐    ┌──────────────────────────────┐  │
│   │  kube-       │    │           etcd               │  │
│   │  apiserver   │◄──►│  (persistent key-value store)│  │
│   └──────┬───────┘    └──────────────────────────────┘  │
│          │                                              │
│    ┌─────┴──────┬──────────────────┐                   │
│    ▼            ▼                  ▼                   │
│ ┌──────────┐ ┌──────────────┐ ┌──────────────────────┐ │
│ │  kube-   │ │   kube-      │ │  cloud-controller-   │ │
│ │scheduler │ │controller-   │ │  manager (optional)  │ │
│ └──────────┘ │manager       │ └──────────────────────┘ │
│              └──────────────┘                          │
└─────────────────────────────────────────────────────────┘
             │ (all components talk only to API server)
             ▼
┌─────────────────────────────────────────────────────────┐
│                     WORKER NODES                        │
│                                                         │
│  ┌──────────┐   ┌────────────┐   ┌──────────────────┐  │
│  │ kubelet  │   │ kube-proxy │   │ Container Runtime│  │
│  └──────────┘   └────────────┘   └──────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**Key design principle**: Every component communicates **only with the API server**, never directly with each other. The API server is the single hub. This keeps the system decoupled and auditable.

---

## 3) kube-apiserver — the front door

### What it does

The API server is the **only component that reads from and writes to etcd**. Everything else (controllers, scheduler, kubelet, kubectl) talks to the API server over HTTPS.

```
kubectl apply -f deployment.yaml
        │
        ▼
API Server
  ├── Authentication  (who are you?)
  ├── Authorization   (are you allowed to do this?)
  ├── Admission       (mutate/validate the object)
  └── Persist to etcd (store the object)
        │
        ▼
Watch notifications sent to controllers/scheduler/kubelet
```

### Request lifecycle through the API server

**1. Authentication** — verifies the identity of the caller.

| Method | How it works |
|---|---|
| Client certificates (x509) | Certificate signed by the cluster CA, common name becomes the username |
| Bearer tokens | ServiceAccount tokens (JWTs signed by API server) or OIDC tokens |
| Webhook auth | API server calls an external service to verify the token |
| Bootstrap tokens | Short-lived tokens for joining new nodes |

If none of the authenticators can identify the caller, the request is rejected as `401 Unauthorized`.

**2. Authorization** — decides if the authenticated identity can perform the action.

The default mode is **RBAC (Role-Based Access Control)**:

```
Principal (User / Group / ServiceAccount)
    └── bound to → RoleBinding / ClusterRoleBinding
                        └── references → Role / ClusterRole
                                             └── contains → rules:
                                                   - apiGroups: [apps]
                                                   - resources: [deployments]
                                                   - verbs: [get, list, create, update]
```

RBAC is **deny by default** — if no rule explicitly allows an action, it is denied.

**3. Admission control** — a chain of plugins that can mutate or validate objects before they are persisted.

| Type | What it does | Example |
|---|---|---|
| Mutating Admission Webhooks | Modify the incoming object | Inject a sidecar container into every Pod automatically |
| Validating Admission Webhooks | Reject objects that don't meet policy | Reject Pods without resource limits |
| Built-in plugins | Enforced by API server binary | `LimitRanger` (default limits), `ResourceQuota` (namespace quotas), `NamespaceLifecycle` |

Admission webhooks are how tools like **Istio** (injects the Envoy sidecar) and **OPA/Gatekeeper** (policy enforcement) work.

**4. Persist to etcd** — if all stages pass, the object is serialized and written to etcd.

### What happens after persistence — the Watch mechanism

```
etcd write
    │
    ▼
API server broadcasts change to all watchers
    ├── kube-scheduler   (new Pod with no nodeName → schedule it)
    ├── kube-controller-manager  (ReplicaSet count changed → reconcile)
    └── kubelet on each node     (Pod assigned to my node → run it)
```

This **watch/notify** model is how Kubernetes avoids polling. Controllers open a long-lived HTTP watch stream to the API server and receive events (Added, Modified, Deleted) as they happen.

### API server is horizontally scalable

Multiple API server replicas can run simultaneously. They are **stateless** (all state is in etcd). A load balancer sits in front of them. This is how production clusters achieve high availability for the control plane.

---

## 4) etcd — the source of truth

### What it is

`etcd` is a distributed key-value store that Kubernetes uses as its **only persistent store**. Every object in your cluster — Pods, Deployments, Secrets, ConfigMaps, Nodes, ServiceAccounts — is stored here as a serialized protobuf value.

```
/registry/pods/default/my-pod         → Pod object
/registry/deployments/prod/my-api     → Deployment object
/registry/secrets/default/my-secret   → Secret object
/registry/nodes/worker-node-1         → Node object
```

### etcd consensus — Raft

etcd uses the **Raft consensus algorithm** to replicate data across multiple etcd members. This means:
- Writes go to the **leader**, which replicates to followers before acknowledging.
- A write is committed only when a **majority (quorum)** of members acknowledge it.
- If the leader fails, a new leader is elected among the remaining members.

```
Quorum formula: majority = (N/2) + 1

Members   Quorum   Failures tolerated
   1         1            0
   3         2            1       ← minimum for HA
   5         3            2       ← most common production setup
   7         4            3
```

> You should always run an **odd number** of etcd members. Adding a 4th member actually reduces fault tolerance compared to 3 (both require quorum of 3, but 4 has more machines to fail).

### etcd and the API server

- Only the API server talks to etcd directly. No other component does.
- The API server uses **optimistic concurrency** with a `resourceVersion` field on every object. If two controllers try to update the same object simultaneously, the one with the stale `resourceVersion` gets a `409 Conflict` error and must retry.

### Why etcd health is critical

If etcd loses quorum:
- The API server stops accepting writes (the cluster becomes read-only).
- No new Pods can be scheduled.
- Running Pods continue running (kubelet is autonomous once it has the spec), but no new deployments or changes can happen.

> **etcd backups are mandatory** in production. `etcdctl snapshot save` creates a point-in-time snapshot that can restore the entire cluster state.

```bash
# Take a snapshot
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify snapshot
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-20260504.db
```

---

## 5) kube-controller-manager — the reconciliation engine

### What it is

The controller manager is a single binary that runs **dozens of controllers** as goroutines. Each controller watches a specific set of objects and reconciles actual state toward desired state.

```
kube-controller-manager
├── DeploymentController
├── ReplicaSetController
├── NodeController
├── JobController
├── CronJobController
├── ServiceAccountController
├── EndpointSliceController
├── NamespaceController
├── PersistentVolumeController
├── StatefulSetController
├── DaemonSetController
└── ... (dozens more)
```

### The reconciliation loop pattern

Every controller follows the same pattern:

```
┌─────────────────────────────────────────────┐
│              Reconciliation Loop             │
│                                             │
│  1. Watch API server for changes to         │
│     relevant objects (via informer/cache)   │
│              │                              │
│              ▼                              │
│  2. Compare desired state (spec)            │
│     vs actual state (status)                │
│              │                              │
│              ▼                              │
│  3. Take action to close the gap            │
│     (create/update/delete objects)          │
│              │                              │
│              ▼                              │
│  4. Update status field with result         │
│                                             │
│  (repeat on every relevant event)           │
└─────────────────────────────────────────────┘
```

### Key controllers explained

**ReplicaSet Controller**

Watches ReplicaSets and their Pods. If `status.replicas < spec.replicas`, it creates new Pods. If too many Pods match the selector, it deletes the excess.

```
ReplicaSet spec.replicas = 3
Actual running Pods      = 2  (one crashed)
        │
        ▼
ReplicaSet Controller detects gap
        │
        ▼
Creates 1 new Pod → writes to API server
        │
        ▼
Scheduler picks it up → assigns to a node
        │
        ▼
Kubelet starts the container
```

**Deployment Controller**

Watches Deployments. When the Pod template changes (e.g., new image), it:
1. Creates a new ReplicaSet with the updated template.
2. Scales up the new ReplicaSet while scaling down the old one (rolling update).
3. Keeps old ReplicaSets around for rollback (default: last 10).

**Node Controller**

Monitors node health. If a node stops reporting (kubelet stops heartbeating), the Node Controller:
1. Marks the node as `NotReady` after 40 seconds.
2. After 5 minutes of `NotReady`, evicts all Pods from that node (adds `NoExecute` taint).
3. The evicted Pods are rescheduled elsewhere by their ReplicaSet/Deployment controllers.

```bash
# Node heartbeat timeout defaults
node-monitor-period:          5s   # how often Node Controller checks
node-monitor-grace-period:   40s   # time before marking NotReady
pod-eviction-timeout:         5m   # time before evicting pods from NotReady node
```

**EndpointSlice Controller**

Keeps Service endpoints up to date. When Pods are added/removed/marked not ready, this controller updates the EndpointSlice objects. kube-proxy watches EndpointSlices to program the node's network rules.

### Leader election

Only **one instance** of each controller should run at a time, even in an HA control plane (otherwise two controllers might fight over the same objects). The controller manager uses **leader election** via a Lease object in the API server:

```
3 controller-manager replicas running
        │
        ▼
All three compete for a Lease lock in API server
        │
        ▼
One wins → becomes Leader → runs all controllers
Other two → Standby → watch the Lease, ready to take over
        │
        ▼
Leader fails → Lease expires → Standby wins new election
```

---

## 6) kube-scheduler

The scheduler is covered in depth in `kubernetes-scheduler-deep-dive.md`. In the context of the control plane:

- It watches the API server for Pods that have `spec.nodeName` unset (unscheduled Pods).
- It runs Filter → Score → Bind to assign a node.
- Like the controller manager, it uses **leader election** so only one scheduler instance makes decisions at a time.
- It is **stateless** — all state is in the API server/etcd.

```
Control Plane
├── API Server    ← state store hub
├── etcd          ← persistent storage
├── Controller Mgr ← drives desired state
└── Scheduler      ← assigns Pods to nodes
```

---

## 7) cloud-controller-manager — cloud integration

On cloud providers (AWS, GCP, Azure), a fourth control plane component runs: the **cloud-controller-manager (CCM)**. It was split out from `kube-controller-manager` to decouple cloud-specific logic from the core Kubernetes binary.

### What it manages

| Controller | What it does |
|---|---|
| Node controller | Annotates nodes with cloud instance metadata (instance ID, zone, region), deletes Node objects when cloud instances are terminated |
| Route controller | Configures network routes in the cloud VPC so Pod IPs are routable between nodes |
| Service controller | Provisions cloud load balancers (AWS ELB, GCP GLBL) when a `Service type=LoadBalancer` is created |

```yaml
# When you create this:
apiVersion: v1
kind: Service
spec:
  type: LoadBalancer
  selector:
    app: my-api
  ports:
    - port: 80
```

The cloud-controller-manager calls the cloud provider API to provision an actual load balancer, gets the IP/hostname, and writes it back to `service.status.loadBalancer.ingress`.

---

## 8) How all components work together — full request trace

Here is what happens when you run `kubectl apply -f deployment.yaml` with 3 replicas:

```
Step 1: kubectl apply -f deployment.yaml
│
│  kubectl serializes the YAML, sends HTTP POST to API server
│
▼
Step 2: API Server — Authentication
│  Verifies your kubeconfig certificate against cluster CA
│
▼
Step 3: API Server — Authorization (RBAC)
│  Checks if your user/role can create Deployments in this namespace
│
▼
Step 4: API Server — Admission
│  Mutating webhooks may inject labels/sidecars
│  LimitRanger may add default resource limits
│  Validating webhooks may enforce policy rules
│
▼
Step 5: API Server — Persist to etcd
│  Deployment object written to etcd
│  API server sends watch notification: "Deployment created"
│
▼
Step 6: Deployment Controller (in kube-controller-manager)
│  Receives watch event: new Deployment detected
│  Creates a ReplicaSet (spec.replicas=3, pod template from Deployment)
│  API server persists ReplicaSet, sends watch notification
│
▼
Step 7: ReplicaSet Controller (in kube-controller-manager)
│  Receives watch event: new ReplicaSet with 0 Pods
│  Creates 3 Pod objects (no nodeName set yet)
│  API server persists Pods, sends watch notification
│
▼
Step 8: kube-scheduler
│  Receives watch event: 3 Pods with no nodeName
│  For each Pod: Filter nodes → Score nodes → Bind (write nodeName)
│
▼
Step 9: kubelet on the assigned node
│  Watch: Pod with nodeName=this-node appears
│  Pulls container image from registry
│  Creates and starts the container via container runtime
│  Updates Pod status (Running, ContainerStatuses, etc.)
│
▼
Step 10: EndpointSlice Controller
   If a Service selects these Pods, updates EndpointSlice
   kube-proxy on each node updates iptables/ipvs rules
   Traffic can now reach the Pods through the Service
```

The entire sequence — from `kubectl apply` to containers running — typically takes **2–10 seconds** in a healthy cluster.

---

## 9) High availability control plane

In production, the control plane itself must be resilient. A single-node control plane is a single point of failure.

### HA topology: stacked etcd

All control plane components (including etcd) run on the same control plane nodes.

```
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  Control Plane 1│  │  Control Plane 2│  │  Control Plane 3│
│                 │  │                 │  │                 │
│  API Server     │  │  API Server     │  │  API Server     │
│  Ctrl Manager   │  │  Ctrl Manager   │  │  Ctrl Manager   │
│  Scheduler      │  │  Scheduler      │  │  Scheduler      │
│  etcd (member)  │  │  etcd (member)  │  │  etcd (member)  │
└─────────────────┘  └─────────────────┘  └─────────────────┘
         │                    │                    │
         └─────────── Load Balancer ───────────────┘
                  (kubectl and kubelets connect here)
```

- **etcd**: 3 members, quorum of 2. Can tolerate 1 node failure.
- **API servers**: 3 active simultaneously (stateless, all behind LB).
- **Controller Manager / Scheduler**: Leader election — one active, two standby.

### HA topology: external etcd

etcd runs on dedicated nodes separate from the API server nodes. More expensive (more machines) but better isolation — an etcd overload doesn't affect API server CPU, and vice versa.

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ API Server 1 │  │ API Server 2 │  │ API Server 3 │
│ Ctrl Mgr     │  │ Ctrl Mgr     │  │ Ctrl Mgr     │
│ Scheduler    │  │ Scheduler    │  │ Scheduler    │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                  │
       └─────────────────┼──────────────────┘
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
   ┌────────────┐ ┌────────────┐ ┌────────────┐
   │  etcd 1    │ │  etcd 2    │ │  etcd 3    │
   └────────────┘ └────────────┘ └────────────┘
```

---

## 10) Control plane component health — useful commands

```bash
# Check control plane Pod status (kubeadm clusters run them as static Pods)
kubectl get pods -n kube-system

# NAME                                    READY   STATUS    RESTARTS   AGE
# etcd-control-plane                      1/1     Running   0          10d
# kube-apiserver-control-plane            1/1     Running   0          10d
# kube-controller-manager-control-plane   1/1     Running   0          10d
# kube-scheduler-control-plane            1/1     Running   0          10d

# Check component status (deprecated in newer versions but still readable)
kubectl get componentstatuses

# Check API server health endpoint
curl -k https://<apiserver>:6443/healthz
curl -k https://<apiserver>:6443/readyz

# Check etcd health
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

# Check leader election (who is the active scheduler?)
kubectl get lease -n kube-system
# NAME                      HOLDER                             AGE
# kube-scheduler            control-plane-1_abc123             10d
# kube-controller-manager   control-plane-1_abc123             10d
```

---

## 11) Static Pods — how control plane components run on kubeadm clusters

In clusters bootstrapped with `kubeadm`, the control plane components themselves run as **static Pods**. Static Pods are managed directly by the kubelet on the control plane node — not by the API server. This is how Kubernetes avoids a chicken-and-egg problem (the API server can't schedule itself).

```
/etc/kubernetes/manifests/
├── etcd.yaml                    ← kubelet starts etcd
├── kube-apiserver.yaml          ← kubelet starts API server
├── kube-controller-manager.yaml ← kubelet starts controller manager
└── kube-scheduler.yaml          ← kubelet starts scheduler
```

- kubelet watches this directory and creates/updates/deletes Pods when files change.
- If you edit these YAML files, kubelet automatically restarts the component.
- Static Pod manifests are the **authoritative** way to configure control plane components in kubeadm clusters (not `kubectl edit`).

```bash
# Upgrade the API server (example: change a flag)
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
# kubelet will automatically restart the API server within seconds
```

---

## 12) Real-world examples

### Example A — Cluster is read-only: etcd quorum lost

**Symptom**: `kubectl apply` returns `etcdserver: request timed out`. `kubectl get` still works.

```bash
kubectl apply -f deployment.yaml
# Error from server: etcdserver: request timed out

kubectl get pods   # this works — API server serves reads from cache
```

**Diagnosis**: etcd lost quorum (e.g., 2 out of 3 members are down).

```bash
ETCDCTL_API=3 etcdctl endpoint health --endpoints=...
# https://10.0.0.1:2379 is healthy
# https://10.0.0.2:2379 failed: dial tcp: connect: connection refused
# https://10.0.0.3:2379 failed: dial tcp: connect: connection refused
```

**Fix**: Restore the failed etcd members from snapshot or bring them back online. Until quorum is restored, the cluster is **frozen** (no writes possible).

---

### Example B — Pods stuck in Pending: controller manager crashed

**Symptom**: You apply a Deployment, but no Pods are ever created. The Deployment exists, but `kubectl get pods` returns nothing.

```bash
kubectl get deployment my-api
# NAME     READY   UP-TO-DATE   AVAILABLE
# my-api   0/3     0            0

kubectl get pods
# (no output)

kubectl get replicasets
# (no output)  ← ReplicaSet was never created
```

**Diagnosis**: The Deployment Controller (inside controller manager) is not running.

```bash
kubectl get pods -n kube-system | grep controller-manager
# kube-controller-manager-control-plane   0/1   CrashLoopBackOff   5   10m

kubectl logs -n kube-system kube-controller-manager-control-plane
# ... check the logs for the root cause
```

**Fix**: Investigate the controller manager logs. On kubeadm clusters, check `/etc/kubernetes/manifests/kube-controller-manager.yaml` for misconfiguration.

---

### Example C — Checking who has access (RBAC audit)

```bash
# Can the "ci-bot" service account create Deployments in prod namespace?
kubectl auth can-i create deployments \
  --as=system:serviceaccount:prod:ci-bot \
  --namespace=prod
# no

# List all permissions for a service account
kubectl auth can-i --list \
  --as=system:serviceaccount:prod:ci-bot \
  --namespace=prod
```

---

### Example D — Watching the reconciliation loop in action

```bash
# Start a watch on pods in one terminal
kubectl get pods -w

# In another terminal, delete a pod
kubectl delete pod my-api-abc123

# In the watch terminal you see:
# my-api-abc123   Running → Terminating → (gone)
# my-api-xyz789   Pending → ContainerCreating → Running
# ↑ ReplicaSet Controller saw the gap and created a replacement within seconds
```

---

## 13) Common pitfalls

| Pitfall | What goes wrong | Fix |
|---|---|---|
| No etcd backup | etcd data lost in disaster → entire cluster state gone | Set up automated `etcdctl snapshot save` on a schedule (daily minimum) |
| Even number of etcd members | Adding a 4th member doesn't improve HA — quorum is still 3, but you have more machines to fail | Use odd numbers: 1, 3, or 5 etcd members |
| RBAC too permissive (`cluster-admin` for everything) | Service accounts or users have more access than they need; blast radius of a compromise is large | Apply least-privilege: grant only the specific verbs/resources each identity needs |
| Ignoring `resourceVersion` conflicts | Controllers retrying on stale objects can cause loops or missed updates | Trust the client-go retry mechanism; don't suppress 409 errors |
| Modifying static Pod manifests without verifying | A bad change to `/etc/kubernetes/manifests/kube-apiserver.yaml` crashes the API server | Test YAML syntax first; keep a backup of working manifests |
| Not monitoring etcd disk I/O | etcd is write-intensive; slow disks cause election timeouts and latency spikes | Use SSDs for etcd; monitor `etcd_disk_wal_fsync_duration_seconds` |
| Forgetting control plane node taints | Workload Pods accidentally land on control plane nodes, competing for resources | Control plane nodes have `node-role.kubernetes.io/control-plane:NoSchedule` by default; verify it is present |
| Admission webhook timeout brings down the cluster | A mutating/validating webhook is unavailable; `failurePolicy: Fail` causes all object writes to fail | Set `failurePolicy: Ignore` for non-critical webhooks; ensure webhooks have low latency and high availability |

---

## 14) Quick mental model for revision

```
Control Plane = 4 components, all talk through API server

  kube-apiserver
    → Front door: authenticate → authorize → admit → persist to etcd
    → Watch notifications to all other components
    → Horizontally scalable (stateless)

  etcd
    → Only persistent store; holds all cluster state
    → Raft consensus: needs quorum (majority) of members for writes
    → Back up regularly; losing etcd = losing your cluster

  kube-controller-manager
    → Dozens of reconciliation loops in one binary
    → Each loop: watch → compare desired vs actual → act to close gap
    → Leader election: one active replica at a time

  kube-scheduler
    → Watches for Pods with no nodeName
    → Filter (hard rules) → Score (soft preferences) → Bind (write nodeName)
    → Leader election: one active replica at a time
```

Key facts to remember:
- **API server is the hub** — no component talks directly to another, or directly to etcd.
- **etcd quorum** — odd number of members; losing majority = cluster frozen.
- **Controllers are loops** — they never stop; they always drive toward desired state.
- **Scheduler is stateless** — it just moves nodeName fields; Kubelet does the actual work.
- **Static Pods** (kubeadm) — control plane components are run by kubelet from `/etc/kubernetes/manifests/`, not by the API server.
- **Leader election** — controller manager and scheduler elect a leader among replicas; API servers run fully active in parallel.
- **Admission webhooks** — the hook point for policy engines (OPA, Kyverno) and service meshes (Istio sidecar injection).
