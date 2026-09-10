# Cloud Health Check Mapping Guide

This document maps the local `/health` (liveness) and `/ready` (readiness) HTTP endpoints to production container orchestration environments, specifically **Google Cloud Run** and **Kubernetes**.

---

## Endpoint Summary

| Endpoint | Type | Purpose | Behavior |
| --- | --- | --- | --- |
| `GET /health` | **Liveness Probe** (Shallow) | Verifies the node process itself is running and responding to HTTP traffic. | Returns `200 OK` without touching downstream dependencies. |
| `GET /ready` | **Readiness Probe** (Deep) | Verifies the application and its critical dependencies (PostgreSQL DB) are operational. | Returns `200 READY` when DB ping succeeds; returns `503 NOT READY` when DB connection fails. |

---

## 1. Mapping to Google Cloud Run

Google Cloud Run supports container health checks using **Startup Probes** and **Liveness Probes**.

### A. Liveness Probe (`GET /health`)
- **Purpose**: Detects if the container process has crashed, frozen, or entered a deadlocked state.
- **Action on Failure**: Cloud Run terminates and replaces the unhealthy container instance.
- **Target Endpoint**: `GET /health` (shallow check).
- **Configuration (gcloud / YAML)**:
  ```yaml
  livenessProbe:
    httpGet:
      path: /health
      port: 3000
    initialDelaySeconds: 5
    periodSeconds: 10
    timeoutSeconds: 3
    failureThreshold: 3
  ```

### B. Startup Probe (`GET /ready` or `/health`)
- **Purpose**: Gives slow-starting applications time to boot up and verify DB connectivity before receiving traffic.
- **Action on Failure**: Container initialization fails and deployment is aborted.
- **Target Endpoint**: `GET /ready` (or `/health`).
- **Configuration**:
  ```yaml
  startupProbe:
    httpGet:
      path: /health
      port: 3000
    initialDelaySeconds: 2
    periodSeconds: 5
    failureThreshold: 10
  ```

> **Note**: Cloud Run manages traffic routing automatically. If a container instance becomes unhealthy via the liveness probe, it is restarted.

---

## 2. Mapping to Kubernetes (k8s)

Kubernetes natively separates container management into three distinct probe types.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: orders-api
          image: orders-api:latest
          ports:
            - containerPort: 3000

          # Liveness Probe: Process health only
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3

          # Readiness Probe: Dependency health (DB)
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 2

          # Startup Probe: Initial bootstrap
          startupProbe:
            httpGet:
              path: /health
              port: 3000
            periodSeconds: 5
            failureThreshold: 12
```

---

## 3. Key Architectural Takeaways

1. **Never put DB checks in Liveness Probes**:
   If a database experiences temporary downtime, a deep liveness probe will cause Kubernetes/Cloud Run to restart **all API containers** simultaneously (thundering herd problem), creating cascading outages.
2. **Use Readiness Probes for Traffic Control**:
   When the database goes down, readiness probes fail, causing the orchestrator to remove the pod from Service routing (no incoming user traffic is sent to a container that cannot serve requests). Once the database recovers, readiness probes pass and traffic automatically resumes.
