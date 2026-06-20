---
name: kubernetes-operations
description: Running workloads on Kubernetes — deployments, Helm charts, autoscaling, troubleshooting, and resource management
---

# Operating Kubernetes

## A deployment manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  labels:
    app: api-server
    version: v1
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
        version: v1
    spec:
      containers:
        - name: api
          image: registry.example.com/api:1.2.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 15
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: url
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: api-server
```

Set resource requests and limits on every container, and use topology spread constraints to keep replicas from landing on the same node.

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
  tag: "1.2.0"
  pullPolicy: IfNotPresent
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilization: 70
```

## HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
```

## Troubleshooting from the CLI

```bash
# Pod diagnostics
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -c <container> --previous
kubectl exec -it <pod-name> -- /bin/sh

# Resource usage
kubectl top pods -n <namespace> --sort-by=memory
kubectl top nodes

# Network debugging
kubectl run debug --image=nicolaka/netshoot --rm -it -- bash
nslookup <service-name>.<namespace>.svc.cluster.local

# Events sorted by time
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Find pods not running
kubectl get pods -A --field-selector=status.phase!=Running
```

## What to avoid

- Containers running as root with no `securityContext.runAsNonRoot: true`
- Missing requests/limits, which breaks scheduling and invites noisy neighbors
- The `latest` tag instead of a pinned image version
- No `PodDisruptionBudget` on critical workloads
- Secrets stuffed into ConfigMaps rather than Secrets or an external manager
- Skipping pod anti-affinity on replicated deployments

## Before you ship

- [ ] Every container declares resource requests and limits
- [ ] Liveness and readiness probes are configured
- [ ] Images are pinned to a version tag, not `latest`
- [ ] Secrets live in Kubernetes Secrets or an external vault
- [ ] Production workloads have a PodDisruptionBudget
- [ ] NetworkPolicies restrict cross-namespace traffic
- [ ] HA is covered by topology spread constraints or anti-affinity
- [ ] Helm values are split per environment (staging, production)
