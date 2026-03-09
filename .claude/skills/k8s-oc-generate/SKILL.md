---
name: k8s-oc-generate
description: Generate OpenChoreo GitOps YAML files from a K8s migration inventory. Produces the namespaces/ directory structure matching openchoreo/sample-gitops conventions.
disable-model-invocation: true
allowed-tools: Read, Write, Glob
---

# k8s-oc-generate

Generate OpenChoreo GitOps YAML files from a migration inventory produced by `k8s-oc-explore`.

## Usage

```
/k8s-oc-generate [inventory-file] [output-dir]
```

- `$ARGUMENTS[0]` = inventory JSON file (default: `./oc-migration-inventory.json`)
- `$ARGUMENTS[1]` = output directory (default: `./openchoreo-gitops/`)

---

## Step 1: Read Inventory

Read the inventory JSON file. Parse all projects, components, environments, and connections.

If file not found, exit with error:
> "Inventory file not found at <path>. Run /k8s-oc-explore first to generate it."

## Step 2: Validate Inventory

Check for required fields:
- Each project has a `name` and at least one `component`
- Each component has `name`, `ocComponentType`, `image`
- Environments list is non-empty

Warn (but don't fail) for:
- Components with empty `connections`
- Components with warnings in their `warnings` array

## Step 3: Generate Directory Structure

For each project in the inventory, generate the following files under `<output-dir>/namespaces/<ocNamespace>/`:

```
namespaces/
└── <ocNamespace>/
    ├── namespace.yaml
    └── projects/
        └── <project>/
            ├── project.yaml
            └── components/
                └── <component>/
                    ├── component.yaml
                    ├── workload.yaml
                    └── release-bindings/
                        ├── <component>-development.yaml
                        ├── <component>-staging.yaml
                        └── <component>-production.yaml
```

## Step 4: Generate namespace.yaml

```yaml
# namespaces/<ocNamespace>/namespace.yaml
apiVersion: core.choreo.dev/v1alpha1
kind: Namespace
metadata:
  name: <ocNamespace>
spec: {}
```

## Step 5: Generate project.yaml

```yaml
# namespaces/<ocNamespace>/projects/<project>/project.yaml
apiVersion: core.choreo.dev/v1alpha1
kind: Project
metadata:
  name: <project-name>
  namespace: <ocNamespace>
spec:
  description: "Migrated from K8s namespace(s): <sourceNamespaces joined by ', '>"
```

## Step 6: Generate component.yaml

Select `spec.type` from `component.ocComponentType`.

For standard component:
```yaml
apiVersion: core.choreo.dev/v1alpha1
kind: Component
metadata:
  name: <component-name>
  namespace: <ocNamespace>
  labels:
    core.choreo.dev/project-name: <project-name>
spec:
  type: <ocComponentType>
  # e.g. "deployment/service", "cronjob/scheduled-task", "statefulset/service"

  description: "Migrated from K8s <sourceKind> '<sourceResourceName>'"
```

For components with traits (PVC, HPA), add to spec:
```yaml
spec:
  type: statefulset/service
  traits:
    - type: persistent-volume
      properties:
        size: <pvcs[0].size>
        mountPath: <pvcs[0].mountPath>
        storageClass: <pvcs[0].storageClassName>
    - type: autoscaler
      properties:
        minReplicas: <hpa.minReplicas>
        maxReplicas: <hpa.maxReplicas>
        targetCPUUtilizationPercentage: <hpa.targetCPUUtilizationPercentage>
```

## Step 7: Generate workload.yaml

This is the core workload definition. Reference `yaml-templates.md` for complete field reference.

### Base workload structure:

```yaml
apiVersion: core.choreo.dev/v1alpha1
kind: Workload
metadata:
  name: <component-name>
  namespace: <ocNamespace>
  labels:
    core.choreo.dev/project-name: <project-name>
    core.choreo.dev/component-name: <component-name>
spec:
  owner:
    projectName: <project-name>
    componentName: <component-name>

  container:
    image: <image>
    # Include command/args only if non-null
    # command: [...]
    # args: [...]

    resources:
      requests:
        cpu: <containers[0].resources.requests.cpu or "100m">
        memory: <containers[0].resources.requests.memory or "128Mi">
      limits:
        cpu: <containers[0].resources.limits.cpu or "500m">
        memory: <containers[0].resources.limits.memory or "512Mi">

    env:
      # See env generation rules below

    # Include files only if volume mounts from ConfigMap/Secret exist
    # files:
    #   - path: /etc/config/file.conf
    #     value: |
    #       ...plain content if available...
    #     valueFrom:
    #       secretRef:
    #         name: <secret-name>
    #         key: <key>

  endpoints:
    # See endpoint generation rules below

  connections:
    # See connection generation rules below
```

### For CronJob components, add:
```yaml
spec:
  schedule: "<cronSchedule>"
```

### Env generation rules:

For each env var in `containers[0].env`:

1. **Plain value** (`valueType: plain`):
```yaml
- name: <name>
  value: "<value>"
```

2. **Secret reference** (`valueType: secretRef`):
```yaml
- name: <name>
  valueFrom:
    secretRef:
      name: <secretName>
      key: <secretKey>
```

3. **ConfigMap reference** (`valueType: configMapRef`):
```yaml
- name: <name>
  value: "<value>"
  # Note: OC does not have a direct configMapRef equivalent
  # Inline the value if known, or add a comment
```

**IMPORTANT**: Env vars that are K8s service connection vars (matching a `connections` entry's `envVarName`) should be preserved as-is in `env` AND also appear in `connections[].envBindings`. Do not remove them from env — OC will override them via connections at runtime.

### Endpoint generation rules:

For each service in `component.services`:
```yaml
endpoints:
  - name: <service-port-name or "http">
    port: <service port targetPort>
    # For ClusterIP (visibility: project):
    visibility:
      - project
    # For LoadBalancer/NodePort (visibility: external):
    visibility:
      - external
    # For Ingress rules:
    basePath: <ingress.path>
```

If both a ClusterIP service AND an Ingress rule exist for the same port, generate one endpoint with `visibility: [external]` and set `basePath` from the Ingress rule.

If no Service exists for the component, omit `endpoints`.

### Connection generation rules:

For each entry in `component.connections`:
```yaml
connections:
  - component: <toComponent>
    endpoint: <ocEndpoint>
    visibility: <ocVisibility>
    envBindings:
      address: <envVarName>
```

## Step 8: Generate release-bindings/<component>-<env>.yaml

One ReleaseBinding file per environment per component.

```yaml
apiVersion: core.choreo.dev/v1alpha1
kind: ReleaseBinding
metadata:
  name: <component-name>-<environment>
  namespace: <ocNamespace>
  labels:
    core.choreo.dev/project-name: <project-name>
    core.choreo.dev/component-name: <component-name>
    core.choreo.dev/environment-name: <environment>
spec:
  owner:
    projectName: <project-name>
    componentName: <component-name>
    environmentName: <environment>

  workloadRefName: <component-name>

  componentTypeEnvOverrides:
    # Include only fields that differ from base workload
    # For Kustomize repos, use overlay-specific values
    # For env-namespace repos, use namespace-specific values

    replicas: <replicas for this env, or omit if same as default>

    container:
      image: <env-specific image tag, or omit if same>
      resources:
        requests:
          cpu: <env-specific or omit>
          memory: <env-specific or omit>
        limits:
          cpu: <env-specific or omit>
          memory: <env-specific or omit>
      env:
        # Only env vars that differ per environment
        - name: APP_ENV
          value: "<environment>"
```

**Rules for ReleaseBinding generation:**

1. If `kustomizeOverlays` present in component:
   - Use `kustomizeOverlays.<env>.image` for `container.image`
   - Use `kustomizeOverlays.<env>.replicas` for `replicas`
   - Use `kustomizeOverlays.<env>.resources` for `container.resources`
   - Use `kustomizeOverlays.<env>.additionalEnv` for extra env vars

2. If namespace-per-environment pattern (multiple `sourceNamespaces`):
   - Find the namespace matching this environment
   - Use that namespace's workload spec values

3. If single namespace/no overlays:
   - Generate minimal ReleaseBinding with just owner + workloadRefName
   - Add a comment: `# No environment-specific overrides detected`

## Step 9: Generate kustomization.yaml Files (Flux/GitOps compatibility)

At each directory level, generate a `kustomization.yaml`:

**Root kustomization** (`<output-dir>/kustomization.yaml`):
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespaces/<ocNamespace>/namespace.yaml
  - namespaces/<ocNamespace>/projects/<project>/project.yaml
  # ... all component and release-binding files
```

**Or use recursive include pattern:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespaces/
```

## Step 10: Print Migration Summary

```
## OpenChoreo GitOps Generation Summary

Output directory: <output-dir>

### Generated Resources
- Namespaces: 1 (default)
- Projects: N
- Components: N
- Workloads: N
- ReleaseBindings: N (across M environments)
- kustomization.yaml files: N

### File Tree
<output-dir>/
└── namespaces/
    └── default/
        ├── namespace.yaml
        └── projects/
            └── payment/
                ├── project.yaml
                └── components/
                    ├── api-server/
                    │   ├── component.yaml
                    │   ├── workload.yaml
                    │   └── release-bindings/
                    │       ├── api-server-development.yaml
                    │       ├── api-server-staging.yaml
                    │       └── api-server-production.yaml
                    └── ...

### Warnings Carried Over
- [from component warnings]

### Next Steps
1. Review generated files in <output-dir>
2. Adjust any manually-flagged items (cross-namespace refs, init containers, etc.)
3. Run /k8s-oc-apply <output-dir> to apply to an OC cluster, or use:
   kubectl apply -f <output-dir> --recursive
```

---

## YAML Template Reference

See `yaml-templates.md` for complete field reference and examples for all resource types.
