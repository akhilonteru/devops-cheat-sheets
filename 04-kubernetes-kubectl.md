# Kubernetes (kubectl) Cheat Sheet

> Complete kubectl command reference for DevOps engineers — pods, deployments, services, config, storage, networking, debugging, and one-liners.

---

## Table of Contents

- [Cluster & Context](#1-cluster--context)
- [Imperative Commands](#2-imperative-commands)
- [Declarative (YAML) Workflow](#3-declarative-yaml-workflow)
- [Pods](#4-pods)
- [Deployments](#5-deployments)
- [Services & Ingress](#6-services--ingress)
- [ConfigMaps & Secrets](#7-configmaps--secrets)
- [Storage (PV / PVC)](#8-storage-pv--pvc)
- [Namespaces](#9-namespaces)
- [Nodes](#10-nodes)
- [Jobs & CronJobs](#11-jobs--cronjobs)
- [RBAC](#12-rbac)
- [Debugging & Troubleshooting](#13-debugging--troubleshooting)
- [Useful One-Liners](#14-useful-one-liners)

---

## 1. Cluster & Context

```
kubectl cluster-info                        # Cluster endpoints
kubectl config get-contexts                 # List contexts
kubectl config use-context prod             # Switch context
kubectl config current-context              # Show current context
kubectl config set-context --current --namespace=dev
kubectl get nodes -o wide                   # Nodes with IPs/OS
kubectl api-resources                       # All API objects (shortnames)
kubectl api-versions                        # Supported API versions
kubectl explain pod.spec.containers         # Docs for object fields
kubectl version --short                     # Client/server versions
```

## 2. Imperative Commands

```
kubectl run nginx --image=nginx:alpine                 # Create pod
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl create deployment web --image=nginx:1.27 --replicas=3
kubectl expose deployment web --port=80 --target-port=8080 --type=ClusterIP
kubectl create namespace staging
kubectl create configmap app-config --from-literal=ENV=prod --from-file=app.conf
kubectl create secret generic db-secret --from-literal=PASSWORD=s3cret
kubectl create job backup --image=busybox -- date
kubectl create cronjob report --image=busybox --schedule="0 2 * * *" -- date
kubectl set image deployment/web nginx=nginx:1.28    # Rolling update
kubectl scale deployment web --replicas=5
kubectl autoscale deployment web --min=2 --max=10 --cpu-percent=70
```

## 3. Declarative (YAML) Workflow

```
kubectl apply -f manifest.yaml              # Create/update from file
kubectl apply -f ./manifests/               # Whole directory
kubectl apply -f app.yaml -n staging        # Specific namespace
kubectl diff -f manifest.yaml               # Preview changes
kubectl delete -f manifest.yaml             # Delete from file
kubectl replace -f manifest.yaml            # Replace (requires existing object)
kubectl get deploy web -o yaml > web-backup.yaml   # Export live object
```

## 4. Pods

```
kubectl get pods                            # Pods in current namespace
kubectl get pods -A                         # All namespaces
kubectl get pods -o wide                    # With node & IP
kubectl get pods -n staging -l app=web      # Filter by label
kubectl get pods --field-selector=status.phase=Running
kubectl get pod web -o yaml                 # Full manifest
kubectl describe pod web                    # Events & detailed state
kubectl logs web                            # Pod logs
kubectl logs -f web                         # Follow
kubectl logs web --tail=100 --timestamps
kubectl logs web -c sidecar                 # Specific container (multi-container)
kubectl logs web --previous                 # Previous crashed container's logs
kubectl exec -it web -- bash                # Shell into pod
kubectl exec web -- env                     # Run single command
kubectl exec -it web -c app -- sh           # Specific container
kubectl port-forward pod/web 8080:80        # Local access to pod
kubectl cp web:/var/log/app.log ./app.log   # Copy from pod
kubectl cp ./file.txt web:/tmp/file.txt     # Copy to pod
kubectl top pod -A                          # CPU/memory usage (metrics-server)
kubectl delete pod web                      # Delete (Deployment recreates it)
```

## 5. Deployments

```
kubectl get deployments                     # List deployments
kubectl get deploy web -o wide
kubectl describe deploy web
kubectl rollout status deployment/web       # Watch rollout progress
kubectl rollout history deployment/web      # Revision history
kubectl rollout history deployment/web --revision=2
kubectl rollout undo deployment/web         # Rollback to previous revision
kubectl rollout undo deployment/web --to-revision=1
kubectl rollout restart deployment/web      # Restart all pods (e.g., after ConfigMap change)
kubectl scale deployment web --replicas=3
kubectl set image deployment/web app=myapp:1.2 --record
kubectl get rs                              # ReplicaSets created by deployments
kubectl delete deployment web
```

## 6. Services & Ingress

```
kubectl get svc                             # List services
kubectl get svc -o wide
kubectl describe svc web                    # Endpoints & selector info
kubectl get endpoints web                   # Backend pod IPs behind a service
kubectl expose deployment web --type=LoadBalancer --port=80
kubectl get ingress                         # List ingress rules
kubectl describe ingress main
kubectl port-forward svc/web 8080:80        # Forward to service
```

**Service types:** `ClusterIP` (internal only), `NodePort` (exposes port on each node), `LoadBalancer` (cloud LB), `Headless` (`clusterIP: None` — direct pod DNS).

## 7. ConfigMaps & Secrets

```
# ConfigMaps
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info \
  --from-file=nginx.conf
kubectl get configmaps
kubectl describe configmap app-config
kubectl get configmap app-config -o yaml
kubectl delete configmap app-config

# Secrets
kubectl create secret generic db-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=s3cret123
kubectl create secret tls tls-secret --cert=cert.pem --key=key.pem
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=user --docker-password=pass
kubectl get secrets
kubectl describe secret db-secret           # Shows keys, not values
kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d; echo
kubectl edit secret db-secret
```

**Using in pods:**
```yaml
envFrom:
  - configMapRef: { name: app-config }
  - secretRef: { name: db-secret }
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef: { name: db-secret, key: DB_PASSWORD }
```

## 8. Storage (PV / PVC)

```
kubectl get pv                              # PersistentVolumes (cluster-scoped)
kubectl get pvc                             # PersistentVolumeClaims
kubectl get pvc -n staging
kubectl get storageclass                    # StorageClasses
kubectl describe pvc data
kubectl delete pvc data                     # Deletes PV if policy is Delete
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: data }
spec:
  accessModes: ["ReadWriteOnce"]
  resources: { requests: { storage: 10Gi } }
  storageClassName: standard
```

## 9. Namespaces

```
kubectl get namespaces                      # List
kubectl create namespace staging
kubectl delete namespace staging            # Deletes everything inside
kubectl get all -n staging                  # Objects in namespace
kubectl config set-context --current --namespace=staging   # Set default
```

## 10. Nodes

```
kubectl get nodes -o wide
kubectl describe node worker-1              # Capacity, conditions, allocated pods
kubectl top nodes                           # CPU/memory usage
kubectl cordon worker-2                     # Mark unschedulable (no new pods)
kubectl uncordon worker-2                   # Restore schedulability
kubectl drain worker-2 --ignore-daemonsets --delete-emptydir-data   # Evacuate for maintenance
kubectl get events --sort-by='.lastTimestamp'
```

## 11. Jobs & CronJobs

```
kubectl get jobs
kubectl logs job/backup
kubectl get cronjobs
kubectl create cronjob cleanup --image=busybox --schedule="*/30 * * * *" -- rm -rf /tmp/*
kubectl delete cronjob cleanup
kubectl get jobs --watch
```

## 12. RBAC

```
kubectl get roles
kubectl get rolebindings
kubectl get clusterroles
kubectl get clusterrolebindings
kubectl describe clusterrole admin
kubectl auth can-i get pods                     # Check own permissions
kubectl auth can-i delete deployments --as=system:serviceaccount:dev:ci-bot -n dev
```

## 13. Debugging & Troubleshooting

```
kubectl describe pod web                        # Events section = #1 tool
kubectl logs web --previous                     # Why did it crash?
kubectl exec -it web -- sh                      # Look inside
kubectl debug -it web --image=busybox --target=app   # Ephemeral debug container
kubectl debug node/worker-1 -it --image=ubuntu  # Debug a node
kubectl get events -A --sort-by='.lastTimestamp'
kubectl port-forward svc/web 8080:80            # Bypass ingress, test service directly
kubectl get pod web -o jsonpath='{.status.containerStatuses[*].state}'   # Container state
kubectl get pods -A | grep -v Running           # Non-running pods
kubectl top pods -A --sort-by=cpu
```

## 14. Useful One-Liners

```bash
# Pods sorted by restart count (find crash-loopers)
kubectl get pods -A --sort-by='.status.containerStatuses[0].restartCount'

# All pods not Running
kubectl get pods -A --field-selector=status.phase!=Running

# Find which node a pod runs on
kubectl get pod web -o wide

# Decode any secret value
kubectl get secret <name> -o go-template='{{index .data "key"}}' | base64 -d

# Watch pods live
kubectl get pods -w -n staging

# Export running deployment back to YAML
kubectl get deploy web -o yaml | grep -v '^\s*status:' > web.yaml

# Force delete a stuck pod (last resort)
kubectl delete pod web --force --grace-period=0

# Restart all deployments in a namespace (pick up new ConfigMaps/images)
kubectl rollout restart deployment -n staging

# Resource usage top-5 pods
kubectl top pods -A --sort-by=cpu | head -6
```

---
