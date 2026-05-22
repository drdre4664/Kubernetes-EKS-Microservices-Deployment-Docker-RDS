# Kubernetes Concepts — Understanding the Foundation

This file documents the core Kubernetes concepts behind the deployment in this repo. The README covers what was deployed and how — this file covers why things work the way they do.

---

## Kubernetes Architecture

Kubernetes is a cluster made up of two types of machines: the Control Plane and Worker Nodes.

```
┌─────────────────────────────────────────────────────────────────┐
│                        KUBERNETES CLUSTER                        │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                     CONTROL PLANE                         │   │
│  │                                                           │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │   │
│  │  │  API Server │  │  Scheduler  │  │  Controller Mgr │  │   │
│  │  │             │  │             │  │                 │  │   │
│  │  │ Front door  │  │ Decides     │  │ Watches cluster │  │   │
│  │  │ to cluster  │  │ which node  │  │ state and fixes │  │   │
│  │  │ kubectl     │  │ runs a pod  │  │ drift           │  │   │
│  │  │ talks here  │  │             │  │                 │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘  │   │
│  │                                                           │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │  etcd — distributed key-value store              │    │   │
│  │  │  Stores ALL cluster state — nodes, pods,         │    │   │
│  │  │  configs, secrets. If etcd is lost, the          │    │   │
│  │  │  cluster loses its memory.                       │    │   │
│  │  └──────────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────┐    ┌──────────────────────┐           │
│  │     WORKER NODE 1    │    │     WORKER NODE 2    │           │
│  │                      │    │                      │           │
│  │  ┌────────────────┐  │    │  ┌────────────────┐  │           │
│  │  │     kubelet    │  │    │  │     kubelet    │  │           │
│  │  │ Agent that     │  │    │  │ Agent that     │  │           │
│  │  │ talks to       │  │    │  │ talks to       │  │           │
│  │  │ control plane  │  │    │  │ control plane  │  │           │
│  │  └────────────────┘  │    │  └────────────────┘  │           │
│  │                      │    │                      │           │
│  │  ┌────────────────┐  │    │  ┌────────────────┐  │           │
│  │  │  kube-proxy    │  │    │  │  kube-proxy    │  │           │
│  │  │ Manages network│  │    │  │ Manages network│  │           │
│  │  │ rules for      │  │    │  │ rules for      │  │           │
│  │  │ pod traffic    │  │    │  │ pod traffic    │  │           │
│  │  └────────────────┘  │    │  └────────────────┘  │           │
│  │                      │    │                      │           │
│  │  [Pod] [Pod] [Pod]   │    │  [Pod] [Pod]         │           │
│  └──────────────────────┘    └──────────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

| Component | What it does |
|-----------|-------------|
| **API Server** | The front door to the cluster — every kubectl command goes through here |
| **Scheduler** | Decides which worker node a new pod should run on based on available resources |
| **Controller Manager** | Continuously watches the cluster and fixes anything that drifts from the desired state |
| **etcd** | The cluster's database — stores all state (nodes, pods, configs, secrets) |
| **kubelet** | Agent running on every worker node — receives instructions from control plane and manages pods |
| **kube-proxy** | Manages networking rules on each node so pods can talk to each other and to services |

---

## Pod

A Pod is the smallest deployable unit in Kubernetes. It wraps one or more containers that share the same network and storage.

```
┌──────────────────────────────┐
│            POD               │
│                              │
│  ┌──────────┐ ┌──────────┐  │
│  │Container │ │Container │  │
│  │  (app)   │ │(sidecar) │  │
│  └──────────┘ └──────────┘  │
│                              │
│  Shared network (same IP)    │
│  Shared storage volumes      │
└──────────────────────────────┘
```

**Key facts:**
- Every pod gets its own IP address inside the cluster
- Containers inside the same pod communicate via localhost
- Pods are ephemeral — they can die and be replaced at any time
- You almost never create pods directly — you use a Deployment which manages them

In this repo, the backend and frontend each run as pods:
```yaml
# From k8s/backend/deployment.yaml
containers:
  - name: backend
    image: dre4664/book-review-backend:v1
    ports:
      - containerPort: 3001
```

---

## ReplicaSet

A ReplicaSet ensures a specified number of identical pod copies are always running. If a pod crashes, the ReplicaSet creates a replacement.

```
ReplicaSet (replicas: 3)
        │
        ├── Pod 1 ✓ running
        ├── Pod 2 ✓ running
        └── Pod 3 ✗ crashed → ReplicaSet creates Pod 4 immediately
```

**You rarely create ReplicaSets directly.** Deployments manage ReplicaSets for you and add rolling update capability on top.

---

## Deployment

A Deployment manages ReplicaSets and adds the ability to do rolling updates and rollbacks. It is the standard way to run stateless applications in Kubernetes.

```
Deployment
    │
    └── ReplicaSet (current version)
              │
              ├── Pod (v2)
              ├── Pod (v2)
              └── Pod (v2)

During a rolling update:
    │
    ├── ReplicaSet (old — v1) → scales down
    └── ReplicaSet (new — v2) → scales up
              │
              ├── Pod (v2) ← new
              ├── Pod (v2) ← new
              └── Pod (v1) ← being terminated
```

**Deployment vs ReplicaSet:**

| | ReplicaSet | Deployment |
|--|-----------|-----------|
| Keeps pods running | Yes | Yes (via ReplicaSet) |
| Rolling updates | No | Yes |
| Rollback | No | Yes |
| Use directly | Rarely | Yes — this is what you use |

In this repo both backend and frontend use Deployments with 2 replicas:
```yaml
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
```

---

## Services

A Service gives a stable DNS name and IP address to a set of pods. Without a Service, you would need to track individual pod IPs — which change every time a pod restarts.

```
Without Service:              With Service:

Client → Pod IP (changes)     Client → Service (stable)
         ↑ pod dies                        │
         new IP issued                     ├── Pod 1
                                           ├── Pod 2
                                           └── Pod 3 (load balanced)
```

There are three main Service types:

### ClusterIP (default)

Only reachable from inside the cluster. No external access.

```
Internet ✗
             ┌────────────────────────────────┐
             │         CLUSTER                 │
             │                                 │
             │  Pod A ──► ClusterIP Service    │
             │               ──► Pod B         │
             │                                 │
             └────────────────────────────────┘
```

**Use when:** Internal services that should never be exposed publicly — databases, internal APIs.

```yaml
spec:
  type: ClusterIP
  ports:
    - port: 3306
```

### NodePort

Opens a specific port on every worker node. External traffic can reach pods via NodeIP:NodePort.

```
Internet
    │
    ▼ Node IP : 30080    (NodePort range: 30000–32767)
    │
    ▼ Service → Pod
```

**Use when:** Development, testing, or when you don't have a LoadBalancer available.

```yaml
spec:
  type: NodePort
  ports:
    - port: 80
      nodePort: 30080
```

### LoadBalancer

Provisions a cloud load balancer (AWS ALB, Azure LB) with a public IP. This is the production standard for exposing services to the internet.

```
Internet
    │
    ▼ AWS Load Balancer (public IP/DNS)
    │
    ▼ Service → Pod 1
       → Pod 2
```

**Use when:** Production — exposing a service publicly on AWS, Azure, or GCP.

```yaml
spec:
  type: LoadBalancer
```

In this repo both frontend and backend use LoadBalancer because the backend is in a different VPC from the RDS — an internal ClusterIP couldn't reach it:
```yaml
# k8s/backend/service.yaml
spec:
  type: LoadBalancer
  ports:
    - port: 3001
      targetPort: 3001
```

### Service Type Summary

| Type | Accessible from | Use case |
|------|----------------|---------|
| ClusterIP | Inside cluster only | Internal services, databases |
| NodePort | Node IP + port | Dev/test, no cloud LB available |
| LoadBalancer | Public internet via cloud LB | Production external traffic |

---

## ConfigMap

A ConfigMap stores non-sensitive configuration as key-value pairs. Instead of hardcoding values in your container image, you inject them at runtime.

```
ConfigMap (backend-config)
    │
    ├── DB_HOST: database-1.rds.amazonaws.com
    ├── DB_NAME: book_review_db
    └── PORT: "3001"
         │
         ▼ injected as environment variables
    Pod → container reads DB_HOST from env
```

**Why:** If the database endpoint changes, update the ConfigMap and redeploy. The image stays the same.

```yaml
envFrom:
  - configMapRef:
      name: backend-config
```

---

## Secret

A Secret stores sensitive data (passwords, tokens, keys) encoded in base64. It works like a ConfigMap but Kubernetes treats it differently — it is not shown in logs and can be encrypted at rest.

```yaml
# k8s/backend/secret.yaml
stringData:
  DB_PASS: YOUR_PASSWORD
  JWT_SECRET: YOUR_SECRET
```

**Important:** Secrets are base64 encoded, not encrypted by default. Never commit real secret.yaml files to git — only commit secret.yaml.example with placeholder values.

---

## Health Probes

Health probes tell Kubernetes whether a container is ready to receive traffic and whether it is still alive.

### Readiness Probe

Controls whether the pod receives traffic from a Service. If the readiness probe fails, the pod is removed from the load balancer rotation — traffic stops going to it.

```
Pod starts
    │
    ▼ Readiness probe: GET /api/books
    │
    ├── 200 OK → pod added to Service, receives traffic
    └── failure → pod excluded from Service, not killed
```

### Liveness Probe

Controls whether the pod should be restarted. If the liveness probe fails, Kubernetes kills and restarts the container.

```
Pod running
    │
    ▼ Liveness probe: GET /api/books every 10s
    │
    ├── 200 OK → pod stays running
    └── failure → Kubernetes restarts the container
```

In this repo both probes are configured on the backend:
```yaml
readinessProbe:
  httpGet:
    path: /api/books
    port: 3001
  initialDelaySeconds: 10
  periodSeconds: 5

livenessProbe:
  httpGet:
    path: /api/books
    port: 3001
  initialDelaySeconds: 15
  periodSeconds: 10
```

| Probe | Fails → | Use for |
|-------|--------|---------|
| Readiness | Remove from load balancer | App not ready yet (warming up, DB connecting) |
| Liveness | Restart container | App is stuck or deadlocked |

---

## Horizontal Pod Autoscaler (HPA)

HPA automatically scales the number of pod replicas based on CPU or memory usage.

```
Low traffic:                 High traffic:
replicas: 2                  replicas: 6
                                   ▲
                     HPA detects CPU > 70%
                     scales up automatically

Traffic drops: HPA scales back down to minimum replicas
```

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-deployment
  minReplicas: 2
  maxReplicas: 8
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## Rolling Update Strategy

When you update a Deployment (new image, config change), Kubernetes replaces pods gradually so there is no downtime.

```
Before update:    Pod v1  Pod v1  Pod v1

During update:    Pod v2  Pod v1  Pod v1   ← v2 created, v1 terminating
                  Pod v2  Pod v2  Pod v1
                  Pod v2  Pod v2  Pod v2

After update:     Pod v2  Pod v2  Pod v2
```

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # max extra pods during update
    maxUnavailable: 0  # no pods removed until new one is ready
```

---

## How Everything Connects in This Repo

```
kubectl apply -f k8s/
        │
        ▼ Deployment → creates ReplicaSet → creates Pods
        │                              │
        │                              ▼
        │                    Pods read config from:
        │                    - ConfigMap (DB_HOST, PORT)
        │                    - Secret (DB_PASS, JWT_SECRET)
        │
        ▼ Service (LoadBalancer) → AWS creates a Load Balancer
        │                      with public DNS
        │
        ▼ Internet → AWS LB → Service → Pod 1
                            → Pod 2
        │
        ▼ Health probes run every 5-10s
If pod fails → removed from LB (readiness)
             → restarted (liveness)
```

---

## Key Concepts Summary

| Concept | What it does | When to use |
|---------|-------------|------------|
| Pod | Runs containers | Managed by Deployment |
| ReplicaSet | Keeps N pods running | Managed by Deployment |
| Deployment | Manages ReplicaSets, enables rolling updates | All stateless apps |
| ClusterIP | Internal service, no external access | Databases, internal APIs |
| NodePort | Exposes on node IP:port | Dev/test only |
| LoadBalancer | Public cloud load balancer | Production external traffic |
| ConfigMap | Non-sensitive config injection | DB hosts, ports, URLs |
| Secret | Sensitive config injection | Passwords, tokens, keys |
| Readiness Probe | Controls traffic routing to pod | Apps with slow startup |
| Liveness Probe | Controls pod restart | Apps that can deadlock |
| HPA | Auto-scales replicas on load | Production workloads |
