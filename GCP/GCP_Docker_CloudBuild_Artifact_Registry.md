# GCP — Automated Docker Image Build Pipeline
## Complete Guide: Cloud Build · Artifact Registry · Cloud Run · GKE

> Every time you push code, GCP automatically builds the Docker image, stores it securely, and deploys it — zero manual steps.

---

## Pipeline Overview

```
Git push → Cloud Build → Artifact Registry → Cloud Run / GKE
```

| Service | Role |
|---|---|
| **Cloud Build** | CI engine — executes `cloudbuild.yaml`, runs tests, builds Docker image, pushes to registry |
| **Artifact Registry** | Managed Docker registry inside GCP — stores images with vulnerability scanning and IAM access control |
| **Cloud Build Triggers** | Watch your repo and fire builds automatically on push, PR, or tag |
| **Cloud Run / GKE** | Deployment target — Cloud Build deploys the freshly built image directly |

> **Cost:** Cloud Build gives you 120 free build-minutes per day. A typical Spring Boot build takes 3–5 minutes, so you can run 20–30 builds/day free.

---

## Step 1 — One-time Setup

Run these commands once per project before any automation works.

### 1.1 Enable required APIs

```bash
gcloud services enable \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  run.googleapis.com \
  container.googleapis.com
```

### 1.2 Create an Artifact Registry repository

```bash
gcloud artifacts repositories create my-docker-repo \
  --repository-format=docker \
  --location=us-central1 \
  --description="Docker images for my app"

# Your image URL pattern will be:
# us-central1-docker.pkg.dev/PROJECT_ID/my-docker-repo/IMAGE_NAME:TAG
```

### 1.3 Grant Cloud Build the required IAM permissions

Cloud Build has its own service account — it needs explicit permission to push images and deploy. Without these bindings, every automated build will fail with "permission denied."

```bash
PROJECT_ID=$(gcloud config get-value project)
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID \
  --format='value(projectNumber)')
CB_SA="${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com"

# Allow Cloud Build to push to Artifact Registry
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${CB_SA}" \
  --role="roles/artifactregistry.writer"

# Allow Cloud Build to deploy to Cloud Run
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:${CB_SA}" \
  --role="roles/run.admin"

# Allow Cloud Build to act as the compute service account
gcloud iam service-accounts add-iam-policy-binding \
  "${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --member="serviceAccount:${CB_SA}" \
  --role="roles/iam.serviceAccountUser"
```

### 1.4 Verify the repository

```bash
gcloud artifacts repositories list --location=us-central1
```

---

## Step 2 — Dockerfile Best Practices

A well-written Dockerfile dramatically reduces build times because Cloud Build caches Docker layers between builds. **Put things that change rarely near the top (base image, dependency install) and things that change often near the bottom (your source code).**

### 2.1 Spring Boot — multi-stage Dockerfile

```dockerfile
# Stage 1: build the JAR
FROM maven:3.9-eclipse-temurin-17-alpine AS builder
WORKDIR /app

# Copy dependencies first — cached unless pom.xml changes
COPY pom.xml .
RUN mvn dependency:go-offline -q

# Copy source and build
COPY src ./src
RUN mvn package -DskipTests -q

# Stage 2: minimal runtime image (no JDK in production)
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app

# Non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

COPY --from=builder /app/target/*.jar app.jar

# Cloud Run sends requests to port 8080 by default
EXPOSE 8080

# JVM flags optimised for containers
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-jar", "app.jar"]
```

### 2.2 Angular — multi-stage Dockerfile

```dockerfile
# Stage 1: build Angular SPA
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --silent           # faster than npm install, uses lockfile
COPY . .
RUN npm run build -- --output-path=dist --configuration=production

# Stage 2: Nginx serves the static files
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 8080
CMD ["nginx", "-g", "daemon off;"]
```

### 2.3 .dockerignore — always add this

```
target/
node_modules/
.git/
*.log
.env
test-results/
.dockerignore
```

Excluding these directories keeps the build context small and upload to Cloud Build fast.

---

## Step 3 — cloudbuild.yaml

This file lives in the root of your repo. Cloud Build reads it and executes each step in order when a trigger fires.

### 3.1 Built-in substitution variables

| Variable | Value |
|---|---|
| `$PROJECT_ID` | Your GCP project ID |
| `$SHORT_SHA` | First 7 characters of the commit hash |
| `$COMMIT_SHA` | Full commit hash |
| `$BRANCH_NAME` | Git branch name |
| `$TAG_NAME` | Git tag (if trigger is tag-based) |
| `$BUILD_ID` | Unique build ID |

### 3.2 Basic cloudbuild.yaml (test + build + push + deploy)

```yaml
# cloudbuild.yaml — Spring Boot + Cloud Run
steps:

  # 1. Run unit tests — fail fast before building the image
  - name: 'maven:3.9-eclipse-temurin-17'
    id: 'run-tests'
    entrypoint: mvn
    args: ['test', '-B', '-q', '--no-transfer-progress']
    env:
      - 'MAVEN_OPTS=-Xmx1g'

  # 2. Build Docker image with layer caching
  - name: 'gcr.io/cloud-builders/docker'
    id: 'build-image'
    waitFor: ['run-tests']
    args:
      - build
      - --cache-from
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/api:latest'
      - -t
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/api:$SHORT_SHA'
      - -t
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/api:latest'
      - .

  # 3. Push both tags to Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    id: 'push-image'
    waitFor: ['build-image']
    args:
      - push
      - '--all-tags'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/api'

  # 4. Deploy to Cloud Run
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    id: 'deploy'
    waitFor: ['push-image']
    entrypoint: gcloud
    args:
      - run deploy my-api
      - --image=us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/api:$SHORT_SHA
      - --region=us-central1
      - --platform=managed
      - --allow-unauthenticated
      - --min-instances=0
      - --max-instances=10
      - --memory=512Mi

options:
  machineType: 'E2_HIGHCPU_8'
  logging: CLOUD_LOGGING_ONLY

timeout: '1200s'
```

> **Key concept:** Use `$SHORT_SHA` as your image tag so every image is traceable to a specific commit. Never overwrite the same tag in production — always use the commit hash.

---

## Step 4 — Cloud Build Triggers

Triggers watch your Git repo and automatically fire a build when conditions are met — no manual `gcloud builds submit` needed.

### 4.1 Connect GitHub and create triggers

```bash
# Push-to-main trigger (most common)
gcloud builds triggers create github \
  --name="build-on-push-main" \
  --repo-name="my-app" \
  --repo-owner="my-github-org" \
  --branch-pattern="^main$" \
  --build-config="cloudbuild.yaml" \
  --description="Build and deploy on push to main"

# PR trigger — test only, no deploy
gcloud builds triggers create github \
  --name="test-on-pr" \
  --repo-name="my-app" \
  --repo-owner="my-github-org" \
  --pull-request-pattern=".*" \
  --build-config="cloudbuild-test-only.yaml" \
  --description="Run tests on every pull request"

# Release tag trigger — build on tags like v1.2.3
gcloud builds triggers create github \
  --name="build-on-release-tag" \
  --repo-name="my-app" \
  --repo-owner="my-github-org" \
  --tag-pattern="^v[0-9]+\.[0-9]+\.[0-9]+$" \
  --build-config="cloudbuild-release.yaml"

# File path filter — only trigger when source files change (monorepo pattern)
gcloud builds triggers create github \
  --name="build-api-on-src-change" \
  --repo-name="my-monorepo" \
  --repo-owner="my-github-org" \
  --branch-pattern="^main$" \
  --build-config="services/api/cloudbuild.yaml" \
  --included-files="services/api/**" \
  --description="Only rebuild API when api source changes"
```

### 4.2 Useful trigger management commands

```bash
# List all triggers
gcloud builds triggers list

# Manually run a trigger (useful for testing)
gcloud builds triggers run build-on-push-main --branch=main

# View recent builds
gcloud builds list --limit=10

# Stream logs of a specific build
gcloud builds log BUILD_ID --stream
```

### 4.3 Branch pattern reference

| Pattern | Matches |
|---|---|
| `^main$` | Only the `main` branch |
| `^main$\|^release/.*` | `main` and any `release/` branch |
| `^feature/.*` | Any feature branch |
| `.*` | Every branch (usually too broad) |

---

## Step 5 — Deployment Targets

Cloud Build can deploy the built image to Cloud Run, GKE, or any target reachable with a `gcloud` command.

### 5.1 Cloud Run — standard deploy

```yaml
- name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
  entrypoint: gcloud
  args:
    - run deploy my-api
    - --image=us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/api:$SHORT_SHA
    - --region=us-central1
    - --allow-unauthenticated
    - --min-instances=2
    - --max-instances=50
    - --cpu=2
    - --memory=1Gi
```

### 5.2 Cloud Run — canary deploy (no-traffic pattern)

```yaml
# Deploy new revision without sending any traffic to it
- name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
  id: 'deploy-canary'
  entrypoint: gcloud
  args:
    - run deploy my-api
    - --image=us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/api:$SHORT_SHA
    - --region=us-central1
    - --no-traffic                  # new revision gets 0% traffic
    - --tag=candidate-$SHORT_SHA    # label this revision

# Send 20% traffic to the new revision
- name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
  id: 'shift-canary-traffic'
  waitFor: ['deploy-canary']
  entrypoint: gcloud
  args:
    - run services update-traffic my-api
    - --region=us-central1
    - --to-tags=candidate-$SHORT_SHA=20

# After monitoring — promote to 100%:
# gcloud run services update-traffic my-api --region=us-central1 --to-latest
```

### 5.3 GKE — rolling update

```yaml
- name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
  entrypoint: bash
  args:
    - '-c'
    - |
      gcloud container clusters get-credentials my-cluster \
        --region=us-central1
      kubectl set image deployment/my-api \
        api=us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/api:$SHORT_SHA
      kubectl rollout status deployment/my-api --timeout=5m
```

### 5.4 Inject secrets at deploy time

Store all credentials in Secret Manager — never put them in `cloudbuild.yaml` as plain text.

```yaml
# cloudbuild.yaml — reference Secret Manager secrets
availableSecrets:
  secretManager:
    - versionName: projects/$PROJECT_ID/secrets/db-password/versions/latest
      env: 'DB_PASSWORD'
    - versionName: projects/$PROJECT_ID/secrets/sonar-token/versions/latest
      env: 'SONAR_TOKEN'

steps:
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    id: 'deploy-with-secrets'
    secretEnv: ['DB_PASSWORD']
    entrypoint: gcloud
    args:
      - run deploy my-api
      - --image=us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/api:$SHORT_SHA
      - --update-secrets=DB_PASSWORD=db-password:latest
      - --region=us-central1
```

---

## Step 6 — Full Production Pipeline

Production-grade `cloudbuild.yaml` with all stages: test → SonarQube → build → push → vulnerability scan → staging deploy → integration tests → production canary deploy.

```yaml
# cloudbuild.yaml — production pipeline
steps:

  # ── STAGE 1: TEST ─────────────────────────────────────────────
  - name: 'maven:3.9-eclipse-temurin-17'
    id: 'unit-tests'
    entrypoint: mvn
    args: ['test', '-B', '-q', '--no-transfer-progress']
    env: ['MAVEN_OPTS=-Xmx1g']

  - name: 'maven:3.9-eclipse-temurin-17'
    id: 'sonar-scan'
    waitFor: ['unit-tests']
    entrypoint: mvn
    args:
      - 'sonar:sonar'
      - '-Dsonar.host.url=https://sonar.example.com'
      - '-Dsonar.login=$$SONAR_TOKEN'
    secretEnv: ['SONAR_TOKEN']

  # ── STAGE 2: BUILD IMAGE ───────────────────────────────────────
  # Pull previous image for layer cache (allowed to fail if first build)
  - name: 'gcr.io/cloud-builders/docker'
    id: 'pull-cache'
    waitFor: ['-']
    args:
      - pull
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/api:latest'
    allowFailure: true

  - name: 'gcr.io/cloud-builders/docker'
    id: 'build-image'
    waitFor: ['unit-tests', 'pull-cache']
    args:
      - build
      - --cache-from
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/api:latest'
      - -t
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/api:$SHORT_SHA'
      - -t
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/api:latest'
      - .

  # ── STAGE 3: PUSH AND SCAN ────────────────────────────────────
  - name: 'gcr.io/cloud-builders/docker'
    id: 'push-image'
    waitFor: ['build-image']
    args: ['push', '--all-tags',
           'us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/api']

  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    id: 'vulnerability-scan'
    waitFor: ['push-image']
    entrypoint: bash
    args:
      - '-c'
      - |
        gcloud artifacts docker images describe \
          us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/api:$SHORT_SHA \
          --show-package-vulnerability 2>/dev/null || true

  # ── STAGE 4: DEPLOY TO STAGING ────────────────────────────────
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    id: 'deploy-staging'
    waitFor: ['push-image']
    entrypoint: gcloud
    args:
      - run deploy staging-api
      - --image=us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/api:$SHORT_SHA
      - --region=us-central1
      - --allow-unauthenticated
      - --set-env-vars=ENVIRONMENT=staging
      - --update-secrets=DB_PASSWORD=staging-db-password:latest
      - --min-instances=0
      - --max-instances=3

  # ── STAGE 5: INTEGRATION TESTS AGAINST STAGING ────────────────
  - name: 'node:20-alpine'
    id: 'integration-tests'
    waitFor: ['deploy-staging']
    entrypoint: sh
    args:
      - '-c'
      - |
        npm ci --silent
        STAGING_URL=$(gcloud run services describe staging-api \
          --region=us-central1 --format='value(status.url)')
        API_BASE_URL=$STAGING_URL npm run test:integration

  # ── STAGE 6: DEPLOY TO PRODUCTION ─────────────────────────────
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    id: 'deploy-prod'
    waitFor: ['integration-tests']
    entrypoint: gcloud
    args:
      - run deploy production-api
      - --image=us-central1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/api:$SHORT_SHA
      - --region=us-central1
      - --no-traffic
      - --tag=candidate-$SHORT_SHA
      - --update-secrets=DB_PASSWORD=prod-db-password:latest
      - --min-instances=2
      - --max-instances=50
      - --cpu=2
      - --memory=1Gi

  # Send 20% traffic to new revision (canary)
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    id: 'canary-traffic'
    waitFor: ['deploy-prod']
    entrypoint: gcloud
    args:
      - run services update-traffic production-api
      - --region=us-central1
      - --to-tags=candidate-$SHORT_SHA=20

# ── SECRETS ───────────────────────────────────────────────────────
availableSecrets:
  secretManager:
    - versionName: projects/$PROJECT_ID/secrets/sonar-token/versions/latest
      env: 'SONAR_TOKEN'

# ── BUILD OPTIONS ─────────────────────────────────────────────────
options:
  machineType: 'E2_HIGHCPU_8'
  logging: CLOUD_LOGGING_ONLY
  env:
    - 'DOCKER_BUILDKIT=1'

timeout: '1800s'
```

---

## Monitoring and Troubleshooting

```bash
# View recent builds with status
gcloud builds list --limit=10

# Stream logs for a running build
gcloud builds log BUILD_ID --stream

# View a specific build's details
gcloud builds describe BUILD_ID

# Check what's in Artifact Registry
gcloud artifacts docker images list \
  us-central1-docker.pkg.dev/PROJECT_ID/my-docker-repo

# Check vulnerability scan results
gcloud artifacts docker images describe \
  us-central1-docker.pkg.dev/PROJECT_ID/my-docker-repo/api:TAG \
  --show-package-vulnerability

# Check Cloud Run revision status after deploy
gcloud run revisions list --service=my-api --region=us-central1

# Promote canary to 100% after monitoring
gcloud run services update-traffic my-api \
  --region=us-central1 \
  --to-latest

# Rollback — send traffic back to a previous revision
gcloud run services update-traffic my-api \
  --region=us-central1 \
  --to-revisions=PREVIOUS_REVISION_NAME=100
```

---

## Quick Reference

| What you want | Command |
|---|---|
| Submit a build manually | `gcloud builds submit --config=cloudbuild.yaml .` |
| List triggers | `gcloud builds triggers list` |
| Run a trigger manually | `gcloud builds triggers run TRIGGER_NAME --branch=main` |
| View build history | `gcloud builds list --limit=20` |
| Stream build logs | `gcloud builds log BUILD_ID --stream` |
| List images in registry | `gcloud artifacts docker images list REPO_URL` |
| Scan image for CVEs | `gcloud artifacts docker images describe IMAGE_URL --show-package-vulnerability` |
| List Cloud Run revisions | `gcloud run revisions list --service=SERVICE_NAME --region=REGION` |
| Promote canary to 100% | `gcloud run services update-traffic SERVICE --to-latest --region=REGION` |
| Rollback to previous | `gcloud run services update-traffic SERVICE --to-revisions=NAME=100 --region=REGION` |

---

*GCP Docker Build Automation · Cloud Build · Artifact Registry · Cloud Run · GKE · May 2025*
