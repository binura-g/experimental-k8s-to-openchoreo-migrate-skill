# K8s → OpenChoreo Resource Mapping Table

Complete reference for the `k8s-oc-generate` skill when selecting OC resource types and fields.

---

## Primary Resource Mapping

| K8s Resource | OC Resource(s) | Notes |
|---|---|---|
| Namespace (app boundary) | Project | May need grouping logic for env-suffix patterns |
| Namespace (env pattern: `app-dev`) | Environment in ReleaseBinding | Detected by suffix analysis |
| Deployment | Component + Workload | Default componentType: `deployment/service` |
| StatefulSet | Component + Workload | componentType: `statefulset/service` |
| CronJob | Component + Workload | componentType: `cronjob/scheduled-task` |
| Job | Component + Workload | componentType: `deployment/service` (one-shot) |
| DaemonSet | **UNSUPPORTED** | Flag with warning, skip |
| Service (ClusterIP) | Workload endpoint | `visibility: [project]` |
| Service (LoadBalancer) | Workload endpoint | `visibility: [external]` |
| Service (NodePort) | Workload endpoint | `visibility: [external]` |
| Ingress rule | Workload endpoint | `visibility: [external]`, set `basePath` |
| ConfigMap (envFrom/env) | `container.env[].value` | Inline values if accessible |
| ConfigMap (volumeMount) | `container.files[].value` | Inline content or reference |
| Secret (env) | `container.env[].valueFrom.secretRef` | Preserve secret name + key |
| Secret (volumeMount) | `container.files[].valueFrom.secretRef` | Preserve secret name + key |
| resources.requests/limits | `componentTypeEnvOverrides.container.resources` | Per-env in ReleaseBinding |
| replicas | `componentTypeEnvOverrides.replicas` | Per-env in ReleaseBinding |
| PVC / PersistentVolumeClaim | `Component.spec.traits[persistent-volume]` | |
| HPA | `Component.spec.traits[autoscaler]` | |
| Inter-service DNS in env vars | `Workload.spec.connections[]` | Key detection logic |
| RBAC / ServiceAccount | **SKIP** | OC platform responsibility |
| NetworkPolicy | **SKIP** | OC platform responsibility |
| ClusterRole/RoleBinding | **SKIP** | OC platform responsibility |
| ImagePullSecrets | **SKIP** | OC handles at platform level |
| PodDisruptionBudget | **SKIP** | OC handles availability |
| Custom CRDs | **SKIP** | Flag with warning |

---

## Namespace → Project Mapping Patterns

### Pattern 1: Environment-Suffix Namespaces
```
payment-dev, payment-staging, payment-prod
→ Project: payment
  Environments: development, staging, production
  (each namespace's workloads become components with per-env ReleaseBindings)
```

### Pattern 2: Domain/Team Namespaces
```
payments, orders, auth
→ Project: payments (1 env: default or from Kustomize overlays)
→ Project: orders
→ Project: auth
```

### Pattern 3: Monorepo (Single Namespace)
```
app (namespace with many workloads)
→ Project: app
  Components: all Deployments/StatefulSets/CronJobs in namespace
  Environments: from Kustomize overlays or single "production"
```

### Pattern 4: Kustomize Overlays (No Namespace Env Distinction)
```
base/ + overlays/dev/ + overlays/staging/ + overlays/prod/
→ Project: <inferred from app name>
  Environments: development, staging, production (from overlay names)
  Components: base workloads with per-overlay ReleaseBindings
```

---

## Environment Name Normalization

| K8s Suffix / Overlay Dir | OC Environment Name |
|---|---|
| `-dev`, `dev`, `development` | `development` |
| `-staging`, `staging`, `stage`, `stg` | `staging` |
| `-prod`, `prod`, `production` | `production` |
| `-test`, `test`, `qa`, `uat` | `testing` |
| `-sandbox`, `sandbox` | `development` |
| `-demo`, `demo` | `staging` |

---

## ComponentType Selection Heuristics

| Condition | Selected ComponentType | Priority |
|---|---|---|
| K8s kind = CronJob | `cronjob/scheduled-task` | 1 (highest) |
| K8s kind = StatefulSet | `statefulset/service` | 1 |
| K8s kind = Job | `deployment/service` | 1 |
| Image contains: `postgres`, `postgresql` | `deployment/database` | 2 |
| Image contains: `mysql`, `mariadb` | `deployment/database` | 2 |
| Image contains: `mongodb`, `mongo` | `deployment/database` | 2 |
| Image contains: `redis` | `deployment/database` | 2 |
| Image contains: `elasticsearch`, `opensearch` | `deployment/database` | 2 |
| Image contains: `kafka`, `zookeeper`, `rabbitmq` | `deployment/service` | 2 (message broker) |
| Default (Deployment) | `deployment/service` | 3 (lowest) |

---

## Service Port → Endpoint Protocol Mapping

| Port Number | Protocol | OC Endpoint Name Suggestion |
|---|---|---|
| 80, 8080, 3000, 8000, 8081 | HTTP | `http` |
| 443, 8443, 9443 | HTTPS | `https` |
| 5432 | PostgreSQL TCP | `tcp` |
| 3306 | MySQL TCP | `tcp` |
| 6379 | Redis TCP | `tcp` |
| 27017 | MongoDB TCP | `tcp` |
| 9200, 9300 | Elasticsearch | `tcp` |
| 5672, 15672 | RabbitMQ | `tcp` |
| 9092 | Kafka | `tcp` |
| 2181 | Zookeeper | `tcp` |
| 50051 | gRPC | `grpc` |
| Other | TCP | `tcp` |

---

## Connection Detection Patterns

### Pattern 1: Full K8s DNS
```
Value: "postgres-svc.payment-dev.svc.cluster.local:5432"
→ Service: postgres-svc, Namespace: payment-dev
→ Connection to component: postgres (lookup by service name in registry)
```

### Pattern 2: Namespace-qualified DNS
```
Value: "postgres-svc.payment-dev"
→ Service: postgres-svc, Namespace: payment-dev
```

### Pattern 3: Short Name (same namespace)
```
Value: "postgres-svc" or "postgres-svc:5432"
→ Service: postgres-svc (in same namespace)
```

### Pattern 4: K8s Injected Env Vars
```
Env var name: "POSTGRES_SVC_SERVICE_HOST"
→ Strip _SERVICE_HOST suffix
→ Convert POSTGRES_SVC → postgres-svc (uppercase to lower, _ to -)
→ Match against service registry
```

### Pattern 5: URL with Service Name
```
Value: "postgresql://postgres-svc:5432/mydb"
Value: "redis://redis-service:6379"
Value: "http://api-svc/endpoint"
→ Extract hostname component, match against service registry
```

---

## Kustomize → ReleaseBinding Field Mapping

| Kustomize Patch | ReleaseBinding Field |
|---|---|
| `images[].newTag` | `componentTypeEnvOverrides.container.image` |
| `replicas[].count` | `componentTypeEnvOverrides.replicas` |
| `patchesStrategicMerge` resources | `componentTypeEnvOverrides.container.resources` |
| `configMapGenerator` additions | `componentTypeEnvOverrides.container.env` |
| `secretGenerator` additions | (preserve as secretRef — warn about secret mgmt) |

---

## Field: Workload `container.files[]`

Used for ConfigMap/Secret volume mounts that serve file content.

```yaml
files:
  # Inline value (when ConfigMap data is available)
  - path: /etc/nginx/nginx.conf
    value: |
      server {
        listen 80;
        ...
      }

  # Secret reference (when value is sensitive)
  - path: /etc/ssl/certs/tls.crt
    valueFrom:
      secretRef:
        name: tls-secret
        key: tls.crt

  # ConfigMap reference when content unavailable
  - path: /etc/config/app.properties
    valueFrom:
      configMapRef:
        name: app-config
        key: app.properties
```

---

## Traits Reference

### persistent-volume trait
```yaml
traits:
  - type: persistent-volume
    properties:
      size: 10Gi
      mountPath: /data
      storageClass: standard
      accessMode: ReadWriteOnce  # optional
```

### autoscaler trait
```yaml
traits:
  - type: autoscaler
    properties:
      minReplicas: 2
      maxReplicas: 10
      targetCPUUtilizationPercentage: 70
      # Optional: targetMemoryUtilizationPercentage: 80
```

---

## What OC Handles Natively (Do Not Migrate)

These K8s concerns are handled transparently by the OpenChoreo platform:
- **Service mesh / mTLS** — OC injects Envoy sidecars automatically
- **NetworkPolicies** — OC enforces project-level isolation
- **RBAC** — OC manages component-level permissions
- **ImagePullSecrets** — OC handles registry credentials at platform level
- **PodDisruptionBudgets** — OC manages availability guarantees
- **Liveness/Readiness probes** — OC applies component-type defaults (document original values in comments)
- **Node affinity/tolerations** — OC handles scheduling

When these are found in the source K8s repo, document them in the migration report but do not generate OC YAML for them.
