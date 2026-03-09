---
name: k8s-oc-explore
description: Analyze a Kubernetes GitOps repository (raw YAML or Kustomize) and produce a migration inventory. Detects workloads, services, configs, secrets, inter-service connections, and namespace patterns.
disable-model-invocation: true
allowed-tools: Read, Glob, Grep
---

# k8s-oc-explore

Analyze a Kubernetes GitOps repository and produce a structured migration inventory for OpenChoreo.

## Usage

```
/k8s-oc-explore [k8s-repo-path]
```

`$ARGUMENTS` = path to K8s repo (defaults to current directory if omitted)

---

## Step 1: Determine Repo Path

Use `$ARGUMENTS` as the target directory. If empty, use `.`.

## Step 2: Detect Repo Type

Search for `kustomization.yaml` files:
- If found at root or in subdirectories → **Kustomize repo**
- If none found → **Raw YAML repo**
- If found only in subdirectories (base/ + overlays/) → **Kustomize with overlays**

Use Glob: `<path>/**/kustomization.yaml`

## Step 3: Discover All K8s Resources

Use Glob to find all YAML files: `<path>/**/*.yaml` and `<path>/**/*.yml`

For each file, Read it and parse the `kind` field. Skip files where:
- `kind` is empty or unrecognized
- File contains Helm-managed resources (`app.kubernetes.io/managed-by: Helm`) — flag these with a warning

Collect resources by kind into these categories:

### Workloads
- `Deployment` → standard deployments
- `StatefulSet` → stateful workloads
- `CronJob` → scheduled tasks
- `Job` → one-time tasks
- `DaemonSet` → **UNSUPPORTED** — add to warnings list

### Networking
- `Service` → collect name, type (ClusterIP/LoadBalancer/NodePort), ports, selector labels
- `Ingress` → collect rules (host, path, backend service name)

### Configuration
- `ConfigMap` → collect name, namespace, keys; note if referenced as envFrom/env or volume mount
- `Secret` → collect name, namespace, keys; note if referenced as envFrom/env or volume mount

### Storage
- `PersistentVolumeClaim` → collect name, namespace, storageClassName, accessModes, size

### Scaling
- `HorizontalPodAutoscaler` → collect target workload name, min/max replicas, metrics

### Ignored (document in report)
- `ServiceAccount`, `ClusterRole`, `ClusterRoleBinding`, `Role`, `RoleBinding` → SKIP (OC platform responsibility)
- `NetworkPolicy` → SKIP (OC platform responsibility)
- Unknown/Custom CRDs → flag with warning

## Step 4: Build Service Registry

Create a map of all Services:
```
serviceRegistry = {
  "<service-name>": {
    namespace: "...",
    type: "ClusterIP|LoadBalancer|NodePort",
    ports: [{port: 80, targetPort: 8080, protocol: "TCP"}],
    selector: {app: "..."}
  }
}
```

Also add short-name patterns for K8s DNS: `<name>`, `<name>.<namespace>`, `<name>.<namespace>.svc.cluster.local`

## Step 5: Detect Namespace Patterns

Collect all namespaces found across resources.

Analyze namespace names for environment suffixes:
- Suffixes to detect: `-dev`, `-development`, `-staging`, `-stage`, `-prod`, `-production`, `-test`, `-qa`, `-uat`
- Example: `payment-dev`, `payment-staging`, `payment-prod` → group into project `payment` with environments `development`, `staging`, `production`

If no environment-suffix namespaces found, treat each namespace as an independent project.

For Kustomize repos: examine overlay directory names for environment hints (`dev`, `staging`, `prod`, etc.)

## Step 6: Build Connection Graph

For each Deployment/StatefulSet, inspect the container `env` array:

1. **Direct DNS references**: value matches `<service>.<ns>.svc.cluster.local` or `<service>.<ns>.svc` → extract service name
2. **Short name references**: value exactly matches a known service name from the registry
3. **K8s auto-injected vars**: env var name matches pattern `<SERVICE_NAME>_SERVICE_HOST` or `<SERVICE_NAME>_SERVICE_PORT` → extract service name by converting `_SERVICE_HOST` suffix and underscore-to-hyphen to find matching service

For each detected connection, record:
```json
{
  "fromComponent": "<deployment-name>",
  "toService": "<service-name>",
  "envVarName": "<ENV_VAR_NAME>",
  "envVarValue": "<original-value>",
  "detectionMethod": "dns|shortname|k8s-injected"
}
```

## Step 7: Process Kustomize Structure (if applicable)

For Kustomize repos:
1. Read `base/kustomization.yaml` — collect base resources
2. For each `overlays/<env>/kustomization.yaml` — collect patches
3. Identify patch types:
   - Image tag changes → will become per-environment image in ReleaseBinding
   - Replica count patches → `componentTypeEnvOverrides.replicas`
   - Resource limit patches → `componentTypeEnvOverrides.resources`
   - ConfigMap/Secret patches → per-environment config

Map overlay directory names to OC environments:
- `dev`, `development` → `development`
- `staging`, `stage` → `staging`
- `prod`, `production` → `production`
- `test`, `qa`, `uat` → `testing` (with note)

## Step 8: Propose Project/Namespace Mapping

Based on namespace analysis, propose a mapping:

```
Proposed Project Mapping:
------------------------
K8s Namespace(s)              → OC Project    OC Environments
payment-dev, payment-staging,   payment         development, staging, production
  payment-prod
orders                          orders          (from Kustomize overlays or single env)
auth                            auth            (from Kustomize overlays or single env)
```

Ask the user:
> "I've analyzed the K8s repository and proposed the above Project/namespace mapping.
> Does this look correct? You can adjust project names, merge/split projects, or change environment names.
> Reply 'yes' to accept, or describe any changes."

Wait for user confirmation before proceeding.

## Step 9: Build Inventory JSON

After user confirms the mapping, write `oc-migration-inventory.json` to the current working directory.

Schema defined in `inventory-schema.md`. Key structure:

```json
{
  "sourceRepo": "<path>",
  "repoType": "raw|kustomize|mixed",
  "ocNamespace": "default",
  "projects": [
    {
      "name": "<project-name>",
      "sourceNamespaces": ["<ns1>", "<ns2>"],
      "environments": ["development", "staging", "production"],
      "components": [
        {
          "name": "<component-name>",
          "sourceKind": "Deployment|StatefulSet|CronJob|Job",
          "ocComponentType": "deployment/service|cronjob/scheduled-task|statefulset/service",
          "image": "<image>",
          "containers": [...],
          "services": [...],
          "configMaps": [...],
          "secrets": [...],
          "pvcs": [...],
          "hpa": null,
          "connections": [...],
          "traits": [...],
          "kustomizeOverlays": {...},
          "warnings": [...]
        }
      ]
    }
  ],
  "warnings": [...],
  "skippedResources": [...]
}
```

## Step 10: Output Report

Print a structured markdown report:

```
## K8s Migration Inventory Report

**Source**: <path>
**Repo Type**: raw YAML | Kustomize
**Scanned Files**: N

### Projects Detected
- <project-name>: N components, environments: [dev, staging, prod]

### Components Summary
| Component | Kind | Image | Connections | Traits | Warnings |
|-----------|------|-------|-------------|--------|----------|
| ...       | ...  | ...   | ...         | ...    | ...      |

### Warnings
- [UNSUPPORTED] DaemonSet <name> in namespace <ns> — no OC equivalent, will be skipped
- [HELM] Resource <name> appears Helm-managed — metadata stripped in output
- [MULTI-CONTAINER] Deployment <name> has sidecars: [<names>] — only main container migrated
- [CROSS-NS] Deployment <name> references Service <svc> in another namespace — manual review needed

### Skipped Resources
- ServiceAccount, ClusterRole, RoleBinding — handled by OC platform
- NetworkPolicy — handled by OC platform
- CRD: <name> — custom resource, no OC equivalent

### Output
Inventory written to: oc-migration-inventory.json
```

---

## Edge Cases

See `../k8s-oc-migrate/reference/edge-cases.md` for full details.

Key behaviors:
- **Multi-container pods**: use first non-sidecar container as main, log others as warnings
- **Init containers**: document as limitation, skip in migration
- **Empty/invalid YAML**: skip file with warning, continue processing
- **Cross-namespace Service refs**: flag for manual review, still attempt to create connection entry
- **Helm labels**: strip `helm.sh/*` and `app.kubernetes.io/managed-by: Helm` annotations from output
