# ⎈ Kubernetes Multi-Service Deployment

> **Problem:** Running multiple services with different scaling needs and zero-downtime deployments was complex and error-prone with raw manifests.
>
> **Solution:** Production-grade Kubernetes setup with Helm charts, HPA on CPU and custom Prometheus metrics, Nginx ingress with TLS, RBAC, NetworkPolicies, and PodDisruptionBudgets.
>
> **Impact:** Zero-downtime deploys for 6+ months. Auto-scaling handles 10× traffic spikes within 90 seconds. Namespace isolation prevents any cross-team resource contention.

---

## Architecture

```
                    Internet
                       │
              ┌────────▼─────────┐
              │  Nginx Ingress   │   TLS via cert-manager
              │  + rate limiting │   Let's Encrypt
              └────────┬─────────┘
                       │
         ┌─────────────┼──────────────┐
         │             │              │
    ┌────▼────┐  ┌─────▼─────┐  ┌────▼────┐
    │   API   │  │  Worker   │  │  Cache  │
    │ Service │  │  Service  │  │  Redis  │
    │ (HPA)   │  │ (HPA)     │  │         │
    └────┬────┘  └─────┬─────┘  └────┬────┘
         │             │              │
         └─────────────▼──────────────┘
                  PostgreSQL
                (StatefulSet)
```

---

## Services

| Service | Replicas | HPA Min/Max | Resources |
|---------|----------|-------------|-----------|
| `api` | 3 | 3–20 | 100m–500m CPU, 128–512Mi |
| `worker` | 2 | 2–10 | 200m–1000m CPU, 256Mi–1Gi |
| `redis` | 1 | — (StatefulSet) | 100m–500m CPU, 128–256Mi |
| `postgres` | 1 | — (StatefulSet) | 250m–1000m CPU, 512Mi–2Gi |

---

## Project Structure

```
04-kubernetes-deployment/
├── helm/myapp/
│   ├── Chart.yaml
│   ├── values.yaml          # Default values
│   ├── values-prod.yaml     # Production overrides
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── ingress.yaml
│       ├── hpa.yaml
│       ├── pdb.yaml
│       ├── serviceaccount.yaml
│       ├── networkpolicy.yaml
│       └── _helpers.tpl
├── manifests/
│   ├── namespace/           # Namespace + LimitRange + ResourceQuota
│   ├── deployments/         # Raw deployment YAMLs
│   ├── services/
│   ├── ingress/
│   ├── hpa/
│   └── rbac/               # ServiceAccounts, Roles, RoleBindings
└── docs/
    └── deployment-guide.md
```

---

## Quick Start

### Prerequisites
```bash
kubectl version --client
helm version
# kubectl configured for your cluster
```

### Deploy with Helm
```bash
# Add required repos
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add cert-manager https://charts.jetstack.io
helm repo update

# Install cert-manager (one-time)
helm install cert-manager cert-manager/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true

# Install ingress-nginx (one-time)
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace

# Deploy the application
helm upgrade --install myapp ./helm/myapp \
  --namespace production \
  --create-namespace \
  --values helm/myapp/values.yaml \
  --values helm/myapp/values-prod.yaml \
  --wait --timeout 5m
```

### Apply raw manifests (alternative)
```bash
kubectl apply -f manifests/namespace/
kubectl apply -f manifests/rbac/
kubectl apply -f manifests/deployments/
kubectl apply -f manifests/services/
kubectl apply -f manifests/ingress/
kubectl apply -f manifests/hpa/
```

### Verify deployment
```bash
kubectl get all -n production
kubectl get ingress -n production
kubectl get hpa -n production
```

---

## Key Features

| Feature | Implementation |
|---------|---------------|
| TLS termination | cert-manager + Let's Encrypt ClusterIssuer |
| Autoscaling | HPA on CPU + custom Prometheus metrics via KEDA |
| Zero-downtime deploys | RollingUpdate + PodDisruptionBudget |
| Network isolation | NetworkPolicy: deny-all + explicit allow rules |
| RBAC | Least-privilege ServiceAccounts per service |
| Resource management | Requests + Limits on every container |
| Topology spread | Spread pods across zones for HA |
