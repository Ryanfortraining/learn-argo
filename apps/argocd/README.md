# ArgoCD Helm Chart for EKS

> **Linus says**: "Talk is cheap. Show me the code."
> This chart deploys ArgoCD to your EKS cluster with AWS LoadBalancer access.

## Prerequisites

- Kubernetes cluster (EKS)
- Helm 3.x installed
- `kubectl` configured to access your cluster

## Quick Start

### 1. Update Dependencies

First, download the official argo-cd chart:

```bash
cd argocd
helm dependency update
```

### 2. Install ArgoCD

```bash
# Create namespace
kubectl create namespace argocd

# Install the chart
helm install argocd . --namespace argocd
```

### 3. Get Access Information

```bash
# Wait for LoadBalancer to be ready (takes ~2 minutes on EKS)
kubectl get svc -n argocd argocd-argo-cd-server --watch

# Get LoadBalancer URL
export ARGOCD_SERVER=$(kubectl get svc -n argocd argocd-argo-cd-server -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "ArgoCD URL: http://$ARGOCD_SERVER"

# Get initial admin password
kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
echo
```

### 4. Login

- **URL**: Use the LoadBalancer hostname from step 3
- **Username**: `admin`
- **Password**: From the secret above

## Configuration

Edit `values.yaml` to customize your deployment.

### Common Customizations

#### Use Internal LoadBalancer (VPC-only access)

```yaml
argo-cd:
  server:
    service:
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-scheme: "internal"
```

#### Enable Insecure Mode (HTTP only - for testing)

```yaml
argo-cd:
  configs:
    params:
      server.insecure: true
```

## Upgrade

```bash
helm dependency update
helm upgrade argocd . --namespace argocd
```

## Uninstall

```bash
helm uninstall argocd --namespace argocd
kubectl delete namespace argocd
```

## Structure

```
argocd/
├── Chart.yaml          # Chart metadata and dependencies
├── values.yaml         # Configuration values
├── .helmignore         # Files to ignore when packaging
└── README.md           # This file
```

## Troubleshooting

### LoadBalancer stuck in Pending

```bash
# Check service events
kubectl describe svc -n argocd argocd-argo-cd-server

# Check AWS Load Balancer Controller logs
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```

### Cannot access ArgoCD UI

```bash
# Check if pods are running
kubectl get pods -n argocd

# Check server logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server
```

## References

- [ArgoCD Official Docs](https://argo-cd.readthedocs.io/)
- [ArgoCD Helm Chart](https://github.com/argoproj/argo-helm/tree/main/charts/argo-cd)
