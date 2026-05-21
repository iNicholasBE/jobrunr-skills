---
name: jobrunr-deployment-kubernetes
description: Deploy JobRunr workers and the dashboard to Kubernetes — replica
  topology, probes, secrets, and rolling-update safety.
tier: oss
version: 8.6.0+
frameworks: [spring-boot, micronaut, quarkus]
---

# JobRunr on Kubernetes

Use this skill when deploying a JobRunr-backed Java service to Kubernetes.

## Prerequisites

- JobRunr installed in your application
- A managed database (Postgres, MySQL, MongoDB) reachable from the cluster
- Container image of your service published somewhere `kubectl` can pull from

## Topology

Workers are stateless and don't talk to each other — they poll the
shared storage and claim jobs there. Use a `Deployment` with N replicas.

The dashboard is also stateless. Either enable it on all replicas behind
a `Service` (simplest) or pin it to one dedicated `Deployment` with one
replica (cleaner for SSO).

## Working example — worker Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jobrunr-worker
spec:
  replicas: 3
  selector:
    matchLabels: { app: jobrunr-worker }
  template:
    metadata:
      labels: { app: jobrunr-worker }
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: app
          image: registry.example.com/myapp:latest
          env:
            - name: JOBRUNR_BACKGROUND_JOB_SERVER_ENABLED
              value: "true"
            - name: JOBRUNR_DASHBOARD_ENABLED
              value: "false"     # one-dashboard deployment serves the UI
            - name: SPRING_DATASOURCE_URL
              valueFrom:
                secretKeyRef: { name: app-db, key: url }
            - name: SPRING_DATASOURCE_USERNAME
              valueFrom:
                secretKeyRef: { name: app-db, key: username }
            - name: SPRING_DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef: { name: app-db, key: password }
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 10
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
          resources:
            requests: { cpu: "500m", memory: "768Mi" }
            limits:   { cpu: "2000m", memory: "1536Mi" }
```

For Spring Boot, the `jobrunr-spring-boot-3-starter` / `-4-starter`
contributes a JobRunr health indicator to the actuator health endpoint.

## `terminationGracePeriodSeconds`

This is the most important production detail. When a pod is killed during
a rolling update, Kubernetes sends SIGTERM and waits this many seconds
before SIGKILL. JobRunr's worker uses that window to:

1. Stop claiming new jobs.
2. Finish jobs already in `PROCESSING`.
3. Let interrupted jobs (`Thread.interrupt()`) propagate cleanly.

If your longest-running job takes ~45 seconds, set the grace period to
60+ seconds. If you have multi-minute jobs and a short grace period,
Kubernetes will SIGKILL the JVM mid-job and JobRunr will retry it on
another worker — which may or may not be safe depending on your job's
idempotency.

## Dashboard Service (option: one dashboard, one replica)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: jobrunr-dashboard }
spec:
  replicas: 1
  selector: { matchLabels: { app: jobrunr-dashboard } }
  template:
    metadata: { labels: { app: jobrunr-dashboard } }
    spec:
      containers:
        - name: app
          image: registry.example.com/myapp:latest
          env:
            - name: JOBRUNR_BACKGROUND_JOB_SERVER_ENABLED
              value: "false"
            - name: JOBRUNR_DASHBOARD_ENABLED
              value: "true"
            - name: JOBRUNR_DASHBOARD_USERNAME
              valueFrom: { secretKeyRef: { name: dashboard-auth, key: username } }
            - name: JOBRUNR_DASHBOARD_PASSWORD
              valueFrom: { secretKeyRef: { name: dashboard-auth, key: password } }
          ports:
            - containerPort: 8000
              name: dashboard
```

Then a `Service` of type `ClusterIP` and an `Ingress` (or port-forward
when you only need it for ops).

## Common mistakes

- **Using a `StatefulSet`.** Workers are stateless. Don't pay the StatefulSet
  tax (ordered rollout, stable storage) for no benefit.
- **Short termination grace period.** Default is 30s. Jobs longer than that
  get SIGKILLed and retried — often a surprise in production. Tune to your
  P99 job duration.
- **Dashboard exposed via `LoadBalancer` without auth.** Always
  basic-auth or front it behind your existing IDP.
- **Forgetting JVM heap tuning.** JobRunr workers + Spring Boot + a
  database driver baseline ~512MB. Set memory requests/limits to match;
  starve it and the OOM killer ends jobs mid-flight.

## Sources

- <https://www.jobrunr.io/en/documentation/background-methods/dashboard/>
- <https://www.jobrunr.io/en/documentation/background-methods/deleting-jobs/>
  (covers the thread-interrupt model that grace-period termination relies on)
