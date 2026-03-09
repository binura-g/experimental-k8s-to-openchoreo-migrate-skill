# Enterprise Edge Cases — K8s → OpenChoreo Migration

Catalog of edge cases and their handling strategies for the migration skills.

---

## 1. Multi-Container Pods

**Detection:** `spec.template.spec.containers` has more than one entry.

**Handling:**
- Identify the "main" container: first container, or the one with the largest resource requests, or the one whose name matches the Deployment name
- Include only the main container in the OC Workload's `container` field
- List sidecars in `component.sidecars[]` in the inventory
- Add a warning: `"Sidecar containers [<names>] not migrated — OC manages service mesh at platform level"`

**Common sidecars to expect in K8s:**
- `envoy`, `istio-proxy`, `linkerd-proxy` → OC handles service mesh automatically
- `fluentd`, `fluent-bit`, `logstash` → OC handles log aggregation at platform level
- `filebeat` → OC handles log forwarding
- `cloudsql-proxy` → Note: manual handling needed, flag for review

**Output in inventory:**
```json
"sidecars": [
  {
    "name": "istio-proxy",
    "image": "docker.io/istio/proxyv2:1.17.0",
    "warning": "Service mesh sidecar — OC injects Envoy automatically"
  }
]
```

---

## 2. Init Containers

**Detection:** `spec.template.spec.initContainers` is non-empty.

**Handling:**
- Skip init containers in migration
- Add a warning: `"Init container '<name>' not supported in OC — requires manual migration"`
- Document the init container's purpose (from name/command analysis):
  - `db-migrate`, `migration` → database migration, suggest running as a pre-deploy Job
  - `wait-for-*`, `check-*` → readiness check, may not be needed in OC (OC handles ordering)
  - `init-config`, `copy-config` → config setup, may be replaceable with OC file mounts

**Output in inventory:**
```json
"initContainers": [
  {
    "name": "db-migrate",
    "image": "myapp:latest",
    "command": ["./migrate"],
    "warning": "Init container not supported in OC. Consider: run as a separate Job component triggered before release, or use OC pre-deploy hooks if available."
  }
]
```

---

## 3. Helm-Managed Resources

**Detection:** Any of:
- `metadata.labels["app.kubernetes.io/managed-by"] = "Helm"`
- `metadata.annotations["helm.sh/chart"]` exists
- `metadata.annotations["meta.helm.sh/release-name"]` exists

**Handling:**
- Process the resource normally (extract spec)
- Strip Helm-specific labels and annotations from generated OC YAML
- Add a migration-level warning: `"Resource '<name>' appears Helm-managed. Template variables ({{ }}) have been stripped — verify generated values are correct."`

**Labels/annotations to strip:**
```
helm.sh/*
meta.helm.sh/*
app.kubernetes.io/managed-by (if value is "Helm")
app.kubernetes.io/version (Helm-managed version tracking)
```

**Important:** If a resource has `{{ .Values.* }}` template syntax in its values, it cannot be statically analyzed. Flag these values with:
```
# WARNING: This value was a Helm template variable. Set manually.
- name: SOME_VAR
  value: "HELM_TEMPLATE_VARIABLE_NEEDS_MANUAL_REVIEW"
```

---

## 4. Cross-Namespace Service References

**Detection:** Env var value matches a Service DNS name from a different namespace than the Deployment's namespace.

**Handling:**
- Create a connection entry in the inventory with a cross-namespace warning
- Note the target namespace and service name
- If the target namespace belongs to a different project in the OC structure, flag as potentially requiring external endpoint visibility

**Output in inventory:**
```json
"connections": [
  {
    "fromComponent": "api-server",
    "toService": "auth-svc",
    "toNamespace": "auth",
    "toComponent": null,
    "warning": "Cross-namespace Service reference. If 'auth-svc' is in a different OC project, set visibility to 'external' and update connection endpoint.",
    "crossNamespace": true
  }
]
```

**Generated YAML note:**
```yaml
connections:
  - component: auth-service      # Assumes auth-service is in same project
    endpoint: http
    visibility: project
    # WARNING: Original ref was cross-namespace (auth/auth-svc)
    # If auth-service is in a different OC project, change visibility to 'external'
    envBindings:
      address: AUTH_SERVICE_URL
```

---

## 5. StatefulSets with PVCs

**Detection:** `kind: StatefulSet` with `spec.volumeClaimTemplates` OR separate `PersistentVolumeClaim` resources.

**Handling:**
- Set component type to `statefulset/service`
- Add `persistent-volume` trait to Component spec
- Map PVC spec to trait properties:
  - `spec.resources.requests.storage` → `properties.size`
  - `spec.storageClassName` → `properties.storageClass`
  - Volume mount path → `properties.mountPath`

**Multiple PVCs:** OC supports one persistent-volume trait per component. If multiple PVCs exist:
- Use the largest/primary PVC for the trait
- Add a warning for additional PVCs: `"Multiple PVCs detected — only primary PVC '<name>' migrated. Additional PVC '<name2>' requires manual configuration."`

**Generated component.yaml:**
```yaml
spec:
  type: statefulset/service
  traits:
    - type: persistent-volume
      properties:
        size: 20Gi
        mountPath: /var/lib/postgresql/data
        storageClass: standard
```

---

## 6. HPA → Autoscaler Trait

**Detection:** `kind: HorizontalPodAutoscaler` targeting a known Deployment.

**Handling:**
- Add `autoscaler` trait to the Component spec
- Map HPA spec:
  - `spec.minReplicas` → `properties.minReplicas`
  - `spec.maxReplicas` → `properties.maxReplicas`
  - `spec.metrics[].resource.target.averageUtilization` (CPU) → `properties.targetCPUUtilizationPercentage`

**v2 HPA metrics:**
```yaml
# K8s HPA v2
spec:
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```
→ Extract `averageUtilization: 70` for the trait.

For custom metrics HPAs (non-CPU): Add warning `"HPA uses custom metrics — autoscaler trait only supports CPU/memory targets. Custom metrics require manual configuration."`

---

## 7. Namespace-as-Environment Pattern

**Detection:** Multiple namespaces with common prefix + environment suffix.

**Example:**
```
payment-dev
payment-staging
payment-prod
```

**Handling:**
1. Identify the common prefix (`payment`)
2. Identify environment suffixes (`dev`, `staging`, `prod`)
3. Map to OC environments (`development`, `staging`, `production`)
4. Cross-reference Deployments across namespaces by name — same-named Deployments in different namespaces = same component in different environments
5. Use base values from any namespace's Deployment (they should be similar)
6. Extract per-environment differences (image tags, replica counts, resource limits) → ReleaseBindings

**Challenge:** Services, ConfigMaps, and Secrets may have duplicate names across namespaces with different values. Per-env secrets become per-env `componentTypeEnvOverrides` in ReleaseBindings.

---

## 8. ImagePullSecrets

**Detection:** `spec.template.spec.imagePullSecrets` is non-empty.

**Handling:**
- Skip entirely — add to skipped resources list
- Add note: `"ImagePullSecrets handled by OC platform. Configure registry credentials in the OC control plane."`

---

## 9. External Secret References (Vault, AWS SM, etc.)

**Detection:**
- `kind: ExternalSecret` (external-secrets operator)
- `kind: SecretProviderClass` (Secrets Store CSI Driver)
- Annotations like `vault.hashicorp.com/agent-inject: "true"`

**Handling:**
- For ExternalSecret: preserve the secret name reference in `valueFrom.secretRef`
- Add a warning: `"Secret '<name>' is managed by an external secrets operator. Ensure the equivalent secret exists in the OC environment."`
- Note: OC supports ExternalSecrets — document the secret name so it can be recreated

**Do NOT attempt to inline secret values.**

---

## 10. Custom CRDs

**Detection:** `kind` is not in the known K8s resource list and not in OC resource list.

**Known K8s core kinds to recognize:**
```
Pod, ReplicaSet, Deployment, StatefulSet, DaemonSet, Job, CronJob,
Service, Endpoints, Ingress, IngressClass,
ConfigMap, Secret,
PersistentVolume, PersistentVolumeClaim, StorageClass,
ServiceAccount, ClusterRole, ClusterRoleBinding, Role, RoleBinding,
NetworkPolicy, PodDisruptionBudget,
HorizontalPodAutoscaler, VerticalPodAutoscaler,
Namespace, Node,
CustomResourceDefinition
```

**Handling:**
- Add to `skippedResources` list with `reason: "Custom CRD — no OC equivalent"`
- List all unique custom CRDs found in migration report

---

## 11. Multiple Ingress Rules / Hosts

**Detection:** `kind: Ingress` with multiple rules or paths.

**Handling:**
- Create one endpoint per unique (host, path) combination
- If multiple hosts: create multiple external endpoints
- Use path as `basePath`

**Example:**
```yaml
# K8s Ingress
rules:
  - host: api.example.com
    http:
      paths:
        - path: /v1
          backend: {service: api-svc, port: 80}
        - path: /v2
          backend: {service: api-v2-svc, port: 80}
  - host: admin.example.com
    http:
      paths:
        - path: /
          backend: {service: admin-svc, port: 8080}
```

**Generated endpoints on `api-server` workload:**
```yaml
endpoints:
  - name: api-v1
    port: 8080
    visibility: [external]
    basePath: /v1
  - name: api-v2
    port: 8080
    visibility: [external]
    basePath: /v2
```

---

## 12. Large Repos (100+ Workloads)

**Handling:**
- Process in batches by namespace/project
- Report progress every 20 resources: `"Processing components 21-40 of 142..."`
- Generate inventory in chunks if needed
- Recommend splitting large projects into smaller OC projects for manageability

---

## 13. Deployment with No Service

**Detection:** A Deployment with no matching Service (by selector label).

**Handling:**
- Generate Workload without `endpoints` section
- Add a note in inventory: `"No Service found for Deployment '<name>' — no endpoints generated"`
- This is valid for worker/consumer processes that don't expose a port

---

## 14. Service with No Matching Deployment

**Detection:** A Service whose selector labels don't match any Deployment.

**Handling:**
- Skip the Service (it has no component to attach to)
- Add a warning: `"Service '<name>' has no matching Deployment — skipped"`

---

## 15. DaemonSet

**Detection:** `kind: DaemonSet`

**Handling:**
- Skip entirely
- Add to `skippedResources` with `reason: "DaemonSet has no equivalent in OC"`
- Add a migration-level warning: `"[UNSUPPORTED] DaemonSet '<name>' in namespace '<ns>' — OC does not support DaemonSets. If this is a logging/monitoring agent, OC handles it at platform level."`

**Common DaemonSet purposes and OC equivalents:**
- Log collectors (fluentd, filebeat) → OC handles log aggregation
- Monitoring agents (node-exporter, datadog-agent) → OC handles platform monitoring
- Network proxies (Calico, Weave) → OC handles networking

---

## 16. Liveness and Readiness Probes

**Detection:** `spec.template.spec.containers[].livenessProbe` or `readinessProbe`

**Handling:**
- Preserve probe configuration in inventory (for documentation)
- Do NOT include in OC Workload YAML (OC manages health checking per component type)
- Add a comment in the generated workload.yaml:
```yaml
# Note: Original K8s probes preserved for reference:
# livenessProbe: httpGet /healthz port 8080 initialDelaySeconds 30
# readinessProbe: httpGet /ready port 8080 initialDelaySeconds 5
```

---

## 17. Environment Variables with URLs Referencing External Services

**Detection:** Env var value is a URL that doesn't match any known internal service (e.g., `https://api.stripe.com`, `https://s3.amazonaws.com`)

**Handling:**
- Keep as plain `value` in the workload env section
- Do NOT create a connection entry
- These are legitimate external service URLs that should be preserved as-is

---

## 18. Empty Namespaces or Namespaces with Only Non-Workload Resources

**Detection:** A namespace containing only ConfigMaps, Secrets, RBAC resources, etc. — no Deployments/StatefulSets/CronJobs.

**Handling:**
- Skip namespace for project generation
- Log: `"Namespace '<name>' has no workloads — skipped for project generation. ConfigMaps/Secrets are referenced by other components."`
