# What are most important points add in kubernetes yml file for deployment?

The most critical elements of a Kubernetes deployment YAML file ensure that your application is **discoverable, stable, and highly available**. [[1](https://www.linkedin.com/posts/bharathkumar-reddy-n_kubernetes-devops-cloud-activity-7369223511233474560-V36N)]

Here are the most important points you must include, organized by their functional purpose.

## 1. The Core Identity (Metadata & Matching)

Without these, Kubernetes cannot link your deployment configuration to the actual running containers.

- **apiVersion & kind**: Must be set to apps/v1 and Deployment.
- **metadata.name**: The unique name for your deployment.
- **spec.selector.matchLabels**: **Crucial.** This tells the deployment which Pods it owns. It must *exactly* match the labels defined under spec.template.metadata.labels. [[1](https://octopus.com/devops/kubernetes-deployments/kubernetes-yaml/), [2](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/), [3](https://medium.com/the-programmer/working-with-deployment-in-kubernetes-8450989e4f80), [4](https://www.digitalocean.com/community/tutorials/deploying-an-express-application-on-a-kubernetes-cluster), [5](https://kubegrade.com/kubernetes-how-to-deploy/)]

## 2. High Availability & Scaling

These control how many instances run and how they update without causing downtime. [[1](https://kubegrade.com/kubernetes-how-to-deploy/), [2](https://www.gravitee.io/blog/aws-kubernetes-deployment-guide), [3](https://www.plural.sh/blog/kubernetes-storage-guide/), [4](https://medium.com/@kesaralive/how-to-deploy-a-high-available-kubernetes-cluster-on-azure-without-aks-4f2f2f7f099f)]

- **spec.replicas**: The number of Pod instances to run simultaneously for redundancy.
- **spec.strategy**: Defines how updates roll out. Use RollingUpdate with maxSurge (how many extra pods can be created during an update) and maxUnavailable (how many pods can be offline during an update) to achieve zero-downtime deployments. [[1](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/), [2](https://medium.com/@thisara.weerakoon2001/demystifying-kubernetes-yaml-ef9e92acf3df), [3](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/), [4](https://www.plural.sh/blog/deploy-to-kubernetes-guide/), [5](https://kubegrade.com/kubernetes-upgrade-strategies/)]

## 3. Application Reliability (Health Checks)

Never let Kubernetes assume a Pod is working just because the container started. You must define **Probes**: [[1](https://oneuptime.com/blog/post/2026-02-20-kubernetes-deployments-rolling-updates/view), [2](https://devtron.ai/blog/kubernetes-deployment-best-practices/)]

- **livenessProbe**: Checks if the app crashed or deadlocked. If it fails, Kubernetes kills and restarts the container.
- **readinessProbe**: Checks if the app is fully initialized and ready to accept traffic. If it fails, Kubernetes stops sending network requests to it.
- **startupProbe**: (For slow-starting apps) Disables liveness/readiness checks until the app finishes its initial, heavy boot-up sequence. [[1](https://devopscube.com/kubernetes-deployment-tutorial/), [2](https://andrewlock.net/deploying-asp-net-core-applications-to-kubernetes-part-6-adding-health-checks-with-liveness-readiness-and-startup-probes/), [3](https://www.fairwinds.com/blog/a-guide-to-understanding-kubernetes-liveness-probes-best-practices), [4](https://rohithykrohith.medium.com/kubernetes-k8s-explanation-1e56b48f8b78), [5](https://komodor.com/learn/14-kubernetes-best-practices-you-must-know-in-2025/)]

## 4. Resource Governance (Protection)

Prevent a single misbehaving container from taking down your entire node. Always set **resources**: [[1](https://levelup.gitconnected.com/understanding-kubernetes-out-of-memory-oom-6c6cf21cf9dc)]

- **requests**: The minimum CPU and Memory guaranteed to the container to start up.
- **limits**: The maximum CPU and Memory the container is allowed to consume. If it exceeds the memory limit, it will be killed via an Out-Of-Memory (OOM) error. [[1](https://www.plural.sh/blog/kubernetes-definition-explained/), [2](https://medium.com/@vasanthancomrads/kubernetes-resource-limits-and-requests-practical-tuning-tips-d7d0ac0f0127), [3](https://kubegrade.com/kubernetes-how-to-deploy/), [4](https://kubegrade.com/kubernetes-resource-management-2/), [5](https://jc1175.medium.com/a-crash-course-in-running-kubernetes-locally-7c573dd64933)]

## 5. Security Context

Enforce the principle of least privilege to secure your container environment.

- **securityContext**: Drop root privileges inside the container using runAsNonRoot: true, runAsUser: 1000, and set readOnlyRootFilesystem: true to prevent unauthorized file modifications.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
  labels:
    app: my-app
spec:
  replicas: 3 # High Availability
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0 # Zero-downtime settings
  selector:
    matchLabels:
      app: my-app # Must match the template label below
  template:
    metadata:
      labels:
        app: my-app
    spec:
      securityContext:
        runAsNonRoot: true # Security hardening
        runAsUser: 1000
      containers:
      - name: web-container
        image: my-app:v1.2.0
        ports:
        - containerPort: 8080

        resources: # Resource Governance
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"

        livenessProbe: # Reliability
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20

        readinessProbe: # Traffic Control
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10

how to inject sensitive database credentials into this deployment using Secrets/ConfigMaps
To inject sensitive credentials into your Kubernetes Deployment, use a Kubernetes Secret for sensitive data (like passwords, keys, and tokens) and a ConfigMap for non-sensitive settings (like database URLs, usernames, or ports). [1, 2, 3, 4, 5]
Here is the exact process to create them and mount them into your application.

Step 1: Create the Secret and ConfigMap YAMLs
First, define your environment variables. [1]
Note: Data inside a Kubernetes Secret must be Base64 encoded. [1]
```
```yaml
# 1. secret.yaml (For Sensitive Data)
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  # Values must be base64 encoded.
  # Example: "supersecret123" becomes "c3VwZXJzZWNyZXQxMjM="
  db-password: c3VwZXJzZWNyZXQxMjM=

---

# 2. configmap.yaml (For Non-Sensitive Data)
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
data:
  db-url: "jdbc:mysql://db-service:3306/mydb"
  db-username: "app_user"
```

### Step 2: Inject Them Into the Deployment

You can inject these values into your containers in two ways: as **Environment Variables** or as **Mounted Files**. [[1](https://oneuptime.com/blog/post/2025-11-27-configmaps-secrets/view), [2](https://mobycast.fm/secrets-handling-for-containerized-applications-running-on-ecs/)]

### Option A: Inject as Environment Variables (Most Common) [1, 2]

This reads specific keys from the Secret and ConfigMap and maps them directly to traditional environment variables inside your container. [[1](https://www.plural.sh/blog/kubernetes-secret-guide/), [2](https://apxml.com/courses/getting-started-with-kubernetes/chapter-5-configuration-and-persistent-storage/managing-configuration-with-configmaps), [3](https://www.alibabacloud.com/help/en/cs/user-guide/using-a-secret-dictionary-in-a-container-group)]

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: web-container
        image: my-app:v1.2.0
        ports:
        - containerPort: 8080

        # --- INJECTING ENVIRONMENT VARIABLES ---
        env:
          # 1. Injecting non-sensitive data from ConfigMap
          - name: DATABASE_URL
            valueFrom:
              configMapKeyRef:
                name: db-config
                key: db-url
          - name: DB_USER
            valueFrom:
              configMapKeyRef:
                name: db-config
                key: db-username

          # 2. Injecting sensitive data from Secret
          - name: DB_PASSWORD
            valueFrom:
              secretKeyRef:
                name: db-credentials
                key: db-password
```

### Option B: Inject as Mounted Files (Most Secure)

Some applications prefer reading credentials from a file on disk. This is considered more secure because environment variables can sometimes be accidentally leaked in application crash logs. [[1](https://www.groundcover.com/blog/kubernetes-secret-management), [2](https://blog.gitguardian.com/how-to-handle-secrets-in-docker/), [3](https://cloudification.io/cloud-blog/manage-secrets-with-hashicorp-vault/), [4](https://www.plural.sh/blog/hashicorp-vault-kubernetes-guide/)]

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: web-container
        image: my-app:v1.2.0

        # --- MOUNT FILES INTO CONTAINER ---
        volumeMounts:
        - name: secret-volume
          mountPath: "/etc/secrets" # Files will appear here
          readOnly: true

      # --- DEFINE THE VOLUMES ---
      volumes:
      - name: secret-volume
        secret:
          secretName: db-credentials # Must match Secret metadata.name
```

*In this setup, your application code would read the password string directly from the file path /etc/secrets/db-password.*

### Production Best Practices

- **Do not commit Secrets to Git:** Plain Kubernetes Secrets are only encoded in Base64, **not encrypted**. Never push your secret.yaml to public repositories. Use tools like **Sealed Secrets**, **HashiCorp Vault**, or AWS/GCP Secret Managers for production GitOps pipelines. [[1](https://www.ibm.com/docs/en/spems/6.3.0?topic=overview-encrypting-identity-service-credentials-ocp-deployment), [2](https://medium.com/@bavicnative/managing-kubernetes-secrets-securely-kubernetes-secrets-azure-key-vault-and-hashicorp-vault-8bf4411b831a), [3](https://infisical.com/blog/how-to-manage-secrets-in-kubernetes), [4](https://www.mongodb.com/docs/atlas/operator/current/ak8so-secret-storage/), [5](https://www.wiz.io/it-it/academy/application-security/ci-cd-security-best-practices)]
- **Auto-Refreshes:** If you update a ConfigMap or Secret, variables injected as **Environment Variables will NOT update** until the Pod restarts. However, variables mounted as **Files will update automatically** on disk within a minute without restarting the Pod. [[1](https://www.suse.com/c/observability-how-to-create-and-manage-secrets-in-kubernetes/), [2](https://oneuptime.com/blog/post/2026-03-19-rancher-configmaps-secrets/view), [3](https://igboie.medium.com/k8s-deployment-automatic-rollout-restart-when-referenced-secrets-and-configmaps-are-updated-0c74c85c1b4a), [4](https://www.linkedin.com/pulse/day-55-secrets-management-microservices-using-azure-key-chikkela-ultpe), [5](https://docs.bytebase.com/get-started/self-host/external-postgres)]

Would you like to see how to trigger an automatic rolling restart of your deployment whenever a ConfigMap changes, or should we set up a Kubernetes Service to route traffic to these pods? Yes

Let's tackle **both** of these tasks to make your deployment production-ready.

First, we will set up an automatic rolling restart so your pods pick up configuration changes instantly. Then, we will create a Kubernetes Service to expose those pods and route traffic to them safely. [[1](https://help.splunk.com/en/splunk-enterprise/administer/distributed-search/10.4/manage-search-head-clustering/restart-the-search-head-cluster), [2](https://medium.com/@PlanB./keeping-kubernetes-pods-in-sync-with-vault-secrets-automatically-74d5c252fc15), [3](https://medium.com/@nikhilngarg/a-quick-implementation-guide-for-blue-green-deployment-in-kubernetes-70b763dc5f21), [4](https://developer20.com/periodically-restarting-apps-on-k8s/)]

### Part 1: Trigger Automatic Rolling Restarts on ConfigMap Changes

By default, if you update a ConfigMap, pods using it as environment variables **will not** notice the change until they are manually restarted. [[1](https://oneuptime.com/blog/post/2026-02-09-roll-out-configmap-changes-deployments/view), [2](https://medium.com/@paolocarta_it/behavior-of-configmaps-in-kubernetes-when-we-update-them-0ee6d6863cb4), [3](https://labex.io/questions/how-to-update-a-configmap-in-kubernetes-18463)]

The industry-standard trick to automate this is adding a **SHA-256 hash annotation** of your ConfigMap data into the Deployment's Pod template metadata. When the ConfigMap updates, the hash changes, forcing Kubernetes to trigger a zero-downtime rolling update. [[1](https://discuss.google.dev/t/restart-pod-automatically-on-configmap-changes/180274), [2](https://oneuptime.com/blog/post/2026-02-09-configmap-generators-helm-rolling-hash/view)]

### The Deployment Configuration:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
      annotations:
        # This annotation forces a rolling restart whenever the ConfigMap updates
        configmap-version-hash: "${CONFIGMAP_HASH}"
    spec:
      containers:
      - name: web-container
        image: my-app:v1.2.0
        # ... your env/resources configurations here ...
```

### How to automate this in your pipeline:

If you are using a CI/CD tool or a tool like **Helm** or **Kustomize**, they can compute this hash automatically. [[1](https://levelup.gitconnected.com/ci-cd-for-microservices-automating-builds-tests-and-deployments-8084d14f2d17), [2](https://medium.com/teamsnap-engineering/load-testing-a-service-with-20-000-requests-per-second-with-locust-helm-and-kustomize-ea9bea02ae28)]

If you are using plain kubectl, you can inject the hash dynamically in your deploy script using this bash command before applying:

```bash
# Get the hash of the ConfigMap and patch the deployment to force a restart
CONFIG_HASH=$(kubectl get configmap db-config -o yaml | md5sum | awk '{print $1}')
kubectl patch deployment my-app-deployment -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"configmap-version-hash\":\"$CONFIG_HASH\"}}}}}"
```

### Part 2: Create a Kubernetes Service to Route Traffic

Now that your pods can restart cleanly, they need a single, stable entry point to accept network traffic. Pods die and get replaced constantly, but a **Service** gives them a permanent internal IP and DNS name. [[1](https://stackify.com/how-to-restart-a-kubernetes-pod-using-kubectl/), [2](https://www.plural.sh/blog/deploy-to-kubernetes-guide/), [3](https://www.scaler.com/topics/kubernetes/kubernetes-best-practices/), [4](https://www.harness.io/blog/complete-guide-for-gitops-on-kubernetes)]

We will use a **ClusterIP** service for internal traffic (microservice-to-microservice) or a **LoadBalancer** service if your application needs to be accessible from the public internet. [[1](https://www.asptoday.com/p/aspnet-core-and-kubernetes-from-development), [2](https://www.getlimina.ai/en/blog/how-to-autoscale-kubernetes-pods-based-on-gpu), [3](https://ashadali.medium.com/network-policies-hands-on-securing-traffic-in-kubernetes-61353829af03)]

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  type: ClusterIP # Use 'LoadBalancer' instead if you are on cloud (AWS/GCP) and want public access
  selector:
    app: my-app # CRITICAL: This must match the matchLabels in your deployment
  ports:
    - protocol: TCP
      port: 80         # The port exposed by this Service to other applications
      targetPort: 8080 # The port your actual container is listening on inside the pod
```

### How They Work Together

- An internal request hits http://my-app-service:80.
- The Service looks at its selector (app: my-app).
- It finds all running pods managed by your deployment that carry that specific label.
- It load-balances the traffic evenly across those pods, sending the request to container port 8080.
- If a pod is currently restarting due to a ConfigMap change, its **Readiness Probe** will fail, and the Service will automatically stop sending traffic to that specific pod until it is fully ready again. [[1](https://dev.to/pavanbelagatti/kubernetes-deployments-rolling-vs-canary-vs-blue-green-4k9p), [2](https://seifrajhi.github.io/blog/unsung-kubernetes-features-reliable-resilient/), [3](https://www.scaler.com/topics/kubernetes/kubernetes-service-discovery/), [4](https://www.ecosmob.com/blog/kubernetes-traffic-routing-examples/), [5](https://devopscon.io/blog/demystifying-kubernetes-pods-deployments-and-services/)]
Would you like to see how to add an **Ingress Controller** on top of this Service to route external custom domain names (like ://mycompany.com) with SSL termination?

To route a custom domain name (like ://mycompany.com) to your application with automatic SSL termination, you use a **Kubernetes Ingress resource** managed by an Ingress Controller (like NGINX).

The Ingress resource acts as an intelligent entry point that reads incoming HTTP/HTTPS hostnames and paths, manages SSL certificates, and routes traffic down to your Kubernetes Service.

### Step 1: Create the Secret for SSL Certificates

Before setting up the Ingress rule, your Kubernetes cluster needs access to your SSL certificate and private key. You bundle these into a special kubernetes.io/tls type Secret.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-app-tls-secret
  namespace: default
type: kubernetes.io/tls
data:
  # Base64 encoded contents of your certificate file
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0t...
  # Base64 encoded contents of your private key file
  tls.key: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0t...
```

*(Tip: In a true production environment, you can automate certificate management and renewal using* ***Cert-Manager*** *combined with Let's Encrypt).*

### Step 2: Create the Ingress Configuration

This configuration tells the Ingress Controller to listen for requests hitting ://mycompany.com, terminate the SSL encryption using your TLS secret, and forward the plain traffic down to your my-app-service on port 80.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    # Specifies that the NGINX Ingress Controller should manage this rule
    kubernetes.io/ingress.class: "nginx"

    # Optional: Force HTTP to HTTPS redirection
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Optional: Increase body size limit if your API handles large file uploads
    nginx.ingress.kubernetes.io/proxy-body-size: "20m"
spec:
  tls:
  - hosts:
    - ://mycompany.com
    secretName: my-app-tls-secret # Matches the Secret created in Step 1
  rules:
  - host: ://mycompany.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-service # Matches your Kubernetes Service name
            port:
              number: 80         # Matches the port exposed by your Service
```

### How the Complete Traffic Flow Works

Now, your entire architecture ties together in this lifecycle:

- **DNS Lookup:** A client visits https://mycompany.com. The DNS points to the external IP address of your cluster's **Ingress Controller**.
- **SSL Handshake:** The Ingress Controller intercept the request, looks up my-app-tls-secret, terminates the SSL connection safely, and reads the raw request data.
- **Routing Rule:** The Ingress matches the host (://mycompany.com) and path (/), then forwards the traffic internally to your **Service** (my-app-service:80).
- **Load Balancing:** The Service balances the request down to an available, healthy **Pod** on port 8080.
- **Zero Downtime:** If you update a ConfigMap, your pipeline updates the pod template annotation hash, triggering a **Rolling Restart**. The old pods are killed *only after* new pods pass their **Readiness Probes**, ensuring the Ingress and Service never forward web traffic to a broken or offline instance.
Would you like to explore setting up an **HPA (Horizontal Pod Autoscaler)** next, so your deployment automatically adds more pod replicas when web traffic spikes?

You said: yes

To handle sudden spikes in web traffic automatically, you use a **Horizontal Pod Autoscaler (HPA)**.

The HPA continuously monitors the resource usage (like CPU or Memory) of your pods. If the average usage crosses a threshold you define, it instructs your Deployment to spin up more replicas. When traffic dies down, it scales them back down to save resources.

### Step 1: Ensure Your Deployment Defines Resource "Requests"

The HPA cannot scale your pods if it doesn't know what "100% usage" means. You must have resources.requests explicitly defined in your Deployment YAML (from our earlier step), otherwise the HPA will fail to calculate percentages.

```yaml
# Quick reminder of what must be in your Deployment container spec:
resources:
  requests:
    cpu: "250m"      # HPA uses this baseline number to calculate CPU %
    memory: "256Mi"
```

### Step 2: Create the Horizontal Pod Autoscaler YAML

This configuration ensures your application scales dynamically between 3 and 10 replicas based on real-time demands.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app-deployment # Must exactly match your Deployment name
  minReplicas: 3            # Absolute minimum pods to run during quiet periods
  maxReplicas: 10           # Maximum pods allowed during peak traffic
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70 # Scales up if average CPU across all pods exceeds 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80 # Scales up if average Memory exceeds 80%
```

### Step 3: Add a Stabilization Window (Production Pro-Tip)

By default, the HPA can sometimes scale down too quickly if traffic fluctuates wildly for a brief moment—a problem known as "flapping." You can add a behavior block to make scaling down gradual and smooth.

```yaml
spec:
  # Add this inside your HPA spec to control scaling speed
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300 # Wait 5 minutes before deciding to delete pods
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60 # Scale down by a maximum of 10% of pods per minute
```

### Crucial Requirement: The Metrics Server

For the HPA to actually work, your Kubernetes cluster must have the **Kubernetes Metrics Server** installed. It is the component that collects kubectl top pods data and feeds it to the HPA.

If you are using a managed cloud provider (like AWS EKS, Google GKE, or Azure AKS), this is usually built-in or a one-click install. If you are running a custom or local cluster, you must deploy it using:

```bash
kubectl apply -f https://github.com
```

### Checking Your HPA Status

Once deployed, you can watch your HPA scale your cluster in real-time using this command:

```bash
kubectl get hpa my-app-hpa --watch
```

You will see the target percentages, current utilization, and the active replica count shift up and down as your Ingress routes traffic into the cluster.

Would you like to see how to gracefully shut down your application instances during a scale-down or update using **PreStop lifecycle hooks** so no active user requests are abruptly dropped?

You said: yes

To prevent active user requests from being abruptly cut off when your Horizontal Pod Autoscaler (HPA) scales down or during a rolling update, you must implement a **Graceful Shutdown** mechanism.

By default, when Kubernetes decides to kill a pod, it sends a SIGTERM signal to your application and immediately tells the Service/Ingress to stop sending traffic. However, because network changes take a few seconds to propagate, new traffic might still hit the pod right after it receives the SIGTERM. If your application shuts down instantly, users will see 502 Bad Gateway errors.

To fix this, you combine a **preStop lifecycle hook** with your container's configuration.

### Step 1: Add the preStop Hook to Your Deployment

The preStop hook executes *before* Kubernetes sends the termination signal to your application code. The industry-standard approach is to make the pod sleep for 5 to 15 seconds. This gives the network routers (Ingress, Services, and IPTables) enough time to stop routing new traffic to this pod while your app continues to process any requests that were already mid-flight.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
spec:
  replicas: 3
  template:
    spec:
      # --- CRITICAL STEP 2: GIVE THE POD ENOUGH TIME TO SHUT DOWN ---
      # Default is 30s. If your app needs 15s to drain + 5s sleep, 30s is fine.
      terminationGracePeriodSeconds: 30

      containers:
      - name: web-container
        image: my-app:v1.2.0

        # --- CRITICAL STEP 1: LIFECYCLE HOOKS ---
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
```

### Step 2: Handle SIGTERM In Your Application Code

Once the preStop hook finishes its sleep 15, Kubernetes finally sends the SIGTERM signal to Process 1 (PID 1) inside your container. Your application code must catch this signal and stop accepting *new* connections, while completely finishing any *existing* database or web queries currently processing.

Here is how you handle it across different language runtimes:

### In Node.js (Express)

```javascript
const server = app.listen(8080);

process.on('SIGTERM', () => {
  console.log('SIGTERM signal received: closing HTTP server');
  server.close(() => {
    console.log('HTTP server closed. Exiting process safely.');
    process.exit(0);
  });
});
```

### In Java (Spring Boot)

Spring Boot has built-in support for this. You don't even need code; you just add these properties to your application.properties or application.yml file:

```properties
# Enable graceful shutdown
server.shutdown=graceful

# Timeout period for ongoing requests to finish
spring.lifecycle.timeout-per-shutdown-phase=20s
```

### The Complete Production Timeline of a Pod Shutdown

When your HPA scales down your cluster from 10 pods to 3, each deleted pod goes through this safe timeline:

- **HPA triggers scale down:** Kubernetes sets the Pod status to Terminating.
- **Endpoint removal:** The Pod is removed from the Endpoints list of your **Kubernetes Service** and **Ingress**.
- **PreStop Hook executes:** The container runs sleep 15. During these 15 seconds, the pod is still alive. If any straggling network packets arrive due to a delay in network synchronization, your app handles them seamlessly.
- **SIGTERM is issued:** After 15 seconds, Kubernetes sends a SIGTERM. Your app code stops accepting new connections and safely finishes writing active database records.
- **SIGKILL clean up:** If your app is still running after the terminationGracePeriodSeconds (30 seconds total) expires, Kubernetes sends a SIGKILL to force-stop it.
With this final piece, your cluster now has **secure credentials**, **automated routing**, **SSL encryption**, **traffic autoscaling**, and **zero-downtime graceful updates**.
