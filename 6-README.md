# Kubernetes Store Orchestrator

A production-ready platform for provisioning and managing eCommerce stores (WooCommerce & Medusa) on Kubernetes.

## 🎯 Overview

This system provides a self-service dashboard for creating isolated eCommerce stores on Kubernetes. Each store runs in its own namespace with resource quotas, persistent storage, and automatic provisioning.

### Key Features

- **Multi-store support**: Provision WooCommerce or Medusa stores on demand
- **Namespace isolation**: Each store gets its own namespace with resource quotas
- **Kubernetes-native**: Uses Deployments, StatefulSets, Services, Ingress, PVCs
- **Helm-based**: Same charts work for local development and production
- **Production-ready**: RBAC, security contexts, resource limits, autoscaling
- **Idempotent**: Safe to retry operations, recovers from failures
- **Clean teardown**: Deleting a store removes all resources

## 📋 Prerequisites

### Local Development
- Docker
- Kind, k3d, or Minikube
- kubectl
- Helm 3
- Node.js 18+ (for local development)

### Production (VPS)
- k3s installed
- kubectl configured
- Helm 3
- Domain name with DNS configured

## 🚀 Quick Start (Local)

### 1. Create Local Kubernetes Cluster

```bash
# Using Kind
kind create cluster --name store-orchestrator

# OR using k3d
k3d cluster create store-orchestrator

# OR using Minikube
minikube start --profile store-orchestrator
```

### 2. Install NGINX Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Wait for ingress controller to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

### 3. Build Docker Images

```bash
# Build backend
cd backend
docker build -t store-orchestrator-backend:latest .

# Build dashboard
cd ../dashboard
docker build -t store-orchestrator-dashboard:latest .

# Load images into Kind (if using Kind)
kind load docker-image store-orchestrator-backend:latest --name store-orchestrator
kind load docker-image store-orchestrator-dashboard:latest --name store-orchestrator
```

### 4. Install the Helm Chart

```bash
cd ../helm/store-orchestrator

# Create namespace
kubectl create namespace store-orchestrator

# Install chart
helm install orchestrator . \
  --namespace store-orchestrator \
  --values values.yaml
```

### 5. Configure Local DNS

Add to `/etc/hosts`:
```
127.0.0.1  orchestrator.local.dev
127.0.0.1  store-*.local.dev
```

### 6. Access the Dashboard

```bash
# Port forward if needed
kubectl port-forward -n store-orchestrator svc/orchestrator-dashboard 8080:80

# Open browser
open http://orchestrator.local.dev:8080
```

## 🌐 Production Deployment (VPS with k3s)

### 1. Install k3s

```bash
curl -sfL https://get.k3s.io | sh -

# Copy kubeconfig
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
```

### 2. Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### 3. Build and Push Images

```bash
# Tag images for your registry
docker tag store-orchestrator-backend:latest your-registry/store-orchestrator-backend:1.0.0
docker tag store-orchestrator-dashboard:latest your-registry/store-orchestrator-dashboard:1.0.0

# Push to registry
docker push your-registry/store-orchestrator-backend:1.0.0
docker push your-registry/store-orchestrator-dashboard:1.0.0
```

### 4. Configure DNS

Point your domain to your VPS IP:
```
A  orchestrator.yourdomain.com  →  YOUR_VPS_IP
A  *.yourdomain.com             →  YOUR_VPS_IP
```

### 5. Install cert-manager (Optional, for TLS)

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Create Let's Encrypt issuer
cat <<EOF | kubectl apply -f -
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
          class: nginx
EOF
```

### 6. Deploy with Production Values

```bash
# Update values-prod.yaml with your domain and registry

helm install orchestrator ./helm/store-orchestrator \
  --namespace store-orchestrator \
  --create-namespace \
  --values helm/store-orchestrator/values-prod.yaml
```

## 📖 How to Use

### Create a Store

1. Open the dashboard
2. Click "Create New Store"
3. Choose WooCommerce or Medusa
4. Wait for provisioning (status will change to "Ready")
5. Click "Open Store" to visit your new store

### Test End-to-End

#### For WooCommerce:
1. Visit store URL
2. Go to WordPress admin: `http://store-xxxxx.local.dev/wp-admin`
3. Default credentials: admin/admin (configure in Helm values)
4. Add a product in WooCommerce
5. Add to cart and checkout
6. Verify order in WooCommerce admin

#### For Medusa:
1. Visit storefront URL
2. Browse products (sample products auto-created)
3. Add to cart
4. Complete checkout
5. Verify order via Medusa admin

### Delete a Store

1. Click "Delete" on the store card
2. Confirm deletion
3. All resources (namespace, PVCs, secrets) are cleaned up

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                       User Browser                          │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                  NGINX Ingress Controller                   │
│         orchestrator.local.dev → Dashboard                  │
│         store-*.local.dev → Store Instances                 │
└──────────────┬──────────────────────────────────────────────┘
               │
       ┌───────┴──────┐
       │              │
       ▼              ▼
┌─────────────┐  ┌─────────────┐
│  Dashboard  │  │   Backend   │
│  (React)    │  │  (Node.js)  │
└─────────────┘  └──────┬──────┘
                        │
                        ▼
               ┌─────────────────┐
               │ Kubernetes API  │
               └────────┬────────┘
                        │
         ┌──────────────┼──────────────┐
         │              │              │
         ▼              ▼              ▼
    ┌────────┐    ┌────────┐    ┌────────┐
    │ Store  │    │ Store  │    │ Store  │
    │Namespace│    │Namespace│    │Namespace│
    │   #1   │    │   #2   │    │   #3   │
    └────────┘    └────────┘    └────────┘
```

### Components

- **Dashboard (React)**: User interface for managing stores
- **Backend (Node.js)**: REST API for store lifecycle management
- **Store Orchestrator**: Creates and manages store namespaces
- **Kubernetes Resources**: Namespaces, Deployments, Services, PVCs, Ingress
- **RBAC**: Service account with least-privilege permissions

### Store Isolation

Each store gets:
- Dedicated namespace
- Resource quotas (CPU, memory, storage limits)
- Limit ranges (default resource requests/limits)
- Isolated secrets
- Separate persistent volumes
- Network policies (optional)

## 🔒 Security

### Implemented

- **RBAC**: Least privilege service account
- **Security contexts**: Run as non-root user
- **Container hardening**: Drop all capabilities, read-only filesystem where possible
- **Secret management**: No hardcoded secrets
- **Rate limiting**: API rate limits per IP
- **Resource quotas**: Prevent resource exhaustion
- **Namespace isolation**: Each store isolated

### Additional Hardening (Production)

```bash
# Network policies for store isolation
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: store-xxxxx
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress
  namespace: store-xxxxx
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
EOF
```

## 📊 Scaling

### Horizontal Scaling

Enabled in production via HPA:

```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70
```

### What Scales

- ✅ Dashboard pods
- ✅ Backend API pods
- ⚠️  Store instances (manual - each in own namespace)

### Provisioning Throughput

- Concurrent provisioning supported
- Rate limits prevent abuse
- Timeout controls (default 10 minutes)
- Queue-based system for high load (future enhancement)

## 🔄 Upgrade & Rollback

### Upgrade Platform

```bash
# Update images
helm upgrade orchestrator ./helm/store-orchestrator \
  --namespace store-orchestrator \
  --values values-prod.yaml \
  --set backend.image.tag=1.1.0 \
  --set dashboard.image.tag=1.1.0
```

### Rollback

```bash
# List releases
helm history orchestrator -n store-orchestrator

# Rollback to previous
helm rollback orchestrator -n store-orchestrator

# Rollback to specific revision
helm rollback orchestrator 3 -n store-orchestrator
```

### Upgrade Store

```bash
# Update store image versions
kubectl set image deployment/wordpress \
  wordpress=wordpress:6.4 \
  -n store-xxxxx
```

## 🐛 Troubleshooting

### Stores stuck in "Provisioning"

```bash
# Check orchestrator logs
kubectl logs -n store-orchestrator deployment/orchestrator-backend -f

# Check store namespace
kubectl get pods -n store-xxxxx

# Describe pods
kubectl describe pod <pod-name> -n store-xxxxx
```

### Images not pulling

```bash
# For Kind
kind load docker-image store-orchestrator-backend:latest --name store-orchestrator
kind load docker-image store-orchestrator-dashboard:latest --name store-orchestrator

# Check image pull status
kubectl describe pod <pod-name> -n store-orchestrator
```

### Ingress not working

```bash
# Check ingress controller
kubectl get pods -n ingress-nginx

# Check ingress resource
kubectl get ingress -n store-orchestrator
kubectl describe ingress orchestrator-ingress -n store-orchestrator

# Port forward for local testing
kubectl port-forward -n store-orchestrator svc/orchestrator-dashboard 8080:80
```

## 📝 Configuration Reference

### Environment Variables (Backend)

- `LOG_LEVEL`: Logging level (debug, info, warn, error)
- `MAX_STORES_PER_USER`: Maximum stores per user (default: 10)
- `PROVISIONING_TIMEOUT`: Timeout for store provisioning in ms (default: 600000)
- `INGRESS_DOMAIN`: Base domain for store URLs

### Resource Quotas (Per Store)

Default quotas applied to each store namespace:

```yaml
requests.cpu: 2
requests.memory: 4Gi
limits.cpu: 4
limits.memory: 8Gi
persistentvolumeclaims: 5
```

## 🎬 Demo Video Script

Record a 10-15 minute video covering:

1. **Introduction** (1 min)
   - Show architecture diagram
   - Explain components

2. **Local Setup** (3 min)
   - Create Kind cluster
   - Install ingress controller
   - Deploy with Helm
   - Show running pods

3. **Create Store** (3 min)
   - Open dashboard
   - Create WooCommerce store
   - Watch provisioning
   - Show created namespace and resources

4. **End-to-End Test** (3 min)
   - Access store
   - Add product to cart
   - Complete checkout
   - Show order in admin

5. **Delete Store** (2 min)
   - Delete from dashboard
   - Verify namespace deletion
   - Show cleanup

6. **Production Story** (2 min)
   - Show values-prod.yaml differences
   - Explain VPS deployment
   - Discuss scaling and security

7. **Architecture Deep Dive** (2 min)
   - RBAC setup
   - Isolation strategy
   - Idempotency and recovery

## 📄 License

MIT

## 🙏 Acknowledgments

Built for Urumi AI assessment - demonstrating Kubernetes orchestration, Helm packaging, and production-grade deployment practices.
