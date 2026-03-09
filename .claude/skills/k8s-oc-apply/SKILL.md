---
name: k8s-oc-apply
description: Apply an OpenChoreo GitOps directory to a running OpenChoreo instance using available MCP tools. Discovers OC MCP tools at runtime and falls back to kubectl if MCP tools are unavailable.
argument-hint: "[openchoreo-gitops-dir]"
disable-model-invocation: true
---

# k8s-oc-apply

Apply a generated OpenChoreo GitOps directory to a live OpenChoreo instance.

## Usage

```
/k8s-oc-apply [openchoreo-gitops-dir]
```

`$ARGUMENTS` = path to generated GitOps directory (default: `./openchoreo-gitops/`)

---

## Step 1: Validate Input Directory

Check that the directory exists and contains expected structure:
- `namespaces/` directory present
- At least one `namespace.yaml` file

If directory not found or invalid structure, exit with:
> "Directory <path> not found or does not contain valid OpenChoreo GitOps structure.
> Run /k8s-oc-generate first to produce the GitOps files."

## Step 2: Discover Available Tools

**List all available tools in the current session.**

Look for tools with prefix `mcp__openchoreo__` or similar OpenChoreo-specific MCP tools.

Common OC MCP tool name patterns to look for:
- `mcp__openchoreo__create_namespace`
- `mcp__openchoreo__create_project`
- `mcp__openchoreo__create_component`
- `mcp__openchoreo__create_workload`
- `mcp__openchoreo__create_release_binding`
- `mcp__openchoreo__apply_resource`
- `mcp__openchoreo__*` (any tool with openchoreo prefix)

**Decision:**
- If OC MCP tools found → **Use MCP path** (Step 3a)
- If no OC MCP tools → **Use kubectl fallback** (Step 3b)

Report to user:
```
Found OpenChoreo MCP tools: [list tool names]
Applying via MCP...
```
OR:
```
No OpenChoreo MCP tools found in this session.
Falling back to kubectl apply.
```

---

## Step 3a: Apply via MCP Tools

Read the apply order from `apply-order.md`. Apply resources in strict dependency order:

### Apply Order
1. Namespaces first
2. Projects (depend on Namespace)
3. Components (depend on Project)
4. Workloads (depend on Component)
5. ReleaseBindings (depend on Workload + Environment)

### Discovery Pattern

For each resource file (in order), read the YAML and:
1. Identify `kind` and `metadata.name`
2. Call the appropriate MCP tool based on kind:
   - `kind: Namespace` → call namespace creation tool
   - `kind: Project` → call project creation tool
   - `kind: Component` → call component creation tool
   - `kind: Workload` → call workload creation tool
   - `kind: ReleaseBinding` → call release binding creation tool
3. Pass the resource YAML content to the tool
4. Report success or failure

**If a resource already exists:** check if the MCP tool supports upsert/update. If not, skip with a note: `"Resource <name> already exists — skipped (use update if changes needed)."`

### Error Handling
- If a resource fails to apply, log the error and continue with the next resource
- After all resources, report total success/failure counts
- If a critical resource fails (e.g., Project), warn that dependent resources (Components, Workloads) may also fail

---

## Step 3b: Apply via kubectl (Fallback)

Confirm with user before running kubectl commands:

> "I'll apply the OpenChoreo GitOps files using kubectl. This will run:
> ```
> kubectl apply -f <gitops-dir>/namespaces/ --recursive
> ```
> Proceed? (yes/no)"

If user confirms, execute kubectl in dependency order:

```bash
# Apply all resources recursively (kubectl handles ordering within a directory)
kubectl apply -f <gitops-dir>/namespaces/ --recursive
```

If kubectl is not available, provide manual instructions:
```
kubectl is not available. To apply manually:

1. Apply namespace:
   kubectl apply -f <gitops-dir>/namespaces/<ns>/namespace.yaml

2. Apply projects:
   kubectl apply -f <gitops-dir>/namespaces/<ns>/projects/<project>/project.yaml

3. Apply components:
   kubectl apply -f <gitops-dir>/namespaces/<ns>/projects/<project>/components/<comp>/component.yaml

4. Apply workloads:
   kubectl apply -f <gitops-dir>/namespaces/<ns>/projects/<project>/components/<comp>/workload.yaml

5. Apply release bindings:
   kubectl apply -f <gitops-dir>/namespaces/<ns>/projects/<project>/components/<comp>/release-bindings/ --recursive

Or apply all at once:
   kubectl apply -f <gitops-dir>/ --recursive
```

---

## Step 4: Verify Applied Resources

After applying, attempt to verify resources were created:

**Via MCP:** Call available list/get tools to confirm resources exist.

**Via kubectl:**
```bash
kubectl get namespaces,projects,components,workloads,releasebindings -n <oc-namespace>
```

Report status of each applied resource.

---

## Step 5: Print Apply Summary

```
## OpenChoreo Apply Summary

Applied from: <gitops-dir>
Method: MCP | kubectl

### Results
✓ Namespaces applied: N
✓ Projects applied: N
✓ Components applied: N
✓ Workloads applied: N
✓ ReleaseBindings applied: N
✗ Failed: N (see errors below)

### Errors (if any)
- ReleaseBinding api-server-production: <error message>

### Next Steps
- Monitor deployment status in the OC dashboard
- Verify component health: check that all ReleaseBindings show "Ready"
- Review any failed resources and apply manually if needed
```

---

## Reference

See `apply-order.md` for detailed resource dependency graph and ordering rules.
