# ☸️ Kubernetes — Complete Notes
## Beginner to Expert | Interview Ready

---

## 📋 Table of Contents

1. [Introduction](#1-introduction)
2. [Architecture](#2-architecture)
3. [Workloads](#3-workloads)
4. [Networking & Services](#4-networking--services)
5. [Storage](#5-storage)
6. [ConfigMaps & Secrets](#6-configmaps--secrets)
7. [Security — RBAC](#7-security--rbac)
8. [Scaling & Auto-scaling](#8-scaling--auto-scaling)
9. [Helm](#9-helm)
10. [Troubleshooting & Debugging](#10-troubleshooting--debugging)
11. [Interview Q&A](#11-interview-qa)
12. [kubectl Quick Reference](#12-kubectl-quick-reference)

---

## 1. Introduction

### What is Kubernetes?

Kubernetes (K8s) is an open-source **container orchestration platform**. It automates deployment, scaling, and management of containerized applications. Created by Google in 2014, donated to CNCF. The name comes from Greek meaning "helmsman" or "pilot".

### Why Kubernetes? The Problem It Solves

| Without Kubernetes | With Kubernetes |
|---|---|
| Manual container management | Automatic scheduling on nodes |
| No automatic restart on crash | Self-healing — restarts crashed pods |
| Manual load balancing | Built-in load balancing |
| No self-healing | Auto-scaling based on CPU/memory |
| Hard to scale up/down | Rolling updates with zero downtime |
| Downtime during deployments | Declarative configuration |

### Key Features

- **Self-healing** — Restarts failed containers, replaces unresponsive pods, kills unhealthy containers
- **Auto-scaling** — HPA scales pods, VPA resizes pods, Cluster Autoscaler adds nodes
- **Rolling updates** — Zero downtime deployments, canary deployments, easy rollbacks
- **Service discovery** — Built-in DNS, load balancing between pods
- **Storage orchestration** — Automatically mount storage from local, cloud, or network

### Kubernetes vs Docker

```bash
# Docker = runs containers on ONE machine
docker run -d nginx

# Kubernetes = orchestrates containers across MANY machines
# Schedules, heals, scales, load balances automatically
kubectl apply -f deployment.yaml  # K8s handles the rest
```

> **Note:** K8s does NOT replace Docker. Docker builds and runs containers. K8s orchestrates those containers across a cluster of machines.

### Core Concepts Hierarchy

```
Cluster → contains → Nodes → run → Pods → contain → Containers
```

---

## 2. Architecture

### Control Plane Components

#### API Server (kube-apiserver)
- Frontend to the control plane
- All `kubectl` commands go here first
- Validates and processes all requests
- **Only component that reads/writes to etcd**
- Exposes REST API on port 6443

#### etcd
- Distributed key-value store
- Stores **ALL cluster state** — the source of truth
- Must be backed up regularly
- Run in odd-numbered clusters (3, 5, 7) for quorum
- Backup: `ETCDCTL_API=3 etcdctl snapshot save backup.db`

#### Scheduler (kube-scheduler)
- Watches for unscheduled pods
- Scores nodes based on: available CPU/memory, affinity rules, taints/tolerations
- Assigns pod to the best scoring node
- Does NOT start the pod — just binds it to a node

#### Controller Manager (kube-controller-manager)
- Runs multiple controller loops in one process
- **Node controller** — monitors node health
- **Replication controller** — maintains correct pod count
- **Endpoint controller** — manages Service endpoints
- **Service Account controller** — creates default service accounts

### Worker Node Components

#### kubelet
- Agent running on every worker node
- Watches API Server for pods assigned to its node
- Starts/stops containers via container runtime
- Reports node and pod status back to API Server
- Does NOT manage containers not created by Kubernetes

#### kube-proxy
- Network proxy on each node
- Manages iptables/IPVS rules for Service routing
- Enables communication to Services from inside/outside cluster
- Does NOT proxy actual container traffic — sets up routing rules

#### Container Runtime
- Software that runs containers (containerd, CRI-O)
- Implements Container Runtime Interface (CRI)
- Pulls images from registry
- Creates and manages container lifecycle

### What Happens When You Run `kubectl apply`

```
Step 1:  kubectl         → sends authenticated HTTP request to API Server
Step 2:  API Server      → validates request → runs admission controllers
Step 3:  API Server      → persists desired state to etcd
Step 4:  Controller Mgr  → detects new desired state → creates ReplicaSet → creates Pod objects
Step 5:  Scheduler       → watches unscheduled pods → scores nodes → binds pod to node
Step 6:  kubelet         → sees pod assigned to its node → pulls image → starts container
Step 7:  kubelet         → reports pod status back to API Server → etcd updated
Step 8:  kube-proxy      → updates iptables rules for Service routing
```

### Essential Architecture Commands

```bash
kubectl cluster-info                    # show cluster endpoints
kubectl get nodes                       # list all nodes
kubectl get nodes -o wide               # nodes with IP, OS, runtime
kubectl describe node <node-name>       # detailed node info
kubectl get componentstatuses           # control plane component health
```

---

## 3. Workloads

### Pod — The Smallest Deployable Unit

A Pod is a group of one or more containers that share network and storage. Pods are ephemeral — they can die and be replaced.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
  labels:
    app: my-app       # labels are KEY for selectors!
spec:
  containers:
    - name: my-app
      image: nginx:1.21
      ports:
        - containerPort: 80
      resources:              # ALWAYS set resources!
        requests:
          cpu: "100m"         # 0.1 CPU core
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "512Mi"
```

> **Rule:** Never create bare Pods in production. If a Pod dies, nothing recreates it. Always use Deployment or StatefulSet.

### Deployment — Manages Stateless Apps

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app       # must match pod template labels
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # max pods above desired during update
      maxUnavailable: 0     # zero downtime!
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:1.0
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 20
            failureThreshold: 3
```

### Workload Types Comparison

| Type | Use For | Key Feature |
|---|---|---|
| **Deployment** | Stateless apps, APIs | Rolling updates, rollback |
| **StatefulSet** | Databases, Kafka, Redis | Stable names, ordered start/stop |
| **DaemonSet** | Log agents, monitoring | One pod per node |
| **Job** | Batch tasks | Run to completion |
| **CronJob** | Scheduled batch | Cron-based scheduling |

### StatefulSet — Stateful Applications

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
spec:
  serviceName: kafka-headless    # requires headless service
  replicas: 3
  selector:
    matchLabels:
      app: kafka
  template:
    spec:
      containers:
        - name: kafka
          image: kafka:3.5
          volumeMounts:
            - name: data
              mountPath: /data
  volumeClaimTemplates:          # each pod gets its OWN PVC!
    - metadata:
        name: data
      spec:
        storageClassName: fast-ssd
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 100Gi
```

**StatefulSet guarantees:**
- Stable pod names: `kafka-0`, `kafka-1`, `kafka-2`
- Ordered startup: `kafka-0` starts, then `kafka-1`, then `kafka-2`
- Ordered deletion: `kafka-2` deleted first (reverse order)
- Stable network identity via DNS
- Persistent storage per pod (not shared)

### Probes — Critical for Zero-Downtime

```yaml
# Startup Probe — disables liveness until app starts (slow JVM apps)
startupProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30    # 30 × 10s = 5 minutes to start
  periodSeconds: 10

# Readiness Probe — controls traffic (removes from Service endpoints on failure)
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  successThreshold: 1
  failureThreshold: 3

# Liveness Probe — controls restart (kills container on failure)
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 20
  periodSeconds: 10
  failureThreshold: 3
```

| Probe | On Failure | Use For |
|---|---|---|
| Startup | Restart pod | Slow app initialization |
| Readiness | Remove from Service | Traffic gating |
| Liveness | Restart pod | Deadlock / hung process |

### Deployment Commands

```bash
kubectl apply -f deployment.yaml
kubectl get deployments -n production
kubectl rollout status deployment/my-app     # watch rollout progress
kubectl rollout history deployment/my-app    # see revision history
kubectl rollout undo deployment/my-app       # rollback to previous!
kubectl rollout undo deployment/my-app --to-revision=2  # rollback to v2
kubectl scale deployment my-app --replicas=5
kubectl set image deployment/my-app my-app=my-app:2.0
```

---

## 4. Networking & Services

### Service Types

#### ClusterIP (default)
- Internal only — no external access
- Stable virtual IP inside cluster
- DNS: `svc-name.namespace.svc.cluster.local`
- **Use for:** microservice-to-microservice communication

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: my-app          # matches pod labels!
  ports:
    - port: 80           # service port
      targetPort: 8080   # pod port
```

#### NodePort
- Exposes on each node's IP at a static port (30000–32767)
- External: `nodeIP:nodePort`
- **Use for:** dev/testing only — NOT for production

#### LoadBalancer
- Provisions cloud load balancer (ELB, GLB, Azure LB)
- Each service gets its OWN external LB — expensive!
- **Use for:** non-HTTP services (TCP, UDP) or simple setups
- **Prefer:** Ingress for HTTP services

#### Headless Service (clusterIP: None)
- No stable IP — DNS returns individual pod IPs
- **Use for:** StatefulSets, Kafka, Cassandra, databases
- DNS: `pod-0.headless-svc.namespace.svc.cluster.local`

### Ingress — HTTP Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/limit-rps: "100"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.company.com
      secretName: app-tls
  rules:
    - host: app.company.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

> **Cost tip:** 1 Ingress Controller = 1 cloud LB routes to ALL services. LoadBalancer per Service = 1 LB per service. Use Ingress!

### NetworkPolicy — Pod-to-Pod Traffic Control

```yaml
# Default deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector: {}           # applies to all pods
  policyTypes: [Ingress]

---
# Allow only frontend to reach backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
        - namespaceSelector:
            matchLabels:
              name: production
```

### DNS in Kubernetes

```
Format: <service>.<namespace>.svc.cluster.local

Examples:
  my-api.default.svc.cluster.local          → ClusterIP
  my-api.default.svc.cluster.local:8080     → with port
  pod-0.kafka-headless.kafka.svc.cluster.local → StatefulSet pod
```

---

## 5. Storage

### Storage Hierarchy

```
StorageClass → dynamically provisions → PersistentVolume → bound to → PVC → mounted in → Pod
```

### PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-storage
spec:
  accessModes: [ReadWriteOnce]    # RWO: 1 node at a time
  storageClassName: fast-ssd      # references StorageClass
  resources:
    requests:
      storage: 20Gi
```

### Mount PVC in Pod

```yaml
spec:
  volumes:
    - name: db-data
      persistentVolumeClaim:
        claimName: db-storage
  containers:
    - name: postgres
      volumeMounts:
        - name: db-data
          mountPath: /var/lib/postgresql/data
```

### Access Modes

| Mode | Short | Description | Use Case |
|---|---|---|---|
| ReadWriteOnce | RWO | Read/write by ONE node | Databases, single-instance |
| ReadOnlyMany | ROX | Read by MANY nodes | Shared config, static assets |
| ReadWriteMany | RWX | Read/write by MANY nodes | Shared filesystems (NFS) |

### StorageClass — Dynamic Provisioning

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

### Volume Types

```yaml
# emptyDir — temporary storage, deleted when pod dies
volumes:
  - name: cache
    emptyDir: {}

# hostPath — mounts directory from node (dangerous!)
volumes:
  - name: logs
    hostPath:
      path: /var/log

# ConfigMap as volume
volumes:
  - name: config
    configMap:
      name: app-config

# Secret as volume (preferred over env vars)
volumes:
  - name: secrets
    secret:
      secretName: db-credentials
      defaultMode: 0400     # read-only
```

> **StatefulSet note:** Each pod gets its OWN PVC via `volumeClaimTemplates`. Pod-0 gets pod-0 storage, Pod-1 gets pod-1 storage. They do NOT share storage.

---

## 6. ConfigMaps & Secrets

### ConfigMap — Non-Sensitive Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DB_HOST: "postgres-service"
  LOG_LEVEL: "INFO"
  MAX_CONNECTIONS: "100"
  app.properties: |          # multi-line file as value
    server.port=8080
    cache.ttl=300
    feature.newui=true
```

### Consuming ConfigMap

```yaml
# Method 1: All keys as env vars
envFrom:
  - configMapRef:
      name: app-config

# Method 2: Specific key as env var
env:
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: LOG_LEVEL

# Method 3: Mount as file volume (updates WITHOUT pod restart!)
volumeMounts:
  - name: config
    mountPath: /etc/config
    readOnly: true
volumes:
  - name: config
    configMap:
      name: app-config
```

### Secret — Sensitive Data

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  # Values must be base64 encoded!
  # echo -n "mypassword" | base64
  username: bXl1c2Vy
  password: bXlwYXNzd29yZA==
```

```bash
# Create secret from command line
kubectl create secret generic db-creds \
  --from-literal=username=myuser \
  --from-literal=password=mypassword

# Create from file
kubectl create secret generic tls-cert \
  --from-file=tls.crt=./cert.crt \
  --from-file=tls.key=./cert.key
```

> **CRITICAL:** Secrets are base64 encoded — NOT encrypted! Anyone with `kubectl get secret` can decode them.

### Production Secret Management

```yaml
# External Secrets Operator — syncs from AWS SSM / Vault / Azure KV
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore
  target:
    name: db-credentials     # creates this K8s Secret
  data:
    - secretKey: password
      remoteRef:
        key: prod/db/password  # path in AWS SSM
```

**Security Best Practices for Secrets:**
1. Enable Encryption at Rest for etcd
2. Use External Secrets Operator (AWS SSM / HashiCorp Vault)
3. Prefer volume mounts over env vars (env vars visible in `/proc` and logs)
4. Restrict access with RBAC
5. Audit log who accessed which secret
6. Rotate secrets regularly (External Secrets auto-rotates)
7. Never commit secrets to Git — even encrypted

---

## 7. Security — RBAC

### RBAC Building Blocks

```
ServiceAccount/User/Group → RoleBinding → Role → permissions on resources
```

| Object | Scope | Purpose |
|---|---|---|
| ServiceAccount | Namespace | Identity for pods |
| Role | Namespace | Permissions in a namespace |
| ClusterRole | Cluster-wide | Permissions across all namespaces |
| RoleBinding | Namespace | Binds Role to User/SA in namespace |
| ClusterRoleBinding | Cluster-wide | Binds ClusterRole cluster-wide |

### Role and RoleBinding Example

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-readonly
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "pods/status", "services"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/exec"]   # allow exec for debugging
    verbs: ["create"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: production
subjects:
  - kind: User
    name: john@company.com
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: dev-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-readonly
  apiGroup: rbac.authorization.k8s.io
```

### Verify Permissions

```bash
kubectl auth can-i get pods \
  --as=john@company.com -n production    # yes

kubectl auth can-i delete pods \
  --as=john@company.com -n production    # no

kubectl auth can-i --list \
  --as=john@company.com -n production    # list all
```

### ServiceAccount for Pods

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: production
  annotations:
    # AWS IRSA — pod assumes AWS IAM role
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/app-role

---
# Use in deployment
spec:
  serviceAccountName: app-service-account
```

### Pod Security Context

```yaml
spec:
  securityContext:                    # pod-level
    runAsNonRoot: true                # never run as root!
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
    - name: app
      securityContext:                # container-level
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: [ALL]                 # remove all Linux capabilities
          add: ["NET_BIND_SERVICE"]   # add only what's needed
```

### Security Checklist for Production

```
✅ Enable RBAC (default in modern K8s)
✅ Use ServiceAccounts with minimal permissions
✅ Never use default ServiceAccount for apps
✅ Run containers as non-root
✅ Set readOnlyRootFilesystem: true
✅ Drop all Linux capabilities, add only what's needed
✅ Enable NetworkPolicy to restrict pod-to-pod traffic
✅ Encrypt Secrets at rest in etcd
✅ Use External Secrets Operator for secret management
✅ Enable audit logging
✅ Disable automountServiceAccountToken if not needed
✅ Use Pod Security Standards (Restricted profile)
```

---

## 8. Scaling & Auto-scaling

### HPA — Horizontal Pod Autoscaler

Scales the NUMBER of pods based on metrics.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70      # scale up if avg CPU > 70%
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 4                    # add max 4 pods at once
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300 # wait 5min before scale-down
      policies:
        - type: Percent
          value: 10                   # remove max 10% pods at once
          periodSeconds: 60
```

### VPA — Vertical Pod Autoscaler

Adjusts CPU/memory REQUESTS of pods (restarts pods to apply).

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"     # Auto=restarts, Off=recommendations only
  resourcePolicy:
    containerPolicies:
      - containerName: my-app
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: "4"
          memory: 4Gi
```

### KEDA — Event-Driven Autoscaler

Scales based on external event sources. Can scale **to ZERO**.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer-scaler
spec:
  scaleTargetRef:
    name: order-processor
  minReplicaCount: 0              # scale to ZERO when no messages!
  maxReplicaCount: 50
  triggers:
    # Scale based on Kafka consumer lag
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        consumerGroup: order-processors
        topic: orders
        lagThreshold: "100"
    # Cron-based scaling
    - type: cron
      metadata:
        timezone: Asia/Kolkata
        start: "0 8 * * 1-5"     # 8am weekdays
        end: "0 20 * * 1-5"      # 8pm weekdays
        desiredReplicas: "10"
```

### Scaler Comparison

| Feature | HPA | VPA | KEDA |
|---|---|---|---|
| Scales | Pod count | Pod size | Pod count |
| Scale to zero | No | No | Yes |
| Based on | CPU/Memory | Usage history | External events |
| Best for | Web APIs | Right-sizing | Queue consumers |
| Restarts pods | No | Yes | No |

> **Warning:** Do NOT use HPA + VPA together on the same resource targets — they conflict. HPA + KEDA work well together.

### PodDisruptionBudget

Protects application during voluntary disruptions (node drain, upgrades).

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 2           # always keep 2 pods running
  # OR:
  # maxUnavailable: 1       # allow max 1 pod down at a time
  selector:
    matchLabels:
      app: my-app
```

### Node Management

```bash
kubectl cordon node-1                           # stop scheduling to node
kubectl drain node-1 \
  --ignore-daemonsets \
  --delete-emptydir-data                        # safely evict pods
kubectl uncordon node-1                         # re-enable scheduling
kubectl taint nodes node-1 key=value:NoSchedule # prevent scheduling
kubectl taint nodes node-1 key=value:NoSchedule- # remove taint
```

---

## 9. Helm

### What is Helm?

Helm is the **package manager for Kubernetes**. A Chart is a package of K8s manifests with templating. Values files parameterize charts for different environments. Helm manages releases and rollbacks.

### Chart Structure

```
my-app/
├── Chart.yaml               # chart metadata (name, version, description)
├── values.yaml              # default values
├── values-staging.yaml      # staging overrides
├── values-prod.yaml         # production overrides
└── templates/
    ├── _helpers.tpl         # reusable template functions
    ├── deployment.yaml      # uses {{ .Values.* }}
    ├── service.yaml
    ├── ingress.yaml
    ├── hpa.yaml
    ├── configmap.yaml
    ├── serviceaccount.yaml
    └── NOTES.txt            # post-install instructions
```

### Chart.yaml

```yaml
apiVersion: v2
name: my-app
description: My Application
type: application
version: 1.2.0        # chart version
appVersion: "2.5.1"   # application version
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

### values.yaml

```yaml
replicaCount: 2

image:
  repository: myregistry/my-app
  tag: "1.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8080

ingress:
  enabled: true
  host: app.example.com

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

hpa:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  cpuTarget: 70
```

### Template Example

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          {{- if .Values.hpa.enabled }}
          readinessProbe:
            httpGet:
              path: /health
              port: {{ .Values.service.port }}
          {{- end }}
```

### Helm Commands

```bash
# Install
helm install my-app ./my-app \
  -f values-prod.yaml \
  --namespace production \
  --create-namespace

# Upgrade (--atomic rolls back automatically if fails)
helm upgrade my-app ./my-app \
  -f values-prod.yaml \
  --namespace production \
  --atomic \
  --timeout 5m

# Rollback
helm rollback my-app 1 --namespace production

# Diff (requires helm-diff plugin)
helm diff upgrade my-app ./my-app -f values-prod.yaml

# Dry run / preview YAML
helm template my-app ./my-app -f values-prod.yaml

# Lint
helm lint ./my-app

# List and history
helm list -n production
helm history my-app -n production

# Uninstall
helm uninstall my-app -n production
```

---

## 10. Troubleshooting & Debugging

### CrashLoopBackOff Debugging Steps

```bash
# Step 1: Check pod status
kubectl get pods -n production
# NAME             READY  STATUS             RESTARTS
# my-app-xyz       0/1    CrashLoopBackOff   5

# Step 2: Describe pod for events and exit code
kubectl describe pod my-app-xyz -n production
# Look for: Events section, Last State exit code

# Exit codes:
# 0   → clean exit (logic bug)
# 1   → application error (check logs)
# 137 → OOMKilled (increase memory limit)
# 139 → segfault
# 143 → SIGTERM (graceful shutdown)

# Step 3: Logs from previous crashed container
kubectl logs my-app-xyz -n production --previous
kubectl logs my-app-xyz -n production -f       # follow live

# Step 4: Events sorted by time
kubectl get events -n production \
  --sort-by='.lastTimestamp'

# Step 5: Resource usage
kubectl top pod my-app-xyz -n production
kubectl top nodes

# Step 6: Exec into running pod
kubectl exec -it my-app-xyz -n production -- /bin/bash

# Step 7: Debug with ephemeral container (distroless images)
kubectl debug -it my-app-xyz \
  --image=busybox:latest \
  --target=my-app \
  -n production

# Step 8: Run debug copy of pod
kubectl debug my-app-xyz \
  --copy-to=debug-pod \
  --image=my-app:debug \
  -n production
```

### Service Not Routing — Debug Steps

```bash
# Step 1: Check endpoints (KEY check!)
kubectl get endpoints my-service -n production
# If NONE → selector mismatch!

# Step 2: Compare selector vs pod labels
kubectl get svc my-service -o yaml | grep -A5 selector
kubectl get pods --show-labels -n production

# Step 3: Test DNS resolution from inside cluster
kubectl run debug --image=busybox -it --rm -- sh
nslookup my-service.production.svc.cluster.local
wget -qO- http://my-service.production.svc.cluster.local:8080/health

# Step 4: Test direct pod IP (bypass service)
kubectl get pods -o wide -n production    # get pod IP
# Then test pod IP directly

# Step 5: Port-forward for local testing
kubectl port-forward svc/my-service 8080:8080 -n production
curl http://localhost:8080/health

# Step 6: Check NetworkPolicy
kubectl get networkpolicy -n production
kubectl describe networkpolicy -n production
```

### Common Issues and Fixes

| Status | Cause | Fix |
|---|---|---|
| `CrashLoopBackOff` | App crash on start | Check `logs --previous` |
| `OOMKilled` (exit 137) | Memory limit hit | Increase memory limit |
| `ImagePullBackOff` | Wrong tag/no pull secret | Check image name, add imagePullSecret |
| `Pending` | No node fits | Check resources, taints, affinity |
| `Terminating` (stuck) | Finalizers | `kubectl patch pod -p '{"metadata":{"finalizers":[]}}` |
| Endpoints empty | Label mismatch | Fix selector in Service spec |
| `0/1 Ready` | Readiness probe failing | Check app health endpoint |

### Production Ops Commands

```bash
# Force delete stuck pod
kubectl delete pod stuck-pod --force --grace-period=0

# Get all resources in namespace
kubectl get all -n production

# Watch pods in real time
kubectl get pods -A -w

# Top resource consumers
kubectl top pods -n production --sort-by=memory

# Copy files from pod
kubectl cp my-pod:/app/logs/app.log ./app.log

# Apply with dry-run (validate before apply)
kubectl apply -f deployment.yaml --dry-run=server
```

---

## 11. Interview Q&A

### Q1: What is the difference between a Pod, ReplicaSet, and Deployment?

**Answer:**
- **Pod** — smallest deployable unit, contains one or more containers, ephemeral
- **ReplicaSet** — maintains N replicas of a pod, but no rolling update support
- **Deployment** — manages ReplicaSets, enables rolling updates, rollbacks, declarative config

Always use Deployment. Deployment creates RS(v1) → Pods. On update creates RS(v2), scales up while scaling down RS(v1). Old RS kept for rollback.

---

### Q2: Explain liveness vs readiness vs startup probes.

**Answer:**
- **Startup probe** — disables liveness/readiness until app starts. Once passes once, hands off to liveness. Critical for slow JVM/Spring Boot apps.
- **Readiness probe** — controls **traffic**. Failure removes pod from Service endpoints. Recovery adds it back. Check downstream dependencies, warmup.
- **Liveness probe** — controls **restart**. Failure kills and restarts container. Use for deadlocks, infinite loops, corrupted state.

Key: readiness = traffic gate. liveness = restart trigger. startup = startup gate.

---

### Q3: What is etcd and why is it critical?

**Answer:**
etcd is a distributed key-value store holding ALL cluster state. It is the source of truth. Only the API Server reads/writes etcd directly. Without etcd, the cluster cannot accept changes or recover from failures.

Best practices: backup regularly with `etcdctl snapshot save`, run in odd-numbered clusters (3/5/7) for quorum, enable encryption at rest.

---

### Q4: How does Kubernetes achieve zero-downtime deployments?

**Answer:**
1. `RollingUpdate` strategy with `maxUnavailable: 0` — no pods killed before new ones ready
2. `maxSurge: 1` — one extra pod runs during transition
3. Readiness probe — new pod not added to Service until it passes
4. PodDisruptionBudget — prevents too many pods removed at once
5. Controller scales up new ReplicaSet while scaling down old one, checking readiness at each step

---

### Q5: What is the difference between ConfigMap and Secret?

**Answer:**
ConfigMap stores non-sensitive config (URLs, flags). Secret stores sensitive data (passwords, tokens). Both are key-value stores. Key difference: Secrets are base64 encoded — NOT encrypted by default. To secure: enable Encryption at Rest in etcd, use External Secrets Operator, restrict with RBAC, prefer volume mounts over env vars.

---

### Q6: Explain RBAC in Kubernetes.

**Answer:**
RBAC has 4 objects: Role (permissions in namespace), ClusterRole (cluster-wide), RoleBinding (binds Role to User/SA in namespace), ClusterRoleBinding (binds ClusterRole cluster-wide).

Flow: Who (ServiceAccount/User/Group) can do What (verbs: get, list, create, delete) on Which resources (pods, deployments) in Where (namespace).

Always follow least-privilege. Verify: `kubectl auth can-i get pods --as=username -n namespace`.

---

### Q7: What is a StatefulSet? When do you use it?

**Answer:**
StatefulSet manages stateful applications needing: stable pod names (pod-0, pod-1), stable network identity (DNS per pod), ordered start/stop, persistent storage per pod (via volumeClaimTemplates).

Use for: databases (MySQL, PostgreSQL), message brokers (Kafka, RabbitMQ), distributed systems (Elasticsearch, Cassandra, ZooKeeper).

Requires a headless Service (`clusterIP: None`) for stable DNS.

---

### Q8: How do you troubleshoot a CrashLoopBackOff pod?

**Answer:**
1. `kubectl describe pod` → check Events and Last State exit code
2. Exit code 137 = OOMKilled (increase memory), exit code 1 = app error
3. `kubectl logs pod --previous` → logs from last crashed container
4. `kubectl get events --sort-by=lastTimestamp` → cluster events
5. `kubectl top pod` → check resource usage
6. Common fixes: increase memory limit, add startupProbe for slow apps, check missing Secret/ConfigMap

---

### Q9: What is the difference between HPA, VPA, and KEDA?

**Answer:**
- **HPA** — scales pod COUNT based on CPU/memory. Cannot scale to zero. Best for stateless web APIs.
- **VPA** — adjusts pod CPU/memory REQUESTS. Requires pod restart. Good for right-sizing. Don't use with HPA on same targets.
- **KEDA** — scales based on external events (Kafka lag, queue depth). Can scale TO ZERO. Best for queue consumers.

HPA + KEDA work well together. HPA + VPA on same resources = conflict.

---

### Q10: What happens when a node goes down?

**Answer:**
1. kubelet stops sending heartbeats to API Server
2. Node controller marks node as NotReady after ~40 seconds
3. After node-eviction-timeout (default 5 min), pods marked for eviction
4. Scheduler reschedules pods onto healthy nodes
5. New pods start — fast if images cached
6. Services automatically route to new pod IPs once readiness passes

This is automatic. Kubernetes self-heals. PodDisruptionBudget ensures minimum available pods throughout.

---

### Q11: What is a DaemonSet?

**Answer:**
DaemonSet ensures exactly ONE pod runs on every node (or a subset based on nodeSelector). Auto-adds to new nodes. Use for: log collectors (Fluentd, Filebeat), monitoring agents (Prometheus node-exporter), network plugins (Calico, Weave), storage drivers. Unlike Deployments, you don't specify replica count — K8s manages it per node.

---

### Q12: How does Service discovery work in Kubernetes?

**Answer:**
Every Service gets a stable DNS name: `service-name.namespace.svc.cluster.local`. CoreDNS runs as a pod in `kube-system` namespace and handles DNS resolution. Pods get DNS configured automatically via `/etc/resolv.conf`. kube-proxy maintains iptables/IPVS rules mapping Service ClusterIP → Pod IPs. When pods come and go, endpoints are updated automatically and iptables rules refreshed.

---

## 12. kubectl Quick Reference

### Essential Commands

```bash
# ── Context ───────────────────────────────────────
kubectl config get-contexts
kubectl config use-context prod
kubectl config current-context

# ── Namespaces ────────────────────────────────────
kubectl get namespaces
kubectl create namespace staging
kubectl delete namespace staging

# ── Pods ──────────────────────────────────────────
kubectl get pods -A                           # all namespaces
kubectl get pods -o wide                      # with node/IP
kubectl get pods -l app=my-app                # by label
kubectl describe pod <pod>
kubectl logs <pod> -f                         # follow logs
kubectl logs <pod> --previous                 # crashed logs
kubectl logs <pod> -c <container>             # specific container
kubectl exec -it <pod> -- /bin/bash
kubectl exec -it <pod> -c <container> -- sh
kubectl delete pod <pod> --force --grace-period=0
kubectl top pods --sort-by=cpu

# ── Deployments ───────────────────────────────────
kubectl get deployments
kubectl apply -f deployment.yaml
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>
kubectl rollout undo deployment/<name> --to-revision=2
kubectl scale deployment <name> --replicas=5
kubectl set image deployment/<name> app=app:2.0

# ── Services ──────────────────────────────────────
kubectl get services
kubectl get endpoints
kubectl describe service <name>
kubectl port-forward svc/<name> 8080:80

# ── ConfigMaps & Secrets ──────────────────────────
kubectl get configmaps
kubectl get secrets
kubectl create configmap app-config --from-file=./config
kubectl create secret generic db-creds \
  --from-literal=password=mypassword
kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 -d

# ── Nodes ─────────────────────────────────────────
kubectl get nodes
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets
kubectl uncordon <node>
kubectl top nodes

# ── Debugging ─────────────────────────────────────
kubectl get events --sort-by='.lastTimestamp'
kubectl auth can-i get pods --as=user -n ns
kubectl debug -it <pod> --image=busybox --target=<container>

# ── YAML Output ───────────────────────────────────
kubectl get pod <pod> -o yaml
kubectl get deployment <name> -o json
kubectl explain deployment.spec.strategy

# ── Apply / Delete ────────────────────────────────
kubectl apply -f .                            # all files in dir
kubectl apply -f . --dry-run=server           # validate only
kubectl delete -f deployment.yaml
kubectl delete pod -l app=my-app              # by label
```

### Resource Abbreviations

| Full Name | Short |
|---|---|
| pods | po |
| deployments | deploy |
| replicasets | rs |
| services | svc |
| namespaces | ns |
| configmaps | cm |
| persistentvolumeclaims | pvc |
| persistentvolumes | pv |
| horizontalpodautoscalers | hpa |
| nodes | no |

---

## 📊 Summary — Key Concepts at a Glance

```
Control Plane:  API Server → etcd → Scheduler → Controller Manager
Worker Node:    kubelet → kube-proxy → Container Runtime → Pods

Workloads:      Deployment (stateless) | StatefulSet (stateful) | DaemonSet (per-node)
Networking:     ClusterIP → NodePort → LoadBalancer → Ingress → NetworkPolicy
Storage:        StorageClass → PV → PVC → Volume Mount
Config:         ConfigMap (non-sensitive) | Secret (sensitive, use External Secrets)
Security:       RBAC → ServiceAccount → PodSecurityContext → NetworkPolicy
Scaling:        HPA (count) | VPA (size) | KEDA (events) | PDB (protection)
Packaging:      Helm Charts → Values → Templates → Releases
```

---

*Notes compiled for interview preparation — Kubernetes v1.28+*
