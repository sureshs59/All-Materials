# OpenShift — Production-Grade Platform Deep Dive
### For a Senior Engineer (Java / Spring Boot / Microservices, Automotive & Logistics domain)

> **Scope.** This is a platform-wide guide. It assumes you know vanilla Kubernetes (Deployments, Services, HPA) and focuses on what OpenShift *adds* on top — the build system, image abstractions, native routing, security model, operators, and pipelines. Where a concept is identical to Kubernetes, this guide says so and moves on rather than re-teaching it.
>
> **Mental model:** OpenShift = Kubernetes + an opinionated developer platform (builds, registry, CI/CD), a hardened security model (SCCs, no root), and native operational tooling (Operators, monitoring, Routes), all wrapped by the `oc` CLI and a web console.

---

## Table of Contents
1. Platform Architecture & Projects
2. Source-to-Image (S2I) & BuildConfig
3. ImageStreams
4. Workloads: Deployment vs DeploymentConfig
5. Routes & the Ingress Controller
6. Security Context Constraints (SCC)
7. Operators & OLM
8. OpenShift Pipelines (Tekton) & GitOps (ArgoCD)
9. Performance Benchmarks & Best Practices
10. Comparison with Alternatives
11. Common Mistakes & Debugging Techniques
12. 10 Interview Questions with Model Answers

---

# 1. Platform Architecture & Projects

**Definition.** OpenShift is an enterprise Kubernetes distribution that adds a control plane of operators, an integrated registry and build system, and a multi-tenancy layer called Projects on top of standard Kubernetes.

**How it works.** Every OpenShift cluster runs the same Kubernetes API server and etcd, but the control plane is managed declaratively by the Cluster Version Operator and a suite of second-level operators (each owning a subsystem like networking, monitoring, or the registry). A Project is a Kubernetes namespace with extra annotations, default quotas, RBAC, and an SCC binding — the unit of multi-tenancy. The `oc` CLI is a superset of `kubectl` that adds platform verbs (`oc new-app`, `oc start-build`, `oc rollout`).

**Why it matters (production impact).** The operator-managed control plane means upgrades, certificate rotation, and component self-healing are automated rather than manual — a major operational difference from rolling your own Kubernetes. Projects give you hard tenant boundaries (quota + RBAC + network policy) so multiple teams or domains share a cluster safely, which is exactly what a large org running both automotive and logistics workloads needs.

**ASCII diagram — control plane & project tenancy:**
```
   +----------------------------------------------------------+
   |                  OpenShift Control Plane                 |
   |                                                          |
   |  Cluster Version Operator (CVO)                          |
   |    +-- Network Operator     +-- Ingress Operator         |
   |    +-- Monitoring Operator  +-- Image Registry Operator  |
   |    +-- Authentication Op.    +-- Machine API Operator     |
   |                                                          |
   |  kube-apiserver  +  etcd   (standard Kubernetes core)    |
   +----------------------------------------------------------+
              |                |                |
        +-----------+   +-----------+    +-----------+
        | Project A |   | Project B |    | Project C |
        | (telemetry)|  | (shipment)|    | (shared)  |
        |  quota     |  |  quota    |    |  quota    |
        |  RBAC      |  |  RBAC     |    |  RBAC     |
        |  SCC bind  |  |  SCC bind |    |  SCC bind |
        |  netpolicy |  |  netpolicy|    |  netpolicy|
        +-----------+   +-----------+    +-----------+
            ^               ^                ^
        team A only     team B only      shared services
```

**Working code example (oc / YAML):**
```bash
# A Project is the tenancy unit; oc new-project wraps namespace + defaults
oc new-project ford-telemetry --display-name="Ford SCA-V Telemetry"

# Scope a developer to just this project
oc adm policy add-role-to-user edit alice -n ford-telemetry
```
```yaml
# Quota + limits define the tenant's resource envelope
apiVersion: v1
kind: ResourceQuota
metadata: { name: telemetry-quota, namespace: ford-telemetry }
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    pods: "60"
```

**Real scenario (Ford / FedEx integration).** A shared cluster hosts the Ford SCA-V connected-vehicle telemetry workloads in a `ford-telemetry` Project and the FedEx shipment-integration services in a `fedex-shipment` Project. Each Project carries its own quota, RBAC, and NetworkPolicy so the two domains can't reach each other's pods or exhaust each other's capacity — the telemetry burst during a vehicle fleet sync can't starve shipment processing. A third `shared-platform` Project holds common services (a config service, an auth gateway) exposed cross-project via explicit NetworkPolicy.

**Interview explanation.** "OpenShift is Kubernetes with an operator-managed control plane and a developer platform layered on. The thing I lean on operationally is that the cluster components are themselves operators reconciled by the Cluster Version Operator — upgrades and self-healing are declarative. For multi-tenancy, a Project is a namespace plus quota, RBAC, SCC binding, and network policy, which is how we isolate, say, automotive and logistics workloads on one cluster."

---

# 2. Source-to-Image (S2I) & BuildConfig

**Definition.** Source-to-Image (S2I) is an OpenShift build framework that produces a runnable container image directly from application source code plus a language-specific builder image, with no hand-written Dockerfile.

**How it works.** A `BuildConfig` defines the source (Git repo), the build strategy (S2I, Docker, or pipeline), and the output (an ImageStream tag). On S2I, OpenShift pulls the builder image (e.g. a Java/OpenJDK builder), injects your source, runs the builder's `assemble` script to compile and embed the app, and commits the result as a new image — then pushes it to the integrated registry. A build can be triggered by a webhook, an image change, or `oc start-build`.

**Why it matters (production impact).** S2I standardizes and secures image production: builds run inside the cluster from a curated, patched builder image, so you get consistent, reproducible, CVE-managed base layers without every team maintaining Dockerfiles. For a regulated shop this is a supply-chain control — the org governs the builder images, and app teams just bring source.

**ASCII diagram — S2I build flow:**
```
   Git push --> webhook --> BuildConfig (strategy: Source/S2I)
        |
        v
   +----------------------------------------------------+
   |  Build pod (ephemeral)                             |
   |                                                    |
   |  1. pull builder image (openjdk-17 S2I builder)    |
   |  2. inject app source into builder                 |
   |  3. run assemble script:                           |
   |        mvn package -> produces app.jar             |
   |        embed jar into image at /deployments        |
   |  4. run 'save-artifacts' (incremental build cache) |
   |  5. commit new image layer                         |
   +----------------------------------------------------+
        |
        v
   push --> Integrated Registry --> ImageStream "telemetry-svc:latest"
        |
        v
   (ImageChange trigger) --> Deployment/DC redeploys with new image
```

**Working code example (BuildConfig for a Spring Boot service):**
```yaml
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: telemetry-svc
  namespace: ford-telemetry
spec:
  source:
    type: Git
    git: { uri: "https://git.internal/ford/telemetry-svc.git", ref: main }
  strategy:
    type: Source                      # S2I
    sourceStrategy:
      from:
        kind: ImageStreamTag
        namespace: openshift
        name: java:openjdk-17-ubi8    # curated, patched builder image
  output:
    to: { kind: ImageStreamTag, name: "telemetry-svc:latest" }
  triggers:
    - type: GitHub
      github: { secretReference: { name: gh-webhook-secret } }
    - type: ImageChange                # rebuild if the builder image is patched
```
```bash
oc start-build telemetry-svc -n ford-telemetry --follow   # manual trigger + stream logs
```

**Real scenario (Ford / FedEx integration).** The Ford telemetry services are built with S2I from the org's curated OpenJDK-17 UBI builder image. When the platform team patches that builder for a CVE, an `ImageChange` trigger rebuilds every dependent service automatically — no app-team action — and the rebuilt images flow through ImageStreams into redeploys. The FedEx integration team, which had legacy services with custom native dependencies, uses the Docker build strategy for those few cases while keeping S2I for everything standard, so the supply-chain governance still applies to 90% of the estate.

**Interview explanation.** "S2I builds a runnable image from source plus a curated builder image, no Dockerfile. The `assemble` script compiles and embeds the app; the result is committed and pushed to the integrated registry as an ImageStream tag. The production value is supply-chain control — the org owns the builder base images, patches them centrally, and an ImageChange trigger cascades rebuilds. You drop to the Docker strategy only for the edge cases S2I can't express."

---

# 3. ImageStreams

**Definition.** An ImageStream is an OpenShift abstraction that tracks a set of related container image tags and their underlying digests, decoupling a workload's image reference from the registry it lives in.

**How it works.** An ImageStream holds named tags (`latest`, `v2`, `prod`), each resolving to an immutable image digest. Workloads and BuildConfigs reference an ImageStreamTag rather than a registry URL; when a new image is pushed or imported, the tag advances to a new digest, and any `ImageChange` trigger watching that tag fires a redeploy. This gives you tag-based promotion and instant rollback by repointing a tag.

**Why it matters (production impact).** ImageStreams are the indirection that makes OpenShift's build-and-deploy loop work and gives you atomic, auditable promotion: `dev → staging → prod` is a tag repoint, and rollback is repointing the tag back to the prior digest — no rebuild, no registry juggling. Because tags pin digests, you also avoid the "`latest` moved under me" class of drift.

**ASCII diagram — ImageStream tags, digests, triggers, promotion:**
```
   ImageStream: telemetry-svc
   +------------------------------------------------+
   |  tag        -> digest                          |
   |  latest     -> sha256:aa11...   (newest build) |
   |  v2         -> sha256:aa11...                  |
   |  prod       -> sha256:99ff...   (last good)    |
   +------------------------------------------------+
        ^                         |
        | new build pushes        | ImageChange trigger watches a tag
        | -> latest = sha256:bb22 |
        v                         v
   build output             Deployment/DC redeploys when its watched tag moves

   PROMOTION (no rebuild, just repoint):
     oc tag telemetry-svc:v2 telemetry-svc:prod
        prod -> sha256:aa11...   (now serving v2)
   ROLLBACK:
     oc tag telemetry-svc@sha256:99ff... telemetry-svc:prod
        prod -> sha256:99ff...   (back to last good)
```

**Working code example (oc):**
```bash
# Import an external image into an ImageStream (mirror + track)
oc import-image telemetry-svc:latest \
  --from=image-registry.internal/ford/telemetry-svc:latest --confirm

# Promote v2 to prod by repointing the tag (atomic, no rebuild)
oc tag telemetry-svc:v2 telemetry-svc:prod

# Roll back prod to a known-good digest
oc tag telemetry-svc@sha256:99ff... telemetry-svc:prod
```
```yaml
# A Deployment referencing the ImageStream via the internal registry DNS
# (the image field resolves to the digest the tag currently points to)
image: image-registry.openshift-image-registry.svc:5000/ford-telemetry/telemetry-svc:prod
```

**Real scenario (Ford / FedEx integration).** The Ford telemetry pipeline promotes builds through `dev → staging → prod` tags on a single ImageStream. A release to production is `oc tag telemetry-svc:staging telemetry-svc:prod`, which advances the digest the prod Deployment's `ImageChange` trigger watches, redeploying in seconds. When a bad telemetry parser shipped, rollback was a one-line tag repoint to the prior digest — faster and safer than a rebuild, with the digest history giving an audit trail of exactly which image ran in prod when.

**Interview explanation.** "An ImageStream is a layer of indirection over registry tags that pins each tag to an immutable digest. Workloads reference an ImageStreamTag, not a registry URL, so promotion and rollback are tag repoints rather than rebuilds, and ImageChange triggers turn a tag move into a redeploy. The win is atomic, auditable promotion and instant rollback to a known digest."

---

# 4. Workloads: Deployment vs DeploymentConfig

**Definition.** OpenShift runs standard Kubernetes `Deployment`s, and historically added its own `DeploymentConfig` (now deprecated) with ImageStream triggers and lifecycle hooks.

**How it works.** A `Deployment` is identical to Kubernetes (manages a ReplicaSet, RollingUpdate strategy, `maxSurge`/`maxUnavailable`). A `DeploymentConfig` manages ReplicationControllers, adds `ImageChange`/`ConfigChange` triggers and pre/mid/post lifecycle hooks, and is reconciled by OpenShift rather than the Kubernetes controller. New work should use `Deployment`.

**Why it matters (production impact).** This distinction is the most common OpenShift interview filter and a real migration concern: legacy projects use DC's automatic redeploy-on-image-push, and replicating that with a plain Deployment requires a pipeline (GitOps) or an external trigger. (This guide's companion document covers the workload primitives, Services, and autoscaling in full depth; here it's summarized.)

**ASCII diagram — the two control paths:**
```
   DEPLOYMENT (recommended)
     manifest change / new tag --> Deployment --> ReplicaSet --> pods
                                   (k8s controller)

   DEPLOYMENTCONFIG (deprecated)
     ImageStream tag moves --> ImageChange trigger --> DeploymentConfig
                                   --> ReplicationController --> pods
                                   (+ pre/mid/post hooks; OpenShift-driven)
```

**Working code example (TypeScript-side health endpoints the probes hit):**
```typescript
// Spring Boot exposes /actuator/health/{readiness,liveness}; a Node/TS service equivalent:
import express from 'express';
const app = express();
let ready = false;

app.get('/health/liveness', (_req, res) => res.sendStatus(200));      // process alive
app.get('/health/readiness', (_req, res) =>                            // ready to serve
  ready ? res.sendStatus(200) : res.sendStatus(503));

warmup().then(() => { ready = true; });   // flip ready only after caches/pools warm
app.listen(8080);
```
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }   # zero-downtime
```

**Real scenario (Ford / FedEx integration).** Legacy FedEx shipment services ran as DeploymentConfigs whose ImageChange triggers auto-redeployed on each S2I build — a self-contained CI/CD loop. New Ford telemetry services use standard Deployments promoted via the GitOps pipeline (section 8), with the readiness probe gating traffic until the Spring Boot app's Kafka consumers and connection pools are warm, so a rolling update never sends telemetry to a cold pod.

**Interview explanation.** "Use `Deployment`; `DeploymentConfig` is deprecated. DC's value was the ImageStream `ImageChange` trigger — push an image, auto-redeploy — plus lifecycle hooks. Migrating off DC means moving that trigger responsibility into a pipeline. Everything else (rolling update, zero-downtime via `maxUnavailable: 0`, readiness-gated traffic) is the same as Kubernetes."

---

# 5. Routes & the Ingress Controller

**Definition.** A Route is OpenShift's native object for exposing a Service to external HTTP(S) traffic through the built-in HAProxy-based Ingress Controller, with first-class TLS termination modes.

**How it works.** A Route maps an external hostname to a Service; the Ingress Controller (HAProxy) reconciles Routes into router config. TLS modes are `edge` (terminate at router, plaintext to pod), `passthrough` (encrypted to pod, router doesn't decrypt), and `reencrypt` (terminate then re-encrypt to pod). Routes support weighted `alternateBackends` for canary/blue-green. Standard Kubernetes `Ingress` also works and generates a managed Route underneath.

**Why it matters (production impact).** Routes are the OpenShift-native ingress answer and carry compliance-relevant behavior: `reencrypt`/`passthrough` keep traffic encrypted end-to-end to the pod, while `edge` does not. Weighted backends give you canary releases without a service mesh.

**ASCII diagram — Route, Service, TLS modes:**
```
   external --> HAProxy Ingress Controller --(Route host -> Service)--> pods
                                                                       (ClusterIP)
   TLS MODES:
     edge        client--TLS-->[router]--PLAINTEXT-->pod   (not E2E)
     passthrough client--------TLS-------->pod             (router never decrypts)
     reencrypt   client--TLS-->[router]--TLS-->pod         (E2E; PHI/PII-friendly)

   CANARY (weighted):
     Route --> svc-v1 (weight 90)
           \-> svc-v2 (weight 10)   <-- 10% of traffic to the new version
```

**Working code example (Route YAML):**
```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata: { name: shipment-api, namespace: fedex-shipment }
spec:
  to: { kind: Service, name: shipment-api, weight: 90 }
  alternateBackends:
    - { kind: Service, name: shipment-api-v2, weight: 10 }   # canary
  port: { targetPort: 8080 }
  tls:
    termination: reencrypt
    insecureEdgeTerminationPolicy: Redirect
```

**Real scenario (Ford / FedEx integration).** The FedEx shipment API is exposed with a `reencrypt` Route so shipment data stays encrypted from client to pod through the router — required because the payload carries customer address and tracking data. A risky shipment-API rewrite was rolled out as a 10% canary via `alternateBackends`, watched on the OpenShift monitoring dashboards, then shifted to 100% by adjusting weights. Internal Ford telemetry services have no Route at all — ClusterIP-only, reachable only inside the cluster.

**Interview explanation.** "A Route is OpenShift's native ingress, backed by HAProxy, with TLS modes and weighted backends built in. For regulated payloads I use `reencrypt` or `passthrough` so traffic is encrypted to the pod, not just to the router. Canary is `alternateBackends` weighting — no mesh required. Ingress also works and creates a Route underneath, so I use Ingress only when I want cross-distro-portable manifests."

---

# 6. Security Context Constraints (SCC)

**Definition.** A Security Context Constraint is an OpenShift policy object that controls the security attributes a pod may request — running as root, host access, volume types, Linux capabilities, and the UID range it runs under.

**How it works.** SCCs are admission-time policies bound to service accounts. The default `restricted` (now `restricted-v2`) SCC denies root, drops most capabilities, disallows host networking/ports, and assigns each pod a *random high UID* from the namespace's allocated range. A workload needing more (e.g. a specific UID or a host mount) must be granted a more permissive SCC explicitly via RBAC — it's never implicit.

**Why it matters (production impact).** SCCs are the #1 reason an image that runs on vanilla Kubernetes fails on OpenShift: the image assumes a fixed user (often root) but gets a random UID and can't write where it expects. Building UID-agnostic images is mandatory. The upside is a hardened-by-default posture — no accidental root containers — which is a real security control for a regulated estate.

**ASCII diagram — SCC admission & the UID problem:**
```
   pod create request
        |
        v
   +-------------------------------+
   | SCC admission                 |
   |  service account -> allowed   |
   |  SCC (default: restricted-v2) |
   +-------------------------------+
        |                |
   allowed?           denied? --> pod rejected (e.g. asked runAsUser:0)
        |
        v  injects random UID, e.g. 1000680000
   container runs as UID 1000680000, GID 0
        |
        +-- writes to /app owned by root:root  --> PERMISSION DENIED  (X)
        +-- writes to /app group-writable, GID 0 --> OK               (V)

   FIX: build image so writable dirs are group-writable & owned by GID 0,
        never hardcode USER <n> / runAsUser
```

**Working code example (Dockerfile made OpenShift-safe + granting an SCC):**
```dockerfile
# OpenShift-safe image: group-writable, GID 0, no hardcoded UID
FROM registry.access.redhat.com/ubi8/openjdk-17
COPY target/telemetry-svc.jar /deployments/app.jar
# make runtime dirs writable by arbitrary assigned UID (group root / GID 0)
RUN mkdir -p /deployments/work && chgrp -R 0 /deployments && chmod -R g=u /deployments
# NO 'USER 1000' line — let OpenShift assign the UID
```
```bash
# If a workload genuinely needs a fixed UID or host access, grant an SCC explicitly:
oc adm policy add-scc-to-user anyuid -z telemetry-sa -n ford-telemetry
```

**Real scenario (Ford / FedEx integration).** A FedEx legacy image hardcoded `USER 1000` and wrote logs to a root-owned path; it ran fine on the team's old Kubernetes cluster and `CrashLoopBackOff`ed immediately on OpenShift with permission-denied. The fix was rebuilding it UID-agnostic (GID 0, group-writable dirs) rather than the tempting-but-wrong shortcut of granting `anyuid`. Ford telemetry images were built UID-agnostic from the start against the UBI OpenJDK base, so they moved onto OpenShift with no SCC changes.

**Interview explanation.** "An SCC is admission-time policy controlling what security context a pod can request. The default `restricted-v2` denies root and assigns a random UID, so images must be UID-agnostic — writable dirs group-owned by GID 0, no hardcoded `USER`. The classic failure is 'runs on Kubernetes, crashes on OpenShift with permission denied,' and the right fix is the image, not granting `anyuid`, which throws away the hardened default."

---

# 7. Operators & OLM

**Definition.** An Operator is a custom controller that encodes operational knowledge for an application as code, and the Operator Lifecycle Manager (OLM) is the OpenShift subsystem that installs, updates, and manages operators.

**How it works.** An Operator watches Custom Resources (CRs) defined by a CRD and reconciles real-world state to match — e.g. a Kafka Operator watches a `Kafka` CR and provisions/scales/heals a cluster. OLM handles discovery (via the OperatorHub catalog), versioned install, dependency resolution, and automatic or manual upgrade channels. OpenShift's own platform components are operators managed this way.

**Why it matters (production impact).** Operators turn day-2 operations (backups, failover, scaling, version upgrades) of complex stateful systems into declarative CRs instead of runbooks. For an org running Kafka, databases, and service meshes alongside its apps, operators mean those platforms self-heal and upgrade consistently rather than depending on tribal ops knowledge.

**ASCII diagram — operator reconciliation loop & OLM:**
```
   OperatorHub catalog --> OLM installs Operator (Subscription + channel)
        |
        v
   +---------------------------------------------+
   |  Operator (custom controller)               |
   |   watch: Kafka CR                            |
   |   reconcile loop:                            |
   |     desired (CR spec) vs actual (cluster)    |
   |     -> create/scale/heal StatefulSets, etc.  |
   +---------------------------------------------+
        ^                          |
        | user edits CR            v
   apiVersion: kafka.strimzi.io    creates: StatefulSet, Services,
   kind: Kafka                     ConfigMaps, PVCs, etc.
   spec: { replicas: 3 }           and continuously repairs drift
```

**Working code example (a Kafka CR consumed by an operator):**
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata: { name: telemetry-bus, namespace: ford-telemetry }
spec:
  kafka:
    replicas: 3
    storage: { type: persistent-claim, size: 200Gi }
    config: { "min.insync.replicas": 2, "default.replication.factor": 3 }
  zookeeper: { replicas: 3, storage: { type: persistent-claim, size: 50Gi } }
```
```yaml
# OLM Subscription — how the operator itself is installed/upgraded
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata: { name: strimzi-kafka, namespace: openshift-operators }
spec:
  channel: stable
  name: strimzi-kafka-operator
  source: community-operators
  installPlanApproval: Manual     # require human approval for upgrades in prod
```

**Real scenario (Ford / FedEx integration).** Ford's SCA-V telemetry ingests vehicle data through a Kafka cluster managed by the Strimzi Operator — defined as a 3-broker `Kafka` CR with `min.insync.replicas: 2` for durability. When a broker pod dies, the operator reconciles it back automatically; a broker version upgrade is a CR edit with an OLM-approved install plan, not a manual rolling-restart runbook. The same pattern manages the Postgres the FedEx shipment service uses, so both stateful backends have consistent, declarative day-2 operations.

**Interview explanation.** "An Operator is a controller that encodes an app's operational knowledge and reconciles a Custom Resource to real state; OLM installs and upgrades operators with versioned channels and dependency resolution. The point is day-2 automation — a Kafka or DB cluster becomes a declarative CR that self-heals and upgrades, instead of a pile of runbooks. OpenShift's own components are operators, which is why the platform self-manages."

---

# 8. OpenShift Pipelines (Tekton) & GitOps (ArgoCD)

**Definition.** OpenShift Pipelines is a Kubernetes-native CI/CD system based on Tekton that runs each pipeline step as a pod, and OpenShift GitOps is an ArgoCD-based system that continuously syncs cluster state to a Git repository.

**How it works.** Tekton defines `Task`s (sequences of steps, each a container) composed into `Pipeline`s, triggered by webhooks; everything runs as pods with no central CI server. ArgoCD watches a Git repo of manifests and continuously reconciles the cluster to match — Git is the single source of truth, and drift is auto-corrected or flagged. Together: Tekton builds/tests and updates the Git manifests; ArgoCD deploys what's in Git.

**Why it matters (production impact).** Serverless, pod-based CI removes the maintain-a-Jenkins-server burden and scales per-build. GitOps makes deployments auditable and reversible (every change is a Git commit) and eliminates config drift — the cluster *is* what Git says. For regulated change management, "the deploy is a reviewed, signed Git commit" is a strong control.

**ASCII diagram — Tekton + ArgoCD flow:**
```
   git push (app code)
        |
        v  webhook (EventListener / Trigger)
   +-------------------- Tekton Pipeline (pods) --------------------+
   |  Task: git-clone -> Task: build/test (S2I) -> Task: push image |
   |                                            -> Task: bump tag in |
   |                                               GitOps manifest   |
   +----------------------------------------------------------------+
        |
        v  commit to config repo
   +------------------+        watches        +---------------------+
   |  Git config repo |<----------------------|  ArgoCD (GitOps)     |
   |  (k8s manifests) |                       |  reconcile cluster   |
   +------------------+                       |  to match Git        |
        ^  drift?                             +---------------------+
        |                                           |
        +------ auto-sync / self-heal --------------+--> Deployment updated
```

**Working code example (a Tekton Task + ArgoCD Application):**
```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata: { name: maven-build, namespace: ford-telemetry }
spec:
  steps:
    - name: build
      image: registry.access.redhat.com/ubi8/openjdk-17
      script: |
        #!/bin/sh
        mvn -B clean package -DskipTests=false   # compile + test in a pod
```
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata: { name: telemetry-svc, namespace: openshift-gitops }
spec:
  project: default
  source:
    repoURL: https://git.internal/ford/telemetry-gitops.git
    path: overlays/prod
    targetRevision: main
  destination: { server: https://kubernetes.default.svc, namespace: ford-telemetry }
  syncPolicy:
    automated: { prune: true, selfHeal: true }   # auto-correct drift
```

**Real scenario (Ford / FedEx integration).** Ford telemetry uses a Tekton pipeline that clones, runs the Maven build and tests in pods, builds via S2I, pushes the image, and commits the new ImageStream tag into a GitOps repo. ArgoCD sees the commit and reconciles the `prod` overlay, with `selfHeal: true` reverting any manual `oc edit` drift back to the Git state — so the cluster can't diverge from the reviewed manifests. A production rollback is `git revert` of the deploy commit; ArgoCD redeploys the prior state automatically, giving change management a clean audit trail.

**Interview explanation.** "Tekton runs CI as pods — Tasks composed into Pipelines, no standing CI server — and ArgoCD does GitOps: Git is the source of truth and the cluster is continuously reconciled to it, with self-heal correcting drift. The combination gives reproducible builds and auditable, reversible deploys: every production change is a reviewed Git commit, and rollback is `git revert`."

---

# 9. Performance Benchmarks & Best Practices

> Numbers are directional orders of magnitude from typical enterprise OpenShift estates, **not** guarantees. Always measure on your own cluster with the built-in monitoring (Prometheus/Grafana) and load tests before acting on them.

**Representative impact:**
```
 Technique                              Typical effect
 -------------------------------------  ----------------------------------------
 Set CPU/memory requests accurately     Scheduler packs nodes well; HPA works at all
 S2I incremental builds (save-artifacts) Rebuilds drop from full to delta (often 2-5x)
 reencrypt vs re-establishing TLS/app   Offloads handshake to router; pod CPU down
 Route/HAProxy connection reuse          Fewer TLS handshakes; lower tail latency
 ImageStream tag promotion vs rebuild    Promotion seconds vs minutes (no rebuild)
 OnPush of monitoring to user workloads  Avoids scraping overhead on idle namespaces
 Cluster Autoscaler right-sized          Pay for peak only during peak (cost, not latency)
```

**Best practices checklist:**
- Always set **resource requests and limits** — the scheduler and HPA are meaningless without requests; missing CPU requests silently disable CPU autoscaling.
- Build **UID-agnostic images** (GID 0, group-writable dirs); never grant `anyuid` as a shortcut.
- Prefer **standard Deployments**; treat DeploymentConfig as legacy-only.
- Use **`reencrypt`/`passthrough`** Routes for any sensitive payload; `edge` only for non-sensitive traffic.
- Drive deploys through **GitOps (ArgoCD) with `selfHeal`**; the cluster should equal Git.
- Use **ImageStream tag promotion** for `dev→staging→prod`, not per-environment rebuilds.
- Pin OLM operator upgrades to **`Manual` install-plan approval** in production channels.
- Scope every team to a **Project with quota + RBAC + NetworkPolicy**; don't share a flat namespace.
- Put **readiness probes** on real warmup signals (pools/consumers ready), not just process-up — they gate rollout *and* Service endpoints.
- Pair **HPA with the Cluster/MachineAutoscaler**; HPA-scaled pods are useless if nodes are full.

---

# 10. Comparison with Alternatives

```
 Aspect            OpenShift                 Vanilla Kubernetes        Managed K8s (EKS/GKE/AKS)
 ----------------  ------------------------  ------------------------  --------------------------
 Control plane     Operator-managed, opinion. You assemble it          Cloud-managed core only
 Builds/registry   Built-in (S2I, registry)  None (bring your own)     None (bring CI + registry)
 Ingress           Routes (native) + Ingress Ingress only              Ingress + cloud LB
 Security default  Hardened (SCC, no root)   Permissive                Permissive-ish
 CI/CD             Bundled (Tekton, ArgoCD)  Add-on                    Add-on
 Day-2 / upgrades  Operator-driven, guided   Manual / DIY              Cloud-assisted
 Cost / licensing  Red Hat subscription      Free (self-supported)     Cloud + support fees
 Best fit          Regulated enterprise,     Max flexibility, custom   Cloud-native teams wanting
                   on-prem/hybrid, batteries platforms, cost-sensitive  managed core + own tooling
```

**Take for your profile.** For a regulated automotive/logistics enterprise — on-prem or hybrid, strict security posture, multiple teams sharing clusters — OpenShift's batteries-included, hardened, operator-driven model is the same kind of bet as choosing Spring Boot over assembling a stack: you pay (subscription, opinionation) to get builds, registry, CI/CD, security defaults, and day-2 automation that you'd otherwise integrate yourself. Vanilla Kubernetes wins on flexibility and cost when you have the platform team to own all of that; managed K8s wins for cloud-native teams happy to bring their own tooling around a managed core.

---

# 11. Common Mistakes & Debugging Techniques

```
 Mistake                               Symptom                          Fix
 ------------------------------------  -------------------------------  --------------------------
 Image assumes fixed/root UID          CrashLoop, permission denied     build UID-agnostic (GID 0)
                                       on OpenShift only
 Granting anyuid to "fix" SCC          works but loses hardening        fix the image instead
 No CPU request set                    HPA never scales                 set requests on containers
 Mixing oc rollout dc vs deploy        command "does nothing"           check kind: oc get dc,deploy
 edge TLS for sensitive payload        plaintext router->pod            use reencrypt/passthrough
 Route 503 "not available"             Service has no Ready endpoints   fix selector/readiness, not Route
 Manual oc edit under ArgoCD           change reverted "mysteriously"   change Git; selfHeal restores
 HPA scales, pods Pending              no node capacity                 Cluster/MachineAutoscaler
 latest tag drift                      "it changed under me"            pin digests / promote by tag
 Operator upgrade auto-applied in prod surprise version bump            installPlanApproval: Manual
```

**Debugging techniques (oc-centric):**
- **`oc describe <kind>/<name>`** — events are the first stop for scheduling, probe, image-pull, and SCC failures.
- **`oc logs <pod> --previous`** — the crash output from the prior container instance in a CrashLoop.
- **`oc debug deploy/<name>`** — launch a pod with the same spec but an overridden command/shell to poke around the environment, mounts, and UID (`id` shows the assigned UID/GID).
- **`oc get events --sort-by=.lastTimestamp -n <project>`** — chronological view of what the platform just did.
- **`oc rsh <pod>` / `oc exec`** — shell into a running pod; check filesystem permissions against the assigned UID.
- **`oc adm policy who-can <verb> <resource>`** and **`oc get scc`** — diagnose RBAC/SCC denials.
- **`oc get endpoints <svc>`** — empty endpoints explain almost every Route 503 and Service connectivity bug.
- **Built-in monitoring** — the console's Observe → Metrics/Dashboards (Prometheus) for CPU/memory vs requests, HPA behavior, and Route/HAProxy latency; enable *user-workload monitoring* to scrape your own services.
- **`oc adm top pods/nodes`** — quick resource view; failing means the metrics path is broken (the usual HPA `<unknown>` cause).

---

# 12. Interview Questions with Model Answers

**Q1. What does OpenShift add over vanilla Kubernetes?** An operator-managed control plane (self-upgrading, self-healing components), a built-in build system (S2I) and integrated registry, native routing (Routes with TLS modes), a hardened security model (SCCs, no root by default), bundled CI/CD (Tekton + ArgoCD), and Projects for multi-tenancy — all behind the `oc` CLI and a console. Underneath it's the same Kubernetes API.

**Q2. Explain S2I and when you'd not use it.** S2I builds a runnable image from source plus a curated builder image (no Dockerfile) via an `assemble` script, committing and pushing to the registry. It gives supply-chain control through governed base images. You drop to the Docker build strategy only when the app needs something S2I can't express — custom native dependencies, unusual base layers.

**Q3. What problem do ImageStreams solve?** They decouple a workload's image reference from the registry by pinning tags to immutable digests. Promotion (`dev→prod`) and rollback become atomic tag repoints rather than rebuilds, and `ImageChange` triggers turn a tag move into a redeploy — plus a digest audit trail of what ran where.

**Q4. DeploymentConfig vs Deployment?** Use `Deployment`; DC is deprecated. DC added ImageStream triggers (auto-redeploy on push) and lifecycle hooks and managed ReplicationControllers, reconciled by OpenShift. Migrating off DC means moving the trigger into a pipeline; everything else matches Kubernetes.

**Q5. Walk through the Route TLS modes.** `edge` terminates TLS at the router and sends plaintext to the pod (simplest, not end-to-end). `passthrough` sends encrypted traffic straight to the pod; the router never decrypts. `reencrypt` terminates at the router then re-encrypts to the pod. For regulated/PII payloads use `reencrypt` or `passthrough`.

**Q6. Why does an image run on Kubernetes but crash on OpenShift?** Almost always SCC: the default `restricted-v2` assigns a random high UID and denies root, so an image assuming a fixed/root user can't write where it expects and CrashLoops with permission denied. Fix the image to be UID-agnostic (GID 0, group-writable dirs) rather than granting `anyuid`.

**Q7. What is an Operator and what does OLM do?** An Operator is a controller that encodes an application's operational knowledge and reconciles a Custom Resource to real state (e.g. a Kafka cluster from a `Kafka` CR). OLM installs, version-channels, dependency-resolves, and upgrades operators. The result is declarative day-2 operations instead of runbooks; OpenShift's own components are operators.

**Q8. How do Tekton and ArgoCD divide responsibility?** Tekton is pod-based CI (Tasks → Pipelines, no standing server) that builds, tests, and updates Git manifests. ArgoCD is GitOps CD: Git is the source of truth and the cluster is continuously reconciled to it, with self-heal correcting drift. Tekton produces the desired state in Git; ArgoCD makes the cluster match it.

**Q9. How do you do a safe production release on OpenShift?** Promote an ImageStream tag (or commit the tag bump to the GitOps repo), let ArgoCD reconcile, optionally canary via a Route's `alternateBackends` weighting while watching monitoring, then shift to 100%. Rollback is a tag repoint to the prior digest or a `git revert` — both fast and auditable, no rebuild.

**Q10. How is multi-tenancy enforced?** Through Projects: each is a namespace with a ResourceQuota, RBAC role bindings, an SCC binding, and NetworkPolicy. That combination isolates teams/domains on a shared cluster so they can't exhaust each other's resources or reach each other's pods — e.g. separating automotive telemetry from logistics shipment workloads.

---

*End of guide. Benchmarks are directional — measure on your own cluster. OpenShift 4.x evolves quickly (DeploymentConfig deprecation, SCC naming `restricted`→`restricted-v2`, Route/Ingress behavior); verify version-specific details against the Red Hat docs for your cluster's exact version before relying on them.*
