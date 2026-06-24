# Kubernetes — Deployments, Services & Autoscaling
### Production-grade deep dive — CareFirst member portal scenario

> **How the three fit together.** A Deployment answers *"how do I run N identical copies of my app and roll out new versions safely?"* A Service answers *"how do those constantly-changing pods get a stable address so other things can reach them?"* Autoscaling answers *"how do I add and remove copies automatically as load changes?"* The Deployment manages pods, the Service gives them a stable front door, and the HorizontalPodAutoscaler tells the Deployment how many pods to run.

---

## Table of Contents
1. Deployments
2. Services
3. Autoscaling (HorizontalPodAutoscaler)
4. How the three compose
5. Version caveat

---

# 1. Deployments

**Definition.** A Deployment is a controller that declaratively manages a set of identical pods, handling rollouts, rollbacks, and self-healing through an underlying ReplicaSet.

**How it works.** You declare a desired state (image, replica count, pod template); the Deployment creates a ReplicaSet, which creates the pods. The controller continuously reconciles actual state toward desired state — if a pod dies, the ReplicaSet replaces it. On an image change, the Deployment creates a *new* ReplicaSet and shifts pods from old to new according to its rollout strategy (RollingUpdate by default), keeping the old ReplicaSet around so you can roll back.

**Why it matters (production impact).** This is the unit of zero-downtime deploys and self-healing. Rolling updates let you ship a new version without taking the service offline; the retained ReplicaSet history means a bad release is one `rollout undo` away rather than a frantic redeploy. For a regulated healthcare portal, that controlled, reversible rollout is also what makes change management auditable.

**ASCII diagram — reconciliation and rolling update:**
```
   kubectl apply -f deployment.yaml
        |
        v
   +------------------+      manages      +-------------------+
   |   Deployment     |----------------->|  ReplicaSet (v2)  |
   | desired: 4 pods  |                  |  desired: 4       |
   | image: portal:v2 |                  +-------------------+
   +------------------+                       |   |   |   |
        |  keeps history                      v   v   v   v
        |                                    pod pod pod pod  (new)
        v
   +-------------------+   scaling down old
   |  ReplicaSet (v1)  |   4 -> 3 -> 2 -> ... -> 0
   |  image: portal:v1 |
   +-------------------+

   ROLLING UPDATE (maxSurge=1, maxUnavailable=0):
   v1: [P][P][P][P]                    (4 ready)
   v2:                [P+]             surge 1 new, wait Ready
   v1: [P][P][P]                       terminate 1 old
   v2:                [P][P+]          ... repeat ...
   v1:                                 (0)
   v2: [P][P][P][P]                    (4 ready)  -- never dropped below 4
```

**YAML:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: member-portal
  labels: { app: member-portal }
spec:
  replicas: 4
  selector:
    matchLabels: { app: member-portal }      # MUST match template labels below
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1            # at most 1 extra pod above desired during rollout
      maxUnavailable: 0      # never drop below desired -> true zero-downtime
  template:
    metadata:
      labels: { app: member-portal }          # selector targets these
    spec:
      containers:
        - name: portal
          image: registry.internal/member-portal:v2
          ports: [{ containerPort: 8080 }]
          resources:
            requests: { cpu: "250m", memory: "512Mi" }   # used by scheduler + HPA
            limits:   { cpu: "1",    memory: "1Gi" }
          readinessProbe:                        # gates traffic + rollout progression
            httpGet: { path: /actuator/health/readiness, port: 8080 }
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:                         # restarts a wedged pod
            httpGet: { path: /actuator/health/liveness, port: 8080 }
            initialDelaySeconds: 30
            periodSeconds: 10
```

**Real scenario (CareFirst member portal).** The portal's Spring Boot frontend runs as a 4-replica Deployment. `maxUnavailable: 0` with `maxSurge: 1` means during a release the cluster always keeps 4 serving pods and adds the new version one at a time — members never hit a downed instance mid-enrollment. The readiness probe points at Spring Boot Actuator's readiness group, so a new pod only receives traffic after the app has actually finished warming up (connection pools, caches). When a v2 release once shipped with a bad config, `kubectl rollout undo deployment/member-portal` reverted to the retained v1 ReplicaSet in seconds.

**Debugging deployments:**
- **Rollout stuck / `kubectl rollout status` hangs** -> new pods aren't passing readiness. With `maxUnavailable: 0` the rollout *cannot* proceed until a new pod is Ready, so a failing readiness probe freezes it. `kubectl describe pod` and check probe failures and events.
- **`CrashLoopBackOff`** -> the container starts then exits/crashes repeatedly. `kubectl logs <pod> --previous` shows the prior crash output. Common causes: missing env/config, failing liveness probe killing a slow-starting app (raise `initialDelaySeconds`), or OOMKill (memory limit too low — check `kubectl describe` for `OOMKilled`).
- **`ImagePullBackOff`** -> wrong image name/tag or missing registry credentials (`imagePullSecrets`).
- **Pods `Pending`** -> unschedulable. Usually insufficient cluster resources to satisfy `requests`, or a node-selector/taint mismatch. `kubectl describe pod` shows the scheduler's reason.
- **Selector/label mismatch** -> `spec.selector` not matching `template.labels` is rejected at apply time.

**Interview Q&A.**
> **Q: Deployment vs ReplicaSet vs Pod?** A Pod is one running instance (one or more co-located containers). A ReplicaSet keeps N identical pods alive. A Deployment manages ReplicaSets to give you versioned rollouts and rollbacks — you almost never create a ReplicaSet directly; you create a Deployment and it manages the ReplicaSet for you.
>
> **Q: How does a rolling update achieve zero downtime?** `maxSurge` lets it add new pods before removing old ones, and `maxUnavailable: 0` guarantees the ready count never drops below desired. New pods only count as available once their readiness probe passes, so traffic shifts only to pods that can actually serve. The old ReplicaSet is retained for instant rollback.

---

# 2. Services

**Definition.** A Service is a stable network abstraction that provides a single, durable address and load-balances traffic across a dynamic set of pods selected by labels.

**How it works.** Pods are ephemeral — they get new IPs every time they're rescheduled. A Service defines a label selector; the control plane continuously maintains the set of matching pod IPs as Endpoints (EndpointSlices). The Service gets a stable virtual IP (ClusterIP) and a DNS name (`name.namespace.svc.cluster.local`); kube-proxy programs the node's networking so traffic to that VIP is load-balanced across the current healthy endpoints.

**Why it matters (production impact).** It decouples callers from pod lifecycle. Pods scale, die, and move, but `claims-service` is always reachable at the same name — this is what makes Deployments and autoscaling safe, since the churn underneath is invisible to clients. Service types also define your exposure boundary (internal-only vs externally reachable), which matters for a regulated system.

**ASCII diagram — selector -> endpoints -> load balancing, and Service types:**
```
   Service: claims-service (ClusterIP 10.96.0.50)
   selector: app=claims
        |
        |  control plane watches pods, keeps EndpointSlice in sync
        v
   Endpoints: [10.0.2.9:8082, 10.0.2.11:8082, 10.0.2.14:8082]
        |              |              |
        v              v              v
     +------+       +------+       +------+
     | pod  |       | pod  |       | pod  |   (label app=claims)
     +------+       +------+       +------+
   pod dies -> removed from Endpoints automatically
   pod added (scale up) -> added to Endpoints automatically

   SERVICE TYPES (exposure boundary):
   ClusterIP     in-cluster only        member-svc -> claims-svc
   NodePort      <nodeIP>:30000-32767   basic external (rarely direct in prod)
   LoadBalancer  cloud LB -> Service    external entry (or via Ingress)
   (Ingress/Gateway sits in front of ClusterIP Services for HTTP routing)
```

**YAML:**
```yaml
# Internal service — how member-service reaches claims-service
apiVersion: v1
kind: Service
metadata:
  name: claims-service
spec:
  type: ClusterIP                 # internal-only; default
  selector:
    app: claims                   # routes to pods with label app=claims
  ports:
    - port: 80                    # the Service's stable port
      targetPort: 8082            # the container's port
---
# Externally exposed front door for the portal (cloud LoadBalancer)
apiVersion: v1
kind: Service
metadata:
  name: member-portal-lb
spec:
  type: LoadBalancer
  selector:
    app: member-portal
  ports:
    - port: 443
      targetPort: 8080
```

**Real scenario (CareFirst member portal).** Internal services (`claims-service`, `eligibility-service`, `benefits-service`) are all `ClusterIP` — unreachable from outside the cluster, satisfying the requirement that PHI-handling services have no public surface. `member-service` calls `http://claims-service/...` by DNS name; as `claims-service` pods autoscale up during enrollment, the Service's EndpointSlice updates automatically and the new pods receive traffic with no caller change. The only externally exposed piece is the portal front-end, fronted by an Ingress/Gateway in front of a ClusterIP Service rather than exposing each service directly.

**Debugging services:**
- **Service resolves but connection refused / times out** -> almost always a selector/label mismatch. `kubectl get endpoints claims-service` — if it's empty, the Service's selector matches no pods (or no pods are Ready). This is the #1 Service bug.
- **Endpoints exist but traffic fails** -> `targetPort` doesn't match the container's actual listening port, or the pod isn't passing readiness (only Ready pods are added to endpoints).
- **DNS not resolving** -> wrong name/namespace. Cross-namespace calls need the FQDN `claims-service.<namespace>.svc.cluster.local`. Test with `kubectl run tmp --rm -it --image=busybox -- nslookup claims-service`.
- **LoadBalancer stuck `<pending>`** -> no cloud LB provisioner (common on bare-metal/local clusters without MetalLB).
- **Intermittent failures during scale-down** -> in-flight requests to terminating pods; pair with `terminationGracePeriodSeconds` and a preStop hook + readiness flip so pods drain before dying.

**Interview Q&A.**
> **Q: How does a Service track which pods to send traffic to?** Through a label selector. The control plane watches pods matching that selector and maintains their IPs in an EndpointSlice; only Ready pods are included. kube-proxy uses that list to load-balance. So pod churn is handled automatically — the Service is a stable indirection over a moving target.
>
> **Q: ClusterIP vs NodePort vs LoadBalancer?** ClusterIP is internal-only (service-to-service). NodePort opens a port on every node's IP — rarely used directly in prod. LoadBalancer provisions an external cloud load balancer pointing at the Service. In practice external HTTP traffic usually enters through an Ingress/Gateway that fronts internal ClusterIP Services, rather than a LoadBalancer per service.

---

# 3. Autoscaling (HorizontalPodAutoscaler)

**Definition.** The HorizontalPodAutoscaler (HPA) automatically adjusts the replica count of a Deployment based on observed metrics like CPU, memory, or custom metrics.

**How it works.** The HPA controller polls metrics (via the metrics-server for CPU/memory) on a control loop (default ~15s). It compares the current metric value against your target and computes desired replicas with a ratio: `desired = ceil(current_replicas x currentMetric / targetMetric)`. It then updates the Deployment's replica count, bounded by `minReplicas`/`maxReplicas`, with stabilization windows to avoid flapping. Crucially, CPU-based scaling is measured against the pod's CPU **request** — so requests must be set sensibly or the math is meaningless.

**Why it matters (production impact).** Traffic for a member portal is spiky — open enrollment, end-of-year deadlines, Monday mornings. Static replica counts force a bad trade: over-provision (pay for idle capacity) or under-provision (fall over under load). HPA right-sizes continuously, holding latency steady during spikes and releasing capacity when it's quiet.

**ASCII diagram — HPA control loop and scale math:**
```
   +---------------------+        every ~15s        +------------------+
   |  metrics-server     |<------- scrapes ----------|   pods (CPU)     |
   +---------------------+                            +------------------+
            |
            v  current avg CPU = 80% (utilization of REQUEST)
   +-----------------------------------------+
   |  HPA controller                         |
   |  target = 50%   currentReplicas = 4     |
   |  desired = ceil(4 * 80 / 50) = ceil(6.4)|
   |          = 7   (clamped to min..max)    |
   +-----------------------------------------+
            |
            v  patch replicas
   +---------------------+
   |  Deployment         |  4 -> 7 pods
   +---------------------+
            |
            v
   Service EndpointSlice auto-updates -> new pods get traffic

   SCALE DOWN: avg CPU falls -> desired < current
   stabilizationWindow (default 5m down) prevents flapping before shrinking
```

**YAML:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: claims-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: claims-service          # the Deployment it scales
  minReplicas: 2
  maxReplicas: 12
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60   # target 60% of the pod CPU REQUEST
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 75
  behavior:                        # tame flapping
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Percent
          value: 100               # at most double per step
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300   # wait 5m of calm before shrinking
      policies:
        - type: Pods
          value: 1                 # remove at most 1 pod per minute
          periodSeconds: 60
```

**Real scenario (CareFirst member portal).** `claims-service` carries the heaviest, spikiest load. Its HPA holds 2 replicas off-peak and scales toward 12 during enrollment when CPU climbs past 60% of request. The asymmetric `behavior` is deliberate: scale **up** fast (30s window, allowed to double) to protect latency when a surge hits, but scale **down** slowly (5-minute window, one pod per minute) so a brief dip doesn't tear down capacity that's about to be needed again — avoiding the thrash that hurt response times in an earlier tuning. Because the Service's endpoints update automatically, scaled-up pods start serving as soon as they pass readiness, with no change to `member-service`.

**Debugging autoscaling:**
- **HPA shows `<unknown>` for the metric** -> metrics-server isn't installed or isn't reporting. `kubectl top pods` failing confirms it. CPU/memory HPA literally cannot work without metrics-server.
- **HPA never scales despite high load** -> no CPU **request** set on the container, so utilization (a percentage *of the request*) is undefined. This is the most common HPA misconfiguration — HPA math is anchored to requests.
- **Pods scale up but stay `Pending`** -> cluster is out of node capacity; HPA added pod *specs* but there's nowhere to schedule them. HPA scales pods, not nodes — you need the Cluster Autoscaler (or Karpenter) for nodes. The two are different layers and people conflate them.
- **Replica count flapping (thrashing)** -> stabilization windows too short or target too tight. Widen `scaleDown.stabilizationWindowSeconds` and rate-limit with `behavior` policies.
- **Hitting `maxReplicas` and still saturated** -> genuine capacity ceiling; raise the max *and* confirm nodes/Cluster Autoscaler can supply the capacity. Inspect with `kubectl describe hpa` — it shows current vs target and recent scaling events/reasons.

**Interview Q&A.**
> **Q: How does HPA decide the replica count?** It's a ratio: desired = ceil(currentReplicas x currentMetricValue / targetValue), clamped to min/max, with stabilization windows to prevent flapping. For CPU it measures utilization as a percentage of the pod's CPU request — so requests must be set correctly or the calculation is meaningless.
>
> **Q: HPA vs VPA vs Cluster Autoscaler?** HPA scales the *number* of pods (horizontal). VPA (Vertical Pod Autoscaler) adjusts each pod's CPU/memory *requests* (vertical). Cluster Autoscaler scales the number of *nodes* so there's room to schedule pods. HPA and Cluster Autoscaler are complementary and usually run together; HPA and VPA on the same resource metric conflict and shouldn't both target CPU/memory on the same workload.
>
> **Q: Why scale up fast but down slow?** Asymmetric behavior protects latency: a surge needs capacity immediately, but tearing pods down at the first dip risks removing capacity you'll need moments later, causing thrash and cold-start latency. A long scale-down stabilization window trades a little extra cost for stability.

---

# 4. How the three compose

The Deployment manages the pods and their versioned rollout; the Service gives those churning pods a stable address and load-balances across only the Ready ones; the HPA watches load and tells the Deployment how many pods to run. The chain is self-reinforcing: HPA scales the Deployment up -> new pods pass readiness -> the Service's EndpointSlice picks them up automatically -> traffic spreads -> measured CPU drops -> HPA settles. Readiness probes are the connective tissue — they gate rollout progression *and* Service endpoint membership *and* whether a freshly-autoscaled pod counts as usable.

```
  HPA  --(observes CPU, sets replica count)-->  Deployment
                                                    |
                                                    v  manages ReplicaSet -> pods
                                              [pod][pod][pod][pod]
                                                    ^
                                                    | selector + readiness
                                              Service (stable VIP/DNS)
                                                    ^
                                                    | DNS name, load-balanced
                                              callers (member-service, Ingress)
```

---

# 5. Version caveat

Kubernetes API surfaces and defaults move between versions (for example, `autoscaling/v2` is current but older clusters used `v2beta` variants, and probe/`behavior` defaults have shifted). The YAML here targets a recent, stable cluster — verify the API versions and defaults against your cluster's actual version before applying.

*End of guide.*
