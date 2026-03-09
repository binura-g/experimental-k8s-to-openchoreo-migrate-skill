# OpenChoreo YAML Patterns Reference

Canonical YAML patterns for all OpenChoreo resource types with full field documentation.
These patterns are the authoritative reference for the `k8s-oc-generate` skill.

---

## API Version

All OpenChoreo resources use:
```yaml
apiVersion: core.choreo.dev/v1alpha1
```

---

## Namespace

```yaml
apiVersion: core.choreo.dev/v1alpha1
kind: Namespace
metadata:
  name: default          # OC control-plane namespace (typically "default")
spec: {}
```

**Notes:**
- The OC namespace is the top-level organizational unit in the OC control plane
- It is NOT the same as a K8s namespace
- Typically a single namespace named "default" is used for all projects

---

## Project

```yaml
apiVersion: core.choreo.dev/v1alpha1
kind: Project
metadata:
  name: payment                # lowercase, hyphens allowed
  namespace: default           # OC namespace (not K8s namespace)
spec:
  description: "Payment processing service"
  # Optional: display name, tags, etc.
```

**Notes:**
- Maps to a K8s namespace or a group of related namespaces
- Name must be unique within the OC namespace
- All components, workloads, and release bindings reference this project

---

## Component

```yaml
apiVersion: core.choreo.dev/v1alpha1
kind: Component
metadata:
  name: api-server
  namespace: default
  labels:
    core.choreo.dev/project-name: payment     # Required: parent project
spec:
  type: deployment/service    # See ComponentType reference below
  description: "RESTful API for payment processing"

  # Optional: traits for additional platform capabilities
  traits:
    - type: persistent-volume
      properties:
        size: 20Gi
        mountPath: /var/lib/data
        storageClass: standard

    - type: autoscaler
      properties:
        minReplicas: 2
        maxReplicas: 20
        targetCPUUtilizationPercentage: 75
```

### ComponentType Values

| Type | Use Case |
|------|----------|
| `deployment/service` | Standard HTTP/TCP service (Deployment) |
| `deployment/database` | Database workload (PostgreSQL, MySQL, Redis, etc.) |
| `deployment/web-app` | Frontend, static file serving |
| `statefulset/service` | Stateful workload with persistent storage |
| `cronjob/scheduled-task` | Scheduled batch job (CronJob) |

---

## Workload — deployment/service

```yaml
apiVersion: core.choreo.dev/v1alpha1
kind: Workload
metadata:
  name: api-server
  namespace: default
  labels:
    core.choreo.dev/project-name: payment
    core.choreo.dev/component-name: api-server
spec:
  owner:
    projectName: payment
    componentName: api-server

  container:
    image: myregistry.io/payment/api-server:latest

    # Optional: override entrypoint
    # command: ["/bin/sh"]
    # args: ["-c", "exec server"]

    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 512Mi

    env:
      # Plain value
      - name: APP_ENV
        value: "production"
      - name: PORT
        value: "8080"

      # From Secret
      - name: DB_PASSWORD
        valueFrom:
          secretRef:
            name: payment-secrets    # K8s secret name (must exist in OC env)
            key: DB_PASSWORD

      # From Secret (different key)
      - name: JWT_SECRET
        valueFrom:
          secretRef:
            name: payment-secrets
            key: JWT_SECRET

    # File mounts (from ConfigMap or Secret volumes)
    files:
      - path: /etc/app/config.yaml
        value: |
          server:
            port: 8080
            timeout: 30s
          logging:
            level: info

      - path: /etc/ssl/ca.crt
        valueFrom:
          secretRef:
            name: tls-certs
            key: ca.crt

  endpoints:
    # Internal endpoint (project-scoped, for service-to-service)
    - name: http
      port: 8080
      visibility:
        - project

    # External endpoint (internet-accessible)
    - name: public-http
      port: 8080
      visibility:
        - external
      basePath: /api/v1     # Maps to Ingress path

  connections:
    # Connection to another OC component
    - component: postgres-db          # OC component name
      endpoint: tcp                   # Endpoint name on the target component
      visibility: project             # project | external
      envBindings:
        address: DATABASE_URL         # Env var to inject connection address into

    - component: redis-cache
      endpoint: tcp
      visibility: project
      envBindings:
        address: REDIS_URL
```

---

## Workload — cronjob/scheduled-task

```yaml
apiVersion: core.choreo.dev/v1alpha1
kind: Workload
metadata:
  name: cleanup-job
  namespace: default
  labels:
    core.choreo.dev/project-name: payment
    core.choreo.dev/component-name: cleanup-job
spec:
  owner:
    projectName: payment
    componentName: cleanup-job

  schedule: "0 3 * * *"    # Standard cron expression (UTC)

  container:
    image: myregistry.io/payment/cleanup-job:latest
    resources:
      requests:
        cpu: 200m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 512Mi
    env:
      - name: CLEANUP_DAYS
        value: "30"
      - name: DB_URL
        valueFrom:
          secretRef:
            name: job-secrets
            key: DATABASE_URL

  connections:
    - component: postgres-db
      endpoint: tcp
      visibility: project
      envBindings:
        address: DB_URL
```

---

## Workload — statefulset/service

```yaml
apiVersion: core.choreo.dev/v1alpha1
kind: Workload
metadata:
  name: postgres-db
  namespace: default
  labels:
    core.choreo.dev/project-name: payment
    core.choreo.dev/component-name: postgres-db
spec:
  owner:
    projectName: payment
    componentName: postgres-db

  container:
    image: postgres:15-alpine
    resources:
      requests:
        cpu: 250m
        memory: 512Mi
      limits:
        cpu: 2000m
        memory: 2Gi
    env:
      - name: POSTGRES_DB
        value: "payment"
      - name: POSTGRES_USER
        value: "payment_user"
      - name: POSTGRES_PASSWORD
        valueFrom:
          secretRef:
            name: postgres-secret
            key: POSTGRES_PASSWORD
      - name: PGDATA
        value: "/var/lib/postgresql/data/pgdata"

  endpoints:
    - name: tcp
      port: 5432
      visibility:
        - project    # Database only accessible within project
```

---

## ReleaseBinding — development (minimal)

```yaml
apiVersion: core.choreo.dev/v1alpha1
kind: ReleaseBinding
metadata:
  name: api-server-development
  namespace: default
  labels:
    core.choreo.dev/project-name: payment
    core.choreo.dev/component-name: api-server
    core.choreo.dev/environment-name: development
spec:
  owner:
    projectName: payment
    componentName: api-server
    environmentName: development

  workloadRefName: api-server
  # No overrides — development uses workload defaults
```

## ReleaseBinding — staging (with image override)

```yaml
apiVersion: core.choreo.dev/v1alpha1
kind: ReleaseBinding
metadata:
  name: api-server-staging
  namespace: default
  labels:
    core.choreo.dev/project-name: payment
    core.choreo.dev/component-name: api-server
    core.choreo.dev/environment-name: staging
spec:
  owner:
    projectName: payment
    componentName: api-server
    environmentName: staging

  workloadRefName: api-server

  componentTypeEnvOverrides:
    replicas: 2
    container:
      image: myregistry.io/payment/api-server:staging-1.5.0
      env:
        - name: APP_ENV
          value: "staging"
        - name: LOG_LEVEL
          value: "debug"
```

## ReleaseBinding — production (full overrides)

```yaml
apiVersion: core.choreo.dev/v1alpha1
kind: ReleaseBinding
metadata:
  name: api-server-production
  namespace: default
  labels:
    core.choreo.dev/project-name: payment
    core.choreo.dev/component-name: api-server
    core.choreo.dev/environment-name: production
spec:
  owner:
    projectName: payment
    componentName: api-server
    environmentName: production

  workloadRefName: api-server

  componentTypeEnvOverrides:
    replicas: 5

    container:
      image: myregistry.io/payment/api-server:v2.1.0

      resources:
        requests:
          cpu: 500m
          memory: 512Mi
        limits:
          cpu: 2000m
          memory: 2Gi

      env:
        - name: APP_ENV
          value: "production"
        - name: LOG_LEVEL
          value: "warn"
        - name: FEATURE_FLAG_NEWUI
          value: "true"
```

---

## Directory Structure (sample-gitops pattern)

```
namespaces/
└── default/                                    ← OC namespace
    ├── namespace.yaml
    └── projects/
        └── payment/                            ← Project
            ├── project.yaml
            └── components/
                ├── api-server/                 ← Component
                │   ├── component.yaml
                │   ├── workload.yaml
                │   └── release-bindings/
                │       ├── api-server-development.yaml
                │       ├── api-server-staging.yaml
                │       └── api-server-production.yaml
                └── postgres-db/
                    ├── component.yaml
                    ├── workload.yaml
                    └── release-bindings/
                        ├── postgres-db-development.yaml
                        ├── postgres-db-staging.yaml
                        └── postgres-db-production.yaml
```

---

## Label Conventions

All OC resources use these standard labels:

```yaml
labels:
  core.choreo.dev/project-name: <project-name>       # On Component, Workload, ReleaseBinding
  core.choreo.dev/component-name: <component-name>    # On Workload, ReleaseBinding
  core.choreo.dev/environment-name: <env-name>        # On ReleaseBinding only
```

---

## Connections — Full Pattern

```yaml
connections:
  # HTTP service connection
  - component: auth-service
    endpoint: http
    visibility: project
    envBindings:
      address: AUTH_SERVICE_URL    # Env var that will receive the connection address

  # Database connection
  - component: postgres-db
    endpoint: tcp
    visibility: project
    envBindings:
      address: DATABASE_URL

  # External service (if connected to a component with external visibility)
  - component: cdn-service
    endpoint: https
    visibility: external
    envBindings:
      address: CDN_BASE_URL
```

The `envBindings.address` env var will be automatically populated by OC with the correct
endpoint address for the target component in the current environment.
