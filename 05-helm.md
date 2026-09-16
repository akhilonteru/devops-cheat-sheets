# Helm Cheat Sheet

> Helm 3 command reference for Kubernetes package management — repos, install/upgrade, rollback, templating, and packaging.

---

## Table of Contents

- [Repositories](#1-repositories)
- [Search & Inspect Charts](#2-search--inspect-charts)
- [Install & Upgrade](#3-install--upgrade)
- [Release Management](#4-release-management)
- [Values Files](#5-values-files)
- [Template, Lint & Package](#6-template-lint--package)
- [Dependencies](#7-dependencies)
- [Useful Patterns](#8-useful-patterns)

---

## 1. Repositories

```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add prometheus https://prometheus-community.github.io/helm-charts
helm repo list
helm repo update                       # Refresh all repos (run before installs)
helm repo remove bitnami
```

## 2. Search & Inspect Charts

```
helm search repo nginx                 # Search added repos
helm search repo nginx --versions      # All versions
helm show chart bitnami/nginx          # Chart metadata
helm show values bitnami/nginx         # Default values (pipe to file to customize)
helm show values bitnami/nginx > nginx-values.yaml
helm show readme bitnami/nginx
```

## 3. Install & Upgrade

```
helm install my-nginx bitnami/nginx
helm install my-nginx bitnami/nginx -n web --create-namespace
helm install my-nginx bitnami/nginx -f values.yaml
helm install my-nginx bitnami/nginx --set image.tag=1.27 --set service.type=NodePort
helm install my-nginx bitnami/nginx --set replicaCount=3 --set resources.limits.memory=512Mi
helm install my-nginx ./my-chart       # Install from local chart directory

helm upgrade my-nginx bitnami/nginx -f values.yaml
helm upgrade my-nginx bitnami/nginx --reuse-values --set image.tag=1.28
helm upgrade --install my-nginx bitnami/nginx -f values.yaml   # Install if missing, else upgrade
helm upgrade my-nginx ./my-chart -n web
```

## 4. Release Management

```
helm list                              # Releases in current namespace
helm list -A                           # All namespaces
helm list --failed / --pending / --deployed
helm status my-nginx                   # Release status
helm history my-nginx                  # Revision history
helm rollback my-nginx 2               # Rollback to revision 2
helm uninstall my-nginx
helm uninstall my-nginx -n web
helm uninstall my-nginx --keep-history # Uninstall but keep history
helm get values my-nginx               # User-supplied values
helm get values my-nginx --all         # All computed values
helm get manifest my-nginx             # Rendered Kubernetes manifests
helm get hooks my-nginx
```

## 5. Values Files

`values.yaml` (lowest priority → highest):

```yaml
image:
  repository: nginx
  tag: "1.27"
  pullPolicy: IfNotPresent

replicaCount: 2

service:
  type: ClusterIP
  port: 80

resources:
  limits: { cpu: 500m, memory: 512Mi }
  requests: { cpu: 100m, memory: 128Mi }

env:
  - name: APP_ENV
    value: production
```

**Priority:** `--set` (highest) > `-f custom.yaml` > chart's `values.yaml` (lowest).

```bash
helm install app ./chart -f base.yaml -f prod.yaml        # Multiple merged
helm install app ./chart --set env[0].name=DEBUG --set env[0].value=true
```

## 6. Template, Lint & Package

```
helm template my-nginx ./my-chart -f values.yaml     # Render manifests locally (no cluster needed)
helm template my-nginx bitnami/nginx --set service.type=NodePort
helm lint ./my-chart                                 # Validate chart
helm lint ./my-chart -f values.yaml
helm package ./my-chart                              # Create .tgz package
helm package ./my-chart --version 1.2.3 --app-version 2.0.0
helm install my-nginx ./my-chart-1.2.3.tgz
```

## 7. Dependencies

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "15.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
```

```
helm dependency update ./my-chart      # Download/refresh charts/
helm dependency list ./my-chart
helm dependency build ./my-chart       # Build from Chart.lock exactly
```

## 8. Useful Patterns

```bash
# Upgrade only if something changed (CI/CD)
helm upgrade --install app ./chart -f values.yaml --atomic --timeout 5m

# Wait for rollout success
helm upgrade --install app ./chart --wait --timeout 300s

# Rollback on failure
helm upgrade --install app ./chart --atomic        # auto-rollback on failed upgrade

# Render diff before applying
helm template app ./chart -f values.yaml | kubectl diff -f -

# Debug failing install
helm install app ./chart --debug --dry-run
kubectl describe pod -l app.kubernetes.io/instance=app
helm history app && helm rollback app <rev>

# Pull and unpack a remote chart for inspection
helm pull bitnami/nginx --untar
```

| Flag | Description |
| --- | --- |
| `--set key=val` | Override values inline (highest priority) |
| `-f, --values` | Values files (multiple allowed) |
| `-n, --namespace` | Target namespace |
| `--create-namespace` | Create namespace if missing |
| `--atomic` | Auto-rollback on failure |
| `--wait` | Wait until pods are ready |
| `--timeout` | Timeout for --wait/--atomic |
| `--dry-run` | Simulate without installing |
| `--debug` | Verbose output |

---
