# How Volumes and Storage Work in Kubernetes — Deep Dive

---

## 1) What problem does Kubernetes storage solve?

Containers are **ephemeral by design**. When a container crashes and restarts, its filesystem is wiped clean. When a Pod is deleted, everything written inside the container is gone.

This is fine for stateless apps (APIs, web servers), but breaks anything that needs to **persist data**:
- Databases (MySQL, PostgreSQL, MongoDB)
- Message queues (Kafka, RabbitMQ)
- File uploads
- ML model checkpoints

Kubernetes solves this through a layered storage system:

```
Ephemeral storage (dies with Pod)  →  Volumes
Volumes tied to a Pod lifecycle    →  PersistentVolumes (PV)
Requesting storage declaratively   →  PersistentVolumeClaims (PVC)
Automating storage provisioning    →  StorageClass
```

---

## 2) Volumes vs PersistentVolumes — key distinction

| | Volume | PersistentVolume (PV) |
|---|---|---|
| Lifecycle | Tied to the Pod | Independent of Pods |
| Survives Pod restart | Yes | Yes |
| Survives Pod deletion | No (most types) | Yes |
| Defined in | Pod spec directly | Separate cluster resource |
| Who manages it | Developer | Cluster admin / dynamic provisioner |

---

## 3) Ephemeral volumes (die with the Pod)

These are defined directly in the Pod spec under `volumes` and do not outlive the Pod.

### emptyDir — scratch space shared between containers

```yaml
spec:
  volumes:
    - name: shared-data
      emptyDir: {}          # created when Pod starts, deleted when Pod is removed
  containers:
    - name: app
      volumeMounts:
        - name: shared-data
          mountPath: /data
    - name: sidecar-logger
      volumeMounts:
        - name: shared-data
          mountPath: /data     # both containers read/write the same directory
```

- Created fresh when the Pod is scheduled on a node.
- Shared between all containers in the same Pod.
- **Survives container crashes** (only deleted when the Pod itself is removed).
- Use cases: cache, inter-container communication, temp files during processing.
- `emptyDir.medium: Memory` stores it in RAM (tmpfs) — faster but counts against memory limits.

---

### configMap and secret volumes — inject config as files

```yaml
spec:
  volumes:
    - name: app-config
      configMap:
        name: my-app-config      # name of the ConfigMap object
    - name: tls-certs
      secret:
        secretName: my-tls-secret
  containers:
    - name: app
      volumeMounts:
        - name: app-config
          mountPath: /etc/config   # each ConfigMap key becomes a file here
        - name: tls-certs
          mountPath: /etc/tls
          readOnly: true
```

- Keys in the ConfigMap/Secret become filenames; values become file contents.
- Kubernetes **automatically updates** the mounted files when the ConfigMap/Secret changes (within ~1 minute) — no Pod restart needed.
- Contrast with env vars from ConfigMaps: those are **not** updated unless the Pod restarts.

---

### projected — combine multiple sources into one directory

```yaml
spec:
  volumes:
    - name: combined
      projected:
        sources:
          - configMap:
              name: my-config
          - secret:
              name: my-secret
          - serviceAccountToken:
              path: token
              expirationSeconds: 3600
```

Merges ConfigMap keys, Secret keys, and a service account token all into a single mount path. Common for workload identity setups (e.g., AWS IRSA, GCP Workload Identity).

---

## 4) PersistentVolumes (PV) — storage as a cluster resource

A **PersistentVolume** is a piece of storage that exists independently of any Pod. It is a cluster-level resource, like a Node.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-database
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteOnce           # see Section 6
  persistentVolumeReclaimPolicy: Retain   # see Section 7
  storageClassName: standard
  hostPath:                   # for local/dev clusters only
    path: /mnt/data
```

In production you would use a real backend instead of `hostPath`:
```yaml
  awsElasticBlockStore:
    volumeID: vol-0a1b2c3d
    fsType: ext4
# or
  gcePersistentDisk:
    pdName: my-disk
    fsType: ext4
# or
  nfs:
    server: nfs-server.example.com
    path: /exports/data
```

---

## 5) PersistentVolumeClaims (PVC) — requesting storage

A **PersistentVolumeClaim** is a request for storage made by a user/developer. It's the bridge between a Pod and a PV.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: standard
```

### How PVC binding works

```
PVC created
    │
    ▼
Kubernetes finds a PV that:
  - has capacity >= requested
  - has matching accessModes
  - has matching storageClassName
  - is Available (not already bound)
    │
    ▼
PV and PVC are bound to each other (1:1 relationship)
    │
    ▼
Pod references the PVC by name
    │
    ▼
Kubelet mounts the PV's actual storage into the Pod's container
```

### Using a PVC in a Pod

```yaml
spec:
  volumes:
    - name: mysql-storage
      persistentVolumeClaim:
        claimName: mysql-data    # reference the PVC name
  containers:
    - name: mysql
      image: mysql:8.0
      volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
```

The Pod doesn't care whether the underlying storage is AWS EBS, GCP PD, NFS, or anything else. It just uses the PVC name. The actual storage backend is abstracted away.

---

## 6) Access modes

Access modes define how many nodes can mount the volume simultaneously.

| Mode | Short | Meaning |
|---|---|---|
| `ReadWriteOnce` | RWO | Mounted as read-write by **one node** at a time (multiple Pods on the same node can share it) |
| `ReadOnlyMany` | ROX | Mounted as read-only by **many nodes** simultaneously |
| `ReadWriteMany` | RWX | Mounted as read-write by **many nodes** simultaneously |
| `ReadWriteOncePod` | RWOP | Mounted as read-write by **exactly one Pod** (stricter than RWO, added in K8s 1.22) |

> Important: A volume's `accessModes` list describes what the storage *supports*. A PVC requests one mode from that list. The binding only works if the modes are compatible.

### Which backends support which modes

| Storage | RWO | ROX | RWX |
|---|---|---|---|
| AWS EBS | Yes | No | No |
| GCP Persistent Disk | Yes | Yes | No |
| Azure Disk | Yes | No | No |
| NFS | Yes | Yes | Yes |
| CephFS / GlusterFS | Yes | Yes | Yes |
| Local (hostPath) | Yes | No | No |

AWS EBS only supports RWO — this matters if you try to run multiple replicas of a database with a shared volume; it won't work with EBS.

---

## 7) Reclaim policies

When a PVC is deleted, what happens to the PV and its data?

| Policy | Behavior |
|---|---|
| `Retain` | PV and data preserved; admin must manually clean up and re-use |
| `Delete` | PV and the underlying storage (e.g., EBS volume) are **automatically deleted** |
| `Recycle` (deprecated) | Data scrubbed (`rm -rf /`), PV made available again |

```yaml
spec:
  persistentVolumeReclaimPolicy: Retain  # safe default for production databases
```

> Use `Retain` for databases and anything critical. Use `Delete` for ephemeral dev/test storage where auto-cleanup is desirable. `Recycle` is removed in modern Kubernetes.

---

## 8) StorageClass — dynamic provisioning

Manually creating PVs for every PVC is tedious at scale. **StorageClasses** allow Kubernetes to automatically provision storage on demand.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com      # the CSI driver that creates the volume
parameters:
  type: gp3                        # EBS volume type
  iops: "3000"
  throughput: "125"
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true         # allows PVC resize after creation
volumeBindingMode: WaitForFirstConsumer  # don't provision until a Pod needs it
```

### Dynamic provisioning flow

```
PVC created with storageClassName: fast-ssd
        │
        ▼
No matching PV exists yet
        │
        ▼
StorageClass provisioner (EBS CSI driver) is triggered
        │
        ▼
CSI driver calls AWS API → creates EBS volume
        │
        ▼
Kubernetes creates PV object representing that EBS volume
        │
        ▼
PV is bound to PVC automatically
        │
        ▼
Pod mounts PVC → data written to EBS
```

Without a StorageClass, you'd manually create the EBS volume in AWS, then manually create the PV YAML. With a StorageClass, the entire chain is automatic.

### Default StorageClass

```bash
kubectl get storageclass
# NAME              PROVISIONER            RECLAIMPOLICY  VOLUMEBINDINGMODE
# standard (default) kubernetes.io/gce-pd  Delete         Immediate
# fast-ssd           ebs.csi.aws.com       Delete         WaitForFirstConsumer
```

If a PVC doesn't specify a `storageClassName`, Kubernetes uses the one marked as default. If there is no default, the PVC stays Pending.

---

## 9) Volume binding modes

| Mode | When the PV is provisioned |
|---|---|
| `Immediate` | As soon as the PVC is created |
| `WaitForFirstConsumer` | When a Pod that uses the PVC is scheduled |

`WaitForFirstConsumer` is strongly recommended for zone-aware storage like AWS EBS. Without it:
- PVC is created → EBS volume provisioned in zone `us-east-1a`
- Pod is scheduled by Kubernetes to zone `us-east-1b`
- Pod cannot mount the volume → stuck Pending forever

With `WaitForFirstConsumer`, the Scheduler picks a zone for the Pod first, then the EBS volume is created in that same zone.

---

## 10) Volume expansion (resizing a PVC)

If the StorageClass has `allowVolumeExpansion: true`:

```bash
kubectl edit pvc mysql-data
# change spec.resources.requests.storage from 20Gi to 50Gi
```

Kubernetes handles the resize:
1. Calls the CSI driver to expand the underlying volume (e.g., AWS EBS resize API).
2. If the filesystem needs to expand too, it happens when the Pod restarts (for some drivers, it's online — no restart needed).

> You can only **increase** a PVC's size, never decrease it.

---

## 11) StatefulSets and volumeClaimTemplates

A regular Deployment gives all replicas the same PVC (which would be a problem for databases since they need separate storage per replica). **StatefulSets** solve this with `volumeClaimTemplates`.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  replicas: 3
  serviceName: mysql
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
  volumeClaimTemplates:           # creates one PVC per replica automatically
    - metadata:
        name: mysql-data
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 50Gi
```

This creates:
- `mysql-data-mysql-0` PVC → bound to Pod `mysql-0`
- `mysql-data-mysql-1` PVC → bound to Pod `mysql-1`
- `mysql-data-mysql-2` PVC → bound to Pod `mysql-2`

Each Pod gets its own dedicated PVC. When a Pod is deleted and recreated (e.g., node failure), it **reconnects to the same PVC** — its data is preserved.

> PVCs created by volumeClaimTemplates are **NOT deleted** when the StatefulSet is deleted. This is intentional to protect data. You must delete them manually.

---

## 12) CSI — Container Storage Interface

The **CSI (Container Storage Interface)** is the standard plugin system that lets any storage vendor integrate with Kubernetes without modifying the Kubernetes core.

```
Kubernetes (PVC request)
        │
        ▼
  CSI Driver (vendor-provided)
  ├── Controller Plugin  ← talks to cloud API (create/delete/resize volumes)
  └── Node Plugin        ← runs on each node (attach/mount/unmount volumes)
        │
        ▼
  Underlying Storage (AWS EBS, GCP PD, Ceph, NetApp, etc.)
```

### Common CSI drivers

| Cloud/Storage | CSI Driver |
|---|---|
| AWS EBS | `ebs.csi.aws.com` |
| AWS EFS (NFS) | `efs.csi.aws.com` |
| GCP Persistent Disk | `pd.csi.storage.gke.io` |
| Azure Disk | `disk.csi.azure.com` |
| Azure File | `file.csi.azure.com` |
| Ceph RBD | `rbd.csi.ceph.com` |
| CephFS | `cephfs.csi.ceph.com` |
| Local path (dev) | `rancher.io/local-path` |

Before CSI (old in-tree plugins): storage code was compiled directly into Kubernetes. Adding a new storage backend required a Kubernetes release. CSI decouples storage completely.

---

## 13) Real-world examples

### Example A — PostgreSQL database with dynamic EBS provisioning

```yaml
# StorageClass (usually set up once by the platform team)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
# StatefulSet for PostgreSQL
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  replicas: 1
  serviceName: postgres
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
          image: postgres:16
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 100Gi
```

Result:
- EBS volume auto-provisioned in same AZ as the Pod.
- Data persists across Pod restarts and node failures.
- `reclaimPolicy: Retain` means data survives even if the PVC is accidentally deleted.

---

### Example B — Shared configuration with ConfigMap volume + live reload

```yaml
# ConfigMap with nginx config
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    server {
      listen 80;
      location / { proxy_pass http://backend:8080; }
    }
---
# Deployment that mounts it
spec:
  template:
    spec:
      volumes:
        - name: nginx-cfg
          configMap:
            name: nginx-config
      containers:
        - name: nginx
          image: nginx:1.25
          volumeMounts:
            - name: nginx-cfg
              mountPath: /etc/nginx/conf.d
```

When you update the ConfigMap, the file at `/etc/nginx/conf.d/nginx.conf` is updated inside the running Pod within ~60 seconds — without restarting the Pod. Nginx can then reload its config via a sidecar or inotify watcher.

---

### Example C — Sidecar log collector using emptyDir

```yaml
spec:
  volumes:
    - name: log-volume
      emptyDir: {}
  containers:
    - name: app
      image: my-org/my-api:v2
      volumeMounts:
        - name: log-volume
          mountPath: /var/log/app      # app writes logs here
    - name: log-shipper
      image: fluent/fluent-bit:latest
      volumeMounts:
        - name: log-volume
          mountPath: /var/log/app      # shipper reads logs from same path
          readOnly: true
```

Both containers share the same `emptyDir`. The app writes logs; the sidecar ships them to a log aggregator (e.g., Elasticsearch). No network socket between containers needed — just a shared filesystem.

---

## 14) Volume lifecycle — full picture

```
StorageClass exists
        │
        ▼
PVC created → triggers dynamic provisioner
        │
        ▼
PV created and bound to PVC (status: Bound)
        │
        ▼
Pod references PVC → Kubelet mounts volume into container
        │
        ▼
Pod runs, data written to volume
        │
        ▼
Pod deleted → volume unmounted from node
        │
        ▼
PVC deleted →
  ├── reclaimPolicy: Retain → PV status: Released, data safe, manual cleanup
  └── reclaimPolicy: Delete → PV deleted, underlying storage (EBS/PD) deleted
```

---

## 15) Common pitfalls

| Pitfall | What goes wrong | Fix |
|---|---|---|
| Using `Immediate` binding mode with zone-locked storage (EBS) | Volume created in wrong zone → Pod stuck Pending | Use `WaitForFirstConsumer` on StorageClass |
| Using Deployment instead of StatefulSet for databases | All replicas share one PVC → data corruption | Use StatefulSet with `volumeClaimTemplates` |
| `reclaimPolicy: Delete` on production databases | Deleting PVC accidentally wipes all data | Use `Retain` for stateful workloads |
| No `accessModes` match between PVC and PV | PVC stays in Pending state indefinitely | Match `accessModes` exactly; check storage backend's supported modes |
| Mounting a `Secret` as env var instead of volume | Env vars are not updated when Secret changes; also exposed in `kubectl describe` | Use volume mounts for Secrets that rotate |
| Forgetting that PVCs from StatefulSets are not auto-deleted | Orphaned PVCs and EBS volumes pile up, incurring cost | Manually delete PVCs after deleting a StatefulSet |
| Requesting RWX on EBS | EBS doesn't support ReadWriteMany → PVC stays Pending | Use EFS (NFS-backed) for shared multi-node storage on AWS |

---

## 16) Quick mental model for revision

```
Container filesystem  →  ephemeral, wiped on restart
emptyDir             →  ephemeral, shared within Pod, wiped on Pod delete
ConfigMap/Secret vol →  ephemeral, injected config, live-updated
PersistentVolume     →  durable, cluster resource, independent of Pods
PersistentVolumeClaim→  request for a PV (developer-facing)
StorageClass         →  auto-provisions PVs on demand (dynamic provisioning)
StatefulSet + VCT    →  one PVC per replica, PVC survives Pod lifecycle
CSI Driver           →  plugin that connects Kubernetes to real storage backends
```

Key facts to remember:
- `requests` on resources = scheduling currency; `requests` on PVC = storage ask.
- PVC and PV are bound 1:1; one PVC cannot share a PV with another PVC.
- `accessModes` describe per-node concurrency, not per-Pod.
- `WaitForFirstConsumer` is the correct binding mode for zone-aware block storage.
- StatefulSet PVCs must be manually deleted — Kubernetes deliberately protects them.
- CSI drivers decoupled storage backends from Kubernetes core since 1.13 (stable).
