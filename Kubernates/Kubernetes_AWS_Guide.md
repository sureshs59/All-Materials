# Kubernetes on AWS — Complete Beginner's Guide

> Everything you need to learn Kubernetes with AWS EKS, from first principles through production deployment. Includes architecture diagrams, all commands, and practical examples.

---

## Table of Contents

1. [What is Kubernetes?](#1-what-is-kubernetes)
2. [Cluster Architecture — The Two Planes](#2-cluster-architecture--the-two-planes)
3. [Core Building Blocks](#3-core-building-blocks)
   - [Pod](#31-pod--the-smallest-unit)
   - [Deployment](#32-deployment--manages-pod-replicas)
   - [Service](#33-service--stable-network-endpoint)
   - [Namespace](#34-namespace--virtual-cluster)
   - [ConfigMap and Secret](#35-configmap--secret)
4. [Traffic Flow — Request to Pod](#4-traffic-flow--request-to-pod)
5. [Auto Scaling — Two Levels](#5-auto-scaling--two-levels)
6. [Getting Started with AWS EKS](#6-getting-started-with-aws-eks)
7. [Essential kubectl Commands](#7-essential-kubectl-commands)
8. [Complete YAML Reference](#8-complete-yaml-reference)
9. [Kubernetes vs Traditional Deployment](#9-kubernetes-vs-traditional-deployment)
10. [Production Checklist](#10-production-checklist)
11. [Quick Reference](#11-quick-reference)

---

## 1. What is Kubernetes?

### Definition

Kubernetes (K8s) is a **container orchestration system** that automatically manages, scales, and keeps your containerised applications running. Instead of manually starting Docker containers on servers, you tell Kubernetes *what you want* and it figures out *how to make it happen*.

> **The manager analogy:** "Kubernetes is like a restaurant manager. You say 'I need 3 waiters at all times.' The manager hires when someone quits, fires when it's slow, and moves staff around. You never manage individual people — just the rules."

### The Problem It Solves

Running containers manually on VMs breaks when:
- Your app crashes at 3 AM — no one restarts it
- Traffic spikes — you cannot scale fast enough
- A server dies — the containers on it are gone
- You deploy a new version — downtime while you swap containers

Kubernetes solves all of these automatically.

### Docker vs Kubernetes

| | Docker | Kubernetes |
|--|--------|-----------|
| **Scale** | One container on one machine | Thousands of containers across hundreds of machines |
| **Self-healing** | Manual restart if container crashes | Automatic — K8s restarts crashed containers immediately |
| **Scaling** | Manual | Automatic based on CPU/memory/custom metrics |
| **Updates** | Manual — downtime during swap | Rolling update — zero downtime |
| **Service discovery** | Manual configuration | Built-in DNS for every service |

### Kubernetes on AWS — Amazon EKS

Amazon EKS (Elastic Kubernetes Service) runs and manages Kubernetes for you:

- AWS manages the **control plane** (the cluster brain) — you never touch it
- You manage only the **worker nodes** (EC2 instances running your apps)
- EKS integrates with AWS services: ELB, ECR, IAM, EBS, RDS, CloudWatch
- Cost: $0.10/hour for the control plane + EC2 instance costs for nodes

### Why YAML Files?

You describe your **desired state** in YAML files. Kubernetes continuously reconciles actual state to match it:

```yaml
# "I always want 3 replicas of this app running"
spec:
  replicas: 3
```

If 1 pod crashes → K8s sees actual=2, desired=3 → starts a new pod automatically.

---

## 2. Cluster Architecture — The Two Planes

### Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                           │
│                                                                     │
│  ┌──────────────────────────┐    ┌───────────────────────────────┐  │
│  │      Control Plane        │    │         Worker Nodes          │  │
│  │      (the brains)         │    │      (run your apps)          │  │
│  │                           │    │                               │  │
│  │  ┌─────────────────────┐ │    │  ┌─────────────┐  ┌────────┐ │  │
│  │  │     API Server       │ │    │  │   Node 1    │  │ Node 2 │ │  │
│  │  │ (front door for all  │ │    │  │  EC2 t3.sm  │  │EC2 t3  │ │  │
│  │  │  kubectl commands)   │ ├────┤  │             │  │        │ │  │
│  │  └─────────────────────┘ │    │  │ Pod: app v1 │  │Pod: app│ │  │
│  │                           │    │  │ Pod: app v1 │  │Pod: db │ │  │
│  │  ┌──────────┐ ┌────────┐ │    │  │   kubelet   │  │kubelet │ │  │
│  │  │   etcd   │ │Sched-  │ │    │  │  kube-proxy │  │k-proxy │ │  │
│  │  │(state DB)│ │uler    │ │    │  └─────────────┘  └────────┘ │  │
│  │  └──────────┘ └────────┘ │    │                               │  │
│  │                           │    │  ┌──────────────────────────┐ │  │
│  │  ┌─────────────────────┐ │    │  │  Auto Scaling Group (AWS) │ │  │
│  │  │Controller Manager   │ │    │  │  adds/removes EC2 nodes   │ │  │
│  │  │(keeps desired state)│ │    │  └──────────────────────────┘ │  │
│  │  └─────────────────────┘ │    └───────────────────────────────┘  │
│  │                           │                                       │
│  │  ┌─────────────────────┐ │                                       │
│  │  │Cloud Controller(AWS)│ │                                       │
│  │  │creates ELBs, EBS,   │ │                                       │
│  │  │routes on AWS        │ │                                       │
│  │  └─────────────────────┘ │                                       │
│  └──────────────────────────┘                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Control Plane Components

| Component | Role |
|-----------|------|
| **API Server** | The front door. Every `kubectl` command, every component communication goes through here |
| **etcd** | The cluster's database. Stores the entire desired state — every resource definition |
| **Scheduler** | Decides which worker node each new pod runs on (based on resources, affinity, taints) |
| **Controller Manager** | Watches actual state and corrects drift — restarts crashed pods, scales deployments |
| **Cloud Controller (AWS)** | Talks to AWS APIs — creates ELBs when you create a LoadBalancer Service, provisions EBS volumes |

### Worker Node Components

| Component | Role |
|-----------|------|
| **kubelet** | Agent on every node. Talks to the API Server. Starts/stops pods as instructed |
| **kube-proxy** | Handles network rules on each node — routes traffic to the right pod |
| **Container runtime** | Usually containerd. Actually runs Docker containers |

### On AWS EKS

```
You manage:                    AWS manages:
  ┌─────────────────┐            ┌────────────────────────┐
  │  Worker Nodes   │            │    Control Plane        │
  │  (EC2 instances)│            │  (API Server, etcd,     │
  │  Node groups    │            │   Scheduler, Controllers)│
  │  Auto Scaling   │            │   99.95% SLA uptime     │
  └─────────────────┘            └────────────────────────┘
```

You pay $0.10/hour for the control plane. AWS handles its availability, backups, and upgrades.

---

## 3. Core Building Blocks

### 3.1 Pod — The Smallest Unit

A Pod wraps one or more containers that always run together on the same node. They share the same network (IP address) and storage.

> **The shipping container analogy:** "A Pod is like a shipping container. Inside you put your app (and maybe a helper sidecar). The whole unit moves together."

**Key facts:**
- Every pod gets its own unique IP address inside the cluster
- Pods are **ephemeral** — they can die and be replaced at any time
- You rarely create pods directly — Deployments manage them for you
- Pods on the same node can communicate via localhost (same network namespace)

**Minimal pod YAML:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  containers:
  - name: my-app
    image: nginx:latest
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

**Pod commands:**
```bash
# Create a pod directly (for learning — use Deployment in production)
kubectl run my-app --image=nginx:latest --port=80

# See all running pods
kubectl get pods

# See pods with node assignment and IP
kubectl get pods -o wide

# Detailed info including events (use for debugging)
kubectl describe pod my-app

# See live logs
kubectl logs -f my-app

# Shell into a running pod
kubectl exec -it my-app -- /bin/bash

# Delete a pod (Deployment will recreate it immediately)
kubectl delete pod my-app
```

---

### 3.2 Deployment — Manages Pod Replicas

A Deployment is a blueprint that says "always run N copies of this pod." It handles:
- Self-healing — if a pod crashes, immediately starts a replacement
- Rolling updates — new version rolls in while old one rolls out (zero downtime)
- Rollbacks — instantly revert to the previous version

> **The staffing contract analogy:** "A Deployment is a staffing contract: 'Always have 3 waiters. If one quits, hire another immediately.'"

**Complete deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
    version: "1.0.0"
spec:
  replicas: 3                    # always run 3 pods
  selector:
    matchLabels:
      app: my-app               # manages pods with this label
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1               # allow 1 extra pod during update
      maxUnavailable: 0         # never have 0 pods available
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: 123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:1.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        # Health checks — K8s uses these to know if the pod is alive
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60   # wait 60s for Spring Boot to start
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 5
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
```

**Deployment commands:**
```bash
# Apply the deployment
kubectl apply -f deployment.yaml

# Watch the rollout status
kubectl rollout status deployment/my-app

# Scale immediately (no YAML edit needed)
kubectl scale deployment my-app --replicas=6

# Zero-downtime update to new version
kubectl set image deployment/my-app my-app=my-app:2.0.0

# Watch pods rolling update in real time
kubectl get pods -w

# Rollback to previous version (instant)
kubectl rollout undo deployment/my-app

# See rollout history
kubectl rollout history deployment/my-app

# Rollback to specific version
kubectl rollout undo deployment/my-app --to-revision=2
```

---

### 3.3 Service — Stable Network Endpoint

Pods get new IPs every time they are recreated. A Service gives you a stable DNS name and IP that never changes. It load-balances traffic across all matching pods automatically.

**Three service types:**

| Type | Accessible from | Use case |
|------|----------------|----------|
| **ClusterIP** (default) | Inside cluster only | Pod-to-pod communication |
| **NodePort** | Outside via node IP + port | Development/testing only |
| **LoadBalancer** | Internet via AWS ELB | Exposing app to users |

**service.yaml — LoadBalancer (creates AWS ELB):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
  namespace: production
spec:
  type: LoadBalancer           # AWS provisions an ELB automatically
  selector:
    app: my-app                # routes to pods with this label
  ports:
  - protocol: TCP
    port: 80                   # external port users call
    targetPort: 8080           # pod's internal port
```

**ClusterIP (internal communication):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-service
spec:
  type: ClusterIP              # internal only
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
```

**Service commands:**
```bash
# Apply the service
kubectl apply -f service.yaml

# Get the public URL (takes 1-2 minutes to provision ELB)
kubectl get service my-app-service
# EXTERNAL-IP: abc123.us-east-1.elb.amazonaws.com

# Test it
curl http://abc123.us-east-1.elb.amazonaws.com/api/health

# From inside another pod — use DNS name
curl http://my-app-service.production.svc.cluster.local/api/health
# Format: <service-name>.<namespace>.svc.cluster.local
```

---

### 3.4 Namespace — Virtual Cluster

Namespaces split one K8s cluster into separate virtual environments — dev, staging, prod — with different teams and resource limits.

> **The office floor analogy:** "A namespace is like a floor of an office building. Floor 1 = dev team. Floor 2 = prod team. They share the building (cluster) but don't interfere with each other."

```bash
# Create namespaces for each environment
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace production

# Deploy to a specific namespace
kubectl apply -f deployment.yaml -n production

# See all resources in a namespace
kubectl get all -n production

# Default namespaces K8s creates
kubectl get namespaces
# default        — where resources go if you don't specify
# kube-system    — K8s internal components (DNS, metrics, etc.)
# kube-public    — readable by all users
# kube-node-lease — node heartbeats

# Set a default namespace for your session
kubectl config set-context --current --namespace=production
```

**Resource quotas per namespace (limit resource usage):**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
```

---

### 3.5 ConfigMap & Secret

Never hardcode config in your Docker image. ConfigMaps store non-sensitive config; Secrets store sensitive values.

**ConfigMap — non-sensitive configuration:**
```bash
# Create from literals
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=INFO \
  --from-literal=SERVER_PORT=8080 \
  --from-literal=CACHE_TTL=300

# Or from a file
kubectl create configmap app-config --from-file=application.properties
```

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "INFO"
  SERVER_PORT: "8080"
  SPRING_PROFILES_ACTIVE: "production"
```

**Secret — sensitive values:**
```bash
# Create a secret (base64 encoded automatically)
kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD=mypassword \
  --from-literal=JWT_SECRET=myjwtsecretkey \
  --from-literal=API_KEY=sk-myapikey
```

```yaml
# secret.yaml (stringData auto-encodes to base64)
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  DB_PASSWORD: "mypassword"
  JWT_SECRET: "myjwtsecretkey"
```

**Use in deployment.yaml:**
```yaml
containers:
- name: my-app
  image: my-app:1.0.0
  env:
  # From ConfigMap
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: LOG_LEVEL
  # From Secret
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: DB_PASSWORD
  # Load all ConfigMap values at once
  envFrom:
  - configMapRef:
      name: app-config
```

---

## 4. Traffic Flow — Request to Pod

```
User's browser
    │
    │  HTTPS request
    ▼
AWS ELB (Elastic Load Balancer)
    │  Created automatically when Service type=LoadBalancer
    │  Distributes across all available Kubernetes nodes
    ▼
Ingress Controller (optional but recommended)
    │  Routes based on URL path and hostname:
    │  /api     → api-service
    │  /admin   → admin-service
    │  /static  → static-service
    ▼
Kubernetes Service
    │  Stable DNS name: my-app-service.production.svc.cluster.local
    │  Load-balances across all healthy pods with matching label
    │  If a pod dies → immediately stops sending traffic to it
    ▼
     ┌──────────────┬──────────────┐
     ▼              ▼              ▼
  Pod 1          Pod 2          Pod 3
  (running)      (running)      (running)
  app:8080       app:8080       app:8080
```

**Ingress — URL-based routing:**
```yaml
# ingress.yaml — routes different paths to different services
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    kubernetes.io/ingress.class: "alb"        # use AWS ALB
    alb.ingress.kubernetes.io/scheme: "internet-facing"
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

---

## 5. Auto Scaling — Two Levels

### Level 1 — HPA (Horizontal Pod Autoscaler)

Adds more **pods** when CPU or memory is high. Fast — happens in seconds.

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70     # scale when CPU > 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80     # or memory > 80%
```

```bash
# Quick HPA creation
kubectl autoscale deployment my-app \
  --min=2 --max=10 --cpu-percent=70

# Watch HPA in action
kubectl get hpa -w
```

### Level 2 — Cluster Autoscaler (AWS)

Adds more **EC2 nodes** when pods cannot be scheduled (all nodes are full). Slower — takes 2–5 minutes.

```
Traffic spike → HPA adds more pods → No room on existing nodes
  → Pods stuck in "Pending" state
  → Cluster Autoscaler detects pending pods
  → Calls AWS Auto Scaling Group API: "add a node"
  → New EC2 node joins cluster in 2-5 minutes
  → Pending pods scheduled to new node
  → Traffic handled

Traffic drops → HPA removes pods → Nodes become underutilised
  → Cluster Autoscaler removes empty nodes
  → Cost savings automatic
```

**Install Cluster Autoscaler on EKS:**
```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --set autoDiscovery.clusterName=my-cluster \
  --set awsRegion=us-east-1
```

### Scaling Summary

| | HPA | Cluster Autoscaler |
|--|-----|-------------------|
| **What scales** | Pods (containers) | EC2 nodes (machines) |
| **Trigger** | CPU/memory/custom metric | Pods stuck in Pending |
| **Speed** | Seconds | 2–5 minutes |
| **AWS integration** | Metrics from CloudWatch | Auto Scaling Groups |
| **When needed** | Almost always | When nodes fill up |

---

## 6. Getting Started with AWS EKS

### Step 1 — Install tools

```bash
# macOS
brew install awscli kubectl eksctl

# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install kubectl /usr/local/bin/kubectl

curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

### Step 2 — Configure AWS credentials

```bash
aws configure
# AWS Access Key ID:     [your key]
# AWS Secret Access Key: [your secret]
# Default region name:   us-east-1
# Default output format: json
```

### Step 3 — Create EKS cluster (15–20 minutes)

```bash
eksctl create cluster \
  --name my-cluster \
  --region us-east-1 \
  --nodegroup-name my-nodes \
  --node-type t3.small \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 4 \
  --managed

# eksctl automatically configures kubectl — verify:
kubectl get nodes
# NAME                           STATUS   ROLES    AGE
# ip-192-168-1-1.ec2.internal    Ready    <none>   2m
# ip-192-168-1-2.ec2.internal    Ready    <none>   2m
```

### Step 4 — Push your Docker image to ECR

```bash
# Create ECR repository
aws ecr create-repository \
  --repository-name my-springboot-app \
  --region us-east-1

# Set variables
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECR_URI=$AWS_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com
export IMAGE=$ECR_URI/my-springboot-app:1.0.0

# Login Docker to ECR
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin $ECR_URI

# Build, tag, and push
mvn clean package -DskipTests
docker build -t my-springboot-app:1.0.0 .
docker tag my-springboot-app:1.0.0 $IMAGE
docker push $IMAGE
```

### Step 5 — Create Kubernetes YAML files

```
k8s/
├── deployment.yaml      # how many pods, which image, health checks
├── service.yaml         # how to expose the pods (creates ELB)
├── configmap.yaml       # non-sensitive environment config
├── secret.yaml          # sensitive values (passwords, keys)
└── hpa.yaml             # auto-scaling rules
```

### Step 6 — Deploy everything

```bash
# Apply all files in the k8s/ folder at once
kubectl apply -f k8s/

# Watch pods start up (Ctrl+C to stop watching)
kubectl get pods -w

# Watch rollout complete
kubectl rollout status deployment/my-springboot-app

# Get your public URL (takes 1-2 minutes to provision ELB)
kubectl get service my-springboot-app-service
# EXTERNAL-IP: abc123.us-east-1.elb.amazonaws.com

# Test it
curl http://abc123.us-east-1.elb.amazonaws.com/actuator/health
# {"status":"UP"}
```

### Update to new version (zero downtime)

```bash
# Build and push new image
mvn clean package -DskipTests
docker build -t my-springboot-app:2.0.0 .
docker tag my-springboot-app:2.0.0 $ECR_URI/my-springboot-app:2.0.0
docker push $ECR_URI/my-springboot-app:2.0.0

# K8s rolls pods one by one — users never see downtime
kubectl set image deployment/my-springboot-app \
  my-springboot-app=$ECR_URI/my-springboot-app:2.0.0

# If something breaks — instant rollback
kubectl rollout undo deployment/my-springboot-app
```

### Clean up (avoid charges)

```bash
# Delete all K8s resources
kubectl delete -f k8s/

# Delete the entire cluster (deletes EC2 nodes, VPC, everything)
eksctl delete cluster --name my-cluster --region us-east-1
```

---

## 7. Essential kubectl Commands

### Get information

```bash
# See everything running
kubectl get all
kubectl get all -n production

# Pods with IPs and node assignment
kubectl get pods -o wide

# Watch pods in real time (Ctrl+C to stop)
kubectl get pods -w

# Detailed info including events (best debugging tool)
kubectl describe pod <pod-name>
kubectl describe deployment my-app
kubectl describe service my-service

# Live logs (-f = follow, --tail=100 = last 100 lines)
kubectl logs -f <pod-name>
kubectl logs -f <pod-name> --tail=100

# Logs from previous crashed pod
kubectl logs <pod-name> --previous
```

### Deploy and manage

```bash
# Apply YAML file (creates or updates)
kubectl apply -f deployment.yaml

# Apply all files in a folder
kubectl apply -f k8s/

# Delete resources defined in a file
kubectl delete -f deployment.yaml

# Delete a specific resource
kubectl delete pod my-pod
kubectl delete deployment my-app
kubectl delete service my-service

# Scale replicas directly (no YAML edit)
kubectl scale deployment my-app --replicas=5

# Force restart all pods (picks up new env vars, secrets)
kubectl rollout restart deployment/my-app
```

### Debugging

```bash
# Shell into a running pod
kubectl exec -it <pod-name> -- /bin/bash

# Run a command in a pod without interactive shell
kubectl exec my-pod -- cat /etc/config/application.yml

# Copy files to/from a pod
kubectl cp ./local-file.txt my-pod:/app/config/
kubectl cp my-pod:/app/logs/app.log ./downloaded.log

# Port-forward (access pod directly for testing — bypasses Service)
kubectl port-forward pod/my-pod 8080:8080
# Now: curl http://localhost:8080/api/health
kubectl port-forward service/my-service 8080:80

# See resource usage (requires metrics-server)
kubectl top pods
kubectl top nodes
```

### Context and namespace

```bash
# See all contexts (clusters you can switch between)
kubectl config get-contexts

# Switch to a different cluster
kubectl config use-context my-cluster

# Set default namespace for this session
kubectl config set-context --current --namespace=production

# See current config
kubectl config view
```

---

## 8. Complete YAML Reference

### Full deployment with all production settings

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-springboot-app
  namespace: production
  labels:
    app: my-springboot-app
    version: "2.0.0"
    team: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-springboot-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1               # 1 extra pod allowed during update
      maxUnavailable: 0         # never go below desired replicas
  template:
    metadata:
      labels:
        app: my-springboot-app
    spec:
      # Node affinity — prefer different availability zones
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values: [my-springboot-app]
              topologyKey: topology.kubernetes.io/zone
      containers:
      - name: my-springboot-app
        image: 123456789.dkr.ecr.us-east-1.amazonaws.com/my-springboot-app:2.0.0
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        - name: SERVER_PORT
          value: "8080"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DB_PASSWORD
        envFrom:
        - configMapRef:
            name: app-config
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 5
          failureThreshold: 3
        # Graceful shutdown — wait 30s for in-flight requests
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10"]
      terminationGracePeriodSeconds: 40
```

---

## 9. Kubernetes vs Traditional Deployment

| Concern | Traditional (EC2 + manual) | Kubernetes |
|---------|---------------------------|------------|
| **App crash** | Manual restart or CloudWatch alarm | Automatic restart in seconds |
| **Scale up** | Manually add EC2, deploy, configure ELB | `kubectl scale --replicas=10` |
| **Deploy new version** | SSH into servers one by one, downtime | Rolling update, zero downtime |
| **Rollback** | SSH back in, restart old version | `kubectl rollout undo` |
| **Config management** | SSH, update files, restart | ConfigMap update + rolling restart |
| **Service discovery** | Hard-coded IPs or Route 53 entries | Built-in DNS (service-name.namespace) |
| **Resource utilisation** | One app per server, wasteful | Many pods per node, efficient |
| **Multi-region** | Complex manual setup | Cluster per region, same YAML files |

---

## 10. Production Checklist

### Before going live

- [ ] Set resource `requests` and `limits` on every container
- [ ] Configure `livenessProbe` and `readinessProbe` on every container
- [ ] Use `RollingUpdate` strategy with `maxUnavailable: 0`
- [ ] Set `minReplicas: 2` in HPA (never run only 1 pod)
- [ ] Spread pods across Availability Zones (pod anti-affinity)
- [ ] Use `Namespaces` to separate dev/staging/production
- [ ] Store secrets in AWS Secrets Manager or Vault — not in K8s Secrets directly
- [ ] Set up RBAC — limit what each service account can do
- [ ] Enable CloudWatch logging for all pods
- [ ] Set resource quotas per namespace
- [ ] Use ECR image scanning for vulnerability detection
- [ ] Configure `NetworkPolicy` to restrict pod-to-pod traffic

### Useful monitoring commands

```bash
# See which pods are using the most CPU/memory
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory

# See pending pods (no node available)
kubectl get pods --field-selector=status.phase=Pending

# See failed pods
kubectl get pods --field-selector=status.phase=Failed

# Check if HPA is working
kubectl describe hpa my-app-hpa

# Node capacity and allocated resources
kubectl describe nodes | grep -A 5 "Allocated resources"
```

---

## 11. Quick Reference

### Object types

| Object | Created by | Purpose |
|--------|-----------|---------|
| `Pod` | Deployment | One or more containers running together |
| `Deployment` | You | Manages desired number of pod replicas |
| `Service` | You | Stable network endpoint for pods |
| `Ingress` | You | URL-based routing to services |
| `ConfigMap` | You | Non-sensitive configuration |
| `Secret` | You | Sensitive values (passwords, keys) |
| `Namespace` | You | Virtual cluster isolation |
| `HPA` | You | Auto-scale pods based on metrics |
| `PersistentVolumeClaim` | You | Request EBS storage for stateful apps |

### kubectl cheat sheet

```bash
# Shortcuts
kubectl get po       # pods
kubectl get deploy   # deployments
kubectl get svc      # services
kubectl get ing      # ingresses
kubectl get cm       # configmaps
kubectl get secret   # secrets
kubectl get ns       # namespaces
kubectl get hpa      # horizontal pod autoscalers
kubectl get nodes    # cluster nodes

# -n flag sets namespace
kubectl get pods -n production

# -o yaml shows full YAML of any resource
kubectl get deployment my-app -o yaml

# -o jsonpath extracts specific fields
kubectl get pod my-pod -o jsonpath='{.status.podIP}'
```

### AWS EKS + Kubernetes relationship

```
AWS Account
  └── EKS Cluster (my-cluster)
        ├── Control Plane (managed by AWS — $0.10/hr)
        │     ├── API Server
        │     ├── etcd
        │     ├── Scheduler
        │     └── Controller Manager
        └── Node Group (my-nodes)
              ├── EC2 t3.small (Node 1)  ← you pay for these
              ├── EC2 t3.small (Node 2)
              └── Auto Scaling Group (min:1, max:4)
```

---

*Kubernetes v1.29+ · AWS EKS · kubectl · eksctl · Spring Boot integration*
*All commands tested on Amazon Linux 2 and macOS.*
