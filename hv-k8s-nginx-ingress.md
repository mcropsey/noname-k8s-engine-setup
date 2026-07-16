# hv-k8s-nginx-ingress.md

# NGINX Ingress Controller Setup — Kubernetes on Rocky Linux 9.8

## Reference Documentation

| Topic | URL |
|---|---|
| NGINX Ingress Controller (main docs) | https://kubernetes.github.io/ingress-nginx/ |
| Installation Guide | https://kubernetes.github.io/ingress-nginx/deploy/ |
| Bare Metal considerations | https://kubernetes.github.io/ingress-nginx/deploy/baremetal/ |
| IngressClass | https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-class |
| Ingress resource | https://kubernetes.io/docs/concepts/services-networking/ingress/ |
| GitHub releases | https://github.com/kubernetes/ingress-nginx/releases |

---

## Cluster Details

| Role | Hostname | IP |
|---|---|---|
| Control Plane | hv-rocky-linux-1 | 192.168.1.98 |
| Worker | hv-rocky-linux-2 | 192.168.1.99 |
| Worker | hv-rocky-linux-3 | 192.168.1.100 |

---

## Installation

> Reference: https://kubernetes.github.io/ingress-nginx/deploy/#bare-metal-clusters

Bare metal manifest was used (not cloud provider) since there is no external load balancer. This exposes the ingress controller via **NodePort**.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.1/deploy/static/provider/baremetal/deploy.yaml
```

### What this creates

| Resource | Name | Notes |
|---|---|---|
| Namespace | `ingress-nginx` | All ingress resources live here |
| Deployment | `ingress-nginx-controller` | The NGINX ingress controller pod |
| Service | `ingress-nginx-controller` | NodePort — exposes HTTP and HTTPS |
| Service | `ingress-nginx-controller-admission` | Webhook for validating Ingress resources |
| IngressClass | `nginx` | Use `ingressClassName: nginx` in Ingress resources |
| Jobs | admission-create / admission-patch | One-time webhook cert setup jobs |

---

## Installed State

### Pods

```
NAMESPACE       NAME                                        READY   STATUS
ingress-nginx   ingress-nginx-admission-create-j6h4t        0/1     Completed
ingress-nginx   ingress-nginx-admission-patch-64zws         0/1     Completed
ingress-nginx   ingress-nginx-controller-5fc44c4c5b-n57hh   1/1     Running
```

### Services & NodePorts

```
NAME                                 TYPE        CLUSTER-IP      PORT(S)
ingress-nginx-controller             NodePort    10.96.86.159    80:30966/TCP, 443:32537/TCP
ingress-nginx-controller-admission   ClusterIP   10.110.199.22   443/TCP
```

| Protocol | NodePort | Access URL |
|---|---|---|
| HTTP | 30966 | `http://192.168.1.98:30966` |
| HTTPS | 32537 | `https://192.168.1.98:32537` |

> NodePorts are accessible on **any node IP** — use .98, .99, or .100 interchangeably.

---

## Verify Installation

```bash
# Check controller is running
kubectl get pods -n ingress-nginx

# Check NodePorts
kubectl get svc -n ingress-nginx

# Check IngressClass
kubectl get ingressclass

# Test HTTP response (expects 404 — no ingress rules yet)
curl -I http://192.168.1.98:30966
```

---

## Using the Ingress Controller

### Example Ingress resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: my-app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-svc
            port:
              number: 80
```

> Reference: https://kubernetes.io/docs/concepts/services-networking/ingress/

### Apply it

```bash
kubectl apply -f my-app-ingress.yaml
kubectl get ingress
```

---

## Useful Commands

```bash
# Check ingress controller logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller -f

# List all ingress resources across all namespaces
kubectl get ingress -A

# Describe an ingress (useful for troubleshooting)
kubectl describe ingress <name> -n <namespace>

# Check ingress class
kubectl get ingressclass

# Reload NGINX config (happens automatically on Ingress changes)
kubectl rollout restart deployment/ingress-nginx-controller -n ingress-nginx
```

---

## Notes

- **Bare metal NodePort**: Since this cluster has no cloud load balancer, the ingress controller is exposed via NodePort (30966/32537). For production use consider [MetalLB](https://metallb.universe.tf/) to get a proper LoadBalancer IP.
- **IngressClass**: Always specify `ingressClassName: nginx` in Ingress resources — required in Kubernetes 1.18+.
- **Admission webhook**: The two `Completed` jobs (admission-create/patch) are normal — they run once to set up the validating webhook and exit.
- **NodePort access**: The NodePorts are reachable on all three node IPs (98, 99, 100) — Kubernetes routes the traffic internally regardless of which node receives it.
