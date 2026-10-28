# 3. Kubernetes Fundamentals for Application Developers

> Phase 4 · Week 3 · [← Phase index](README.md)

**Why this matters:** You don't need to *administer* Kubernetes, but you'll deploy to it, read its manifests and debug pods that won't start. About 26% of developers use Kubernetes, and "my pod is in CrashLoopBackOff" is a routine Middle-level task.

---

## 3.1 The objects you'll use

| Object | What it is | You care because… |
|---|---|---|
| **Pod** | one or more containers scheduled together; ephemeral | your app runs here; it gets killed and replaced freely |
| **Deployment** | desired state for identical pods (replicas, image, rolling updates) | you change the image tag here to deploy |
| **Service** | a stable virtual IP and DNS name in front of matching pods | other services call `http://shop-api` |
| **Ingress** / Gateway API | HTTP routing from outside the cluster (host, path, TLS) | public traffic enters here |
| **ConfigMap** | non-secret configuration | environment variables or mounted files |
| **Secret** | sensitive configuration (only base64, **not encrypted** by default) | prefer syncing secrets from a vault (CSI driver, External Secrets) |
| **HorizontalPodAutoscaler** | scales replicas by CPU, memory or custom metrics | handles load spikes |
| **Namespace** | a logical grouping and isolation boundary | `dev`, `staging`, team namespaces |
| **Job / CronJob** | run-to-completion workloads | database migrations, nightly exports |

---

## 3.2 A complete manifest for an ASP.NET Core API

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shop-api
  labels: { app: shop-api }
spec:
  replicas: 3
  selector:
    matchLabels: { app: shop-api }
  strategy:
    rollingUpdate: { maxUnavailable: 0, maxSurge: 1 }     # zero-downtime rollouts
  template:
    metadata:
      labels: { app: shop-api }
    spec:
      terminationGracePeriodSeconds: 40                   # > the app's ShutdownTimeout (30 s)
      securityContext:
        runAsNonRoot: true
      containers:
        - name: api
          image: ghcr.io/acme/shop-api:1.4.2              # better: pin the digest @sha256:…
          ports:
            - containerPort: 8080
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: Production
            - name: ConnectionStrings__Shop
              valueFrom:
                secretKeyRef: { name: shop-secrets, key: db-connection }
          envFrom:
            - configMapRef: { name: shop-api-config }
          resources:
            requests: { cpu: 250m, memory: 256Mi }        # what the scheduler reserves
            limits:   { memory: 512Mi }                   # the GC sizes its heap from this
          startupProbe:
            httpGet: { path: /health/live, port: 8080 }
            failureThreshold: 30
            periodSeconds: 2
          livenessProbe:
            httpGet: { path: /health/live, port: 8080 }
            periodSeconds: 10
          readinessProbe:
            httpGet: { path: /health/ready, port: 8080 }
            periodSeconds: 5
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: ["ALL"] }
---
apiVersion: v1
kind: Service
metadata:
  name: shop-api
spec:
  selector: { app: shop-api }
  ports:
    - port: 80
      targetPort: 8080
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: shop-api
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: shop-api }
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 70 }
```

### The probes: get these right

| Probe | Question | On failure | Should check |
|---|---|---|---|
| **Startup** | "has the app finished starting?" | keeps waiting, then restarts | the process is up (gives slow starters time) |
| **Liveness** | "is the process stuck forever?" | **restarts the container** | **only the process itself.** Never the database |
| **Readiness** | "can it serve traffic now?" | **removes the pod from the Service** (no restart) | critical dependencies (database, cache) |

> ⚠️ **A classic outage:** a liveness probe that checks the database. The database blips, **every pod is restarted at once**, and the restarts make the outage longer. Dependencies belong in **readiness**, not liveness.

### Resources and .NET

- **Memory limit:** the GC sets its heap hard limit to about 75% of it. If the limit is too low, you get OOMKilled (exit code 137). If there's no limit, the GC assumes the node's memory and pods get evicted under pressure.
- **CPU limits** throttle the container and can cause latency spikes. Many teams set CPU **requests** but no CPU **limits**. Follow your platform team's policy.
- With `readOnlyRootFilesystem: true`, mount an `emptyDir` volume at `/tmp` if your app (or a library) writes temp files.

---

## 3.3 Graceful shutdown during rolling updates

1. Kubernetes marks the pod as terminating and starts removing it from Service endpoints. This isn't instant.
2. It sends **SIGTERM**. .NET's host stops accepting new connections and waits for in-flight requests, up to `HostOptions.ShutdownTimeout`.
3. After `terminationGracePeriodSeconds`, it sends SIGKILL.

Because step 1 propagates asynchronously, some requests can still arrive just after SIGTERM. A common mitigation is a short delay before shutdown starts (a `preStop` sleep, which needs a shell, or a delay in an `IHostedLifecycleService`).

---

## 3.4 Configuration and secrets

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: shop-api-config
data:
  Payments__BaseUrl: "https://payments.internal"
  Logging__LogLevel__Default: "Information"
```

- Kubernetes Secrets are **base64-encoded, not encrypted**, unless the cluster enables encryption at rest. Restrict who can read them with RBAC.
- Better: keep secrets in **Azure Key Vault or AWS Secrets Manager**, and sync them with the Secrets Store CSI driver or External Secrets Operator. Or have the app read the vault directly using **workload identity** (no stored credentials at all).

---

## 3.5 Database migrations

**Don't** run `Database.Migrate()` at application startup with several replicas: they race, and a failed migration crash-loops the whole app.

**Do** run migrations as a separate step:

- a Kubernetes **Job** (or a Helm pre-install/pre-upgrade hook) that runs an **EF migration bundle** before the rollout, or
- a pipeline step before deployment ([guide 4](04-ci-cd.md#46-database-migrations-in-pipelines)).

Keep migrations **backward-compatible** (expand → migrate → contract), because during a rolling update, old and new pods run at the same time.

---

## 3.6 Helm in five minutes

Helm packages manifests as **templates plus values**:

```
charts/shop-api/
├─ Chart.yaml
├─ values.yaml               # defaults: image, replicas, resources
├─ values-prod.yaml          # environment overrides
└─ templates/deployment.yaml # uses {{ .Values.image.tag }} etc.
```

```powershell
helm upgrade --install shop-api ./charts/shop-api -f ./charts/shop-api/values-prod.yaml --set image.tag=1.4.2 -n shop
helm rollback shop-api 3 -n shop
```

---

## 3.7 Debugging toolkit

```powershell
kubectl get pods -n shop -o wide                     # status, restarts, node
kubectl describe pod shop-api-7d9c… -n shop          # events: image pull errors, probe failures, OOMKilled
kubectl logs shop-api-7d9c… -n shop --previous       # logs of the crashed container
kubectl rollout status deployment/shop-api -n shop
kubectl rollout undo deployment/shop-api -n shop     # roll back
kubectl port-forward svc/shop-api 8080:80 -n shop    # call the service locally
kubectl top pods -n shop                             # CPU and memory (needs metrics-server)
kubectl debug -it shop-api-7d9c… --image=busybox --target=api -n shop   # a shell beside a chiseled container
```

| Status | Typical cause |
|---|---|
| `ImagePullBackOff` | wrong image name or tag, or missing registry credentials |
| `CrashLoopBackOff` | the app exits at startup: missing configuration, a failing migration, a bad connection string. Check `logs --previous` |
| `OOMKilled` (exit code 137) | memory limit too low, or a memory leak |
| Running but never `Ready` | readiness probe failing (a dependency is down, or the wrong path or port) |
| `Pending` | not enough cluster resources for the requests, or node selectors and taints |

---

## Interview questions

<details><summary><b>1. Liveness vs readiness probe?</b></summary>

A failing liveness probe restarts the container, so it should only detect a stuck process. A failing readiness probe removes the pod from load balancing without restarting it, so that's where dependency checks belong. Checking the database in liveness can turn a database blip into restarts across the whole fleet.
</details>

<details><summary><b>2. Your pod is in CrashLoopBackOff. What do you do?</b></summary>

Run `kubectl describe pod` (events, exit code) and `kubectl logs --previous` (the crash output). Look for missing configuration or secrets, bad connection strings, failing startup migrations or OOM kills (exit code 137). Fix it and redeploy, or roll back.
</details>

<details><summary><b>3. How should database migrations run with several replicas?</b></summary>

As a separate, single execution before the rollout: a pipeline step or a Kubernetes Job running an EF migration bundle. Keep schema changes backward-compatible because old and new versions run side by side during a rolling update.
</details>

<details><summary><b>4. How does .NET behave with container memory limits?</b></summary>

The runtime detects cgroup limits, and the GC sets its heap hard limit to about 75% of the memory limit. Too low a limit causes OOM kills. No limit at all makes the GC assume the node's memory.
</details>

---

## Exercises

1. Deploy a lab API to a local cluster (Docker Desktop, kind or k3d) with the manifest above ([Lab 3](10-labs.md#lab-3-deploy-to-a-local-cluster)).
2. Break it on purpose: a wrong image tag, a missing secret, a liveness probe pointing to a database check, and a 64 Mi memory limit. Diagnose each with `describe` and `logs`.
3. Do a rolling update while load testing, and verify there are no failed requests. Then remove the readiness probe and repeat.
4. Package the manifests as a Helm chart with separate values for dev and prod.

## Self-check

- [ ] I can write a Deployment, Service, ConfigMap and HPA for an ASP.NET Core API.
- [ ] I configure startup, liveness and readiness probes correctly.
- [ ] I set resource requests and limits and understand their effect on .NET.
- [ ] I can debug ImagePullBackOff, CrashLoopBackOff, OOMKilled and not-ready pods.
- [ ] I run migrations outside the app's startup.

## Further reading

- kubernetes.io: *Concepts* (workloads, services, configuration), *Configure liveness, readiness and startup probes*.
- Microsoft Learn: *Deploy .NET microservices to Kubernetes* and *.NET container memory considerations*.
- *Kubernetes Up & Running* (Burns, Beda, Hightower, Evenson).
