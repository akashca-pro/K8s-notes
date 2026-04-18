# How the Kubernetes Scheduler Works — Deep Dive

---

## 1) What problem does the Scheduler solve?

When you create a Pod, Kubernetes needs to answer one question:

> **"Which node should this Pod run on?"**

You could assign nodes manually (`nodeName` field), but at scale — with hundreds of nodes and thousands of Pods, each with different CPU/memory needs, affinity rules, and hardware requirements — that's impossible to do by hand.

The **kube-scheduler** automates this decision. It continuously watches for Pods that have no node assigned and picks the best node for each one based on a two-phase process: **Filter** then **Score**.

> One-liner: The Scheduler is a **decision engine** that matches Pods to nodes by eliminating unsuitable nodes, then ranking the rest.

---

## 2) Where does the Scheduler fit?

```
Control Plane
├── API Server       ← stores all cluster state (etcd)
├── etcd             ← persistent key-value store
├── Controller Mgr   ← drives desired state (Deployments, ReplicaSets…)
└── Scheduler        ← assigns Pods to nodes  ← you are here

Worker Nodes
├── Kubelet          ← receives assignment, runs the container
├── Kube-proxy       ← handles Service networking
└── Container Runtime (containerd/CRI-O) ← actually runs containers
```

The Scheduler is **stateless** — it watches the API server, makes a node assignment decision, and writes the decision back. It does not directly tell Kubelet anything; Kubelet watches the API server for Pods assigned to its own node.

---

## 3) The scheduling pipeline — overview

```
New Pod created (no nodeName set)
        │
        ▼
┌───────────────────┐
│  Scheduling Queue │  ← Pods wait here, sorted by priority
└───────────────────┘
        │
        ▼
┌───────────────────┐
│   Phase 1: FILTER │  ← eliminate nodes that cannot run the Pod
└───────────────────┘
        │ (surviving nodes)
        ▼
┌───────────────────┐
│   Phase 2: SCORE  │  ← rank surviving nodes 0-100
└───────────────────┘
        │ (highest-score node wins)
        ▼
┌───────────────────┐
│      BIND         │  ← write nodeName to Pod spec in API server
└───────────────────┘
        │
        ▼
  Kubelet on that node sees the Pod → pulls image → starts container
```

---

## 4) Phase 1 — Filtering (hard constraints)

Filtering removes any node that **cannot** run the Pod. These are binary pass/fail checks. A node either passes all filters or is eliminated.

### Built-in filters (called predicates in older docs)

| Filter | What it checks |
|---|---|
| `NodeResourcesFit` | Node has enough free CPU and memory to satisfy Pod `requests` |
| `NodeName` | If Pod specifies `spec.nodeName`, only that node passes |
| `NodeSelector` | Node has the labels the Pod requires in `nodeSelector` |
| `TaintToleration` | Pod tolerates all taints on the node |
| `NodeAffinity` | Node matches Pod's `nodeAffinity` rules (required ones) |
| `PodAffinity / PodAntiAffinity` | Co-location or separation rules with other Pods |
| `VolumeBinding` | Node can satisfy the Pod's PersistentVolumeClaim requirements |
| `NodeUnschedulable` | Node is not cordoned off (`kubectl cordon`) |

If **no nodes** survive filtering, the Pod stays **Pending** and the Scheduler keeps retrying. You'll see this in:
```bash
kubectl describe pod <name>
# Events: "0/5 nodes are available: 5 Insufficient memory."
```

---

## 5) Phase 2 — Scoring (soft preferences)

From the nodes that passed filtering, the Scheduler scores each one using multiple plugins. Each plugin gives a score of 0–100. The weighted sum across all plugins determines the final score. The **highest-scoring node wins**.

### Important scoring plugins

| Plugin | What it favors |
|---|---|
| `LeastAllocated` | Nodes with the most free CPU/memory → spreads load |
| `MostAllocated` | Nodes with the least free resources → bins Pods tightly (useful to free up nodes for scale-down) |
| `BalancedResourceAllocation` | Nodes where CPU and memory usage are proportionally balanced |
| `InterPodAffinity` | Nodes that match preferred co-location rules |
| `NodeAffinity` | Nodes that match preferred (soft) affinity rules |
| `ImageLocality` | Nodes that already have the container image cached → faster startup |
| `TaintToleration` | Nodes with fewer tolerated taints score higher |
| `NodeResourcesFit` (least waste) | Minimizes wasted resources after Pod placement |

After all plugins score all nodes, scores are summed. If multiple nodes tie, one is picked at random.

---

## 6) How resource requests affect scheduling

```yaml
resources:
  requests:
    cpu: "500m"      # 0.5 CPU core
    memory: "256Mi"
  limits:
    cpu: "1"
    memory: "512Mi"
```

- The Scheduler **only looks at `requests`**, not `limits`.
- `requests` is the guaranteed minimum the Pod needs; it's the unit of scheduling currency.
- A node is considered to have enough room if:
  `allocatable_cpu - sum_of_all_pod_requests_on_node >= pod_request`
- `limits` are enforced at runtime by the Kubelet/cgroups, not by the Scheduler.

> Implication: You can over-commit limits (schedule more than the node has) but you cannot over-commit requests. If you don't set requests, the Scheduler assumes 0 — which can cause nodes to become overloaded.

---

## 7) Node Selector (simplest targeting)

Add a label requirement to a Pod so only labeled nodes are considered.

```yaml
# Label a node
kubectl label node worker-node-1 disktype=ssd

# In the Pod spec
spec:
  nodeSelector:
    disktype: ssd
```

If no node has the label, the Pod stays Pending. Node selectors are **required and binary** — no soft preference here.

---

## 8) Node Affinity (flexible targeting)

Node Affinity is the evolved version of nodeSelector with two modes:

```yaml
spec:
  affinity:
    nodeAffinity:
      # Hard rule — node MUST match (filtering phase)
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: [us-east-1a, us-east-1b]

      # Soft rule — node SHOULD match, but not required (scoring phase)
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80          # higher weight = stronger preference
          preference:
            matchExpressions:
              - key: disktype
                operator: In
                values: [ssd]
```

| Type | Phase used | Behavior if not met |
|---|---|---|
| `requiredDuring…` | Filtering | Pod stays Pending |
| `preferredDuring…` | Scoring | Pod still schedules, just on a less-preferred node |

> "IgnoredDuringExecution" means: if a node's labels change after a Pod is already running there, the Pod is **not evicted**. A future type `RequiredDuringExecution` (in alpha) will evict Pods if the node stops matching.

---

## 9) Pod Affinity and Anti-Affinity (co-location rules)

These rules are relative to **other Pods**, not nodes.

```yaml
spec:
  affinity:
    # Schedule this Pod on a node that already has a cache Pod
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: redis-cache
          topologyKey: kubernetes.io/hostname   # "same node"

    # Never schedule two replicas of my-api on the same node
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: my-api
          topologyKey: kubernetes.io/hostname
```

`topologyKey` defines what "same" means:
- `kubernetes.io/hostname` → same node
- `topology.kubernetes.io/zone` → same availability zone
- `topology.kubernetes.io/region` → same cloud region

---

## 10) Taints and Tolerations (node repulsion)

While affinity/selectors say "I want to go to this node", taints say "I don't want most Pods here". Tolerations say "I can tolerate that taint".

```bash
# Taint a node — only Pods with a matching toleration can be scheduled here
kubectl taint node gpu-node-1 hardware=gpu:NoSchedule
```

```yaml
# Pod that tolerates the taint
spec:
  tolerations:
    - key: "hardware"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"
```

### Taint effects

| Effect | Behavior |
|---|---|
| `NoSchedule` | New Pods without a toleration are not scheduled here. Existing Pods stay. |
| `PreferNoSchedule` | Scheduler tries to avoid this node but will use it if no other option. |
| `NoExecute` | New Pods without toleration rejected AND existing Pods without toleration are evicted. |

### Built-in system taints (Kubernetes sets these automatically)

| Taint | When set | Meaning |
|---|---|---|
| `node.kubernetes.io/not-ready` | Node fails health check | Node is unhealthy |
| `node.kubernetes.io/unreachable` | Kubelet not reporting | Node is unreachable |
| `node.kubernetes.io/memory-pressure` | Node running low on memory | Avoid scheduling |
| `node.kubernetes.io/disk-pressure` | Disk almost full | Avoid scheduling |
| `node-role.kubernetes.io/control-plane` | Control plane nodes | Prevents workload Pods by default |

---

## 11) Priority and Preemption

Not all Pods are equal. You can give Pods a priority so that high-priority Pods can **evict** lower-priority ones when a cluster is under resource pressure.

```yaml
# Define a PriorityClass
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000          # higher integer = higher priority
globalDefault: false
description: "Critical production workloads"
---
# Use it in a Pod/Deployment
spec:
  priorityClassName: high-priority
```

### How preemption works

```
High-priority Pod is Pending (no nodes have free resources)
        │
        ▼
Scheduler looks for a node where evicting
lower-priority Pods would free enough resources
        │
        ▼
Lower-priority Pods are gracefully terminated
        │
        ▼
High-priority Pod is scheduled on that node
```

> System-critical Pods (like CoreDNS, kube-proxy) use very high built-in PriorityClasses (`system-cluster-critical`, value: 2,000,000,001) so they are almost never preempted.

---

## 12) Topology Spread Constraints (even distribution)

Ensures Pods spread evenly across failure domains (zones, nodes, racks).

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1                            # max allowed imbalance between zones
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule      # or ScheduleAnyway
      labelSelector:
        matchLabels:
          app: my-api
```

Example with 3 zones:
- Allowed: zone-a=2, zone-b=2, zone-c=1 (max skew=1)
- Not allowed: zone-a=3, zone-b=1, zone-c=0 (skew=3)

This is more flexible than PodAntiAffinity, which just avoids co-location but doesn't guarantee even spread.

---

## 13) The scheduling pipeline internals (plugin framework)

The Scheduler is built on an **extension point plugin framework** introduced in Kubernetes 1.15 (stable since 1.19). Each scheduling decision passes through a series of extension points:

```
Pod enters queue
      │
  [QueueSort]       ← order Pods in queue (by priority by default)
      │
  [PreFilter]       ← pre-compute data for Filter plugins
      │
  [Filter]          ← eliminate nodes (hard constraints)
      │
  [PostFilter]      ← called only if Filter eliminates ALL nodes
      │               (used for preemption logic)
  [PreScore]        ← pre-compute data for Score plugins
      │
  [Score]           ← rank surviving nodes
      │
  [NormalizeScore]  ← normalize scores to 0-100
      │
  [Reserve]         ← tentatively reserve resources on chosen node
      │
  [Permit]          ← can block/delay binding (used for gang scheduling)
      │
  [PreBind]         ← e.g., provision a volume before binding
      │
  [Bind]            ← write nodeName to Pod in API server
      │
  [PostBind]        ← cleanup after successful bind
```

You can write **custom scheduler plugins** that hook into any of these extension points without forking the Scheduler binary.

---

## 14) Real-world examples

### Example A — GPU workload isolation

A machine learning team needs GPU nodes only for their training jobs.

```bash
# Taint GPU nodes so normal Pods don't land there
kubectl taint node gpu-node-1 gpu=true:NoSchedule
kubectl taint node gpu-node-2 gpu=true:NoSchedule

# Label them
kubectl label node gpu-node-1 hardware=gpu
kubectl label node gpu-node-2 hardware=gpu
```

```yaml
# Training job Pod spec
spec:
  tolerations:
    - key: "gpu"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
  nodeSelector:
    hardware: gpu
  containers:
    - name: trainer
      resources:
        limits:
          nvidia.com/gpu: 1
```

Result: normal Pods never land on GPU nodes (taint blocks them). GPU jobs go exclusively there. GPU nodes are not wasted on CPU workloads.

---

### Example B — High availability across zones

A payment service needs 3 replicas spread across 3 availability zones so that one zone failure doesn't take down all replicas.

```yaml
spec:
  replicas: 3
  template:
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: payment-service
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: payment-service
              topologyKey: kubernetes.io/hostname
```

Result: Kubernetes guarantees one replica per zone, and no two replicas share the same node. Zone us-east-1a goes down → 2/3 replicas still serving traffic.

---

### Example C — Pod stays Pending: debugging it

```bash
kubectl get pods
# NAME           READY  STATUS   RESTARTS  AGE
# web-abc-xyz    0/1    Pending  0         5m

kubectl describe pod web-abc-xyz
# Events:
#   Warning  FailedScheduling  5m  default-scheduler
#             0/8 nodes are available:
#             3 node(s) had untolerated taint {dedicated: gpu},
#             5 Insufficient cpu.
```

Diagnosis:
- 3 nodes are GPU-only (tainted, Pod has no toleration).
- 5 nodes don't have enough CPU (Pod requests too high or nodes are full).

Fix options:
- Lower the CPU `requests` on the Pod.
- Scale the cluster (add more nodes).
- Add a toleration if the GPU nodes have spare CPU and the workload is appropriate.

---

## 15) Common pitfalls

| Pitfall | What goes wrong | Fix |
|---|---|---|
| No `resources.requests` set | Scheduler assumes 0; node appears empty, gets overloaded | Always set requests |
| Overly strict affinity + anti-affinity | Pod can never satisfy all rules simultaneously → perpetual Pending | Use `preferred` rules for non-critical spread; test with `kubectl describe pod` |
| Forgetting zone labels on nodes | `topologyKey: topology.kubernetes.io/zone` matches nothing → all spread rules fail | Ensure cloud provider (or you) labels nodes with zone topology labels |
| Taint without matching toleration | Pods can't land on tainted nodes, confusing if taints are unexpected | Run `kubectl describe node <n>` to see taints; check system-applied taints |
| `whenUnsatisfiable: DoNotSchedule` in topology spread | Pod stays Pending if a perfectly good node exists but would violate maxSkew | Use `ScheduleAnyway` for non-critical spread |
| Not setting `priorityClassName` | All Pods treated equally; during resource crunch, random eviction | Assign PriorityClasses for critical vs non-critical workloads |

---

## 16) Quick mental model for revision

```
Unscheduled Pod enters Scheduling Queue
    → Filter: hard rules eliminate unfit nodes
        (resources, taints, affinity, volumes…)
    → Score: soft rules rank surviving nodes
        (least allocated, image locality, preferred affinity…)
    → Bind: nodeName written to Pod in API server
    → Kubelet on that node sees Pod, runs it
```

Key facts to remember:
- Scheduler watches API server for Pods with no `nodeName`.
- Two phases: **Filter** (eliminate) → **Score** (rank).
- `requests` = scheduling currency; `limits` = runtime enforcement.
- Taints repel; tolerations grant entry.
- Node affinity = node labels; Pod affinity = co-location with other Pods.
- Topology spread = even distribution across failure domains.
- Priority + preemption = high-priority Pods can evict lower-priority ones.
- If a Pod is Pending: `kubectl describe pod` → Events section tells you exactly why.
