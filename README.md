# vlab-charts

Helm charts for VirtuaLab demo tenants. Contains the `tenant-app` chart which provides baseline Kubernetes resources for tenant applications:
- **Deployment** with container probes (readiness/liveness)
- **Service** exposing the application
- **ResourceQuota** for namespace quota enforcement
- **NetworkPolicy** for default-deny with same-namespace and DNS egress
- **RBAC** (Role + RoleBinding) with read-only viewer access

## Quick Start

### Local Validation

Validate the chart locally:

```bash
# Lint the chart
helm lint charts/tenant-app --set image.tag=abc123
# Expected output: "1 chart(s) linted, 0 chart(s) failed"

# Render the template and inspect output
helm template t charts/tenant-app --set image.tag=abc123
```

### Chart Interface

Values consumed by ApplicationSet via `helm.parameters`:

```yaml
tenant: time-payments           # tenant identifier
app: payments-api               # application name (labels + names)
replicas: 1                     # pod count
port: 8080                      # container and service port
probePath: /                    # readiness/liveness probe path
image.repository: <registry>    # container image registry
image.tag: <tag>                # container image tag
imagePullSecret: ghcr-pull      # imagePullSecret name
resources.*                     # cpu/memory limits and requests
quota.*                         # namespace pod/cpu/memory quota
```

### ApplicationSet Usage

Consume this chart via ArgoCD ApplicationSet with helm values override:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: tenant-apps
spec:
  generators:
    - list:
        elements:
          - name: payments-api
            tenant: time-payments
            tag: v1.2.3
  template:
    spec:
      source:
        repoURL: https://github.com/slackerwx/vlab-charts.git
        targetRevision: main
        path: charts/tenant-app
        helm:
          parameters:
            - name: app
              value: "{{ name }}"
            - name: tenant
              value: "{{ tenant }}"
            - name: image.tag
              value: "{{ tag }}"
```

### N+1 Testing

To test multiple tenants from the same chart, pass different values:

```bash
helm template t1 charts/tenant-app \
  --set app=api-1 --set tenant=client-1 --set image.tag=abc123
helm template t2 charts/tenant-app \
  --set app=api-2 --set tenant=client-2 --set image.tag=abc123
```

Both render independently with correct label/name isolation.

## Development

Install dependencies:

```bash
brew install helm chart-testing
```

Run linter:

```bash
ct lint --all --config ct.yaml
```

## Chart Structure

```
charts/tenant-app/
├── Chart.yaml                      # chart metadata
├── values.yaml                     # default values
├── templates/
│   ├── _helpers.tpl               # (empty; values are plain)
│   ├── deployment.yaml            # main application pod
│   ├── service.yaml               # service exposure
│   ├── resourcequota.yaml         # quota enforcement
│   ├── networkpolicy.yaml         # network isolation
│   └── rbac.yaml                  # read-only viewer role
```

## License

Internal use for VirtuaLab project.
