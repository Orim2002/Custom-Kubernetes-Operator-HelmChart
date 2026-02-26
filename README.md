# Custom Operator Helm Chart

Helm chart for deploying the Preview Environment Operator onto a Kubernetes cluster. Packages the CRD, operator deployment, RBAC, Prometheus alerting rules, and ServiceMonitor into a single installable unit.

---

## What's Included

| Template | Description |
|----------|-------------|
| `crd.yaml` | `PreviewEnvironment` CustomResourceDefinition with OpenAPI v3 schema validation |
| `operator-deploy.yaml` | Deployment that runs the operator Pod |
| `rbac.yaml` | ServiceAccount, ClusterRole, and ClusterRoleBinding (least-privilege) |
| `alerts.yaml` | PrometheusRule with 3 alerting rules |
| `metrics-service.yaml` | Service + ServiceMonitor for Prometheus scraping |

---

## Prerequisites

- Kubernetes cluster with:
  - NGINX Ingress Controller
  - cert-manager with a `selfsigned-issuer` ClusterIssuer
  - kube-prometheus-stack (for alerts and metrics)
  - ArgoCD (optional, for GitOps deployment)

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
- `deployments` — create, get, list, watch, patch, update
- `services` — create, get, list, watch, patch, update
- `namespaces` — create, get, list, watch, delete
- `ingresses` — create, get, list, watch, patch, update
- `events` — create, patch (for kubectl describe output)

---

## GitOps with ArgoCD

This chart is designed to be deployed via ArgoCD. Point an ArgoCD Application at this repository and ArgoCD will automatically sync changes to the cluster on every push.
