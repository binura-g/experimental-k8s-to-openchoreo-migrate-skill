# oc-migration-inventory.json Schema

This file is the intermediate artifact passed from `k8s-oc-explore` to `k8s-oc-generate`.
It can be manually edited to adjust the migration before generation.

## Full Schema

```json
{
  "schemaVersion": "1.0",
  "sourceRepo": "/path/to/k8s-repo",
  "repoType": "raw | kustomize | mixed",
  "ocNamespace": "default",
  "generatedAt": "2024-01-01T00:00:00Z",

  "projects": [
    {
      "name": "payment",
      "sourceNamespaces": ["payment-dev", "payment-staging", "payment-prod"],
      "environments": ["development", "staging", "production"],

      "components": [
        {
          "name": "api-server",
          "sourceKind": "Deployment",
          "sourceNamespace": "payment-dev",
          "sourceResourceName": "payment-api",

          "ocComponentType": "deployment/service",
          // Valid values:
          // "deployment/service"     - standard HTTP service
          // "deployment/database"    - database-like workload (postgres, mysql, etc)
          // "deployment/web-app"     - frontend/static serving
          // "cronjob/scheduled-task" - CronJob
          // "statefulset/service"    - StatefulSet

          "image": "myregistry/payment-api:latest",

          "containers": [
            {
              "name": "api-server",
              "image": "myregistry/payment-api:latest",
              "isMain": true,
              "ports": [
                {"containerPort": 8080, "protocol": "TCP"}
              ],
              "env": [
                {
                  "name": "DATABASE_URL",
                  "value": "postgres://...",
                  "valueType": "plain | secretRef | configMapRef",
                  "secretName": null,
                  "secretKey": null,
                  "configMapName": null,
                  "configMapKey": null
                }
              ],
              "volumeMounts": [
                {
                  "name": "config-volume",
                  "mountPath": "/etc/config",
                  "sourceType": "configMap | secret",
                  "sourceName": "api-config"
                }
              ],
              "resources": {
                "requests": {"cpu": "100m", "memory": "128Mi"},
                "limits": {"cpu": "500m", "memory": "512Mi"}
              },
              "command": null,
              "args": null,
              "livenessProbe": null,
              "readinessProbe": null
            }
          ],

          "sidecars": [
            {
              "name": "envoy-proxy",
              "image": "envoyproxy/envoy:v1.28",
              "warning": "Sidecar not migrated — OC handles service mesh at platform level"
            }
          ],

          "initContainers": [
            {
              "name": "db-migrate",
              "image": "myregistry/payment-api:latest",
              "warning": "Init container not supported in OC — manual migration required"
            }
          ],

          "services": [
            {
              "name": "payment-api-svc",
              "type": "ClusterIP | LoadBalancer | NodePort",
              "ports": [
                {"port": 80, "targetPort": 8080, "protocol": "TCP", "name": "http"}
              ],
              "ocVisibility": "project | external"
              // ClusterIP → project
              // LoadBalancer/NodePort → external
            }
          ],

          "ingress": [
            {
              "host": "api.payment.example.com",
              "path": "/v1",
              "pathType": "Prefix",
              "backendService": "payment-api-svc",
              "backendPort": 80,
              "ocVisibility": "external",
              "ocBasePath": "/v1"
            }
          ],

          "configMaps": [
            {
              "name": "api-config",
              "namespace": "payment-dev",
              "usageType": "env | volume",
              "keys": ["APP_ENV", "LOG_LEVEL"],
              "mountPath": "/etc/config"
            }
          ],

          "secrets": [
            {
              "name": "api-secrets",
              "namespace": "payment-dev",
              "usageType": "env | volume",
              "keys": ["DATABASE_PASSWORD", "JWT_SECRET"],
              "mountPath": null
            }
          ],

          "pvcs": [
            {
              "name": "data-pvc",
              "storageClassName": "standard",
              "accessModes": ["ReadWriteOnce"],
              "size": "10Gi",
              "mountPath": "/data"
            }
          ],

          "hpa": {
            "minReplicas": 2,
            "maxReplicas": 10,
            "targetCPUUtilizationPercentage": 70
          },

          "replicas": 3,

          "connections": [
            {
              "fromComponent": "api-server",
              "toService": "postgres-svc",
              "toComponent": "postgres",
              "envVarName": "DATABASE_URL",
              "envVarValue": "postgres://postgres-svc.payment-dev.svc.cluster.local:5432/db",
              "detectionMethod": "dns | shortname | k8s-injected",
              "ocEndpoint": "tcp",
              "ocVisibility": "project"
            }
          ],

          "traits": [
            // Added when PVC detected
            {
              "type": "persistent-volume",
              "params": {
                "size": "10Gi",
                "mountPath": "/data",
                "storageClass": "standard"
              }
            },
            // Added when HPA detected
            {
              "type": "autoscaler",
              "params": {
                "minReplicas": 2,
                "maxReplicas": 10,
                "targetCPUUtilization": 70
              }
            }
          ],

          "cronSchedule": null,
          // For CronJob components: "*/5 * * * *"

          "kustomizeOverlays": {
            // Present only for Kustomize repos
            "base": {
              "image": "myregistry/payment-api:1.0.0",
              "replicas": 1,
              "resources": null
            },
            "development": {
              "image": "myregistry/payment-api:dev-latest",
              "replicas": 1,
              "resources": {
                "requests": {"cpu": "50m", "memory": "64Mi"},
                "limits": {"cpu": "200m", "memory": "256Mi"}
              },
              "additionalEnv": []
            },
            "staging": {
              "image": "myregistry/payment-api:staging-1.2.0",
              "replicas": 2,
              "resources": null,
              "additionalEnv": []
            },
            "production": {
              "image": "myregistry/payment-api:1.2.0",
              "replicas": 5,
              "resources": {
                "requests": {"cpu": "500m", "memory": "512Mi"},
                "limits": {"cpu": "2000m", "memory": "2Gi"}
              },
              "additionalEnv": []
            }
          },

          "warnings": [
            "Multi-container pod: sidecar 'envoy-proxy' not migrated",
            "Init container 'db-migrate' requires manual migration"
          ]
        }
      ]
    }
  ],

  "warnings": [
    "[UNSUPPORTED] DaemonSet 'log-collector' in namespace 'payment-dev' — no OC equivalent, skipped",
    "[HELM] Resources in 'payment-dev' appear Helm-managed — metadata stripped",
    "[CROSS-NS] Deployment 'api-server' references Service 'auth-svc' in namespace 'auth' — manual review required"
  ],

  "skippedResources": [
    {"kind": "ServiceAccount", "name": "payment-sa", "namespace": "payment-dev", "reason": "OC platform responsibility"},
    {"kind": "ClusterRole", "name": "payment-reader", "reason": "OC platform responsibility"},
    {"kind": "NetworkPolicy", "name": "deny-all", "namespace": "payment-dev", "reason": "OC platform responsibility"},
    {"kind": "DaemonSet", "name": "log-collector", "namespace": "payment-dev", "reason": "Unsupported in OC"},
    {"kind": "CustomCRD", "name": "CertificateRequest", "namespace": "payment-dev", "reason": "Custom CRD — no OC equivalent"}
  ]
}
```

## Field Notes

### `ocComponentType` Selection Heuristics

| Condition | Selected Type |
|-----------|--------------|
| Workload kind = CronJob | `cronjob/scheduled-task` |
| Workload kind = StatefulSet | `statefulset/service` |
| Image contains: `postgres`, `mysql`, `mariadb`, `mongodb`, `redis`, `elasticsearch` | `deployment/database` |
| Image contains: `nginx`, `apache`, `caddy` AND no app code evident | `deployment/web-app` |
| Default | `deployment/service` |

### Environment Name Normalization

| K8s Suffix / Overlay Dir | OC Environment Name |
|--------------------------|---------------------|
| `-dev`, `dev`, `development` | `development` |
| `-staging`, `staging`, `stage` | `staging` |
| `-prod`, `prod`, `production` | `production` |
| `-test`, `test`, `qa`, `uat` | `testing` |

### Connection `ocEndpoint` Inference

| Service Port | Protocol Guess | OC Endpoint |
|-------------|---------------|-------------|
| 80, 8080, 3000, 8000 | HTTP | `http` |
| 443, 8443 | HTTPS | `https` |
| 5432 | PostgreSQL | `tcp` |
| 3306 | MySQL | `tcp` |
| 6379 | Redis | `tcp` |
| 27017 | MongoDB | `tcp` |
| Other | TCP | `tcp` |
