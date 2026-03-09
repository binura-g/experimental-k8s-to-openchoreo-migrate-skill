---
name: k8s-oc-migrate
description: Migrate a Kubernetes GitOps repository to OpenChoreo. Analyzes K8s resources (raw YAML or Kustomize overlays) and generates a standards-compliant OpenChoreo GitOps structure.
argument-hint: "[k8s-repo-path] [output-dir]"
disable-model-invocation: true
context: fork
agent: general-purpose
---

# k8s-oc-migrate

Orchestrates a complete Kubernetes → OpenChoreo migration in three phases:
1. **Explore** — analyze the K8s repo and build an inventory
2. **Generate** — produce OC GitOps YAML from the inventory
3. **Apply** (optional) — push resources to a live OC instance

## Usage

```
/k8s-oc-migrate [k8s-repo-path] [output-dir]
```

- `$0` = path to K8s GitOps repo (required — use `.` for current directory)
- `$1` = output directory for OC GitOps files (default: `./openchoreo-gitops/`)

---

## Phase 1: Discovery

Invoke the `k8s-oc-explore` skill with the source path:

```
/k8s-oc-explore <k8s-repo-path>
```

The explore skill will:
- Scan all K8s YAML files
- Build a service registry and connection graph
- Detect namespace/environment patterns
- Propose a Project/namespace mapping
- Ask for user confirmation of the mapping
- Write `oc-migration-inventory.json`

**Wait for the user to confirm the mapping before proceeding.**

If the user requests changes to the mapping:
- Describe the changes to make
- Re-invoke the explore phase if needed, or ask the user to manually edit `oc-migration-inventory.json`
- Proceed once the user confirms the inventory is correct

---

## Phase 2: Generation

After the inventory is confirmed, invoke the `k8s-oc-generate` skill:

```
/k8s-oc-generate oc-migration-inventory.json <output-dir>
```

The generate skill will:
- Read the confirmed inventory
- Generate all OC YAML files in the `namespaces/` structure
- Print a summary of generated files and any warnings

Present the summary to the user. Point out any warnings that require manual attention.

---

## Phase 3: Apply (Optional)

Ask the user:

> "Generation complete! Would you like to apply these files to a running OpenChoreo instance now?
>
> Options:
> - **Yes, use MCP** — I'll use OpenChoreo MCP tools if available
> - **Yes, use kubectl** — apply via `kubectl apply -f <output-dir> --recursive`
> - **No, I'll do it manually** — review files first and apply later"

If the user chooses to apply:

```
/k8s-oc-apply <output-dir>
```

---

## Summary Output

After all phases complete, print:

```
## Migration Complete

Source: <k8s-repo-path>
Output: <output-dir>
Inventory: oc-migration-inventory.json

### Results
- Projects created: N
- Components migrated: N
- ReleaseBindings generated: N (M environments each)
- Warnings requiring manual review: N

### Manual Review Required
<list any items from warnings that need human attention>

### Files Generated
<output-dir tree>
```

---

## Reference Materials

This skill's `reference/` directory contains:
- `mapping-table.md` — complete K8s → OC resource mapping
- `oc-yaml-patterns.md` — canonical OC YAML examples
- `edge-cases.md` — enterprise edge case handling guide
