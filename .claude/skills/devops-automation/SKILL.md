---
name: devops-automation
description: Automating delivery — GitHub Actions pipelines, Docker builds, Kubernetes manifests, Helm, and ArgoCD GitOps
---

# DevOps automation

## Structuring a GitHub Actions workflow

```yaml
name: CI/CD
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'npm'
      - run: npm ci
      - run: npm run lint

  test:
    runs-on: ubuntu-latest
    needs: lint
    strategy:
      matrix:
        node-version: [20, 22]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      - run: npm ci
      - run: npm test -- --coverage
      - uses: actions/upload-artifact@v4
        with:
          name: coverage-${{ matrix.node-version }}
          path: coverage/

  deploy:
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - uses: actions/checkout@v4
      - run: ./deploy.sh
```

Patterns worth carrying over:
- Cancel superseded runs with a `concurrency` group
- Cache dependencies through the setup action's `cache` option
- Chain job order with `needs`
- Gate deploys behind `environment` protection rules
- Cover multiple versions with a matrix

## Multi-stage Docker builds

```dockerfile
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --production

FROM node:22-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22-alpine AS runner
WORKDIR /app
RUN addgroup -g 1001 appgroup && adduser -u 1001 -G appgroup -S appuser
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package.json ./
USER appuser
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "dist/server.js"]
```

Guidelines:
- Pin a specific image tag — never `latest`
- Run the container as a non-root user
- Copy only what the final stage needs
- Add a `HEALTHCHECK` so the orchestrator can see liveness
- Use `.dockerignore` to keep `node_modules`, `.git`, and tests out

## A Kubernetes deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  labels:
    app: api-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
        - name: api
          image: registry.example.com/api:v1.2.3
          ports:
            - containerPort: 3000
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 20
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: api-secrets
                  key: database-url
```

Always declare resource requests and limits, always define readiness and liveness probes, and set `maxUnavailable: 0` for zero-downtime rollouts.

## Laying out a Helm chart

```
chart/
  Chart.yaml
  values.yaml
  values-staging.yaml
  values-production.yaml
  templates/
    deployment.yaml
    service.yaml
    ingress.yaml
    hpa.yaml
    _helpers.tpl
```

```yaml
# values.yaml
replicaCount: 2
image:
  repository: registry.example.com/api
  tag: latest
  pullPolicy: IfNotPresent
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
ingress:
  enabled: true
  host: api.example.com
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilization: 70
```

Override per environment with `values-{env}.yaml`, lint with `helm lint`, and render with `helm template` to eyeball the output before you deploy.

## GitOps with ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api-server
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/k8s-manifests
    targetRevision: main
    path: apps/api-server
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

The GitOps contract:
- Git is the one source of truth for cluster state
- Every change lands through a PR — no ad-hoc `kubectl apply` in production
- ArgoCD continuously syncs Git into the cluster
- Turn on `selfHeal` so manual drift gets reverted automatically
- Keep application code and deployment manifests in separate repos

## The monitoring stack

```yaml
# Prometheus ServiceMonitor
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: api-server
spec:
  selector:
    matchLabels:
      app: api-server
  endpoints:
    - port: metrics
      interval: 15s
      path: /metrics
```

Metrics worth exposing:
- `http_request_duration_seconds` (histogram) — request latency by route and status
- `http_requests_total` (counter) — request volume by route and status
- `process_resident_memory_bytes` (gauge) — memory in use
- `db_query_duration_seconds` (histogram) — database query latency

Alert when: error rate tops 1%, P99 latency tops 2s, memory passes 80% of the limit, or a pod restarts more than three times in ten minutes.

## Pipeline practices that pay off

1. Keep CI under ten minutes — parallelize jobs and cache aggressively
2. Lint and type-check before you run tests
3. Spin up ephemeral environments for PR previews
4. Pin every action to a SHA, not a tag
5. Keep secrets in GitHub Secrets, never in the workflow file
6. Authenticate to cloud providers with OIDC instead of long-lived keys
7. Tag images with the git SHA, never `latest`
8. Scan container images in CI (Trivy, Snyk)
