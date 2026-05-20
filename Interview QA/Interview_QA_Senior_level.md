

"Walk me through your experience with microservices architecture and Spring Boot. Give me a specific example of a system you designed — 
  what the problem was, what decisions you made, and what the outcome was."
=============================================================================================================================================

"At Ford I designed SCA-V — a distributed microservices platform processing millions of vehicle telemetry events per day across four global regions. The legacy system was a monolithic processor that couldn't scale beyond a single thread. I broke it into independently deployable Spring Boot microservices — one per data type: CAN decoding, DID decoding, telemetry ingestion. Each service communicated via Kafka topics partitioned by VIN, so all events for a given vehicle always landed on the same consumer thread and were processed in order. I implemented Resilience4j circuit breakers between services so a failure in the DID decoder didn't cascade into the telemetry ingestion layer. The result was a 40% throughput improvement and zero message loss across three years of production. That architecture became the reference design for two other Ford platform teams."


###############################################################################################################
"Describe a specific performance issue you faced in a Java application. What was the symptom, what was the root cause, and exactly what did you do to fix it?"
=========================================================================================================

At CareFirst we had a production incident where API response time spiked from 60ms to 500ms. Our SLA required resolution within 24 hours before client escalation. I immediately set up a war room with four engineers.
Using CloudWatch, we pulled thread dumps and spotted that one specific service had 90% of its threads in WAITING state. We traced it to a Hibernate query inside a loop — for each of 200 member records we were firing a separate SELECT to fetch benefit data. That's an N+1 query problem — 200 network round-trips instead of one joined query. Under low load it was invisible. Under production load it saturated our connection pool.
The fix was a single JPQL JOIN FETCH that retrieved all 200 benefit records in one query. We deployed to staging, ran a JMeter load test at 5,000 concurrent users, and confirmed response time dropped to 30ms — half the original SLA target. We pushed to production within 14 hours of the alert. Client never escalated.
After the incident I added a rule to our SonarQube quality gate that flagged Hibernate queries inside loops as a blocker — so the same class of bug couldn't reach production again."


#################################################################################################################

"Walk me through your hands-on experience with Docker, Kubernetes, and CI/CD pipelines. One specific system — your decisions, your trade-offs, measurable outcome."
==================================================================================================================


"At CareFirst I designed and built the full CI/CD pipeline for 8 Spring Boot microservices moving from PCF to Kubernetes on AWS EKS.
The pipeline had five stages in Jenkins. First, Maven pulled the code from Git and ran all unit and integration tests. If any test failed, the pipeline stopped — nothing moved forward. Second, SonarQube ran a quality gate — we set an 80% coverage threshold as a hard blocker. Third, Docker built the image using a multi-stage Dockerfile. The first stage used the full JDK for compilation, the second stage used only the JRE Alpine image. That brought image size from 680MB down to 180MB — faster pod startup, smaller attack surface. Fourth, the image was pushed to AWS ECR tagged with the commit SHA so every image was traceable to a specific commit. Fifth, Helm deployed to the target environment.
In Kubernetes I configured liveness and readiness probes on the Spring Actuator health endpoint. The readiness probe was critical — it prevented traffic routing to a pod until the application had fully started and the database connection pool was warm. Before this we had intermittent 502 errors during deployments.
I chose rolling update over blue/green — maxSurge one, maxUnavailable zero. Blue/green doubles infrastructure cost during the deployment window. Rolling update with readiness probes gave us the same safety at half the cost.
The outcome: deployment lead time dropped from 3 days to 4 hours. We moved from weekly to daily releases. Zero downtime across all production deployments after the migration."


explain these concepts:    Kubernetes Features We Implemented
✅ rolling deployments
✅ auto scaling
✅ self healing
✅ blue-green deployments
✅ zero downtime releases
✅ readiness/liveness probes


Let me walk you through these Kubernetes features step-by-step, building from the fundamentals to how they work together.

Rolling Deployments
This is Kubernetes' default deployment strategy that updates your application gradually without downtime. Instead of replacing all pods at once, Kubernetes incrementally replaces old pods with new ones—updating a few at a time. For example, if you have 10 pods running version 1, Kubernetes might replace 2 pods with version 2, monitor their health, then continue until all are updated. This ensures users always have access to a working version during updates.

Auto Scaling
Kubernetes provides three autoscaling mechanisms:

Horizontal Pod Autoscaler (HPA): Automatically adds or removes pod replicas based on CPU, memory, or custom metrics

Vertical Pod Autoscaler (VPA): Adjusts CPU and memory limits for existing pods

Cluster Autoscaler: Adds or removes worker nodes based on resource demands

These work together to optimize resource usage and costs while maintaining performance.

Self-Healing
Kubernetes continuously monitors your cluster and automatically recovers from failures. When a container crashes, Kubernetes restarts it based on the restart policy. If a pod fails health checks, controllers (like Deployments) automatically replace it to maintain your desired replica count. The system also removes unhealthy pods from service endpoints so traffic only goes to healthy instances.

Blue-Green Deployments
This strategy maintains two identical production environments: "blue" (current version) and "green" (new version). You deploy the new version to the green environment, test it thoroughly, then switch all traffic from blue to green instantly. In Kubernetes, you use services and ingress controllers to route traffic between environments. If issues arise, you can instantly rollback by switching traffic back to blue.

Zero Downtime Releases
This combines rolling deployments with readiness probes. Kubernetes ensures new pods are fully ready before routing traffic to them and keeps old pods running until replacements are healthy. This guarantees continuous service availability during updates.

Readiness and Liveness Probes
These are health checks that enable self-healing:

Liveness Probe: Checks if a container is running properly. If it fails, Kubernetes restarts the container

Readiness Probe: Checks if a container can serve requests. If it fails, Kubernetes removes the pod's IP from service endpoints (stops sending traffic) but doesn't restart it

Each probe can use HTTP requests, TCP connections, or command execution to check health.

