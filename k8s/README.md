# Level 4: Resource Constraints

## 🎯 Learning Objectives
- Understand resource requests and limits in Kubernetes
- Learn about CPU and memory management
- Practice production-ready configuration

## 📚 Concepts Covered

### Resource Requests vs Limits

| Type | Purpose | What Happens If Exceeded |
|------|---------|--------------------------|
| **Requests** | Guaranteed minimum resources | Pod won't be scheduled if node can't provide |
| **Limits** | Maximum allowed resources | Container is throttled (CPU) or killed (memory) |

```yaml
resources:
  requests:
    memory: "128Mi"   # Guaranteed 128 MiB
    cpu: "100m"       # Guaranteed 0.1 CPU cores
  limits:
    memory: "256Mi"   # Max 256 MiB (OOMKilled if exceeded)
    cpu: "250m"       # Max 0.25 CPU cores (throttled if exceeded)
```

### CPU Units
- `1` = 1 vCPU/Core
- `100m` = 100 millicores = 0.1 cores
- `250m` = 250 millicores = 0.25 cores

### Memory Units
- `Ki` = Kibibytes (1024 bytes)
- `Mi` = Mebibytes (1024 Ki)
- `Gi` = Gibibytes (1024 Mi)

## 📊 Resource Allocations in This Level

| Component | Requests | Limits | Rationale |
|-----------|----------|--------|-----------|
| **PostgreSQL** | 256Mi / 250m | 512Mi / 500m | Database needs consistent resources |
| **Backend** | 128Mi / 100m | 256Mi / 250m | Flask app, moderate load |
| **Frontend** | 64Mi / 50m | 128Mi / 100m | Static file serving (Nginx) |

## ⚠️ What's Different from Level 3

| Aspect | Level 3 | Level 4 |
|--------|---------|---------|
| Resource limits | None | All containers have limits |
| Resource requests | None | All containers have requests |
| QoS Class | BestEffort | Burstable |

## 🚀 Deployment Steps

```bash
# 1. Apply all manifests
kubectl apply -f k8s/level-4-namespaces-resources/

# 2. Check resource allocations
kubectl describe pod -l app=backend | grep -A 5 "Requests:"

# 3. View resource usage (requires metrics-server)
kubectl top pods
```

## 📁 Files in This Level

| File | Purpose |
|------|---------|
| `postgres-init-configmap.yaml` | Database init script |
| `configmap.yaml` | Application configuration |
| `secret.yaml` | Sensitive credentials |
| `postgres-pv.yaml` | PersistentVolume |
| `postgres-pvc.yaml` | PersistentVolumeClaim |
| `postgres-headless-service.yaml` | Headless service for StatefulSet |
| `postgres-statefulset.yaml` | StatefulSet with resource limits |
| `backend-deployment.yaml` | Backend with resource limits |
| `backend-service.yaml` | Backend ClusterIP service |
| `frontend-deployment.yaml` | Frontend with resource limits |
| `frontend-service.yaml` | Frontend NodePort service |

## 🔍 Verification Commands

```bash
# Check resource allocations
kubectl describe pod -l app=backend | grep -A 5 "Requests:"

# View resource usage (requires metrics-server)
kubectl top pods

# Test resource limits - run the mining operation and watch CPU
kubectl top pods -w &
# Then trigger mining from the frontend

# See what happens when limits are exceeded
kubectl describe pod -l app=backend
# Look for OOMKilled or CPU throttling events
```

## 🧪 Experiments

### 1. Test CPU Throttling
```bash
# Watch CPU usage
kubectl top pods -w

# Trigger mining (CPU-intensive)
kubectl exec -it $(kubectl get pod -l app=frontend -o jsonpath='{.items[0].metadata.name}') -- curl -X POST http://backend-service:5000/api/mine -H "Content-Type: application/json" -d '{"id":"test"}'
```

### 2. Test Memory Limits
Change the backend memory limit to something very small (e.g., 32Mi) and see what happens.

## 💡 Discussion Questions

1. Why should requests typically be lower than limits?
2. What is QoS (Quality of Service) class and how does it affect pod eviction?
3. How would you prevent a single pod from consuming all cluster resources?

## 🎓 Advanced Topics (For Further Learning)

- **LimitRange**: Set default limits for a namespace
- **ResourceQuota**: Limit total resources per namespace
- **PodDisruptionBudget**: Control pod evictions during maintenance
- **Horizontal Pod Autoscaler (HPA)**: Auto-scale based on metrics
