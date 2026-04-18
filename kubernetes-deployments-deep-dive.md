# How Kubernetes Deployments Work — Deep Dive

---

## 1) What problem does a Deployment solve?

Without Deployments you would have to:
- Manually create every Pod.
- Manually restart a Pod if it crashes.
- Manually create a second version of Pods when you ship new code.
- Manually delete old Pods once the new ones are up.

A **Deployment** automates all of this. You declare *what you want* (which image, how many replicas, update strategy), and Kubernetes continuously makes reality match that declaration.

> One-liner: A Deployment is a **declarative, self-healing, rolling-update manager** for stateless apps.

---

## 2) Object hierarchy (who owns whom)

```
Deployment
  └── ReplicaSet  (one per version of your app)
        └── Pod
        └── Pod
        └── Pod
```

- A **Deployment** creates and manages **ReplicaSets**.
- A **ReplicaSet** creates and manages **Pods**.
- When you change the Pod template (e.g., new image), the Deployment creates a **new** ReplicaSet and scales it up while scaling the old one down.
- Old ReplicaSets are kept around (by default the last 10) so you can roll back.

---

## 3) Anatomy of a Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-api            # name of the Deployment object
  namespace: production
spec:
  replicas: 3             # desired number of Pods
  selector:
    matchLabels:
      app: my-api         # the Deployment owns Pods with this label
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1   # at most 1 Pod can be down during update
      maxSurge: 1         # at most 1 extra Pod can exist during update
  template:               # Pod blueprint starts here
    metadata:
      labels:
        app: my-api       # must match selector.matchLabels above
    spec:
      containers:
        - name: api
          image: my-org/my-api:v1.2.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```

### Key YAML fields explained

| Field | What it controls |
|---|---|
| `replicas` | How many identical Pods to keep running |
| `selector.matchLabels` | How the Deployment finds/owns its Pods |
| `template` | The Pod definition; any change here triggers a rollout |
| `strategy.type` | `RollingUpdate` (default) or `Recreate` |
| `maxUnavailable` | Max Pods that can be down at once during update |
| `maxSurge` | Max extra Pods above desired count during update |
| `resources.requests` | Minimum CPU/mem Scheduler needs to find a node |
| `resources.limits` | Hard cap; container is killed if it exceeds memory limit |

---

## 4) Internal mechanics — step by step

### 4a) Creating a Deployment for the first time

```
You: kubectl apply -f deployment.yaml
         │
         ▼
    API Server (stores Deployment object in etcd)
         │
         ▼
  Deployment Controller  ← part of kube-controller-manager
  (watches API server for Deployments)
         │
         ▼
  Creates ReplicaSet rs-abc123 (owns: replicas=3)
         │
         ▼
  ReplicaSet Controller
  (watches for ReplicaSets)
         │
         ▼
  Creates 3 Pod objects (status: Pending)
         │
         ▼
  Scheduler watches for Pending Pods
  → assigns each Pod to a node
         │
         ▼
  Kubelet on each node
  → pulls the container image
  → starts the container
  → reports Pod status = Running
```

The whole chain is **event-driven** — each controller watches the API server for changes to objects it cares about and reacts.

---

### 4b) The reconciliation loop (heart of Kubernetes)

Every controller continuously runs this logic:

```
loop forever:
    desired = read desired state from API server
    actual  = observe current state (running Pods)
    if actual != desired:
        take action to close the gap
```

Example: You set `replicas: 3`. One Pod crashes. The ReplicaSet controller sees `actual=2, desired=3` → creates a new Pod. You did nothing. It just fixed itself.

---

### 4c) Rolling update — what actually happens internally

Scenario: you update your image from `v1.2.0` to `v1.3.0`.

```
Before update:
  ReplicaSet rs-v1 → [Pod1(v1), Pod2(v1), Pod3(v1)]

Step 1: Deployment controller creates a new ReplicaSet rs-v2 with replicas=0

Step 2: Scale rs-v2 up by 1 (maxSurge=1 allows a 4th Pod temporarily)
  rs-v1 → [Pod1(v1), Pod2(v1), Pod3(v1)]
  rs-v2 → [Pod4(v2)]  ← new Pod starting

Step 3: Wait for Pod4(v2) to pass readinessProbe

Step 4: Scale rs-v1 down by 1 (maxUnavailable=1 allows 1 old Pod to leave)
  rs-v1 → [Pod1(v1), Pod2(v1)]
  rs-v2 → [Pod4(v2)]

Step 5: Repeat until rs-v2 has 3 running Pods, rs-v1 has 0
  rs-v1 → []
  rs-v2 → [Pod4(v2), Pod5(v2), Pod6(v2)]

After update:
  rs-v1 kept at replicas=0 (for rollback) ← NOT deleted
  rs-v2 is the active ReplicaSet
```

> The key insight: Kubernetes never kills an old Pod before a new one is healthy. This gives **zero-downtime deploys** out of the box.

---

### 4d) Recreate strategy (the other option)

```yaml
strategy:
  type: Recreate
```

- All old Pods are **deleted first**, then new Pods are created.
- Causes downtime — only use when two versions of your app cannot run simultaneously (e.g., database schema migration with breaking changes).

---

## 5) Readiness vs Liveness probes (critical for safe rollouts)

These are defined on the container spec inside the Pod template.

```yaml
containers:
  - name: api
    image: my-org/my-api:v1.3.0
    readinessProbe:         # "am I ready to receive traffic?"
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
    livenessProbe:          # "am I still alive?"
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 10
```

| Probe | What happens on failure |
|---|---|
| `readinessProbe` | Pod is removed from Service endpoints (traffic stops going to it), but it is NOT restarted. Rolling update waits for readiness before proceeding. |
| `livenessProbe` | Container is restarted by Kubelet. |

**Without a readinessProbe**, Kubernetes sends traffic to a new Pod the moment it starts — even if your app hasn't finished booting. Always define one.

---

## 6) Rollbacks

Every change to the Pod template creates a new revision tracked by Kubernetes.

```bash
# View rollout history
kubectl rollout history deployment/my-api

# Roll back to the previous version
kubectl rollout undo deployment/my-api

# Roll back to a specific revision
kubectl rollout undo deployment/my-api --to-revision=2

# Check rollout status live
kubectl rollout status deployment/my-api
```

Internally, a rollback just scales the old ReplicaSet back up and scales the current one down — same rolling update mechanism, just in reverse.

---

## 7) Scaling

```bash
# Imperatively
kubectl scale deployment/my-api --replicas=10

# Or edit the YAML and re-apply
kubectl apply -f deployment.yaml
```

Scaling only changes the `replicas` field on the current ReplicaSet. No new ReplicaSet is created because the Pod template hasn't changed.

---

## 8) Real-world examples

### Example A — Web API update with zero downtime

You run an e-commerce API with 5 replicas and a live payment service. You ship a bug fix.

```
Config: replicas=5, maxUnavailable=1, maxSurge=1
→ At most 4 Pods down at any time, at most 6 Pods running at any time
→ Kubernetes cycles through all 5 old Pods one at a time
→ Traffic never fully drops; users never see an error
```

### Example B — Bad deploy caught by readinessProbe

A developer pushes a broken image (`v1.4.0`) that starts up but crashes after 3 seconds.

```
New Pod(v1.4.0) starts → passes initialDelay → readinessProbe fails
→ Pod never becomes Ready
→ Rolling update stalls (maxUnavailable=1 is at its limit)
→ Old Pods keep serving traffic
→ On-call engineer runs: kubectl rollout undo deployment/my-api
→ Kubernetes rolls back, old Pods restored
→ Zero customers affected
```

### Example C — Traffic spike, HPA kicks in

Your app normally runs at 3 replicas. A flash sale drives CPU to 80%.

```
HPA monitors CPU via Metrics Server
→ CPU threshold (e.g., 60%) exceeded
→ HPA patches Deployment: replicas=3 → replicas=8
→ Deployment Controller: ReplicaSet scales up 5 new Pods
→ Scheduler places them on available nodes
→ Load is distributed across 8 Pods
→ After sale, CPU drops → HPA scales back to 3
```

---

## 9) Common operational commands

```bash
# Apply or create a Deployment
kubectl apply -f deployment.yaml

# View Deployment status
kubectl get deployment my-api
kubectl describe deployment my-api

# Watch Pods live
kubectl get pods -l app=my-api -w

# Trigger a rollout by updating the image
kubectl set image deployment/my-api api=my-org/my-api:v1.3.0

# Pause a rollout mid-way (e.g., to check canary traffic)
kubectl rollout pause deployment/my-api
kubectl rollout resume deployment/my-api

# Force restart all Pods without changing image (e.g., to pick up new ConfigMap)
kubectl rollout restart deployment/my-api

# View revision history
kubectl rollout history deployment/my-api

# Roll back one version
kubectl rollout undo deployment/my-api
```

---

## 10) Common pitfalls and how to avoid them

| Pitfall | What goes wrong | Fix |
|---|---|---|
| No `readinessProbe` | Traffic hits Pod before app is ready → errors | Always define readinessProbe |
| No resource `requests` | Scheduler can't make good placement decisions, nodes get overloaded | Always set requests |
| Using `latest` image tag | You can't tell what code is running; rollbacks are ambiguous | Use versioned or commit-SHA tags |
| Forgetting `revisionHistoryLimit` | Old ReplicaSets pile up and consume etcd space | Set `revisionHistoryLimit: 3` for non-critical apps |
| Using `Recreate` strategy for stateless apps | Unnecessary downtime | Use `RollingUpdate` (default) unless you have a hard reason not to |
| Not testing rollback | Rollback works in theory but your DB migrations are irreversible | Plan for backward-compatible migrations |

---

## 11) Quick mental model for revision

```
kubectl apply
    → API Server stores desired state
    → Deployment Controller creates/updates ReplicaSet
    → ReplicaSet Controller creates Pods
    → Scheduler assigns Pods to nodes
    → Kubelet pulls image and runs containers
    → readinessProbe passes → Pod joins Service endpoints
    → Old ReplicaSet scaled to zero (but kept for rollback)
```

Key facts to remember:
- Deployment → owns ReplicaSets → own Pods.
- One ReplicaSet per Pod template version.
- Rolling update = scale new RS up, scale old RS down, interleaved.
- Rollback = promote old RS, demote current RS (same mechanism).
- readinessProbe gates when traffic + rolling update proceeds.
- No image change = no new ReplicaSet; only replica count changes.
