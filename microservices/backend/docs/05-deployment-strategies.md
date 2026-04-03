# 05 — Deployment Strategies

## Table of Contents

- [Containerisation with Docker](#containerisation-with-docker)
- [Container Orchestration with Kubernetes](#container-orchestration-with-kubernetes)
- [CI/CD Pipelines](#cicd-pipelines)
- [Deployment Strategies](#deployment-strategies)
- [Health Checks and Readiness](#health-checks-and-readiness)
- [Configuration Management](#configuration-management)
- [Kubernetes Manifests](#kubernetes-manifests)
- [Helm Charts](#helm-charts)
- [Environment Promotion](#environment-promotion)
- [Rollback Procedures](#rollback-procedures)
- [Summary & Next Steps](#summary--next-steps)

---

## Containerisation with Docker

Every microservice ships as a Docker container image. The image is the deployable unit — it packages the application code, runtime, and all dependencies into a reproducible, immutable artifact.

### Writing a Production-Grade Dockerfile

```dockerfile
# ── Stage 1: build ──────────────────────────────────────────────────────────
FROM node:20-alpine AS builder

WORKDIR /app

# Copy dependency files first — layer cache busts only when they change
COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build

# ── Stage 2: runtime ─────────────────────────────────────────────────────────
FROM node:20-alpine AS runtime

# Security: run as non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser  -u 1001 -S appuser -G appgroup

WORKDIR /app

# Copy only what the runtime needs — not the full build toolchain
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=builder --chown=appuser:appgroup /app/package.json ./

USER appuser

# Expose the port your app listens on
EXPOSE 3000

# Prefer exec form — ensures signals (SIGTERM) reach your process, not a shell
CMD ["node", "dist/server.js"]
```

### Multi-Stage Build Benefits

- Stage 1 contains compilers, test tools, and dev dependencies — never shipped
- Stage 2 image is small (often 80–90% smaller) and has a minimal attack surface
- Build cache is maximised by copying `package.json` before source files

### .dockerignore

Always include a `.dockerignore` to prevent bloating the build context:

```
node_modules
.git
*.log
.env*
coverage
dist
*.test.ts
README.md
```

### Image Tagging Strategy

Never deploy `:latest` in production — it's not reproducible and you can't roll back to a specific version.

```bash
# Tag with git SHA for exact traceability
docker build -t order-service:${GIT_SHA} .
docker tag  order-service:${GIT_SHA} registry.example.com/order-service:${GIT_SHA}
docker push registry.example.com/order-service:${GIT_SHA}

# Also tag with semantic version for human readability
docker tag order-service:${GIT_SHA} registry.example.com/order-service:1.4.2
```

---

## Container Orchestration with Kubernetes

Kubernetes (K8s) is the standard platform for running containers in production at scale. It handles scheduling, self-healing, service discovery, scaling, and rolling updates.

### Core Resource Types

| Resource                  | What it does                                                                           |
| ------------------------- | -------------------------------------------------------------------------------------- |
| `Pod`                     | The smallest deployable unit — one or more containers sharing a network namespace      |
| `Deployment`              | Manages a ReplicaSet of identical pods — handles rollouts and rollbacks                |
| `Service`                 | Stable DNS name and virtual IP that load-balances across pod replicas                  |
| `ConfigMap`               | Non-sensitive configuration as key-value pairs — injected as env vars or mounted files |
| `Secret`                  | Sensitive configuration (passwords, tokens) — base64-encoded, access controlled        |
| `HorizontalPodAutoscaler` | Automatically scales pod count based on CPU, memory, or custom metrics                 |
| `Ingress`                 | Routes external HTTP/HTTPS traffic to internal Services                                |
| `Namespace`               | Virtual cluster partition — isolates resources per team or environment                 |
| `PersistentVolumeClaim`   | Requests durable storage for stateful workloads                                        |

### Local Development with Kubernetes

```bash
# Minikube — single-node local cluster
minikube start --driver=docker --cpus=4 --memory=8192

# k3d — lightweight k3s in Docker (faster startup)
k3d cluster create dev --servers 1 --agents 2

# kind — Kubernetes IN Docker
kind create cluster --name dev
```

---

## CI/CD Pipelines

Each microservice has its own independent CI/CD pipeline. Teams deploy on their own schedule without coordinating with other teams — this is the operational realisation of service autonomy.

### Pipeline Stages

A complete pipeline for a microservice:

```yaml
# .github/workflows/deploy.yml
name: Deploy order-service

on:
  push:
    branches: [main]
    paths:
      - "services/order-service/**" # only run when this service changes

env:
  SERVICE: order-service
  REGISTRY: registry.example.com

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Unit tests
        run: npm run test:unit

      - name: Integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: ${{ secrets.TEST_DATABASE_URL }}

  build-image:
    needs: build-and-test
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4

      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.SERVICE }}
          tags: |
            type=sha,prefix=,format=short
            type=semver,pattern={{version}}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    needs: build-image
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Deploy to staging
        run: |
          kubectl set image deployment/${{ env.SERVICE }} \
            ${{ env.SERVICE }}=${{ needs.build-image.outputs.image-tag }} \
            --namespace=staging

      - name: Wait for rollout
        run: |
          kubectl rollout status deployment/${{ env.SERVICE }} \
            --namespace=staging --timeout=5m

      - name: Smoke tests
        run: npm run test:smoke -- --env=staging

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production # requires manual approval
    steps:
      - name: Deploy to production (canary)
        run: |
          # Deploy canary with 10% traffic weight
          kubectl apply -f k8s/canary-deploy.yaml

      - name: Monitor canary metrics
        run: |
          # Check error rate and latency for 5 minutes
          ./scripts/check-canary-metrics.sh --duration=5m --error-threshold=1

      - name: Promote canary to full rollout
        run: |
          kubectl apply -f k8s/production-deploy.yaml
          kubectl delete -f k8s/canary-deploy.yaml
```

### Pipeline Non-Negotiables

Every pipeline must enforce these gates — no exceptions:

1. Linting and static analysis — catches style and type errors before tests run
2. Unit tests with coverage threshold (fail below 80%)
3. Integration tests against real dependencies (test database, broker)
4. Container image security scan (Trivy, Snyk) — fail on HIGH or CRITICAL CVEs
5. Staging deployment with smoke tests before any production promotion
6. Human approval gate before production (for regulated environments)

---

## Deployment Strategies

### Rolling Update (Kubernetes Default)

Kubernetes replaces pods incrementally — a few at a time — until all replicas run the new version.

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0 # never take pods offline during rollout
      maxSurge: 1 # run 1 extra pod during transition
```

Pros: simple, zero additional infrastructure cost, built into Kubernetes.
Cons: old and new versions run simultaneously — requires backward-compatible API and DB schema changes.

### Blue-Green Deployment

Two identical environments run in parallel. All traffic goes to Blue (stable). Green runs the new version. Switch is instant — flip the load balancer. Rollback is equally instant.

```yaml
# blue-deployment.yaml (currently live)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-blue
  labels:
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
      version: blue
  template:
    metadata:
      labels:
        app: order-service
        version: blue
    spec:
      containers:
        - name: order-service
          image: registry.example.com/order-service:1.4.1

---
# service.yaml — point this at blue or green to switch traffic
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
    version: blue # ← change to "green" to switch traffic
  ports:
    - port: 80
      targetPort: 3000
```

```bash
# Switch traffic to green
kubectl patch service order-service \
  -p '{"spec":{"selector":{"version":"green"}}}'

# Roll back instantly if something is wrong
kubectl patch service order-service \
  -p '{"spec":{"selector":{"version":"blue"}}}'
```

Best for: stateless services, high-stakes deployments, teams that prioritise instant rollback over cost.

### Canary Deployment

Route a small percentage of traffic to the new version. Monitor error rates, latency, and business metrics. Gradually increase the percentage if metrics are healthy.

```yaml
# canary-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-canary
spec:
  replicas: 1 # 1 canary pod vs 9 stable pods = ~10% traffic
  selector:
    matchLabels:
      app: order-service
      track: canary
  template:
    metadata:
      labels:
        app: order-service
        track: canary
    spec:
      containers:
        - name: order-service
          image: registry.example.com/order-service:1.5.0
```

For fine-grained traffic splitting (not just pod ratio), use a service mesh or ingress controller:

```yaml
# Istio VirtualService — exact 10% weight regardless of pod count
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
spec:
  http:
    - route:
        - destination:
            host: order-service
            subset: stable
          weight: 90
        - destination:
            host: order-service
            subset: canary
          weight: 10
```

Best for: high-traffic services, features you want to validate with real production data before full rollout.

### Feature Flags

Decouple deployment from release. Deploy code to 100% of instances but control feature activation independently.

```typescript
import { OpenFeature } from "@openfeature/server-sdk";

const client = OpenFeature.getClient("order-service");

async function placeOrder(req: Request): Promise<Response> {
  const user = req.user;

  // New checkout flow gated behind a feature flag
  const useNewCheckout = await client.getBooleanValue(
    "new-checkout-flow",
    false, // default — off
    { targetingKey: user.id }, // can roll out per user segment
  );

  if (useNewCheckout) {
    return newCheckoutHandler(req);
  }
  return legacyCheckoutHandler(req);
}
```

Feature flag providers: LaunchDarkly, Flagsmith, Unleash, OpenFeature (vendor-agnostic SDK).

---

## Health Checks and Readiness

Kubernetes uses two probes to manage pod lifecycle. Implementing them correctly is mandatory — getting them wrong causes failed deployments or dropped traffic.

### Liveness Probe

"Is the process alive?" If the liveness probe fails, Kubernetes kills and restarts the pod.

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 10 # wait before first check — let app start
  periodSeconds: 30
  failureThreshold: 3 # restart after 3 consecutive failures
```

The `/health` endpoint should be trivial — just return 200. It must NOT check downstream dependencies. A database being slow should not cause your pod to restart.

```typescript
app.get("/health", (_req, res) => {
  res.json({
    status: "healthy",
    service: process.env.SERVICE_NAME,
    version: process.env.SERVICE_VERSION,
    uptime: process.uptime(),
  });
});
```

### Readiness Probe

"Is the service ready to accept traffic?" If the readiness probe fails, Kubernetes removes the pod from the Service's endpoints — no traffic is routed to it, but the pod is not restarted.

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3
```

The `/ready` endpoint SHOULD check critical dependencies — database connectivity, cache availability.

```typescript
app.get("/ready", async (_req, res) => {
  const checks: Record<string, string> = {};
  let allHealthy = true;

  try {
    await db.query("SELECT 1");
    checks.database = "ok";
  } catch {
    checks.database = "unavailable";
    allHealthy = false;
  }

  try {
    await redis.ping();
    checks.cache = "ok";
  } catch {
    checks.cache = "unavailable";
    allHealthy = false;
  }

  res
    .status(allHealthy ? 200 : 503)
    .json({ status: allHealthy ? "ready" : "not ready", checks });
});
```

### Startup Probe

For services with slow initialisation (loading ML models, warming caches), a startup probe prevents Kubernetes from killing the pod before it has had time to start.

```yaml
startupProbe:
  httpGet:
    path: /health
    port: 3000
  failureThreshold: 30 # 30 × 10s = 5 minutes maximum startup window
  periodSeconds: 10
```

---

## Configuration Management

Services need configuration that varies between environments (dev, staging, production). Never bake environment-specific values into container images.

### The Twelve-Factor Config Rule

Store all config in environment variables. Never commit secrets to source control.

```typescript
// config.ts — parse and validate at startup, fail fast if required values are missing
import { z } from "zod";

const configSchema = z.object({
  NODE_ENV: z.enum(["development", "staging", "production"]),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  LOG_LEVEL: z.enum(["debug", "info", "warn", "error"]).default("info"),
  SERVICE_VERSION: z.string().default("unknown"),
});

export const config = configSchema.parse(process.env);
// Throws at startup if any required variable is missing or invalid
// Much better than a cryptic runtime error three hours later
```

### Kubernetes ConfigMap and Secret

```yaml
# configmap.yaml — non-sensitive configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
  namespace: production
data:
  NODE_ENV: "production"
  LOG_LEVEL: "info"
  PORT: "3000"

---
# secret.yaml — sensitive configuration (values are base64-encoded)
apiVersion: v1
kind: Secret
metadata:
  name: order-service-secrets
  namespace: production
type: Opaque
stringData: # stringData handles encoding automatically
  DATABASE_URL: "postgresql://user:pass@postgres:5432/orders"
  JWT_SECRET: "your-secret-here"
```

```yaml
# deployment.yaml — inject config into pods
spec:
  template:
    spec:
      containers:
        - name: order-service
          envFrom:
            - configMapRef:
                name: order-service-config
            - secretRef:
                name: order-service-secrets
```

### External Secret Management

For production-grade secret management, use a dedicated secrets store rather than Kubernetes Secrets (which are only base64-encoded, not encrypted at rest by default).

```yaml
# external-secrets-operator — syncs secrets from Vault/AWS SSM/GCP Secret Manager
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: order-service-secrets
spec:
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: order-service-secrets
  data:
    - secretKey: DATABASE_URL
      remoteRef:
        key: production/order-service
        property: database_url
    - secretKey: JWT_SECRET
      remoteRef:
        key: production/order-service
        property: jwt_secret
```

Popular choices: HashiCorp Vault, AWS Secrets Manager, GCP Secret Manager, Azure Key Vault.

---

## Kubernetes Manifests

A minimal but complete set of manifests for a production microservice:

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
  labels:
    app: order-service
    version: "1.4.2"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: order-service
        version: "1.4.2"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: order-service
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
      terminationGracePeriodSeconds: 30 # time for in-flight requests to complete
      containers:
        - name: order-service
          image: registry.example.com/order-service:1.4.2
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: order-service-config
            - secretRef:
                name: order-service-secrets
          resources:
            requests:
              cpu: "100m" # 0.1 CPU guaranteed
              memory: "128Mi"
            limits:
              cpu: "500m" # 0.5 CPU maximum
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"] # drain connections before SIGTERM

---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: production
spec:
  selector:
    app: order-service
  ports:
    - name: http
      port: 80
      targetPort: 3000

---
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70 # scale out when average CPU > 70%
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

### Pod Disruption Budget

Ensures a minimum number of pods remain available during voluntary disruptions (node maintenance, cluster upgrades).

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service-pdb
  namespace: production
spec:
  minAvailable: 2 # always keep at least 2 pods running
  selector:
    matchLabels:
      app: order-service
```

---

## Helm Charts

Helm is the Kubernetes package manager. A Helm chart templatises your manifests so the same chart can deploy to dev, staging, and production with different values.

### Chart Structure

```
order-service/
├── Chart.yaml              # chart metadata
├── values.yaml             # default values
├── values-staging.yaml     # staging overrides
├── values-production.yaml  # production overrides
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── configmap.yaml
    ├── hpa.yaml
    ├── pdb.yaml
    └── _helpers.tpl        # shared template helpers
```

```yaml
# values.yaml — defaults
replicaCount: 2
image:
  repository: registry.example.com/order-service
  tag: latest
  pullPolicy: IfNotPresent
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

```yaml
# values-production.yaml — production overrides
replicaCount: 3
image:
  tag: "" # set at deploy time via --set image.tag=
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
```

```bash
# Deploy to staging
helm upgrade --install order-service ./order-service \
  --namespace staging \
  --values values-staging.yaml \
  --set image.tag=${GIT_SHA}

# Deploy to production
helm upgrade --install order-service ./order-service \
  --namespace production \
  --values values-production.yaml \
  --set image.tag=${GIT_SHA} \
  --atomic \          # roll back automatically if deployment fails
  --timeout 5m
```

---

## Environment Promotion

A well-defined promotion path prevents shipping bugs to production and makes debugging trivial because you know exactly what is running where.

```
dev  →  staging  →  production

dev:        every commit, automatic, ephemeral review environments for PRs
staging:    every merge to main, automatic deploy, mirrors production topology
production: explicit promotion from staging, manual approval gate, canary first
```

### GitOps with ArgoCD / Flux

GitOps externalises cluster state into a Git repository. The cluster continuously reconciles against the desired state in Git.

```yaml
# argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/gitops-repo
    targetRevision: main
    path: apps/order-service/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true # remove resources deleted from git
      selfHeal: true # revert manual kubectl changes
    syncOptions:
      - CreateNamespace=true
```

With GitOps, the deployment process is:

1. Open a PR that updates the image tag in the gitops repo
2. Get review and approval
3. Merge — ArgoCD/Flux automatically syncs the cluster to the new state
4. Full audit trail in Git history — every deployment is a commit

---

## Rollback Procedures

Rollback must be fast, practiced, and non-scary. If rollback is hard, teams avoid deploying — which leads to big-bang releases and more risk.

### Kubernetes Rollback

```bash
# View rollout history
kubectl rollout history deployment/order-service --namespace=production

# Roll back to the previous version (takes ~30 seconds)
kubectl rollout undo deployment/order-service --namespace=production

# Roll back to a specific revision
kubectl rollout undo deployment/order-service --namespace=production --to-revision=3

# Check rollback status
kubectl rollout status deployment/order-service --namespace=production
```

### Helm Rollback

```bash
# View Helm release history
helm history order-service --namespace=production

# Roll back to the previous release
helm rollback order-service --namespace=production

# Roll back to a specific revision
helm rollback order-service 5 --namespace=production

# Rollback with wait
helm rollback order-service --namespace=production --wait --timeout=5m
```

### Database Migration Rollback

This is the hard part. Schema changes are often irreversible. Mitigations:

1. Use the expand/contract pattern (see [03-database-patterns.md](./03-database-patterns.md#data-migration-strategy)) — never add breaking changes in the same deployment that drops the old column
2. Keep `down` migrations alongside every `up` migration and test them in CI
3. For large data migrations, use background jobs with feature flags — don't block the deployment

---

## Summary & Next Steps

Deployment strategy is what makes independent service ownership real. Each service gets its own pipeline, its own schedule, and its own rollback. Containers make builds reproducible. Kubernetes provides the self-healing, scaling, and routing primitives. GitOps provides the audit trail and consistency. The key discipline: every change goes through the pipeline, every production deployment starts as a canary, and rollback is always one command away.

### Recommended Reading Order

| Step | Document                                                           | What you'll learn                                       |
| ---- | ------------------------------------------------------------------ | ------------------------------------------------------- |
| Next | [06-security.md](./06-security.md)                                 | mTLS, JWT, secrets management in production             |
| Also | [07-observability.md](./07-observability.md)                       | How to know when your canary is failing                 |
| Also | [08-resilience.md](./08-resilience.md)                             | Circuit breakers so one service failure doesn't cascade |
| Also | [09-performance-optimization.md](./09-performance-optimization.md) | Tuning resource requests, HPA behaviour, and caching    |

---

_Part of the [Microservices Architecture Guide](../../README.md)_  
_Previous: [04-communication.md](./04-communication.md)_  
_Next: [06-security.md](./06-security.md)_
