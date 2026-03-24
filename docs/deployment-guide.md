# Kubernetes Deployment Guide

## Prerequisites

```bash
# Verify tools
kubectl version --client    # >= 1.28
helm version                # >= 3.12
```

## Cluster Setup (one-time)

### 1. Install cert-manager
```bash
helm repo add cert-manager https://charts.jetstack.io
helm repo update
helm install cert-manager cert-manager/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true \
  --version v1.14.0
```

### 2. Install Nginx Ingress Controller
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.replicaCount=2 \
  --set controller.metrics.enabled=true
```

### 3. Create ClusterIssuer for TLS
```bash
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
EOF
```

### 4. Create application secrets
```bash
kubectl create secret generic myapp-secrets \
  --namespace production \
  --from-literal=DATABASE_URL="postgresql://user:pass@host:5432/db" \
  --from-literal=REDIS_URL="redis://redis:6379" \
  --from-literal=SECRET_KEY="$(openssl rand -hex 32)"
```

## Deployment

```bash
# First deploy
helm upgrade --install myapp ./helm/myapp \
  --namespace production \
  --create-namespace \
  --values helm/myapp/values.yaml \
  --values helm/myapp/values-prod.yaml \
  --wait --timeout 5m

# Upgrade with new image
helm upgrade myapp ./helm/myapp \
  --namespace production \
  --reuse-values \
  --set api.image.tag=v1.2.3 \
  --wait --atomic

# Rollback
helm rollback myapp -n production
```

## Verify

```bash
kubectl get pods,svc,ingress,hpa -n production
kubectl describe hpa -n production
kubectl top pods -n production
```

## Scaling

HPA automatically scales based on CPU/memory. To manually scale:
```bash
kubectl scale deployment myapp-api --replicas=10 -n production
```

## Troubleshooting

```bash
# Pod not starting
kubectl describe pod <pod-name> -n production
kubectl logs <pod-name> -n production --previous

# HPA not scaling
kubectl describe hpa myapp-api -n production
kubectl get events -n production --sort-by='.lastTimestamp'

# Network policy blocking traffic
kubectl describe networkpolicy -n production
```
