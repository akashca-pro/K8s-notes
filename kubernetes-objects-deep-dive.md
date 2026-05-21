# Every Kubernetes Object — Deep Dive

---

## Table of Contents

1. [Pod](#1-pod)
2. [ReplicaSet](#2-replicaset)
3. [Deployment](#3-deployment)
4. [StatefulSet](#4-statefulset)
5. [DaemonSet](#5-daemonset)
6. [Job](#6-job)
7. [CronJob](#7-cronjob)
8. [Service](#8-service)
9. [Ingress](#9-ingress)
10. [ConfigMap](#10-configmap)
11. [Secret](#11-secret)
12. [Namespace](#12-namespace)
13. [ServiceAccount](#13-serviceaccount)
14. [RBAC — Role, ClusterRole, RoleBinding, ClusterRoleBinding](#14-rbac)
15. [NetworkPolicy](#15-networkpolicy)
16. [HorizontalPodAutoscaler (HPA)](#16-horizontalpodautoscaler-hpa)
17. [VerticalPodAutoscaler (VPA)](#17-verticalpodautoscaler-vpa)
18. [PodDisruptionBudget (PDB)](#18-poddisruptionbudget-pdb)
19. [LimitRange](#19-limitrange)
20. [ResourceQuota](#20-resourcequota)
21. [PersistentVolume (PV) & PersistentVolumeClaim (PVC)](#21-persistentvolume-pv--persistentvolumeclaim-pvc)
22. [StorageClass](#22-storageclass)
23. [EndpointSlice](#23-endpointslice)
24. [CustomResourceDefinition (CRD)](#24-customresourcedefinition-crd)
25. [PriorityClass](#25-priorityclass)
26. [Quick Decision Guide — Which Object to Use?](#26-quick-decision-guide)

---

## 1) Pod

### What problem does a Pod solve?

A Pod is the **smallest deployable unit** in Kubernetes. It wraps one or more containers that must be co-located on the same node, share the same network namespace (same IP, same ports), and optionally share storage volumes.

> One-liner: A Pod is a **logical host** for one or more tightly coupled containers.

The key insight is that Kubernetes doesn't schedule individual containers — it schedules Pods. All containers inside a Pod:
- Share `localhost` network (they can communicate via `localhost:port`)
- Share the same lifecycle (created together, destroyed together)
- Can share volumes mounted at the Pod level

You almost never create a bare Pod directly in production. Instead you use a controller (Deployment, StatefulSet, etc.) that creates and manages Pods for you. A bare Pod that dies is **not rescheduled**.

---

### Anatomy of a Pod YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
    version: v1.2.0
spec:
  serviceAccountName: my-app-sa     # which identity this Pod runs as
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  containers:
    - name: app
      image: my-org/my-app:v1.2.0
      ports:
        - containerPort: 8080
      env:
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: host
      resources:
        requests:
          cpu: "250m"
          memory: "256Mi"
        limits:
          cpu: "500m"
          memory: "512Mi"
      readinessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 5
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 15
        periodSeconds: 10
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
  initContainers:
    - name: wait-for-db
      image: busybox
      command: ['sh', '-c', 'until nc -z db-service 5432; do sleep 1; done']
  volumes:
    - name: config-volume
      configMap:
        name: my-app-config
  restartPolicy: Always             # Always | OnFailure | Never
  terminationGracePeriodSeconds: 30
```

### Key fields explained

| Field | What it controls |
|---|---|
| `initContainers` | Run to completion before app containers start; use to wait for dependencies or run migrations |
| `serviceAccountName` | The RBAC identity attached to the Pod |
| `securityContext` | Kernel-level security settings (user ID, read-only filesystem, capabilities) |
| `resources.requests` | Minimum guaranteed resources; used by Scheduler |
| `resources.limits` | Hard cap; OOM-killed if memory exceeds this |
| `readinessProbe` | Gates traffic routing; Pod removed from Service endpoints on failure |
| `livenessProbe` | Gates container health; container restarted on failure |
| `startupProbe` | One-time probe at startup; disables liveness until app is up |
| `restartPolicy` | `Always` = always restart, `OnFailure` = only on non-zero exit, `Never` = never restart |
| `terminationGracePeriodSeconds` | How long kubelet waits for graceful shutdown before SIGKILL |

---

### Pod lifecycle states

```
Pending  → Pod accepted by API server, waiting for scheduler or image pull
Running  → At least one container is running or starting
Succeeded → All containers exited with code 0 (only meaningful for Jobs)
Failed    → All containers terminated, at least one failed
Unknown   → Node lost contact with API server
```

### Multi-container Pod patterns

**Sidecar pattern** — helper container runs alongside main app

```yaml
containers:
  - name: app           # main app writes logs to /var/log/app
  - name: log-shipper   # sidecar reads /var/log/app and ships to Elasticsearch
```

**Init container pattern** — run once before app starts

```yaml
initContainers:
  - name: migrate-db    # runs DB migrations, exits 0, then app container starts
```

**Ambassador pattern** — proxy container handles network concerns

```yaml
containers:
  - name: app           # connects to localhost:6379 (as if Redis is local)
  - name: redis-proxy   # ambassador forwards localhost:6379 to cluster Redis
```

---

### Real production use case

**ML inference service**: A Pod runs two containers — the model server (TensorFlow Serving) and a sidecar that pulls the latest model from S3 and reloads it into shared memory via an `emptyDir` volume. The sidecar handles hot model reloads without restarting the inference server.

---

### How to debug Pods

```bash
# See Pod status and events
kubectl describe pod <pod-name> -n <namespace>

# Live logs (tail)
kubectl logs -f <pod-name> -n <namespace>

# Logs from a specific container in a multi-container Pod
kubectl logs <pod-name> -c <container-name> -n <namespace>

# Logs from a previous crashed container
kubectl logs <pod-name> --previous -n <namespace>

# Shell into a running container
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# Run a debug sidecar on an existing Pod (K8s 1.23+)
kubectl debug -it <pod-name> --image=busybox --target=<container-name>

# Check what's happening on the node
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Describe node the Pod is on
kubectl describe node <node-name>
```

**Common failure reasons and fixes**

| Symptom | Likely cause | Fix |
|---|---|---|
| `ImagePullBackOff` | Wrong image tag or private registry with no pull secret | Fix image name or create `imagePullSecrets` |
| `CrashLoopBackOff` | Container exits immediately | Check `kubectl logs --previous`; fix app startup |
| `OOMKilled` | Exceeded memory limit | Increase `resources.limits.memory` |
| `Pending` forever | No node has enough resources or no matching node selector | Check node capacity, affinity rules |
| `CreateContainerError` | Volume not found, config missing | Check `kubectl describe pod` Events section |
| `RunContainerError` | Security policy violation, bad entrypoint | Check securityContext, image CMD |

---

### Alternatives to bare Pods

Never use bare Pods in production. Use:
- **Deployment** for stateless apps (99% of cases)
- **StatefulSet** for stateful apps (databases, queues)
- **DaemonSet** for per-node agents
- **Job / CronJob** for batch workloads

---

## 2) ReplicaSet

### What problem does a ReplicaSet solve?

A ReplicaSet ensures a **specified number of identical Pod replicas** are running at any given time. If a Pod crashes, the ReplicaSet controller creates a replacement. If too many Pods are running, it deletes the excess.

> One-liner: A ReplicaSet is the **self-healing replica manager**.

In practice, you almost never create a ReplicaSet directly. Deployments create and own ReplicaSets for you, adding rollout and rollback logic on top.

---

### Anatomy of a ReplicaSet YAML

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-app-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app          # owns Pods with this label
  template:                # Pod blueprint
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: my-org/my-app:v1.2.0
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
```

### How ReplicaSet ownership works

The ReplicaSet finds its Pods using the `selector`. Any Pod on the cluster with matching labels is **adopted** by the ReplicaSet — even Pods you created manually. This is a common footgun: if you create a bare Pod with the same labels as an existing ReplicaSet, the RS will try to delete it to bring the count back to `replicas`.

---

### How to debug ReplicaSets

```bash
kubectl get replicaset -n <namespace>
kubectl describe replicaset <rs-name> -n <namespace>

# See which Pods belong to this RS
kubectl get pods -l app=my-app -n <namespace>

# If Pods aren't being created, check RS events
kubectl describe rs <rs-name> -n <namespace>
```

---

### When to use ReplicaSet directly vs Deployment

| Use case | Use |
|---|---|
| Simple replica management, no rollout needed | ReplicaSet directly (rare) |
| Any production workload with code changes | Deployment (wraps RS with rollout logic) |
| You need a fixed RS for a canary split | Deployment + manual RS scaling (advanced) |

---

## 3) Deployment

Deployments are covered in detail in `kubernetes-deployments-deep-dive.md`. Here's a quick reference.

### What it adds over ReplicaSet

- Declarative rolling updates (creates a new RS, scales up, scales old RS down)
- Rollback to any previous revision
- Pause/resume rollouts for canary inspection
- Stores revision history

### Quick YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-api
  namespace: production
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: my-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: my-api
    spec:
      containers:
        - name: api
          image: my-org/my-api:v1.3.0
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
```

### Key debug commands

```bash
kubectl rollout status deployment/my-api
kubectl rollout history deployment/my-api
kubectl rollout undo deployment/my-api
kubectl rollout undo deployment/my-api --to-revision=3
```

---

## 4) StatefulSet

### What problem does a StatefulSet solve?

Deployments treat all Pods as **identical and interchangeable**. This breaks for stateful apps like databases and message brokers that need:

- **Stable network identity**: Pod gets the same DNS name even after rescheduling (`pod-0.my-service.ns.svc.cluster.local`)
- **Stable persistent storage**: Each Pod gets its own PVC that follows it across reschedules (not shared)
- **Ordered startup and shutdown**: Pod-0 starts before Pod-1; Pod-2 terminates before Pod-1 (important for leader election, replication setup)

> One-liner: A StatefulSet is a Deployment where **each Pod has a fixed identity and its own storage**.

---

### Anatomy of a StatefulSet YAML

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: "postgres"      # must match a Headless Service name (see below)
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
  volumeClaimTemplates:        # each Pod gets its OWN PVC from this template
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 50Gi
  updateStrategy:
    type: RollingUpdate        # updates one Pod at a time, from highest ordinal to lowest
```

### Required Headless Service

StatefulSets require a **headless Service** (clusterIP: None) to create stable DNS records per Pod.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres          # matches serviceName in StatefulSet
  namespace: production
spec:
  clusterIP: None         # headless — no virtual IP, DNS returns Pod IPs directly
  selector:
    app: postgres
  ports:
    - port: 5432
```

With this, each Pod gets its own DNS entry:
```
postgres-0.postgres.production.svc.cluster.local
postgres-1.postgres.production.svc.cluster.local
postgres-2.postgres.production.svc.cluster.local
```

---

### StatefulSet Pod naming

Pods are named `<statefulset-name>-<ordinal>`:
- `postgres-0` (first, starts first, terminates last)
- `postgres-1`
- `postgres-2` (last, starts last, terminates first)

---

### StatefulSet vs Deployment

| Property | Deployment | StatefulSet |
|---|---|---|
| Pod names | Random hash (`my-app-7d9f8b-xkz`) | Ordered (`postgres-0`, `postgres-1`) |
| Pod DNS | Not stable | Stable per-Pod DNS |
| Storage | Pods share a PVC or have no persistence | Each Pod gets its own PVC |
| Startup order | All at once (parallel) | In order (0, 1, 2...) |
| Shutdown order | All at once | Reverse order (2, 1, 0) |
| Use case | Stateless apps | Databases, queues, clustered apps |

---

### Real production use case

**Kafka cluster**: 3-broker Kafka cluster deployed as a StatefulSet. Each broker needs a stable identity (`kafka-0`, `kafka-1`, `kafka-2`) so that partitions can be reliably assigned to a specific broker. Each broker gets its own 500Gi SSD PVC for its logs. Zookeeper (or KRaft) uses the stable DNS names to coordinate the cluster. When a broker pod is rescheduled to a different node, it reattaches to the same PVC and rejoins the cluster with the same identity.

---

### How to debug StatefulSets

```bash
# Check StatefulSet status (shows Ready count)
kubectl get statefulset -n <namespace>
kubectl describe statefulset postgres -n <namespace>

# Check Pod startup order (ordinal progression)
kubectl get pods -l app=postgres -n <namespace> -w

# Check individual Pod and its PVC
kubectl describe pod postgres-0 -n <namespace>
kubectl get pvc -l app=postgres -n <namespace>

# Exec into the primary (ordinal 0)
kubectl exec -it postgres-0 -n <namespace> -- psql -U postgres

# Check DNS resolution from inside a Pod
kubectl exec -it postgres-0 -- nslookup postgres-1.postgres.production.svc.cluster.local

# Manual rolling restart
kubectl rollout restart statefulset/postgres -n <namespace>
kubectl rollout status statefulset/postgres -n <namespace>
```

**Common StatefulSet issues**

| Symptom | Likely cause | Fix |
|---|---|---|
| Pod stuck in `Pending` | PVC can't be provisioned (no storage class, quota exceeded) | Check `kubectl describe pvc` |
| Pod-1 never starts | Pod-0 is not `Running` + `Ready` | Fix Pod-0 first; SS won't move to next ordinal |
| Old Pod reattaches to wrong data | PVC naming mismatch after rename | PVCs are named `<volumeClaimTemplate>-<pod-name>` — don't rename the SS |
| Pod rescheduled but PVC not found | Zone mismatch (PVC is in us-east-1a, node is in us-east-1b) | Use topology-aware storage or pin Pods to zones |

---

### Alternatives

| Alternative | When to use |
|---|---|
| Managed databases (RDS, Cloud SQL) | Any production database — offloads ops burden |
| Helm charts (Bitnami PostgreSQL) | Wraps StatefulSet complexity with proven config |
| Operators (Postgres Operator, Strimzi for Kafka) | Complex stateful apps that need lifecycle management beyond SS |

---

## 5) DaemonSet

### What problem does a DaemonSet solve?

A DaemonSet ensures that **exactly one Pod runs on every node** (or a subset of nodes matching a selector). When a new node joins the cluster, the DaemonSet controller automatically creates a Pod on it. When a node is removed, its Pod is garbage-collected.

> One-liner: A DaemonSet is the mechanism for **deploying per-node infrastructure agents**.

---

### Anatomy of a DaemonSet YAML

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  updateStrategy:
    type: RollingUpdate        # or OnDelete
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane   # also run on control plane nodes
          operator: Exists
          effect: NoSchedule
      hostNetwork: true        # bind to host network namespace
      hostPID: true            # access host process namespace (needed for some agents)
      containers:
        - name: node-exporter
          image: prom/node-exporter:v1.7.0
          ports:
            - containerPort: 9100
              hostPort: 9100
          volumeMounts:
            - name: proc
              mountPath: /host/proc
              readOnly: true
            - name: sys
              mountPath: /host/sys
              readOnly: true
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: sys
          hostPath:
            path: /sys
```

### Run on a subset of nodes using nodeSelector

```yaml
spec:
  template:
    spec:
      nodeSelector:
        node-type: gpu            # only schedule on GPU nodes
```

---

### Common DaemonSet use cases

| Agent | Purpose |
|---|---|
| `fluentd` / `fluent-bit` | Collect and ship logs from every node |
| `node-exporter` (Prometheus) | Scrape node-level metrics (CPU, disk, network) |
| `datadog-agent` / `newrelic` | APM, metrics, traces per node |
| `calico-node` / `cilium` | CNI network plugin — must run on every node |
| `kube-proxy` | Manages iptables / ipvs rules for Service routing |
| `csi-node-driver` | Node-side component of a storage CSI driver |
| Falco | Runtime security monitoring, reads kernel syscalls |

---

### Real production use case

**Log shipping**: Fluent Bit DaemonSet runs on every node with `hostPath` volume mounts on `/var/log/containers`. It tails container logs, enriches them with Pod/namespace metadata from the Kubernetes API, and ships them to Elasticsearch. When a new node autoscales in, the DaemonSet Pod lands on it automatically within seconds — zero manual configuration.

---

### How to debug DaemonSets

```bash
# Check DaemonSet rollout (desired vs ready)
kubectl get daemonset -n monitoring
kubectl describe daemonset node-exporter -n monitoring

# Check which nodes have a Pod and which don't
kubectl get pods -l app=node-exporter -n monitoring -o wide

# Node missing a Pod? Check if taint is blocking it
kubectl describe node <node-name> | grep Taint

# Add a toleration or check nodeSelector
kubectl get ds node-exporter -n monitoring -o yaml | grep -A 10 tolerations

# Check rolling update status
kubectl rollout status daemonset/node-exporter -n monitoring
```

---

### Alternatives

| Alternative | When to use |
|---|---|
| Deployment with `nodeAffinity` | If you want agents on specific nodes but don't need exactly-one-per-node |
| Node-level systemd services | For very low-level kernel agents that can't run in containers |
| `initContainers` on other Pods | Lightweight node setup (e.g., sysctl tuning) on first Pod per node |

---

## 6) Job

### What problem does a Job solve?

A Deployment runs Pods that are **meant to run forever** and restarts them on exit. A Job runs Pods that are **meant to complete**, tracks their success or failure, and is done when enough completions are achieved.

> One-liner: A Job manages **one-shot or parallelized batch workloads** to a successful completion.

---

### Anatomy of a Job YAML

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  namespace: production
spec:
  completions: 1          # total successful completions needed
  parallelism: 1          # how many Pods can run in parallel
  backoffLimit: 3         # retry up to 3 times before marking Job failed
  activeDeadlineSeconds: 600   # kill Job if not completed in 10 min
  ttlSecondsAfterFinished: 3600   # auto-delete Job 1 hour after completion
  template:
    spec:
      restartPolicy: OnFailure   # REQUIRED; Jobs cannot use restartPolicy: Always
      containers:
        - name: migrate
          image: my-org/migrations:v1.3.0
          command: ["python", "manage.py", "migrate"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
```

### Parallel Job patterns

```yaml
# Indexed jobs — each Pod gets a unique JOB_COMPLETION_INDEX env var
spec:
  completions: 10
  parallelism: 3
  completionMode: Indexed   # Pods get index 0..9; use to partition work
```

| Pattern | completions | parallelism | Use case |
|---|---|---|---|
| Single run | 1 | 1 | DB migration, one-time script |
| Fixed completion count | N | 1 | N sequential tasks |
| Parallel fixed | N | K | N tasks, K workers at a time |
| Parallel indexed | N | K | Partitioned data processing (shard per index) |

---

### Real production use case

**Database migration on deploy**: In a CI/CD pipeline, before rolling out a new Deployment, a Job runs `flask db upgrade` or `rails db:migrate`. The pipeline uses `kubectl wait job/db-migration --for=condition=Complete --timeout=300s`. If the migration Job fails, the pipeline aborts and the new Deployment never rolls out.

---

### How to debug Jobs

```bash
# Check Job status (Succeeded/Failed)
kubectl get job db-migration -n production
kubectl describe job db-migration -n production

# Find the Job's Pod(s)
kubectl get pods -l job-name=db-migration -n production

# Get logs from the Job Pod
kubectl logs job/db-migration -n production
kubectl logs <pod-name> --previous -n production   # if it restarted

# Watch a Job to completion
kubectl wait job/db-migration --for=condition=Complete --timeout=300s -n production

# Manually trigger a re-run by deleting and recreating
kubectl delete job db-migration -n production
kubectl apply -f db-migration-job.yaml
```

**Common Job issues**

| Symptom | Cause | Fix |
|---|---|---|
| Job stuck, Pod never starts | Resource constraints or node pressure | Check `kubectl describe pod` |
| Job retries infinitely | App exits non-zero for a non-retryable reason | Check exit code; set `backoffLimit` appropriately |
| Old Job blocking new one | Same Job name already exists | `kubectl delete job` before re-applying |
| Job completes but Pods linger | `ttlSecondsAfterFinished` not set | Set it so Pods are auto-cleaned |

---

### Alternatives

| Alternative | When to use |
|---|---|
| CronJob | Scheduled/recurring batch tasks |
| Argo Workflows | Complex DAG pipelines with dependencies, retries, artifacts |
| Tekton | CI/CD native pipeline tasks in K8s |
| Spark Operator | Large-scale distributed data processing |

---

## 7) CronJob

### What problem does a CronJob solve?

A CronJob creates Jobs on a **cron schedule**. It's the Kubernetes equivalent of a Unix cron task — the schedule is specified in standard cron syntax.

> One-liner: A CronJob is a **scheduled Job factory**.

---

### Anatomy of a CronJob YAML

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
  namespace: production
spec:
  schedule: "0 2 * * *"         # every day at 2:00 AM UTC
  timeZone: "America/New_York"  # K8s 1.27+; otherwise all times are UTC
  concurrencyPolicy: Forbid     # Allow | Forbid | Replace
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  startingDeadlineSeconds: 300  # if Job misses schedule by >5 min, skip this run
  jobTemplate:
    spec:
      backoffLimit: 2
      ttlSecondsAfterFinished: 3600
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: my-org/db-backup:latest
              command: ["/backup.sh"]
              env:
                - name: S3_BUCKET
                  value: "my-backups"
              resources:
                requests:
                  cpu: "250m"
                  memory: "512Mi"
```

### concurrencyPolicy options

| Value | Behavior |
|---|---|
| `Allow` | New Job starts even if previous one is still running |
| `Forbid` | Skip new Job if previous one hasn't finished |
| `Replace` | Delete the running Job and start a fresh one |

---

### Cron schedule cheatsheet

```
# ┌──── minute (0-59)
# │ ┌──── hour (0-23)
# │ │ ┌──── day of month (1-31)
# │ │ │ ┌──── month (1-12)
# │ │ │ │ ┌──── day of week (0-6, 0=Sunday)
# * * * * *

"*/5 * * * *"      # every 5 minutes
"0 * * * *"        # every hour at :00
"0 2 * * *"        # daily at 2 AM
"0 2 * * 0"        # every Sunday at 2 AM
"0 2 1 * *"        # first of each month at 2 AM
"0 2 * * 1-5"      # weekdays at 2 AM
```

---

### Real production use case

**Report generation**: A CronJob runs every weekday at 6 AM, queries the analytics database, generates a PDF report, and emails it to stakeholders. `concurrencyPolicy: Forbid` prevents overlap if the DB query runs long. `startingDeadlineSeconds: 300` ensures stale reports (if scheduler was down) are not sent late and confuse readers.

---

### How to debug CronJobs

```bash
# Check CronJob and last schedule time
kubectl get cronjob -n production
kubectl describe cronjob db-backup -n production

# See Jobs created by the CronJob
kubectl get jobs -l app=db-backup -n production

# Manually trigger a run immediately
kubectl create job --from=cronjob/db-backup db-backup-manual -n production

# Check logs of the latest run
kubectl logs job/$(kubectl get jobs -l app=db-backup --sort-by=.metadata.creationTimestamp -o name | tail -1 | cut -d/ -f2) -n production
```

---

## 8) Service

### What problem does a Service solve?

Pods are **ephemeral** — they come and go, and their IP addresses change. A Service provides a **stable virtual IP and DNS name** that load-balances traffic across a set of Pods selected by labels.

> One-liner: A Service is a **stable network endpoint** that abstracts a dynamic set of Pods.

---

### The four Service types

#### ClusterIP (default)

Internal-only virtual IP. Only reachable within the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-api
  namespace: production
spec:
  type: ClusterIP        # default; can omit
  selector:
    app: my-api
  ports:
    - name: http
      port: 80           # port the Service listens on
      targetPort: 8080   # port on the Pod container
      protocol: TCP
```

DNS: `my-api.production.svc.cluster.local` → resolves to `ClusterIP`

---

#### NodePort

Exposes the Service on a static port on every node's IP. Useful for development; not for production internet traffic.

```yaml
spec:
  type: NodePort
  selector:
    app: my-api
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080    # must be 30000–32767; omit to auto-assign
```

Accessible at `<any-node-ip>:30080` from outside the cluster.

---

#### LoadBalancer

Creates an external load balancer in the cloud provider (AWS ELB, GCP LB, Azure LB). The LB forwards traffic to the NodePort of the Service.

```yaml
spec:
  type: LoadBalancer
  selector:
    app: my-api
  ports:
    - port: 443
      targetPort: 8080
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"           # AWS NLB
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
```

Each `LoadBalancer` Service creates a new cloud LB — expensive for many services. Use Ingress + one LB instead.

---

#### Headless Service (clusterIP: None)

Returns individual Pod IPs directly from DNS instead of a virtual IP. Used by StatefulSets and service discovery.

```yaml
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
```

DNS lookup returns all Pod IPs — caller picks one (useful for database drivers doing their own connection management).

---

#### ExternalName

Maps a Service to an external DNS name. No proxying — just DNS CNAME.

```yaml
spec:
  type: ExternalName
  externalName: my-database.rds.amazonaws.com
```

Useful for migrating traffic gradually from external to in-cluster resources or vice versa.

---

### How kube-proxy handles Services

```
Client Pod → sends packet to ClusterIP:80
kube-proxy (iptables/ipvs) intercepts packet
→ DNAT to one of the backend Pod IPs:8080
→ Packet arrives at Pod
```

kube-proxy watches the API server for Service and EndpointSlice changes and updates iptables/ipvs rules on every node in real time.

---

### Session affinity

```yaml
spec:
  sessionAffinity: ClientIP        # same client IP always goes to same Pod
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
```

---

### Real production use case

**Microservices communication**: 15 microservices communicate internally using ClusterIP Services. Service discovery is by DNS name (`payment-service.production.svc.cluster.local`). A single `LoadBalancer` Service or Ingress handles external traffic at the edge. Headless Services front the PostgreSQL and Redis StatefulSets so clients can connect to specific primaries.

---

### How to debug Services

```bash
# Check service endpoints (which Pods are behind the Service)
kubectl get endpoints my-api -n production
kubectl describe service my-api -n production

# Service has no endpoints? Check label selector match
kubectl get pods -n production -l app=my-api    # do these Pods exist?
kubectl get service my-api -n production -o yaml | grep selector -A 5

# DNS resolution from inside a Pod
kubectl exec -it <any-pod> -- nslookup my-api.production.svc.cluster.local

# Port-forward for local testing (bypasses kube-proxy)
kubectl port-forward service/my-api 8080:80 -n production

# Trace the path: pod → service → endpoint
kubectl exec -it <pod> -- curl http://my-api.production.svc.cluster.local/healthz
```

**Common Service issues**

| Symptom | Cause | Fix |
|---|---|---|
| No endpoints | Selector doesn't match Pod labels | Fix `spec.selector` or Pod labels |
| Service resolves but connection times out | kube-proxy not running or iptables issue | Check kube-proxy pod logs |
| `LoadBalancer` stays `<pending>` | No cloud load balancer integration | Check cloud-controller-manager; use `NodePort` in local cluster |
| Wrong Pod receiving traffic | Label mismatch (multiple apps share label) | Use unique label keys per service |

---

## 9) Ingress

### What problem does Ingress solve?

A `LoadBalancer` Service creates one cloud load balancer per Service — expensive and unmanageable at scale. An Ingress provides **HTTP/HTTPS routing rules** at Layer 7, pointing many virtual host routes to different Services behind a **single** load balancer.

> One-liner: Ingress is an **HTTP reverse proxy configuration** backed by an Ingress Controller.

---

### Ingress Controller vs Ingress resource

- **Ingress resource**: A Kubernetes object declaring routing rules (path, host → service)
- **Ingress Controller**: The actual reverse proxy (nginx, traefik, AWS ALB, etc.) that reads Ingress resources and configures itself

The Ingress resource does nothing without a running Ingress Controller.

---

### Anatomy of an Ingress YAML

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"    # auto TLS via cert-manager
spec:
  ingressClassName: nginx                  # which Ingress Controller handles this
  tls:
    - hosts:
        - api.myapp.com
        - admin.myapp.com
      secretName: myapp-tls               # TLS cert stored as Secret
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /v1
            pathType: Prefix
            backend:
              service:
                name: api-v1
                port:
                  number: 80
          - path: /v2
            pathType: Prefix
            backend:
              service:
                name: api-v2
                port:
                  number: 80
    - host: admin.myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-dashboard
                port:
                  number: 80
```

### pathType options

| Value | Behavior |
|---|---|
| `Exact` | Only exact path matches (`/foo` matches only `/foo`) |
| `Prefix` | Prefix match (`/foo` matches `/foo`, `/foo/bar`, `/foo/baz`) |
| `ImplementationSpecific` | Behavior depends on the Ingress Controller |

---

### Popular Ingress Controllers

| Controller | Best for |
|---|---|
| **ingress-nginx** | General purpose; most popular; feature-rich |
| **AWS ALB Ingress Controller** | Native AWS ALB with WAF, ACM, target groups |
| **Traefik** | Automatic service discovery, Let's Encrypt built in |
| **HAProxy Ingress** | High performance, fine-grained TCP/HTTP control |
| **Istio Gateway** | Service mesh traffic management |
| **Contour / Envoy** | Envoy-based, good for gRPC and HTTP/2 |

---

### Real production use case

**SaaS multi-tenant routing**: A single nginx Ingress Controller fronts 20 microservices. Each service has its own path (`api.example.com/users`, `/payments`, `/notifications`). cert-manager automatically provisions and renews Let's Encrypt certificates. The Ingress Controller is the only component with a `LoadBalancer` Service (one cloud LB for the whole cluster).

---

### How to debug Ingress

```bash
# Check Ingress and its address
kubectl get ingress -n production
kubectl describe ingress my-app-ingress -n production

# Check Ingress Controller logs (nginx example)
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx -f

# Verify the backend Service exists and has endpoints
kubectl get svc api-v1 -n production
kubectl get endpoints api-v1 -n production

# Test routing from outside
curl -H "Host: api.myapp.com" http://<ingress-external-ip>/v1/healthz

# Check TLS cert
kubectl get certificate -n production   # cert-manager Certificate object
kubectl describe certificate myapp-tls -n production
kubectl get secret myapp-tls -n production
```

---

### Alternatives

| Alternative | When to use |
|---|---|
| Gateway API (`HTTPRoute`) | Next-gen Ingress with role separation; replaces Ingress long-term |
| Service Mesh (Istio, Linkerd) | Advanced traffic management (canary, retries, circuit breaking) |
| AWS API Gateway + ALB | When you want managed API gateway features |

---

## 10) ConfigMap

### What problem does a ConfigMap solve?

Hardcoding configuration inside container images violates the [12-factor app](https://12factor.net/config) principle. A ConfigMap stores **non-sensitive configuration data** (env vars, config files, command-line args) as key-value pairs, decoupled from the Pod spec.

> One-liner: ConfigMap is a **key-value store for non-sensitive app configuration**.

---

### Anatomy of a ConfigMap YAML

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-config
  namespace: production
data:
  # Simple key-value pairs
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
  FEATURE_FLAGS: "dark_mode=true,beta_ui=false"

  # Multi-line file content (e.g., a config file)
  app.properties: |
    server.port=8080
    server.timeout=30s
    database.pool.size=20

  nginx.conf: |
    server {
      listen 80;
      location / {
        proxy_pass http://localhost:8080;
      }
    }
```

### Two ways to consume a ConfigMap in a Pod

#### 1. As environment variables

```yaml
spec:
  containers:
    - name: app
      envFrom:
        - configMapRef:
            name: my-app-config    # inject all keys as env vars
      env:
        - name: LOG_LEVEL          # inject a single key
          valueFrom:
            configMapKeyRef:
              name: my-app-config
              key: LOG_LEVEL
```

#### 2. As a mounted volume (file)

```yaml
spec:
  volumes:
    - name: config
      configMap:
        name: my-app-config
        items:
          - key: app.properties
            path: app.properties       # mount as /etc/config/app.properties
  containers:
    - name: app
      volumeMounts:
        - name: config
          mountPath: /etc/config
```

Volume-mounted ConfigMaps update **automatically** (within ~1 minute) when the ConfigMap is updated. Environment variable injection does **not** update without a Pod restart.

---

### Real production use case

**Feature flags without deploys**: A ConfigMap holds JSON feature flags (`{"dark_mode": true, "new_checkout": false}`). The app polls the mounted file every 30 seconds. To toggle a feature, an operator runs `kubectl edit configmap feature-flags` — no Pod restart, no deployment needed.

---

### How to debug ConfigMaps

```bash
# View a ConfigMap
kubectl get configmap my-app-config -n production -o yaml

# Check if a Pod sees the right value
kubectl exec -it <pod> -- env | grep LOG_LEVEL
kubectl exec -it <pod> -- cat /etc/config/app.properties

# Edit a ConfigMap live (dangerous in prod — use GitOps instead)
kubectl edit configmap my-app-config -n production
```

---

### Alternatives

| Alternative | When to use |
|---|---|
| Secret | Sensitive values (passwords, tokens, certs) |
| AWS Parameter Store / SSM | External config store; use External Secrets Operator to sync to K8s |
| HashiCorp Vault + Vault Agent | Enterprise secrets management with dynamic credentials |
| Helm values.yaml | Template-driven config management across environments |

---

## 11) Secret

### What problem does a Secret solve?

A Secret stores **sensitive data** — passwords, tokens, SSH keys, TLS certificates — base64-encoded and handled with slightly more care than ConfigMaps (RBAC restrictions, encryption at rest, not logged by default).

> One-liner: Secret is a **ConfigMap for sensitive data** with extra access controls.

**Important**: base64 is encoding, not encryption. Secrets are only as secure as your etcd encryption and RBAC configuration.

---

### Anatomy of a Secret YAML

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque               # generic key-value; other types below
stringData:                # plain text — Kubernetes base64-encodes automatically
  DB_HOST: "postgres.production.svc.cluster.local"
  DB_USER: "myapp"
  DB_PASSWORD: "supersecret"
```

**Do not commit Secret YAMLs with real values to git.** Use Sealed Secrets or External Secrets instead.

### Secret types

| Type | Use |
|---|---|
| `Opaque` | Generic key-value pairs (most common) |
| `kubernetes.io/tls` | TLS certificate and key (`tls.crt` and `tls.key`) |
| `kubernetes.io/dockerconfigjson` | Registry pull credentials |
| `kubernetes.io/service-account-token` | Auto-created for ServiceAccounts |
| `kubernetes.io/ssh-auth` | SSH private key |
| `kubernetes.io/basic-auth` | Username + password |

### TLS Secret (for Ingress)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-tls
  namespace: production
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
```

### Image pull Secret

```yaml
# Create via kubectl (easiest)
kubectl create secret docker-registry regcred \
  --docker-server=registry.myorg.com \
  --docker-username=ci-bot \
  --docker-password=$TOKEN \
  -n production

# Reference in Pod spec
spec:
  imagePullSecrets:
    - name: regcred
```

---

### Real production use case

**Zero-trust credential management**: No passwords are committed to git. The CD pipeline uses External Secrets Operator to sync secrets from AWS Secrets Manager into Kubernetes Secrets. Pods consume them as env vars or mounted files. Rotation happens in AWS — the operator syncs the new value within minutes without any redeploy.

---

### How to debug Secrets

```bash
# List secrets (values hidden)
kubectl get secret -n production

# View a secret's keys (not values)
kubectl describe secret db-credentials -n production

# Decode a secret value
kubectl get secret db-credentials -n production -o jsonpath='{.data.DB_PASSWORD}' | base64 -d

# Check if Pod is consuming secret correctly
kubectl exec -it <pod> -- env | grep DB_PASSWORD
kubectl exec -it <pod> -- cat /etc/secrets/DB_PASSWORD
```

---

### Hardening Secrets

```yaml
# 1. Enable encryption at rest in API server config
# /etc/kubernetes/encryption-config.yaml:
resources:
  - resources: [secrets]
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-32-byte-key>

# 2. Restrict Secret access with RBAC (see RBAC section)
# 3. Use ExternalSecrets Operator + AWS/Vault backend
# 4. Use Sealed Secrets (Bitnami) for GitOps-safe encrypted manifests
```

---

## 12) Namespace

### What problem does a Namespace solve?

A Namespace provides a **virtual cluster** within a physical cluster, scoping names, RBAC rules, resource quotas, and network policies.

> One-liner: Namespace is a **logical partition** of cluster resources for isolation and organization.

---

### Anatomy of a Namespace YAML

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments
  labels:
    team: platform
    environment: production
```

### Built-in namespaces

| Namespace | Purpose |
|---|---|
| `default` | Workloads with no namespace specified |
| `kube-system` | Core Kubernetes components (kube-proxy, CoreDNS, etc.) |
| `kube-public` | Public cluster info (e.g., cluster-info ConfigMap) |
| `kube-node-lease` | Node heartbeat lease objects |

---

### Namespace-scoped vs cluster-scoped resources

| Namespace-scoped | Cluster-scoped |
|---|---|
| Pod, Deployment, Service, ConfigMap, Secret | Node, PersistentVolume, ClusterRole, StorageClass |
| Ingress, NetworkPolicy, HPA | Namespace itself |

---

### Real production use case

**Environment isolation**: Three namespaces — `development`, `staging`, `production` — in the same cluster. `ResourceQuota` limits how much CPU/memory each namespace can use. `NetworkPolicy` blocks cross-namespace traffic except for the monitoring namespace. Each team gets a namespace with RBAC rules granting full access to their namespace but read-only to others.

---

### How to debug Namespace issues

```bash
# List all namespaces
kubectl get namespaces

# Set default namespace for current context
kubectl config set-context --current --namespace=production

# Find objects across ALL namespaces
kubectl get pods --all-namespaces
kubectl get pods -A

# Namespace stuck in Terminating
kubectl get namespace payments -o yaml | grep finalizers
# Remove finalizers if namespace is stuck
kubectl patch namespace payments -p '{"metadata":{"finalizers":[]}}' --type=merge
```

---

## 13) ServiceAccount

### What problem does a ServiceAccount solve?

Pods need to call the Kubernetes API (e.g., list Pods, read Secrets, watch ConfigMaps). A ServiceAccount provides a **machine identity** for the Pod, with a token automatically mounted into the Pod filesystem that the API server uses to authenticate the request.

> One-liner: ServiceAccount is the **identity for Pods calling the Kubernetes API**.

---

### Anatomy of a ServiceAccount YAML

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::123456789:role/my-app-role"   # IRSA on EKS
automountServiceAccountToken: false    # disable auto-mount if not needed
```

### Consuming a ServiceAccount in a Pod

```yaml
spec:
  serviceAccountName: my-app-sa
  automountServiceAccountToken: true    # explicitly opt-in if you need it
```

If you don't specify a `serviceAccountName`, the Pod uses the `default` ServiceAccount in its namespace — which usually has no permissions (and shouldn't have any).

---

### ServiceAccount token projection (K8s 1.20+)

Modern Kubernetes uses **projected service account tokens** — short-lived, audience-bound JWTs — instead of long-lived tokens stored as Secrets.

```yaml
spec:
  volumes:
    - name: token
      projected:
        sources:
          - serviceAccountToken:
              audience: api
              expirationSeconds: 3600
              path: token
  containers:
    - name: app
      volumeMounts:
        - name: token
          mountPath: /var/run/secrets/tokens
```

---

### IRSA (IAM Roles for Service Accounts) on EKS

The `eks.amazonaws.com/role-arn` annotation on a ServiceAccount allows a Pod to assume an AWS IAM role. This is the correct way to give Pods AWS API access without long-lived credentials.

```yaml
# The Pod's service account token is exchanged for temporary AWS credentials
# via OIDC federation — no AWS keys stored anywhere in the cluster
```

---

### How to debug ServiceAccounts

```bash
# List service accounts in a namespace
kubectl get serviceaccount -n production

# Check what token a Pod has
kubectl exec -it <pod> -- cat /var/run/secrets/kubernetes.io/serviceaccount/token

# Test the token against the API server
TOKEN=$(kubectl exec -it <pod> -- cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -H "Authorization: Bearer $TOKEN" https://kubernetes.default.svc/api/v1/namespaces/production/pods
```

---

## 14) RBAC

### What problem does RBAC solve?

By default, if a user or Pod can reach the Kubernetes API server, they can do anything. RBAC (**Role-Based Access Control**) restricts what actions subjects (users, groups, ServiceAccounts) can perform on which resources.

> One-liner: RBAC controls **who can do what to which Kubernetes resources**.

---

### The four RBAC objects

| Object | Scope | Purpose |
|---|---|---|
| `Role` | Namespace | Grant permissions within one namespace |
| `ClusterRole` | Cluster-wide | Grant permissions across all namespaces or on cluster-scoped resources |
| `RoleBinding` | Namespace | Bind a Role or ClusterRole to a subject, within one namespace |
| `ClusterRoleBinding` | Cluster-wide | Bind a ClusterRole to a subject across the entire cluster |

---

### Anatomy of RBAC YAML

```yaml
# Role: read-only access to Pods and ConfigMaps in 'production'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
  - apiGroups: [""]                   # "" = core API group
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]

---
# RoleBinding: bind the Role to a ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: my-app-sa
    namespace: production
  - kind: User                        # or a human user from your IdP
    name: alice@company.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

---
# ClusterRole: manage Nodes across the cluster
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-manager
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch", "patch"]

---
# ClusterRoleBinding: bind to an ops team Group
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-manager-binding
subjects:
  - kind: Group
    name: "platform-ops"              # OIDC group from your SSO provider
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-manager
  apiGroup: rbac.authorization.k8s.io
```

### Available verbs

| Verb | HTTP method | Meaning |
|---|---|---|
| `get` | GET | Read a single object |
| `list` | GET | List objects |
| `watch` | GET (watch) | Stream changes |
| `create` | POST | Create a new object |
| `update` | PUT | Replace an object |
| `patch` | PATCH | Partial update |
| `delete` | DELETE | Delete an object |
| `deletecollection` | DELETE | Delete multiple objects |
| `exec` | POST | Execute command in container |
| `portforward` | POST | Port-forward to Pod |

---

### Real production use case

**CI/CD pipeline access**: The Jenkins ServiceAccount has a RoleBinding to a Role that can only create and delete Jobs in the `ci` namespace. Developers get a RoleBinding to `view` ClusterRole in their team's namespace (read-only). The on-call SRE group gets `edit` ClusterRole via ClusterRoleBinding for emergencies.

---

### How to debug RBAC

```bash
# Check if a subject can perform an action
kubectl auth can-i get pods --as=system:serviceaccount:production:my-app-sa -n production
kubectl auth can-i create deployments --as=alice@company.com -n production
kubectl auth can-i '*' '*' --as=system:serviceaccount:kube-system:default   # cluster-admin check

# See what roles/bindings exist
kubectl get roles,rolebindings -n production
kubectl get clusterroles,clusterrolebindings

# Check which roles a ServiceAccount has
kubectl get rolebindings -n production -o yaml | grep -A 5 my-app-sa

# Who can delete Secrets?
kubectl who-can delete secrets -n production   # requires kubectl-who-can plugin
```

---

## 15) NetworkPolicy

### What problem does NetworkPolicy solve?

By default, all Pods in a Kubernetes cluster can talk to all other Pods — no firewall. A NetworkPolicy defines **firewall rules at the Pod level**, controlling which Pods can send/receive traffic to/from which other Pods or external IPs.

> One-liner: NetworkPolicy is a **Pod-level firewall** rule.

NetworkPolicies are only enforced if a **CNI plugin that supports NetworkPolicy** is installed (Calico, Cilium, Weave). Flannel alone does not enforce NetworkPolicies.

---

### Anatomy of a NetworkPolicy YAML

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payments-isolation
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: payments-service           # applies to Pods with this label
  policyTypes:
    - Ingress
    - Egress

  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api-gateway        # only allow traffic from api-gateway Pods
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring   # allow from monitoring namespace
      ports:
        - protocol: TCP
          port: 8080

  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres           # only talk to postgres
      ports:
        - protocol: TCP
          port: 5432
    - to:                             # allow DNS resolution
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
```

### Default deny-all policy (start here)

```yaml
# Deny all ingress and egress for Pods in a namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}          # applies to ALL pods in namespace
  policyTypes:
    - Ingress
    - Egress
```

Apply this first, then add explicit allow policies. This is the recommended security posture.

---

### Real production use case

**PCI-DSS compliance**: The `payments` namespace has a default-deny NetworkPolicy. Explicit allow rules permit only the `checkout` service to reach the payments service. The payments service can only egress to the payment gateway IP range and the internal PostgreSQL. Compliance auditors get a YAML file showing exactly which traffic flows are permitted.

---

### How to debug NetworkPolicies

```bash
# List network policies
kubectl get networkpolicy -n production
kubectl describe networkpolicy payments-isolation -n production

# Test connectivity (from a debug Pod)
kubectl run test-pod --image=busybox --rm -it --restart=Never -- wget -qO- http://payments-service:8080/healthz

# Cilium specific: inspect policy for a Pod
kubectl exec -n kube-system cilium-<pod> -- cilium policy get

# Calico: trace policy enforcement
calicoctl policy
```

---

## 16) HorizontalPodAutoscaler (HPA)

### What problem does HPA solve?

Manual scaling is reactive and error-prone. HPA **automatically scales the number of Pod replicas** based on observed metrics (CPU, memory, custom metrics from Prometheus, etc.).

> One-liner: HPA **automatically scales replicas** in response to load.

---

### Anatomy of an HPA YAML

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-api
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60    # target 60% CPU utilization across all Pods
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
    - type: Pods                    # custom metric per Pod
      pods:
        metric:
          name: requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30    # react quickly to spikes
      policies:
        - type: Pods
          value: 4                       # add at most 4 Pods at a time
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300   # wait 5 min before scaling down (avoid flapping)
      policies:
        - type: Percent
          value: 10                      # remove at most 10% of Pods at a time
          periodSeconds: 60
```

### Prerequisites

HPA requires the **Metrics Server** to be installed for CPU/memory metrics:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

For custom metrics, use **KEDA** (Kubernetes Event-Driven Autoscaling) or the **Prometheus Adapter**.

---

### How HPA calculates desired replicas

```
desiredReplicas = ceil(currentReplicas × (currentMetricValue / targetMetricValue))

Example:
  currentReplicas = 3
  currentCPU = 90%
  targetCPU = 60%
  desiredReplicas = ceil(3 × (90/60)) = ceil(4.5) = 5
```

---

### Real production use case

**E-commerce flash sale**: Normal load → 3 replicas. Sale announced on social media → traffic spikes 10x in 2 minutes. HPA with `scaleUp.stabilizationWindowSeconds: 30` detects the spike within one evaluation cycle and scales to 15 replicas before most of the traffic arrives. After the sale, the slow scale-down policy prevents oscillation as traffic tapers off.

---

### How to debug HPA

```bash
# Check HPA status and current metrics
kubectl get hpa -n production
kubectl describe hpa my-api-hpa -n production

# Is Metrics Server running?
kubectl top pods -n production
kubectl top nodes

# Why isn't HPA scaling?
kubectl describe hpa my-api-hpa -n production | grep -A 5 "Conditions"
# Look for: AbleToScale, ScalingActive, ScalingLimited

# Watch HPA events
kubectl get events -n production --field-selector involvedObject.kind=HorizontalPodAutoscaler
```

**Common HPA issues**

| Symptom | Cause | Fix |
|---|---|---|
| `unknown` metrics | Metrics Server not installed or not ready | Install Metrics Server |
| HPA not scaling up | `maxReplicas` hit or Pod doesn't have `requests` set | Increase `maxReplicas`; set CPU `requests` |
| HPA flapping (scales up/down rapidly) | `stabilizationWindowSeconds` too low | Increase scaleDown stabilization window |
| Custom metrics not found | Prometheus Adapter / KEDA not configured | Check adapter configuration |

---

## 17) VerticalPodAutoscaler (VPA)

### What problem does VPA solve?

Setting the right CPU and memory `requests`/`limits` is hard. Too low → OOMKilled or throttled. Too high → wasted resources. VPA **automatically recommends and optionally sets** the right resource values based on actual usage history.

> One-liner: VPA **auto-tunes Pod resource requests and limits** based on observed usage.

---

### Anatomy of a VPA YAML

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-api-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-api
  updatePolicy:
    updateMode: "Off"     # Off (recommend only) | Initial | Auto
  resourcePolicy:
    containerPolicies:
      - containerName: api
        minAllowed:
          cpu: "100m"
          memory: "128Mi"
        maxAllowed:
          cpu: "4"
          memory: "8Gi"
        controlledResources: ["cpu", "memory"]
```

### updateMode options

| Mode | Behavior |
|---|---|
| `Off` | Only compute recommendations; never applies them. Use to learn. |
| `Initial` | Set resources only on Pod creation; never update running Pods. |
| `Recreate` | Evict and recreate Pods when recommendations change significantly. |
| `Auto` | Evict and recreate when better values found (same as Recreate currently). |

**VPA in `Auto` mode evicts Pods to apply new values — use with PodDisruptionBudget.**

---

### VPA + HPA caveat

Do **not** use VPA and HPA on the same Deployment for CPU/memory — they conflict. Use VPA for right-sizing; use HPA for scaling. Or use KEDA + VPA with HPA on custom metrics.

---

### Real production use case

**Right-sizing before cost optimization**: Run VPA in `Off` mode for 2 weeks on all production Deployments. Review `kubectl describe vpa` recommendations. Apply the suggested values in YAML. Result: 30% reduction in cluster resource requests without OOMKills.

---

### How to debug VPA

```bash
# View VPA recommendations
kubectl describe vpa my-api-vpa -n production
# Look for: Status.Recommendation.ContainerRecommendations
#           Target, LowerBound, UpperBound, UncappedTarget

# Check VPA components are running
kubectl get pods -n kube-system | grep vpa
```

---

## 18) PodDisruptionBudget (PDB)

### What problem does a PDB solve?

Cluster operations (node drain for upgrades, scale-down) can **voluntarily evict Pods**. Without a PDB, all Pods of a Deployment could be evicted simultaneously, causing downtime. A PDB sets a minimum availability guarantee during voluntary disruptions.

> One-liner: PDB guarantees a **minimum number of Pods stay running** during voluntary disruptions.

---

### Anatomy of a PDB YAML

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-api-pdb
  namespace: production
spec:
  minAvailable: 2          # at least 2 Pods must be running at all times
  # OR:
  # maxUnavailable: 1      # at most 1 Pod can be unavailable at a time
  selector:
    matchLabels:
      app: my-api
```

Use `minAvailable` with an absolute number or percentage (`minAvailable: "80%"`).

---

### What counts as a voluntary disruption?

- `kubectl drain node` (node maintenance, upgrades)
- Cluster autoscaler scale-down
- VPA evicting Pods to update resources
- Manual `kubectl delete pod`

PDB does **not** protect against involuntary disruptions (hardware failure, OOMKill).

---

### Real production use case

**Node upgrade without downtime**: The platform team upgrades node AMIs by draining nodes one at a time. PDB on each service ensures at least `minAvailable: 2` Pods are running. When a drain would violate the PDB, it waits until another node comes up with a replacement Pod before proceeding. Zero service disruption.

---

### How to debug PDBs

```bash
kubectl get pdb -n production
kubectl describe pdb my-api-pdb -n production

# Why is a drain blocked?
kubectl drain <node> --dry-run
# Check: "Cannot evict pod as it would violate the pod's disruption budget"
```

---

## 19) LimitRange

### What problem does LimitRange solve?

If developers don't set resource `requests`/`limits`, Pods get scheduled without any guarantees and can starve other workloads. A LimitRange **enforces default and maximum resource constraints** at the namespace level.

> One-liner: LimitRange **auto-injects and caps resource requests/limits** for all Pods in a namespace.

---

### Anatomy of a LimitRange YAML

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:                  # injected if container doesn't set limits
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:           # injected if container doesn't set requests
        cpu: "100m"
        memory: "128Mi"
      max:                      # hard ceiling per container
        cpu: "4"
        memory: "8Gi"
      min:                      # minimum per container
        cpu: "50m"
        memory: "64Mi"
    - type: Pod
      max:
        cpu: "8"
        memory: "16Gi"
    - type: PersistentVolumeClaim
      max:
        storage: 100Gi
      min:
        storage: 1Gi
```

---

### Real production use case

**Multi-tenant namespace security**: Each team namespace has a LimitRange. A rogue or misconfigured Pod cannot consume more than 4 CPU or 8 GiB memory. New developers who forget to set resource requests get sensible defaults automatically.

---

## 20) ResourceQuota

### What problem does ResourceQuota solve?

LimitRange caps individual containers. ResourceQuota caps **total resource consumption across the entire namespace**.

> One-liner: ResourceQuota is a **namespace-level budget** for total compute, storage, and object count.

---

### Anatomy of a ResourceQuota YAML

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    # Compute
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"

    # Storage
    requests.storage: "500Gi"
    persistentvolumeclaims: "20"

    # Object counts
    pods: "50"
    services: "20"
    secrets: "50"
    configmaps: "50"
    services.loadbalancers: "2"
    services.nodeports: "0"          # prevent NodePort Services in this namespace

    # Scoped quotas (only count Pods in BestEffort QoS class)
    # count/pods: "10"
```

---

### QoS classes (determined by resource settings)

| QoS Class | Condition | Behavior |
|---|---|---|
| `Guaranteed` | `requests == limits` for all containers | Last to be evicted under pressure |
| `Burstable` | `requests < limits` or only requests set | Evicted before Guaranteed |
| `BestEffort` | No requests or limits | First to be evicted |

---

### Real production use case

**Cost management across teams**: Each team namespace has a ResourceQuota matching their monthly budget. The platform team reviews quota utilization reports weekly. Teams that need more capacity raise a request for quota increase rather than accidentally consuming the whole cluster.

---

## 21) PersistentVolume (PV) & PersistentVolumeClaim (PVC)

Volumes and storage are covered in detail in `kubernetes-volumes-storage-deep-dive.md`. Here's a quick reference.

### PersistentVolume (PV) — the storage resource

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: fast-pv-001
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce           # can be mounted as read-write by one node
  persistentVolumeReclaimPolicy: Retain   # Retain | Delete | Recycle
  storageClassName: fast-ssd
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-0abc123def456
```

### PersistentVolumeClaim (PVC) — the request for storage

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-data
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: fast-ssd
```

### Mounting a PVC in a Pod

```yaml
spec:
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: my-data
  containers:
    - name: app
      volumeMounts:
        - name: data
          mountPath: /data
```

### Access modes

| Mode | Short | Meaning |
|---|---|---|
| `ReadWriteOnce` | RWO | Read/write by one node at a time |
| `ReadOnlyMany` | ROX | Read-only by many nodes simultaneously |
| `ReadWriteMany` | RWX | Read/write by many nodes simultaneously (NFS, CephFS) |
| `ReadWriteOncePod` | RWOP | Read/write by one Pod only (K8s 1.22+) |

### How to debug PVCs

```bash
# Check PVC binding status
kubectl get pvc -n production

# PVC stuck in Pending?
kubectl describe pvc my-data -n production
# Check events: "no persistent volumes available for this claim"

# Check available PVs
kubectl get pv

# Check StorageClass
kubectl get storageclass
```

---

## 22) StorageClass

### What problem does StorageClass solve?

Without dynamic provisioning, a cluster admin must manually create PVs before developers can use them. A StorageClass defines a **provisioner and parameters** that allow PVCs to dynamically create their own storage.

> One-liner: StorageClass is a **template for on-demand storage provisioning**.

---

### Anatomy of a StorageClass YAML

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"   # make this the default
provisioner: ebs.csi.aws.com          # the CSI driver that creates volumes
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  kmsKeyId: arn:aws:kms:us-east-1:123456789:key/...
reclaimPolicy: Delete                  # Delete | Retain
allowVolumeExpansion: true             # allow `kubectl patch pvc` to resize
volumeBindingMode: WaitForFirstConsumer   # wait for Pod scheduling before provisioning (zone-aware)
```

### volumeBindingMode options

| Mode | Behavior |
|---|---|
| `Immediate` | Provision volume as soon as PVC is created; may create in wrong zone |
| `WaitForFirstConsumer` | Wait until a Pod is scheduled; provision in the same zone as the Pod |

Always use `WaitForFirstConsumer` in multi-zone clusters to avoid zone affinity issues.

---

## 23) EndpointSlice

### What problem does EndpointSlice solve?

The original `Endpoints` object stored all Pod IPs for a Service in a single object. At 1000+ Pods, any Pod change caused the entire multi-megabyte Endpoints object to be re-synced to every node — O(n²) problem. EndpointSlice shards this into groups of 100 Pods max.

> One-liner: EndpointSlice is the **scalable replacement for Endpoints**, auto-managed by Kubernetes.

---

### Viewing EndpointSlices

```bash
kubectl get endpointslices -n production
kubectl get endpointslices -n production -l kubernetes.io/service-name=my-api
kubectl describe endpointslice my-api-abc12 -n production
```

You rarely create EndpointSlices manually. They're auto-managed. The only time you interact with them directly is debugging Service connectivity.

---

### Manual EndpointSlice (for external services)

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: external-db
  namespace: production
  labels:
    kubernetes.io/service-name: external-db   # ties to a Service with same name
addressType: IPv4
endpoints:
  - addresses:
      - "10.0.1.50"           # external database IP
    conditions:
      ready: true
ports:
  - name: postgres
    protocol: TCP
    port: 5432
```

This allows a Service to abstract external resources with a stable Kubernetes DNS name.

---

## 24) CustomResourceDefinition (CRD)

### What problem does CRD solve?

Kubernetes' core objects (Pod, Service, Deployment) cover general workloads. CRDs allow you to **define your own resource types** that the Kubernetes API serves natively — extending Kubernetes for your domain (databases, ML pipelines, certificates, etc.).

> One-liner: CRD is a **schema registration** that adds a new object type to the Kubernetes API.

---

### Anatomy of a CRD YAML

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.myorg.com          # must be <plural>.<group>
spec:
  group: myorg.com
  versions:
    - name: v1alpha1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                engine:
                  type: string
                  enum: ["postgres", "mysql"]
                version:
                  type: string
                storageGb:
                  type: integer
                  minimum: 10
            status:
              type: object
              properties:
                phase:
                  type: string
                connectionString:
                  type: string
      subresources:
        status: {}
  scope: Namespaced
  names:
    plural: databases
    singular: database
    kind: Database
    shortNames: ["db"]
```

### Creating a custom resource (CR) after registering the CRD

```yaml
apiVersion: myorg.com/v1alpha1
kind: Database
metadata:
  name: orders-db
  namespace: production
spec:
  engine: postgres
  version: "15"
  storageGb: 100
```

### The Operator pattern

A CRD alone is just a schema. An **Operator** (a custom controller) watches CRs and acts on them:

```
You: kubectl apply -f database.yaml
API Server: stores Database CR
Operator controller: watches for Database CRs
  → provisions actual RDS instance or deploys a StatefulSet
  → updates Database CR's status.phase = "Running"
  → writes status.connectionString = "postgres://..."
```

Popular Operators: cert-manager, Prometheus Operator, Strimzi (Kafka), Crossplane (cloud infra), Argo CD.

---

### Real production use case

**Internal developer platform**: Platform team defines a `Database`, `Cache`, and `Queue` CRD. App teams request resources by creating simple CRs. The Operator provisions the actual cloud resources (RDS, ElastiCache, SQS) and writes connection strings back to Secrets. App teams never deal with Terraform or AWS console.

---

### How to debug CRDs

```bash
# List all CRDs in the cluster
kubectl get crd

# Check a specific CR
kubectl get database -n production
kubectl describe database orders-db -n production

# Check Operator logs
kubectl logs -n operators -l app=database-operator -f

# Validate a CR schema issue
kubectl apply -f database.yaml --dry-run=server
```

---

## 25) PriorityClass

### What problem does PriorityClass solve?

When nodes are under resource pressure, the Scheduler needs to **preempt** (evict) lower-priority Pods to make room for higher-priority ones. PriorityClass assigns a numeric priority to Pods.

> One-liner: PriorityClass determines **which Pods get preempted first** under resource pressure.

---

### Anatomy of a PriorityClass YAML

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000           # higher value = higher priority
globalDefault: false
preemptionPolicy: PreemptLowerPriority  # or Never
description: "For production critical workloads"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
globalDefault: false
description: "For batch jobs that can be preempted"
```

### Using PriorityClass in a Pod

```yaml
spec:
  priorityClassName: high-priority
```

### Built-in system priority classes

| Class | Value | Used by |
|---|---|---|
| `system-cluster-critical` | 2000000000 | Kubernetes core components (kube-dns) |
| `system-node-critical` | 2000001000 | Node-level components (kubelet static pods) |

---

### Real production use case

**Batch vs production workloads on the same cluster**: ML training jobs use `low-priority`. The API serving the end users uses `high-priority`. When a burst of training jobs fills the cluster, the Scheduler refuses to schedule them if it would require preempting a production API Pod. Training jobs queue until capacity is available.

---

### How to debug PriorityClass

```bash
kubectl get priorityclass
kubectl describe priorityclass high-priority

# Check preemption events
kubectl get events -A | grep Preempted
```

---

## 26) Quick Decision Guide

### Which workload object to use?

```
Is your app stateless (API, web server, worker)?
  → Deployment

Is your app stateful with per-Pod identity (DB, queue, cluster)?
  → StatefulSet

Do you need one agent per node (log shipper, monitoring)?
  → DaemonSet

Do you need a task to run once and complete?
  → Job

Do you need a task on a recurring schedule?
  → CronJob
```

### Which networking object to use?

```
Internal service-to-service traffic within the cluster?
  → Service (ClusterIP)

Expose HTTP(S) to the internet with host/path-based routing?
  → Ingress (single LB for all services)

Expose TCP/UDP or a single service to the internet?
  → Service (LoadBalancer)

Lock down which Pods can talk to which Pods?
  → NetworkPolicy
```

### Which configuration/secret object to use?

```
Non-sensitive config (env vars, config files)?
  → ConfigMap

Sensitive data (passwords, certs, tokens)?
  → Secret (with External Secrets Operator for production)

Git-safe encrypted secrets?
  → Sealed Secrets (Bitnami) or SOPS
```

### Which scaling object to use?

```
Scale replicas based on CPU/memory/custom metrics?
  → HPA

Right-size CPU/memory requests for a Pod?
  → VPA (in Off mode first, then Auto)

Scale based on queue depth, cron, or external events?
  → KEDA

Protect services during node maintenance?
  → PodDisruptionBudget
```

### Which resource control object to use?

```
Default and max resources per container in a namespace?
  → LimitRange

Total CPU/memory/object budget for a namespace?
  → ResourceQuota

Priority for preemption under pressure?
  → PriorityClass
```

---

## Object quick-reference summary

| Object | API Group | Scope | Key purpose |
|---|---|---|---|
| Pod | core | Namespaced | Smallest deployable unit |
| ReplicaSet | apps | Namespaced | Ensure N replicas running |
| Deployment | apps | Namespaced | Rolling updates + rollback for stateless apps |
| StatefulSet | apps | Namespaced | Ordered, identity-stable replicas for stateful apps |
| DaemonSet | apps | Namespaced | Exactly one Pod per node |
| Job | batch | Namespaced | Run to successful completion |
| CronJob | batch | Namespaced | Scheduled Jobs |
| Service | core | Namespaced | Stable network endpoint for Pods |
| Ingress | networking.k8s.io | Namespaced | HTTP/HTTPS routing rules |
| ConfigMap | core | Namespaced | Non-sensitive key-value config |
| Secret | core | Namespaced | Sensitive key-value data |
| Namespace | core | Cluster | Virtual cluster partitioning |
| ServiceAccount | core | Namespaced | Machine identity for Pods |
| Role | rbac.authorization.k8s.io | Namespaced | Permissions within a namespace |
| ClusterRole | rbac.authorization.k8s.io | Cluster | Cluster-wide permissions |
| RoleBinding | rbac.authorization.k8s.io | Namespaced | Bind role to subject in namespace |
| ClusterRoleBinding | rbac.authorization.k8s.io | Cluster | Bind cluster role to subject cluster-wide |
| NetworkPolicy | networking.k8s.io | Namespaced | Pod-level firewall rules |
| HPA | autoscaling | Namespaced | Auto-scale replica count |
| VPA | autoscaling.k8s.io | Namespaced | Auto-tune resource requests |
| PodDisruptionBudget | policy | Namespaced | Min availability during voluntary disruptions |
| LimitRange | core | Namespaced | Default + max resources per container |
| ResourceQuota | core | Namespaced | Total namespace resource budget |
| PersistentVolume | core | Cluster | Storage resource |
| PersistentVolumeClaim | core | Namespaced | Request for storage |
| StorageClass | storage.k8s.io | Cluster | Dynamic storage provisioning template |
| EndpointSlice | discovery.k8s.io | Namespaced | Scalable Pod IP tracking for Services |
| CustomResourceDefinition | apiextensions.k8s.io | Cluster | Register custom API types |
| PriorityClass | scheduling.k8s.io | Cluster | Preemption priority for Pods |
