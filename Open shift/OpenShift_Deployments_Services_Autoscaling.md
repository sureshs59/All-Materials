# OpenShift — Deployments, Services & Autoscaling
### Production-grade deep dive (the OpenShift delta over Kubernetes) — CareFirst member portal scenario

> **Framing.** OpenShift *is* Kubernetes underneath, so everything from the Kubernetes guide still applies. The interview-relevant skill is knowing exactly where OpenShift diverges. Three big OpenShift-specific things to anchor on: **DeploymentConfig vs Deployment** (OpenShift's older, now-deprecated controller), **Routes vs Ingress** (OpenShift's pre-Ingress external-traffic object), and **stricter security defaults** (SCCs and random UIDs) that change how your images must be built.

---

## Table of Contents
1. Deployments (Deployment vs DeploymentConfig)
2. Services & Routes
3. Autoscaling (HPA + MachineAutoscaler/ClusterAutoscaler)
4. Cross-cutting: Security Context Constraints (SCC)
5. Synthesis — what's actually different
6. Version caveat

---

# 1. Deployments (Deployment vs DeploymentConfig)

**Definition.** OpenShift runs standard Kubernetes `Deployment` objects, but historically shipped its own `DeploymentConfig` (DC) controller that adds deployment triggers, lifecycle hooks, and a distinct rollout engine driven by the OpenShift master rather than a Kubernetes controller.

**How it works.** A standard `Deployment` works exactly as in Kubernetes (ReplicaSet, rolling update). A `DeploymentConfig` instead manages `ReplicationController`s (the pre-ReplicaSet primitive) and adds *triggers* — most notably `ImageChange`, which automatically redeploys when a new image lands in an OpenShift `ImageStream` — plus `Recreate`-with-hooks strategies (pre/mid/post lifecycle hooks). As of recent OpenShift versions, DeploymentConfig is **deprecated**; Red Hat recommends standard Deployments combined with other mechanisms for the trigger behavior.

**Why it matters (production impact).** Knowing the distinction is the single most common OpenShift interview filter. New workloads should use `Deployment`; you'll encounter `DeploymentConfig` in older CareFirst-era projects and need to know why an `oc rollout` behaves differently and why an image push auto-triggers a redeploy. The ImageStream trigger is the genuinely useful DC feature people miss when migrating.

**ASCII diagram — DC vs Deployment control paths:**
```
   DEPLOYMENT (standard k8s, recommended)
   -------------------------------------
   oc apply -f deployment.yaml
        |
        v
   Deployment ---> ReplicaSet ---> pods      (k8s controller reconciles)
   image update = you change the tag / CI patches the manifest

   DEPLOYMENTCONFIG (OpenShift legacy, deprecated)
   -----------------------------------------------
   ImageStream "member-portal:latest"  <--- new image pushed (build / oc import-image)
        |  (ImageChange trigger fires)
        v
   DeploymentConfig ---> ReplicationController(vN) ---> pods
        |                         ^
        |  Recreate/Rolling       | keeps prior RC for rollback
        |  + pre/mid/post hooks   |
        v
   new ReplicationController(vN+1)   <-- rollout driven by OpenShift, not k8s
```

**YAML:**
```yaml
# RECOMMENDED: standard Deployment (works identically to Kubernetes)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: member-portal
spec:
  replicas: 4
  selector:
    matchLabels: { app: member-portal }
  strategy:
    type: RollingUpdate
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }
  template:
    metadata:
      labels: { app: member-portal }
    spec:
      containers:
        - name: portal
          # ImageStream-resolved internal registry reference is common in OpenShift
          image: image-registry.openshift-image-registry.svc:5000/carefirst/member-portal:v2
          ports: [{ containerPort: 8080 }]
          resources:
            requests: { cpu: "250m", memory: "512Mi" }
            limits:   { cpu: "1",    memory: "1Gi" }
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: 8080 }
            initialDelaySeconds: 10
          # NOTE: do NOT hardcode runAsUser — OpenShift assigns a random UID (see SCC)
---
# LEGACY: DeploymentConfig with an ImageChange trigger (you'll see this in old projects)
apiVersion: apps.openshift.io/v1
kind: DeploymentConfig
metadata:
  name: member-portal-dc
spec:
  replicas: 4
  selector: { app: member-portal-dc }
  strategy:
    type: Rolling          # or Recreate, which supports pre/mid/post hooks
  triggers:
    - type: ConfigChange   # redeploy when the DC spec changes
    - type: ImageChange    # redeploy automatically when the ImageStream tag updates
      imageChangeParams:
        automatic: true
        containerNames: [portal]
        from:
          kind: ImageStreamTag
          name: member-portal:latest
  template:
    metadata:
      labels: { app: member-portal-dc }
    spec:
      containers:
        - name: portal
          image: member-portal:latest
          ports: [{ containerPort: 8080 }]
```

**Real scenario (CareFirst member portal).** An older portal service was built around a `DeploymentConfig` with an `ImageChange` trigger wired to an `ImageStream`: the OpenShift `BuildConfig` produced a new image on every merge, pushed it to the internal registry, the ImageStream tag advanced, and the DC auto-redeployed — a built-in CI/CD loop with no external pipeline step. During modernization, new services moved to standard `Deployment`s with the image tag advanced by the external pipeline (Tekton/ArgoCD) instead, because DC is deprecated and GitOps wants the manifest to be the source of truth. The migration gotcha was reproducing the auto-redeploy-on-image-push behavior, which a plain Deployment doesn't do on its own.

**Debugging deployments on OpenShift:**
- **`oc rollout latest` does nothing / unexpected behavior** -> you're mixing DC and Deployment commands. `oc rollout latest dc/name` is DeploymentConfig-only; standard Deployments use `oc rollout restart deployment/name`. Confirm the kind with `oc get dc,deploy`.
- **Pod fails with permission errors / "can't write" / UID issues** -> the #1 OpenShift-specific failure: the image assumes it runs as a fixed user (often root), but OpenShift's default `restricted` SCC assigns a random high UID. Fix the image to be UID-agnostic (group-writable dirs, GID 0); don't force `runAsUser`.
- **ImageChange trigger not firing** -> the ImageStream tag didn't actually advance, or `automatic: false`. Check `oc describe is member-portal` to see whether the tag moved.
- **`CrashLoopBackOff`** -> same as Kubernetes; use `oc logs <pod> --previous`. OpenShift adds `oc debug deploy/member-portal` to launch a debug pod with the same spec but an overridden command.

**Interview Q&A.**
> **Q: DeploymentConfig vs Deployment — which should I use and why?** Use standard `Deployment` for anything new; `DeploymentConfig` is deprecated. DC predates Kubernetes Deployments and adds ImageStream triggers (auto-redeploy on image push) and lifecycle hooks, managing ReplicationControllers instead of ReplicaSets, with rollouts driven by OpenShift rather than a Kubernetes controller. The main reason you'd still touch DC is maintaining older projects or wanting its ImageStream-trigger CI loop — but the modern path replaces that with a GitOps pipeline.
>
> **Q: What's an ImageStream and why does OpenShift add it?** An ImageStream is an OpenShift abstraction over a set of image tags in a registry. It lets you decouple "which image" from "which deployment," trigger redeploys when a tag advances, and roll back by repointing a tag. It's the hook that made DC's automatic redeploy work.

---

# 2. Services & Routes (the big OpenShift divergence)

**Definition.** OpenShift uses the same Kubernetes `Service` for internal load balancing, but for external HTTP(S) traffic it predates and supplements Ingress with its own `Route` object, backed by a built-in HAProxy-based router.

**How it works.** A `Service` (ClusterIP) works exactly as in Kubernetes — selector, EndpointSlice, internal VIP. To expose a Service outside the cluster, OpenShift's `Route` maps an external hostname to the Service and is reconciled by the OpenShift Ingress Controller (HAProxy). A Route handles TLS termination modes natively — `edge` (TLS terminated at the router), `passthrough` (encrypted straight to the pod), and `reencrypt` (terminated then re-encrypted to the pod) — and OpenShift can auto-generate a hostname. Standard Kubernetes `Ingress` also works and, in modern OpenShift, actually creates a managed Route under the hood.

**Why it matters (production impact).** Routes are the OpenShift-native way to expose services and the most likely "how do I get traffic in?" answer in an OpenShift shop. The TLS modes matter for a healthcare portal: `reencrypt` or `passthrough` keeps PHI traffic encrypted end-to-end to the pod, which `edge` (plaintext from router to pod) does not. Knowing which mode preserves in-cluster encryption is a compliance-relevant detail.

**ASCII diagram — Service (internal) + Route (external) and TLS modes:**
```
   EXTERNAL traffic
        |
        v
   +----------------------------+
   | OpenShift Router (HAProxy) |   <-- reconciles Route objects
   +----------------------------+
        |  Route: portal.apps.cluster.example.com  -> Service member-portal
        v
   +------------------+          (ClusterIP, internal)
   | Service          |---- selector app=member-portal ----+
   | member-portal    |                                     |
   +------------------+                                     v
        |                                            [pod][pod][pod][pod]
        v   internal-only services stay ClusterIP (no Route)
   claims-service / eligibility-service  (never exposed externally)

   ROUTE TLS MODES:
   edge        client--TLS-->[router]--PLAINTEXT-->pod   (simplest; not E2E encrypted)
   passthrough client--------TLS-------->pod             (router doesn't decrypt)
   reencrypt   client--TLS-->[router]--TLS-->pod         (E2E encrypted; PHI-friendly)
```

**YAML:**
```yaml
# Internal Service — identical to Kubernetes
apiVersion: v1
kind: Service
metadata:
  name: member-portal
spec:
  selector: { app: member-portal }
  ports:
    - port: 8080
      targetPort: 8080
---
# OpenShift Route — external exposure with reencrypt TLS (keeps PHI encrypted to the pod)
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: member-portal
spec:
  host: portal.apps.cluster.example.com   # omit to let OpenShift auto-generate
  to:
    kind: Service
    name: member-portal
    weight: 100
  port:
    targetPort: 8080
  tls:
    termination: reencrypt                 # edge | passthrough | reencrypt
    insecureEdgeTerminationPolicy: Redirect # force http -> https
---
# Canary / blue-green: Route can split traffic across multiple Services by weight
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: member-portal-canary
spec:
  to:        { kind: Service, name: member-portal-v1, weight: 90 }
  alternateBackends:
    - { kind: Service, name: member-portal-v2, weight: 10 }   # 10% to canary
  port: { targetPort: 8080 }
  tls: { termination: reencrypt }
```

**Real scenario (CareFirst member portal).** The portal front-end is exposed with a `Route` using `reencrypt` termination so traffic stays encrypted from the client through the HAProxy router and re-encrypted on to the pod — satisfying the requirement that PHI never traverses the cluster network in plaintext. Internal services (`claims-service`, `eligibility-service`) have **no Route at all** — they're ClusterIP-only and unreachable from outside, which is the OpenShift-native way to enforce that PHI-handling services have zero public surface. For a risky portal upgrade, the team used a Route's `alternateBackends` weighting to send 10% of traffic to the v2 Service as a canary before shifting fully over — a capability Routes give you without a separate service-mesh.

**Debugging services & routes:**
- **Route returns 503 "Application is not available"** -> the Route's Service has no Ready endpoints. Same root cause as a broken Kubernetes Service — check `oc get endpoints <service>` and the pod readiness. The Route is fine; the backend isn't.
- **Route 503 but Service works internally** -> Route points at the wrong Service name or `targetPort` doesn't match a named Service port. `oc describe route <name>`.
- **TLS errors / cert warnings** -> wrong termination mode. `reencrypt` requires the router to trust the pod's serving cert (`destinationCACertificate`); `passthrough` means the pod must serve TLS itself. A mismatch between what the pod serves and the Route mode is the usual culprit.
- **`oc expose svc/foo` created a Route with http only** -> `oc expose` defaults to no TLS; you must add the `tls:` block or use `oc create route reencrypt`.
- **Works on Route hostname but not custom domain** -> DNS/wildcard issue: the cluster's `*.apps.<cluster>` wildcard covers generated hosts, but a custom host needs its own DNS pointing at the router and a matching Route `host`.

**Interview Q&A.**
> **Q: Route vs Ingress — what's the difference and when do you use each?** A Route is OpenShift's native external-traffic object, older than Ingress, backed by the HAProxy router, with first-class TLS termination modes (edge/passthrough/reencrypt) and weighted traffic splitting. Ingress is the Kubernetes-standard object and also works on OpenShift — in fact it generates a managed Route underneath. Use Routes for OpenShift-native features (TLS modes, weighting) and portability within OpenShift; use Ingress when you want manifests portable across any Kubernetes distro.
>
> **Q: Which TLS termination keeps traffic encrypted end-to-end?** `passthrough` (router never decrypts) and `reencrypt` (router decrypts then re-encrypts to the pod). `edge` terminates at the router and sends plaintext to the pod, so it's not end-to-end — which matters for regulated/PHI traffic.

---

# 3. Autoscaling (HPA — plus what OpenShift adds)

**Definition.** OpenShift uses the same Kubernetes `HorizontalPodAutoscaler`, and adds operator-driven autoscaling layers: a cluster-level autoscaler for nodes (`MachineAutoscaler` + `ClusterAutoscaler`) and, on OpenShift Serverless (Knative), request-driven scaling including scale-to-zero.

**How it works.** The HPA behaves exactly as in Kubernetes — metric ratio against the pod's CPU/memory request, bounded by min/max, with stabilization windows. OpenShift ships the metrics pipeline (Prometheus-based monitoring) out of the box. For nodes, instead of the upstream Cluster Autoscaler you configure a `ClusterAutoscaler` (cluster-wide limits) and per-machine-pool `MachineAutoscaler` objects that scale the underlying machine sets. OpenShift Serverless (Knative Serving) adds the Knative Pod Autoscaler (KPA) for concurrency-based scaling and scale-to-zero.

**Why it matters (production impact).** The HPA scales pods, but pods need nodes — on a managed OpenShift cluster the `MachineAutoscaler`/`ClusterAutoscaler` pairing is how you actually get more nodes when the HPA's new pods would otherwise sit `Pending`. Knowing this two-layer story (pods via HPA, nodes via MachineAutoscaler) is the OpenShift-flavored version of the "HPA vs Cluster Autoscaler" distinction.

**ASCII diagram — two-layer autoscaling on OpenShift:**
```
   LAYER 1: PODS (HorizontalPodAutoscaler — same as k8s)
   ----------------------------------------------------
   OpenShift monitoring (Prometheus/metrics) --> HPA
        |  desired = ceil(replicas * curCPU / targetCPU)
        v
   Deployment: 2 -> 12 pods
        |
        |  pods need somewhere to run...
        v  (if nodes full -> pods Pending)

   LAYER 2: NODES (OpenShift-specific)
   -----------------------------------
   ClusterAutoscaler (cluster-wide min/max cores, memory)
        |
        v
   MachineAutoscaler (per machine pool)  --> MachineSet --> new Nodes (cloud VMs)
        |
        v
   Pending pods now schedule onto fresh nodes

   (OpenShift Serverless / Knative path: KPA scales on concurrency, incl. -> 0)
```

**YAML:**
```yaml
# Pod-level autoscaling — standard Kubernetes HPA (works identically on OpenShift)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: claims-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: claims-service
  minReplicas: 2
  maxReplicas: 12
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 60 }
  behavior:
    scaleUp:   { stabilizationWindowSeconds: 30 }
    scaleDown: { stabilizationWindowSeconds: 300 }
---
# Node-level: ClusterAutoscaler (cluster-wide resource ceilings)
apiVersion: autoscaling.openshift.io/v1
kind: ClusterAutoscaler
metadata:
  name: default
spec:
  resourceLimits:
    maxNodesTotal: 24
    cores:  { min: 8,  max: 128 }
    memory: { min: 32, max: 512 }
  scaleDown:
    enabled: true
    delayAfterAdd: 10m
    unneededTime: 10m
---
# Node-level: MachineAutoscaler (scales a specific machine pool)
apiVersion: autoscaling.openshift.io/v1beta1
kind: MachineAutoscaler
metadata:
  name: worker-us-east-1a
  namespace: openshift-machine-api
spec:
  minReplicas: 2
  maxReplicas: 8
  scaleTargetRef:
    apiVersion: machine.openshift.io/v1beta1
    kind: MachineSet
    name: carefirst-worker-us-east-1a
```

**Real scenario (CareFirst member portal).** During open enrollment, `claims-service`'s HPA scales pods from 2 toward 12 as CPU crosses 60% of request. Without node autoscaling, those extra pods would hit `Pending` once the existing worker nodes filled up. The `MachineAutoscaler` on the worker pool (min 2, max 8 nodes) provisions additional cloud VMs as the scheduler reports unschedulable pods, and the `ClusterAutoscaler` caps total spend with cluster-wide core/memory ceilings. After the surge, scale-down removes pods first, then nodes drain and deprovision after the `unneededTime` window — so CareFirst pays for peak capacity only during the peak. The two layers are configured and reasoned about separately, which is the OpenShift mental model.

**Debugging autoscaling on OpenShift:**
- **HPA `<unknown>` metrics** -> same as Kubernetes (metrics pipeline), but on OpenShift the cause is usually the user-workload monitoring or metrics path, not a missing metrics-server (OpenShift ships monitoring). Confirm with `oc adm top pods`.
- **HPA scales pods but they're `Pending` and no new nodes appear** -> node autoscaling isn't configured or hit its ceiling. Check `oc get clusterautoscaler`, `oc get machineautoscaler -n openshift-machine-api`, and whether `maxNodesTotal`/`cores.max` is reached. This pod-vs-node confusion is the most common OpenShift autoscaling failure.
- **Nodes scale up but never scale down** -> `scaleDown.enabled: false`, or pods with restrictive PodDisruptionBudgets / local storage / `kube-system`-style pods blocking node drain. The ClusterAutoscaler won't remove a node it can't safely drain.
- **MachineAutoscaler not adding machines** -> the underlying MachineSet or cloud credentials are failing; `oc get machines -n openshift-machine-api` shows machines stuck in `Provisioning`/`Failed`.

**Interview Q&A.**
> **Q: How does autoscaling differ on OpenShift vs vanilla Kubernetes?** The pod layer is identical — same HPA. The node layer is OpenShift-specific: instead of the upstream Cluster Autoscaler you use a `ClusterAutoscaler` resource for cluster-wide limits plus `MachineAutoscaler` objects that scale MachineSets (the cloud VMs). OpenShift also bundles the metrics/monitoring stack, and OpenShift Serverless adds Knative concurrency-based scaling with scale-to-zero.
>
> **Q: Your HPA scaled pods but latency didn't improve — what do you check?** Whether the new pods actually scheduled. If they're `Pending`, the node layer is the bottleneck: check the ClusterAutoscaler/MachineAutoscaler limits and machine provisioning. HPA scaling pods is meaningless if there's no node capacity, which is exactly why OpenShift splits the two layers.

---

# 4. Cross-cutting: Security Context Constraints (SCC)

This isn't one of the three core topics, but it's the OpenShift difference most likely to break a deployment that worked on plain Kubernetes. OpenShift's default `restricted` (or `restricted-v2`) SCC **drops root, drops most Linux capabilities, and assigns each pod a random high UID** from the namespace's allocated range. An image that assumes it runs as a known user (e.g. writes to a root-owned directory, or hardcodes `USER 1000`) will fail with permission errors. The fix is to build UID-agnostic images: make writable directories group-writable and owned by GID 0, and never hardcode `runAsUser`. In interviews, "why does my image run on Kubernetes but crash on OpenShift?" almost always traces back to this.

---

# 5. Synthesis — what's actually different

```
   CONCEPT          KUBERNETES              OPENSHIFT ADDS / CHANGES
   ---------------  ----------------------  ----------------------------------------
   Workload         Deployment              Deployment (same) + DeploymentConfig
                                            (legacy/deprecated, ImageStream triggers)
   Internal net     Service (ClusterIP)     Service (identical)
   External net     Ingress                 Route (HAProxy, TLS modes, weighting);
                                            Ingress also works -> creates a Route
   Pod scaling      HPA                     HPA (identical), bundled metrics stack
   Node scaling     Cluster Autoscaler      ClusterAutoscaler + MachineAutoscaler
                                            (scales MachineSets)
   Security         permissive defaults     SCC: no root, random UID (build for it!)
   Images           registry refs           ImageStreams abstract tags + triggers
   CLI              kubectl                 oc (superset: oc rollout, oc debug, oc expose)
```

---

# 6. Version caveat

OpenShift versions move quickly and Red Hat has been actively deprecating DeploymentConfig and evolving the Route/Ingress relationship and SCC naming (`restricted` -> `restricted-v2`) across 4.x releases. The specifics here reflect recent OpenShift 4.x, but verify against your cluster's exact version before relying on deprecation timelines or default-SCC behavior.

*End of guide.*
