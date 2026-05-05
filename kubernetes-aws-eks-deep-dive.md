# How AWS EKS Works — Deep Dive

---

## 1) What problem does EKS solve?

Running Kubernetes yourself means you are responsible for:
- Installing, upgrading, and patching the control plane (API server, etcd, controller manager, scheduler).
- Making etcd highly available across multiple availability zones.
- Securing the control plane (certificates, RBAC, audit logs).
- Backing up etcd and recovering from control plane failures.
- Upgrading Kubernetes versions without downtime.

This is significant operational work before you have even run a single workload.

**Amazon EKS (Elastic Kubernetes Service)** is a managed Kubernetes service that removes all of that. AWS operates the control plane for you — it is highly available, automatically backed up, and covered under the AWS SLA. You only manage the worker nodes (and even those can be managed, with Fargate).

> One-liner: EKS gives you a **fully managed, HA Kubernetes control plane** on AWS so you only think about running workloads, not operating Kubernetes itself.

---

## 2) EKS architecture overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        AWS MANAGED                              │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                EKS Control Plane VPC                    │   │
│   │  ┌──────────────┐   ┌────────┐   ┌──────────────────┐  │   │
│   │  │ kube-apiserver│   │  etcd  │   │ kube-controller- │  │   │
│   │  │ (multi-AZ)   │   │(multi- │   │ manager +        │  │   │
│   │  │              │   │  AZ)   │   │ kube-scheduler   │  │   │
│   │  └──────────────┘   └────────┘   └──────────────────┘  │   │
│   └─────────────────────────────────────────────────────────┘   │
│                             │                                   │
│                 AWS PrivateLink / ENI                           │
└─────────────────────────────┼───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│                     YOUR VPC (Customer)                         │
│                                                                 │
│  AZ us-east-1a          AZ us-east-1b          AZ us-east-1c   │
│  ┌──────────────┐       ┌──────────────┐       ┌─────────────┐ │
│  │ Worker Node  │       │ Worker Node  │       │ Worker Node │ │
│  │ (EC2)        │       │ (EC2)        │       │ (EC2)       │ │
│  │ kubelet      │       │ kubelet      │       │ kubelet     │ │
│  │ kube-proxy   │       │ kube-proxy   │       │ kube-proxy  │ │
│  └──────────────┘       └──────────────┘       └─────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

Key separation: the **control plane runs in an AWS-owned VPC**; your worker nodes run in **your VPC**. They communicate over a cross-account ENI (Elastic Network Interface) that AWS provisions automatically.

---

## 3) EKS control plane — what AWS manages

When you create an EKS cluster, AWS:
- Runs **two or more API server instances** spread across at least two AZs.
- Runs **three etcd members** across three AZs (quorum-safe: tolerates 1 AZ failure).
- Automatically rotates certificates before they expire.
- Automatically backs up etcd.
- Patches and upgrades the control plane OS.
- Integrates with AWS CloudWatch for control plane logs.
- Provides the Kubernetes API behind an AWS-managed NLB endpoint.

You cannot SSH into the control plane nodes. You interact with them only through:
- `kubectl` (via the API server endpoint)
- AWS Console / CLI / CloudFormation / Terraform (for cluster-level settings)

```bash
# The API server endpoint looks like:
https://ABCDEF1234567890.gr7.us-east-1.eks.amazonaws.com

# Your kubeconfig uses this endpoint
aws eks update-kubeconfig --region us-east-1 --name my-cluster
```

---

## 4) Node groups — managing worker nodes

EKS gives you three ways to run worker nodes:

### Option 1 — Self-managed node groups

You manage the EC2 instances yourself using an Auto Scaling Group. You are responsible for:
- Choosing the AMI (must use the Amazon EKS-optimized AMI or configure containerd/kubelet yourself).
- Upgrading the AMI when a new Kubernetes version comes out.
- Draining nodes before terminating them.

Rarely used today unless you need highly customized OS configuration.

### Option 2 — Managed node groups (most common)

AWS manages the lifecycle of the EC2 instances inside an Auto Scaling Group.

```bash
# Create a managed node group
aws eks create-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name my-nodes \
  --node-role arn:aws:iam::123456789:role/eks-node-role \
  --subnets subnet-aaa subnet-bbb subnet-ccc \
  --instance-types t3.medium \
  --scaling-config minSize=2,maxSize=10,desiredSize=3 \
  --ami-type AL2_x86_64
```

AWS handles:
- Rolling updates when you upgrade the node group Kubernetes version.
- Draining nodes gracefully before termination.
- Respecting Pod Disruption Budgets during drains.
- Patching the underlying EC2 AMI.

You still pay for the EC2 instances.

### Option 3 — AWS Fargate

No EC2 instances at all. AWS runs each Pod in its own isolated micro-VM.

```yaml
# You create a Fargate Profile to tell EKS which Pods run on Fargate
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: my-cluster
  region: us-east-1
fargateProfiles:
  - name: default
    selectors:
      - namespace: production    # Pods in this namespace → Fargate
      - namespace: staging
```

| | Managed Node Groups | Fargate |
|---|---|---|
| Billing | Pay per EC2 instance (running or idle) | Pay per Pod vCPU/memory (per second) |
| Node management | AWS manages EC2 lifecycle | No nodes to manage |
| Daemonsets | Supported | Not supported |
| Node-level access | SSH into EC2 | No node access |
| GPU workloads | Supported | Not supported |
| Startup time | Seconds (node already running) | ~30-60s (cold start per Pod) |
| Best for | Predictable, high-throughput workloads | Bursty, variable workloads with no idle cost |

---

## 5) Networking in EKS — the VPC CNI plugin

Standard Kubernetes networking uses a CNI (Container Network Interface) plugin to assign IPs to Pods. In EKS, the default CNI is the **AWS VPC CNI (amazon-vpc-cni-k8s)**.

### How VPC CNI works

Instead of a private overlay network, the VPC CNI assigns **real VPC IP addresses** directly to Pods:

```
Without VPC CNI (overlay network):
  Node IP:  10.0.1.5
  Pod IP:   192.168.1.10  ← private overlay, not routable in VPC
  (traffic must be encapsulated/decapsulated at the node)

With AWS VPC CNI:
  Node IP:  10.0.1.5
  Pod IP:   10.0.1.23   ← real VPC IP, directly routable in VPC
  (no encapsulation needed — packets flow natively through VPC)
```

### How IPs are assigned

Each EC2 instance has a primary network interface (ENI) with a primary IP. The VPC CNI attaches **secondary IPs** and **secondary ENIs** to each node to get a pool of IPs to assign to Pods.

```
EC2 Node (e.g. t3.medium, max 3 ENIs, 6 IPs each)
├── ENI 0 (primary)
│   ├── 10.0.1.5   ← node primary IP
│   ├── 10.0.1.6   ← pre-allocated, given to Pod 1
│   └── 10.0.1.7   ← pre-allocated, given to Pod 2
├── ENI 1 (secondary)
│   ├── 10.0.1.20  ← given to Pod 3
│   └── 10.0.1.21  ← given to Pod 4
└── ENI 2 (secondary)
    ├── 10.0.1.30  ← given to Pod 5
    └── 10.0.1.31  ← reserved (warm pool)
```

**Max Pods per node** = (max ENIs for instance type × max IPs per ENI) - 1

```bash
# Check max pods for your instance type
# t3.medium: 3 ENIs × 6 IPs each = 18 IPs → 17 max Pods (minus 1 for node IP)
# m5.xlarge: 4 ENIs × 15 IPs each = 60 IPs → 58 max Pods

# EKS calculates this automatically and sets --max-pods on kubelet
```

### Prefix delegation (for more Pods per node)

On Nitro-based instances (most modern EC2 types), you can assign a /28 IPv4 prefix (16 IPs) to an ENI slot instead of a single secondary IP. This multiplies available IPs by up to 16×.

```bash
# Enable prefix delegation
kubectl set env daemonset aws-node \
  -n kube-system \
  ENABLE_PREFIX_DELEGATION=true
```

### Security groups for Pods

With VPC CNI, you can assign AWS Security Groups directly to individual Pods (not just nodes). Each Pod gets its own ENI with its own security group rules.

```yaml
apiVersion: vpcresources.k8s.aws/v1beta1
kind: SecurityGroupPolicy
metadata:
  name: payment-service-sgp
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: payment-service
  securityGroups:
    groupIds:
      - sg-0abc123def456789a    # only this SG attached to matching Pods
```

This means you can apply granular AWS firewall rules at the Pod level, just like you do for EC2 instances.

---

## 6) IAM integration — authentication and authorization

Kubernetes has its own RBAC system, but EKS replaces native Kubernetes user certificates with **AWS IAM** for authentication.

### How it works

```
kubectl get pods
        │
        ▼
kubectl calls: aws eks get-token (via exec credential plugin)
        │  Returns a pre-signed STS URL as a bearer token
        ▼
API Server receives token
        │
        ▼
AWS IAM Authenticator (webhook in API server)
  Validates token by calling STS GetCallerIdentity
  Returns: "this is IAM user/role arn:aws:iam::123:user/alice"
        │
        ▼
API Server maps IAM identity → Kubernetes username/groups
  (via aws-auth ConfigMap or EKS Access Entries)
        │
        ▼
Kubernetes RBAC decides what alice can do
```

### aws-auth ConfigMap (legacy method)

```yaml
# kubectl edit configmap aws-auth -n kube-system
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789:role/eks-node-role
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
    - rolearn: arn:aws:iam::123456789:role/dev-team-role
      username: dev-team
      groups:
        - dev-readonly
  mapUsers: |
    - userarn: arn:aws:iam::123456789:user/alice
      username: alice
      groups:
        - system:masters    # cluster-admin equivalent
```

### EKS Access Entries (new method, recommended)

AWS introduced a native API for managing cluster access, replacing the fragile ConfigMap approach.

```bash
# Grant an IAM role cluster-admin access
aws eks create-access-entry \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789:role/dev-team-role \
  --type STANDARD

aws eks associate-access-policy \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::123456789:role/dev-team-role \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster
```

### IRSA — IAM Roles for Service Accounts

Pods often need AWS API access (read from S3, write to DynamoDB, etc.). The wrong approach is storing static AWS credentials in a Secret. The right approach is **IRSA**.

```
How IRSA works:

1. Create an IAM Role with a trust policy scoped to the EKS OIDC provider
2. Annotate a Kubernetes ServiceAccount with the role ARN
3. EKS automatically injects a projected token + env vars into matching Pods
4. AWS SDK in the Pod exchanges the token for temporary IAM credentials via STS
```

```bash
# Step 1: Enable the OIDC provider for your cluster (one-time)
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --approve

# Step 2: Create the IAM Role and ServiceAccount in one command (with eksctl)
eksctl create iamserviceaccount \
  --cluster my-cluster \
  --namespace production \
  --name s3-reader \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
```

```yaml
# The ServiceAccount gets an annotation:
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/s3-reader-role

# Pods using this ServiceAccount automatically get:
# - AWS_ROLE_ARN env var
# - AWS_WEB_IDENTITY_TOKEN_FILE env var
# - A projected token volume mounted at /var/run/secrets/eks.amazonaws.com/serviceaccount/token
```

No static credentials. Tokens rotate automatically. The AWS SDK picks them up with zero code changes.

---

## 7) EKS add-ons — AWS-managed cluster components

EKS add-ons are AWS-managed versions of critical cluster components. AWS tests them against each Kubernetes version and handles upgrades.

| Add-on | What it does |
|---|---|
| `vpc-cni` | AWS VPC CNI plugin — Pod IP assignment |
| `coredns` | DNS for service discovery inside the cluster |
| `kube-proxy` | Service networking (iptables/ipvs rules) |
| `aws-ebs-csi-driver` | Provision EBS volumes as PersistentVolumes |
| `aws-efs-csi-driver` | Provision EFS (shared NFS) as PersistentVolumes |
| `eks-pod-identity-agent` | Alternative to IRSA for Pod IAM credentials |
| `aws-guardduty-agent` | GuardDuty runtime threat detection on nodes |

```bash
# Install or update an add-on
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name aws-ebs-csi-driver \
  --addon-version v1.28.0-eksbuild.1 \
  --service-account-role-arn arn:aws:iam::123456789:role/ebs-csi-role

# List installed add-ons and their versions
aws eks list-addons --cluster-name my-cluster
aws eks describe-addon --cluster-name my-cluster --addon-name vpc-cni
```

---

## 8) Storage in EKS — EBS and EFS

### EBS (Elastic Block Store) — block storage for single-node

EBS volumes are block devices that attach to a single EC2 instance at a time. They are the standard choice for databases and stateful apps where only one Pod needs the volume.

```yaml
# StorageClass for EBS (gp3 is the recommended type)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer   # wait until Pod is scheduled → provision in correct AZ
reclaimPolicy: Retain
---
# PVC that dynamically provisions an EBS volume
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  accessModes: [ReadWriteOnce]    # EBS: one node at a time
  storageClassName: ebs-gp3
  resources:
    requests:
      storage: 100Gi
```

> `WaitForFirstConsumer` is critical for EBS. Without it, the PVC is provisioned in a random AZ; if the Pod is then scheduled in a different AZ, it can't mount the volume.

### EFS (Elastic File System) — shared NFS storage

EFS is a managed NFS file system that can be mounted by many Pods across many nodes and AZs simultaneously.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap          # EFS Access Point per PVC
  fileSystemId: fs-0abc123def456789
  directoryPerms: "700"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-uploads
spec:
  accessModes: [ReadWriteMany]     # EFS: many nodes simultaneously
  storageClassName: efs-sc
  resources:
    requests:
      storage: 10Gi                # EFS is elastic — this is just a label
```

| | EBS | EFS |
|---|---|---|
| Access mode | ReadWriteOnce (one node) | ReadWriteMany (many nodes) |
| AZ constraint | Volume lives in one AZ | Automatically multi-AZ |
| Performance | High IOPS, low latency | Higher latency than EBS |
| Cost | Pay for provisioned size | Pay for data stored |
| Use case | Databases, stateful single-Pod apps | Shared content, ML datasets, config files |

---

## 9) Load balancing in EKS — ALB and NLB

EKS integrates with AWS load balancers through the **AWS Load Balancer Controller** (an add-on that runs as a Deployment in your cluster).

### NLB for Services (Layer 4)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-api
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip   # target Pod IPs directly
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
spec:
  type: LoadBalancer
  selector:
    app: my-api
  ports:
    - port: 443
      targetPort: 8080
```

The AWS Load Balancer Controller provisions an NLB, registers Pod IPs as targets, and keeps them updated as Pods come and go.

### ALB for Ingress (Layer 7)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-api-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm::123:certificate/abc
    alb.ingress.kubernetes.io/ssl-redirect: "443"
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /v1/users
            pathType: Prefix
            backend:
              service:
                name: users-service
                port:
                  number: 80
          - path: /v1/orders
            pathType: Prefix
            backend:
              service:
                name: orders-service
                port:
                  number: 80
```

One ALB can serve multiple Ingress resources in the same cluster via **IngressGroups**:

```yaml
annotations:
  alb.ingress.kubernetes.io/group.name: shared-alb   # multiple Ingresses share one ALB
```

| | NLB | ALB |
|---|---|---|
| OSI layer | 4 (TCP/UDP) | 7 (HTTP/HTTPS) |
| Kubernetes resource | Service (type=LoadBalancer) | Ingress |
| SSL termination | Pass-through or TLS termination | Terminates TLS |
| Path/host routing | No | Yes |
| WebSockets | Yes | Yes |
| gRPC | Yes | Yes |
| Cost | Per LCU, lower for TCP | Per LCU, slightly higher |

---

## 10) Cluster Autoscaler and Karpenter — automatic node scaling

### Cluster Autoscaler (CA)

The traditional node scaler. When Pods are Pending (no room on existing nodes), CA increases the ASG desired count. When nodes are underutilized, CA scales down.

```
Pending Pod (not enough resources on any node)
        │
        ▼
Cluster Autoscaler detects Pending Pod
        │
        ▼
Calls AWS Auto Scaling API to increase desired capacity
        │
        ▼
New EC2 node joins cluster (~2-3 minutes)
        │
        ▼
Scheduler places the Pending Pod on the new node
```

Limitation: CA works within a fixed set of ASG node groups with pre-defined instance types.

### Karpenter (modern replacement)

Karpenter replaces CA with a more flexible approach. It watches Pending Pods and directly provisions the **right-sized EC2 instance** for those Pods — without ASGs.

```yaml
# NodePool defines what kinds of nodes Karpenter can provision
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
        - key: node.kubernetes.io/instance-category
          operator: In
          values: ["c", "m", "r"]     # compute, memory, general-purpose families
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
  limits:
    cpu: 1000                         # max 1000 vCPUs in this pool
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s             # replace 2 half-empty nodes with 1 right-sized node
---
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2023
  role: KarpenterNodeRole-my-cluster
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
```

| | Cluster Autoscaler | Karpenter |
|---|---|---|
| Scale-up speed | ~2-3 minutes (ASG warmup) | ~60 seconds (direct EC2 launch) |
| Instance selection | Fixed ASG instance types | Picks best fit from any instance type |
| Spot handling | Manual, per ASG | Automatic multi-type Spot diversification |
| Bin packing | No | Yes — consolidates underutilized nodes |
| Configuration | ASG-centric | NodePool + EC2NodeClass CRDs |

---

## 11) EKS cluster upgrade — how versioning works

Kubernetes releases a new minor version roughly every 4 months. EKS supports each minor version for ~14 months after it is first available on EKS.

```
Upgrade order (ALWAYS in this sequence):
  1. Control plane first  (EKS upgrades this for you with zero downtime)
  2. Add-ons second       (update each add-on to the version compatible with new K8s)
  3. Node groups last     (rolling replacement of EC2 nodes)
```

### Control plane upgrade

```bash
# Upgrade the EKS control plane from 1.29 → 1.30
aws eks update-cluster-version \
  --name my-cluster \
  --kubernetes-version 1.30

# Monitor the update
aws eks describe-update \
  --name my-cluster \
  --update-id <update-id>
```

AWS performs a blue/green-style upgrade of the control plane components with no downtime. Running Pods are not affected.

### Node group upgrade

```bash
# Upgrade managed node group (rolling replacement of nodes)
aws eks update-nodegroup-version \
  --cluster-name my-cluster \
  --nodegroup-name my-nodes \
  --kubernetes-version 1.30
```

AWS drains each node (respecting PodDisruptionBudgets), terminates it, and launches a replacement with the new AMI. Nodes are replaced one at a time (or per the `maxUnavailable` setting).

> You can only upgrade **one minor version at a time** (e.g., 1.28 → 1.29, not 1.28 → 1.30). Plan upgrade windows accordingly.

---

## 12) Observability — logs, metrics, and traces

### Control plane logs

EKS can ship control plane component logs to CloudWatch:

```bash
aws eks update-cluster-config \
  --name my-cluster \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'
```

| Log type | What it contains |
|---|---|
| `api` | All API server requests (useful for auditing who did what) |
| `audit` | Kubernetes audit log (user, resource, verb, response code) |
| `authenticator` | IAM authentication events and errors |
| `controllerManager` | Controller reconciliation events |
| `scheduler` | Scheduling decisions and failures |

### Node and Pod metrics

The standard stack for EKS observability:

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│  Prometheus (metrics scraping)                      │
│    scrapes: node-exporter, kube-state-metrics,      │
│             cAdvisor (built into kubelet)            │
│                                                     │
│  Grafana (dashboards)                               │
│    dashboards: node CPU/memory, Pod resource usage, │
│                Deployment health, API server latency│
│                                                     │
│  AWS alternative: Amazon Managed Prometheus (AMP)   │
│  + Amazon Managed Grafana (AMG)                     │
│                                                     │
└─────────────────────────────────────────────────────┘
```

```bash
# Install kube-prometheus-stack (includes Prometheus + Grafana + alerting)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

### Container Insights

AWS Container Insights uses the **CloudWatch Agent** or **AWS Distro for OpenTelemetry (ADOT)** to collect and ship metrics and logs from nodes and Pods directly to CloudWatch — no Prometheus needed.

```bash
# Install Container Insights with the CloudWatch Observability add-on
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name amazon-cloudwatch-observability
```

---

## 13) Real-world examples

### Example A — Pod can't pull image: ECR permissions

**Symptom**: Pods stuck in `ImagePullBackOff`.

```bash
kubectl describe pod my-api-abc123
# Events:
#   Warning  Failed  2m  kubelet  Failed to pull image
#   "123456789.dkr.ecr.us-east-1.amazonaws.com/my-api:v1.2":
#   no basic auth credentials
```

**Cause**: The node's IAM role doesn't have permission to pull from ECR.

**Fix**: Attach the `AmazonEC2ContainerRegistryReadOnly` policy to the node IAM role.

```bash
aws iam attach-role-policy \
  --role-name eks-node-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
```

---

### Example B — Pods Pending: hit the max-pods limit per node

**Symptom**: Pods stay Pending even though nodes look underutilized in AWS console.

```bash
kubectl describe pod my-api-xyz
# Events:
#   Warning  FailedScheduling  0/3 nodes available:
#   3 Too many pods.
```

**Cause**: Hit the VPC CNI's max-pods limit (IP exhaustion on the ENI).

**Fix options**:
1. Use larger instance types with more ENI slots.
2. Enable prefix delegation to get more IPs per ENI.
3. Use Karpenter which handles this automatically by selecting appropriately-sized instances.

```bash
# Check current pod capacity
kubectl get node worker-node-1 -o jsonpath='{.status.capacity.pods}'
# 17

# Check pods currently on the node
kubectl get pods --all-namespaces --field-selector spec.nodeName=worker-node-1 | wc -l
```

---

### Example C — IRSA not working: Pod can't call AWS API

**Symptom**: AWS SDK call inside a Pod returns `NoCredentialProviders`.

```bash
kubectl exec -it my-pod -- aws s3 ls
# An error occurred (NoCredentialProviders): no valid providers in chain
```

**Checklist**:

```bash
# 1. Does the ServiceAccount have the annotation?
kubectl get sa my-sa -n production -o yaml | grep role-arn

# 2. Does the Pod actually use that ServiceAccount?
kubectl get pod my-pod -o jsonpath='{.spec.serviceAccountName}'

# 3. Is the OIDC provider associated with the cluster?
aws eks describe-cluster --name my-cluster \
  --query "cluster.identity.oidc.issuer"

# 4. Does the IAM Role trust policy reference the correct OIDC provider and ServiceAccount?
aws iam get-role --role-name my-role \
  --query "Role.AssumeRolePolicyDocument"

# 5. Is the token file actually mounted in the Pod?
kubectl exec -it my-pod -- ls /var/run/secrets/eks.amazonaws.com/serviceaccount/
```

---

### Example D — Tracing a node scale-up with Karpenter

```bash
# Watch Karpenter logs in real time as it provisions nodes
kubectl logs -f -n kube-system -l app.kubernetes.io/name=karpenter \
  | grep -E "provisioned|launched|registered"

# 10:00:01  provisioned: 3 pods, 1 node (m5.xlarge, spot, us-east-1b)
# 10:00:35  node registered: ip-10-0-1-42.ec2.internal
# 10:00:48  pods scheduled: 3/3

# See what nodes Karpenter is managing
kubectl get nodeclaims
```

---

## 14) Common pitfalls

| Pitfall | What goes wrong | Fix |
|---|---|---|
| EBS PVC in wrong AZ | Pod scheduled in us-east-1a, EBS volume in us-east-1b → `FailedMount` | Set `volumeBindingMode: WaitForFirstConsumer` on StorageClass |
| aws-auth ConfigMap corrupted | All non-admin access breaks; may lock you out | Use EKS Access Entries instead of aws-auth; keep a backup; always test with a dry-run |
| No Pod Disruption Budgets | Node drain during upgrade terminates all replicas simultaneously → downtime | Define PDBs for all critical workloads |
| Skipping a Kubernetes minor version | EKS rejects update requests that skip versions | Upgrade one minor version at a time: 1.28 → 1.29 → 1.30 |
| Node IAM role too permissive | A compromised node can access all AWS resources in the account | Use IRSA instead of instance-level IAM roles for Pod-level AWS access |
| Forgetting to upgrade add-ons after control plane | Old VPC CNI or CoreDNS becomes incompatible with new API versions | Upgrade add-ons immediately after each control plane upgrade |
| Max Pods limit hit | Pods Pending despite spare node CPU/memory | Enable prefix delegation or use Karpenter for dynamic instance selection |
| Public API server endpoint with no IP allowlist | The Kubernetes API is exposed to the entire internet | Enable private endpoint or add `publicAccessCidrs` to restrict API server access |

---

## 15) Quick mental model for revision

```
EKS = managed K8s control plane + your choice of worker nodes

Control Plane (AWS manages):
  API server, etcd, controller manager, scheduler
  → Multi-AZ, automatically backed up, AWS SLA
  → You interact via kubectl endpoint + IAM auth

Worker Nodes (you choose):
  Managed Node Groups  → AWS manages EC2 lifecycle, rolling upgrades
  Fargate              → no nodes; per-Pod billing, no DaemonSets
  Self-managed         → full control, full responsibility

Networking (VPC CNI):
  → Real VPC IPs on Pods (no overlay)
  → Max Pods = ENI slots × IPs per ENI
  → Security Groups per Pod for fine-grained firewall rules

IAM Integration:
  → Authentication via STS tokens (not K8s certificates)
  → IRSA = Pod-level IAM roles via OIDC (no static credentials)
  → aws-auth ConfigMap OR EKS Access Entries → K8s RBAC

Storage:
  EBS  → ReadWriteOnce, one AZ, WaitForFirstConsumer mandatory
  EFS  → ReadWriteMany, multi-AZ, shared across Pods

Load Balancing (AWS Load Balancer Controller):
  NLB  → Service type=LoadBalancer (L4, TCP/UDP)
  ALB  → Ingress (L7, HTTP/HTTPS path routing)

Node Scaling:
  Cluster Autoscaler  → ASG-based, slower (~3 min), fixed types
  Karpenter           → direct EC2 launch (~60s), right-sized, bin-packing

Upgrade order: control plane → add-ons → node groups (one minor version at a time)
```

Key facts to remember:
- EKS control plane is AWS-managed — you never SSH into it.
- Authentication is IAM-based (STS tokens), not Kubernetes certificate-based.
- VPC CNI assigns real VPC IPs to Pods — check max-pods limits per instance type.
- IRSA eliminates the need for static AWS credentials in Pods.
- `WaitForFirstConsumer` on EBS StorageClass is mandatory to avoid AZ mismatches.
- Upgrade one minor version at a time, always in order: control plane → add-ons → nodes.
- Karpenter is the modern Cluster Autoscaler replacement — faster, smarter, cheaper.
