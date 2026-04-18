# Version 1: Nginx Ingress Controller (Old Way)

## API Evolution
| K8s Version | API Group | Status |
|---|---|---|
| < 1.14 | `extensions/v1beta1` | Removed in 1.22 |
| 1.14 - 1.18 | `networking.k8s.io/v1beta1` | Removed in 1.22 |
| 1.19+ | `networking.k8s.io/v1` | Current stable |

This folder demonstrates the **v1beta1** style (removed in K8s 1.22) for teaching purposes.

## How It Works
- Students install a **nginx ingress controller** inside the cluster (runs as a pod)
- The controller watches Ingress resources and configures nginx routing rules
- Traffic flow: Internet → LoadBalancer Service → nginx pod → app pod

## Install Nginx Ingress Controller
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/aws/deploy.yaml
```

## Apply Manifests
```bash
kubectl apply -f app1/manifest.yaml
kubectl apply -f app2/manifest.yaml
```

## Key Differences vs v1 API
- No `pathType` field required
- `backend` uses `serviceName` + `servicePort` directly (flat, not nested)
- `ingressClassName` set via annotation (`kubernetes.io/ingress.class`) not spec
