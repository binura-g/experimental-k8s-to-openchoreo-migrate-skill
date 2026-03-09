# OpenChoreo Resource Apply Order

Resources must be applied in dependency order to avoid reference errors.

---

## Dependency Graph

```
Namespace
└── Project (depends on: Namespace)
    └── Component (depends on: Project)
        ├── Workload (depends on: Component)
        └── ReleaseBinding (depends on: Workload + Environment)
```

---

## Ordered Apply Sequence

### Phase 1: Namespace

Apply `namespace.yaml` first. All other resources reference this namespace.

```
namespaces/<ns>/namespace.yaml
```

**Verify:** `kubectl get namespace <ns>` returns `Active`

---

### Phase 2: Projects

Apply all `project.yaml` files. Projects depend on the Namespace existing.

```
namespaces/<ns>/projects/<project>/project.yaml
```

If multiple projects: apply all project.yaml files before moving to components.

**Verify:** `kubectl get projects -n <ns>`

---

### Phase 3: Components

Apply all `component.yaml` files. Components depend on their Project existing.

```
namespaces/<ns>/projects/<project>/components/<comp>/component.yaml
```

Order: apply all components for a project before workloads/release-bindings.

If components have dependencies on each other (via connections), they must all exist before workloads are applied — connections reference component names, not resource IDs.

**Verify:** `kubectl get components -n <ns>`

---

### Phase 4: Workloads

Apply all `workload.yaml` files. Workloads depend on their Component existing.

```
namespaces/<ns>/projects/<project>/components/<comp>/workload.yaml
```

**Important:** All Components must be applied before any Workloads, because Workloads reference other components via `spec.connections[].component`.

**Verify:** `kubectl get workloads -n <ns>`

---

### Phase 5: ReleaseBindings

Apply all `release-bindings/<comp>-<env>.yaml` files. ReleaseBindings depend on:
- The Component existing
- The Workload (referenced by `workloadRefName`) existing
- The Environment existing (OC manages environments at platform level)

Apply order within ReleaseBindings:
1. `development` environment bindings first
2. `staging` environment bindings second
3. `production` environment bindings last

This allows validation at lower environments before production is activated.

```
namespaces/<ns>/projects/<project>/components/<comp>/release-bindings/<comp>-development.yaml
namespaces/<ns>/projects/<project>/components/<comp>/release-bindings/<comp>-staging.yaml
namespaces/<ns>/projects/<project>/components/<comp>/release-bindings/<comp>-production.yaml
```

**Verify:** `kubectl get releasebindings -n <ns>`

---

## Full Apply Script (kubectl)

```bash
#!/bin/bash
# apply-oc-gitops.sh
# Usage: ./apply-oc-gitops.sh <gitops-dir> <oc-namespace>

GITOPS_DIR="${1:-./openchoreo-gitops}"
OC_NS="${2:-default}"

set -e

echo "=== Phase 1: Namespace ==="
kubectl apply -f "$GITOPS_DIR/namespaces/$OC_NS/namespace.yaml"

echo "=== Phase 2: Projects ==="
find "$GITOPS_DIR/namespaces/$OC_NS/projects" -name "project.yaml" \
  | xargs kubectl apply -f

echo "=== Phase 3: Components ==="
find "$GITOPS_DIR/namespaces/$OC_NS/projects" -name "component.yaml" \
  | xargs kubectl apply -f

echo "=== Phase 4: Workloads ==="
find "$GITOPS_DIR/namespaces/$OC_NS/projects" -name "workload.yaml" \
  | xargs kubectl apply -f

echo "=== Phase 5: ReleaseBindings (development) ==="
find "$GITOPS_DIR/namespaces/$OC_NS/projects" -name "*-development.yaml" \
  | xargs kubectl apply -f

echo "=== Phase 5: ReleaseBindings (staging) ==="
find "$GITOPS_DIR/namespaces/$OC_NS/projects" -name "*-staging.yaml" \
  | xargs kubectl apply -f

echo "=== Phase 5: ReleaseBindings (production) ==="
find "$GITOPS_DIR/namespaces/$OC_NS/projects" -name "*-production.yaml" \
  | xargs kubectl apply -f

echo "=== Apply Complete ==="
kubectl get projects,components,workloads,releasebindings -n "$OC_NS"
```

---

## MCP Tool Apply Order

When using OpenChoreo MCP tools, follow the same phase order:

```
Phase 1: mcp__openchoreo__create_namespace (or equivalent)
Phase 2: mcp__openchoreo__create_project (for each project)
Phase 3: mcp__openchoreo__create_component (for each component)
Phase 4: mcp__openchoreo__create_workload (for each workload)
Phase 5: mcp__openchoreo__create_release_binding (dev → staging → prod)
```

**MCP Tool Discovery:** The exact MCP tool names are discovered at runtime by listing all available tools and filtering for those with the `openchoreo` prefix. The skill must not hardcode tool names — OC MCP server versions may use different tool name conventions.

---

## Rollback Strategy

If apply fails partway through:

1. **ReleaseBinding failures**: Safe — delete the ReleaseBinding and re-apply
   ```bash
   kubectl delete releasebinding <name> -n <ns>
   kubectl apply -f <release-binding-file>
   ```

2. **Workload failures**: Delete Workload and ReleaseBindings, then re-apply
   ```bash
   kubectl delete workload <name> -n <ns>
   kubectl delete releasebinding -l core.choreo.dev/component-name=<name> -n <ns>
   # Fix the workload.yaml, then re-apply phases 4 and 5
   ```

3. **Component failures**: Full cleanup required
   ```bash
   kubectl delete component <name> -n <ns>
   # This should cascade to delete Workload and ReleaseBindings
   ```

4. **Project failures**: Full project cleanup
   ```bash
   kubectl delete project <name> -n <ns>
   # This should cascade to delete all Components, Workloads, ReleaseBindings
   ```

---

## Idempotency

All `kubectl apply` commands are idempotent — re-running will update existing resources.

For MCP tools: check if the tool supports upsert. If not, use a separate update tool or delete-then-recreate strategy.

When applying to an existing OC instance with pre-existing resources:
- Existing Namespace/Project: will be updated (safe)
- Existing Component: will be updated (may trigger re-deployment)
- Existing Workload: will be updated (will trigger rolling update)
- Existing ReleaseBinding: will be updated (may trigger new release)

**Always review diffs before applying to production environments.**
