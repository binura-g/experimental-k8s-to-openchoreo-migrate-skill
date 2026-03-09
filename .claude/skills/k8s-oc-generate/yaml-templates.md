# OpenChoreo YAML Templates Reference

Canonical YAML templates for all OpenChoreo resource types. Used by `k8s-oc-generate` as authoritative reference.
Sourced from the `openchoreo/sample-gitops` repository structure.

---

## Namespace

```yaml
apiVersion: core.choreo.dev/v1alpha1
kind: Namespace
metadata:
  name: default
spec: {}
```

---

## Project

```yaml
apiVersion: core.choreo.dev/v1alpha1
kind: Project
metadata:
  name: my-project
  namespace: default
spec:
  description: "A migrated project"
```

---

## Component — deployment/service

```yaml
apiVersion: core.choreo.dev/v1alpha1
kind: Component
metadata:
  name: api-server
  namespace: default
  labels:
    core.choreo.dev/project-name: my-project
spec:
  type: deployment/service
  description: "REST API service"
```

## Component — deployment/database

```yaml
apiVersion: core.choreo.dev/v1alpha1
kind: Component
metadata:
  name: postgres
  namespace: default
  labels:
    core.choreo.dev/project-name: my-project
spec:
  type: deployment/database
  description: "PostgreSQL database"
```

## Component — cronjob/scheduled-task

```yaml
apiVersion: core.choreo.dev/v1alpha1
kind: Component
metadata:
  name: report-generator
  namespace: default
  labels:
    core.choreo.dev/project-name: my-project
spec:
  type: cronjob/scheduled-task
  description: "Generates daily reports"
```

## Component — statefulset/service

```yaml
apiVersion: core.choreo.dev/v1alpha1
kind: Component
metadata:
  name: redis-cache
  namespace: default
  labels:
    core.choreo.dev/project-name: my-project
spec:
  type: statefulset/service
  description: "Redis cache cluster"
  traits:
    - type: persistent-volume
      properties:
        size: 10Gi
        mountPath: /data
        storageClass: standard
```

## Component — with autoscaler trait

```yaml
apiVersion: core.choreo.dev/v1alpha1
kind: Component
metadata:
  name: api-server
  namespace: default
  labels:
    core.choreo.dev/project-name: my-project
spec:
  type: deployment/service
  traits:
    - type: autoscaler
      properties:
        minReplicas: 2
        maxReplicas: 10
        targetCPUUtilizationPercentage: 70
```

---

## Workload — deployment/service (full example)

```yaml
apiVersion: core.choreo.dev/v1alpha1
kind: Workload
metadata:
  name: api-server
  namespace: default
  labels:
    core.choreo.dev/project-name: my-project
    core.choreo.dev/component-name: api-server
spec:
  owner:
    projectName: my-project
    componentName: api-server

  container:
    image: myregistry/api-server:latest

    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 512Mi

    env:
      - name: APP_ENV
        value: "production"
      - name: LOG_LEVEL
        value: "info"
      - name: DATABASE_PASSWORD
        valueFrom:
          secretRef:
            name: api-secrets
            key: DATABASE_PASSWORD
      - name: JWT_SECRET
        valueFrom:
          secretRef:
            name: api-secrets
            key: JWT_SECRET

    files:
      # ConfigMap as file mount
      - path: /etc/config/app.conf
        value: |
          [server]
          port = 8080
          timeout = 30
      # Secret as file mount
      - path: /etc/certs/tls.crt
        valueFrom:
          secretRef:
            name: tls-secret
            key: tls.crt

  endpoints:
    - name: http
      port: 8080
      visibility:
        - project
    - name: public
      port: 8080
      visibility:
        - external
      basePath: /api/v1

  connections:
    - component: postgres
      endpoint: tcp
      visibility: project
      envBindings:
        address: DATABASE_URL
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
  name: report-generator
  namespace: default
  labels:
    core.choreo.dev/project-name: my-project
    core.choreo.dev/component-name: report-generator
spec:
  owner:
    projectName: my-project
    componentName: report-generator

  schedule: "0 2 * * *"
  # Cron schedule: minute hour day month weekday

  container:
    image: myregistry/report-generator:latest

    resources:
      requests:
        cpu: 200m
        memory: 256Mi
      limits:
        cpu: 1000m
        memory: 1Gi

    env:
      - name: REPORT_TYPE
        value: "daily"
      - name: DB_PASSWORD
        valueFrom:
          secretRef:
            name: report-secrets
            key: DB_PASSWORD

  connections:
    - component: postgres
      endpoint: tcp
      visibility: project
      envBindings:
        address: DATABASE_URL
```

---

## Workload — statefulset with persistent volume

```yaml
apiVersion: core.choreo.dev/v1alpha1
kind: Workload
metadata:
  name: postgres
  namespace: default
  labels:
    core.choreo.dev/project-name: my-project
    core.choreo.dev/component-name: postgres
spec:
  owner:
    projectName: my-project
    componentName: postgres

  container:
    image: postgres:15-alpine

    resources:
      requests:
        cpu: 250m
        memory: 256Mi
      limits:
        cpu: 1000m
        memory: 1Gi

    env:
      - name: POSTGRES_DB
        value: "myapp"
      - name: POSTGRES_USER
        value: "postgres"
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
        - project
```

---

## ReleaseBinding — minimal

```yaml
apiVersion: core.choreo.dev/v1alpha1
kind: ReleaseBinding
metadata:
  name: api-server-development
  namespace: default
  labels:
    core.choreo.dev/project-name: my-project
    core.choreo.dev/component-name: api-server
    core.choreo.dev/environment-name: development
spec:
  owner:
    projectName: my-project
    componentName: api-server
    environmentName: development

  workloadRefName: api-server
  # No overrides — use workload defaults for development
```

## ReleaseBinding — with overrides

```yaml
apiVersion: core.choreo.dev/v1alpha1
kind: ReleaseBinding
metadata:
  name: api-server-production
  namespace: default
  labels:
    core.choreo.dev/project-name: my-project
    core.choreo.dev/component-name: api-server
    core.choreo.dev/environment-name: production
spec:
  owner:
    projectName: my-project
    componentName: api-server
    environmentName: production

  workloadRefName: api-server

  componentTypeEnvOverrides:
    replicas: 5

    container:
      image: myregistry/api-server:v1.2.0

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
```

---

## Endpoint Visibility Reference

| K8s Resource | OC Visibility |
|---|---|
| Service type: ClusterIP | `[project]` |
| Service type: LoadBalancer | `[external]` |
| Service type: NodePort | `[external]` |
| Ingress rule | `[external]` |
| Both ClusterIP + Ingress | `[external]` with basePath |

## ComponentType Selection Reference

| Condition | ComponentType |
|---|---|
| K8s kind: CronJob | `cronjob/scheduled-task` |
| K8s kind: StatefulSet | `statefulset/service` |
| Image: postgres/mysql/mariadb/mongodb/redis/elasticsearch | `deployment/database` |
| Default | `deployment/service` |

## Resource Default Fallbacks

When K8s resource requests/limits are not specified, use these defaults:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

## Naming Conventions

- All OC resource names: lowercase, alphanumeric, hyphens only (`[a-z0-9-]+`)
- Convert underscores to hyphens
- Convert dots to hyphens
- Truncate to 63 characters maximum
- Strip `k8s-`, `kube-` prefixes where clearly metadata
- Environment names: `development`, `staging`, `production`, `testing`
