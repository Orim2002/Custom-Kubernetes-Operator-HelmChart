# Custom Operator Helm Chart

Helm chart for deploying the Preview Environment Operator onto a Kubernetes cluster. Packages the CRD, operator deployment, RBAC, NetworkPolicy, Prometheus alerting rules, and ServiceMonitor into a single installable unit.

---

## What's Included

| Template | Description |
|----------|-------------|
| `crd.yaml` | `PreviewEnvironment` CustomResourceDefinition with OpenAPI v3 schema validation |
| `operator-deploy.yaml` | Deployment that runs the operator Pod (non-root, read-only filesystem) |
| `rbac.yaml` | ServiceAccount, ClusterRole, and ClusterRoleBinding (least-privilege) |
| `networkpolicies.yaml` | NetworkPolicy restricting metrics port 8000 to the `monitoring` namespace only |
| `alerts.yaml` | PrometheusRule with 3 alerting rules |
| `metrics-service.yaml` | Service + ServiceMonitor for Prometheus scraping |

---

## Prerequisites

- Kubernetes cluster with:
  - NGINX Ingress Controller
  - cert-manager with a `letsencrypt-issuer` ClusterIssuer
  - kube-prometheus-stack (for alerts and metrics)
  - ArgoCD (optional, for GitOps deployment)
  - Azure Workload Identity webhook (if enabling `workloadIdentity`)

---

## Install

```bash
helm install custom-operator ./operator \
  --namespace operator \
  --create-namespace
```

## Upgrade

```bash
helm upgrade custom-operator ./operator \
  --namespace operator
```

## Uninstall

```bash
helm uninstall custom-operator -n operator
```

---

## Configuration (`values.yaml`)

```yaml
operator:
  image:
    repository: youruser/custom-operator
    tag: "latest"
    pullPolicy: IfNotPresent
  replicaCount: 1
  resources:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "200m"
      memory: "256Mi"

rbac:
  create: true      # set to false if managing RBAC externally

workloadIdentity:
  enabled: false    # set to true to enable Azure Workload Identity
  clientId: ""      # managed identity client ID from `terraform output operator_client_id`

alerting:
  enabled: true     # set to false to skip PrometheusRule creation
```

---

## Alerting Rules

Three PrometheusRule alerts are included (requires `alerting.enabled: true`):

| Alert | Severity | Condition | For |
|-------|----------|-----------|-----|
| `PreviewOperatorDown` | critical | `absent(preview_environments_active)` — operator stopped reporting | 2m |
| `PreviewEnvironmentHighFailureRate` | warning | >3 failures in 5 minutes | 1m |
| `PreviewEnvironmentSlowCreation` | warning | p95 creation time >30s | 5m |

---

## RBAC Permissions

The operator's ClusterRole grants:

- `previewenvironments` — get, list, watch, patch, update, delete
- `previewenvironments/status` — get, patch
- `deployments` — create, get, list, watch, patch
- `services` — create, get, list, watch
- `namespaces` — create, get, list, watch, delete
- `ingresses`, `networkpolicies` — create, get, list, watch, patch
- `events` — create, patch
- `customresourcedefinitions` — get, list, watch (CRD discovery)
- `clusterkopfpeerings`, `kopfpeerings` — full access (kopf leader election)

---

## Security

- Pod runs as non-root user (UID 1000)
- Read-only root filesystem
- All Linux capabilities dropped
- Metrics port 8000 restricted to the `monitoring` namespace via NetworkPolicy

---

## GitOps with ArgoCD

This chart is designed to be deployed via ArgoCD. Point an ArgoCD Application at this repository and ArgoCD will automatically sync changes to the cluster on every push.

To enable Workload Identity after provisioning with Terraform, set parameters in the ArgoCD application:

```yaml
workloadIdentity:
  enabled: true
  clientId: "<value from terraform output operator_client_id>"
```
