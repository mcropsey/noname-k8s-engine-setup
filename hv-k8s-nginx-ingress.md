# hv-k8s-nginx-ingress.md

# NGINX Ingress Controller Setup — Kubernetes on Rocky Linux 9.8

## Reference Documentation

| Topic | URL |
|---|---|
| NGINX Ingress Controller (main docs) | https://kubernetes.github.io/ingress-nginx/ |
| Installation Guide | https://kubernetes.github.io/ingress-nginx/deploy/ |
| Bare Metal considerations | https://kubernetes.github.io/ingress-nginx/deploy/baremetal/ |
| hostNetwork mode | https://kubernetes.io/docs/concepts/workloads/pods/#pod-networking |
| IngressClass | https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-class |
| Ingress resource | https://kubernetes.io/docs/concepts/services-networking/ingress/ |
| GitHub releases | https://github.com/kubernetes/ingress-nginx/releases |

---

## Cluster Details

| Role | Hostname | IP |
|---|---|---|
| Control Plane | hv-rocky-linux-1 | 192.168.1.98 |

---

## Architecture

NGINX ingress runs in **hostNetwork mode**, binding directly to ports 80 and 443 on the node's IP address. This eliminates the need for MetalLB or NodePorts, giving standard port access on a single-node cluster.

```
Client → https://engine.cropseyit.com (resolves to 192.168.1.98)
              │
         Port 443 directly on hv-rocky-linux-1
              │
         NGINX Ingress Controller (hostNetwork)
              │
         Routes by hostname → app pods
```

> **Why hostNetwork instead of MetalLB:** MetalLB requires multiple nodes for reliable HA and quorum. On a single-node cluster hostNetwork mode is simpler, more reliable, and uses standard ports with no additional components.

---

## Step 1 — Ensure Firewall Ports are Open

> This should already be done in the cluster setup. Verify ports 80 and 443 are open:

```bash
sudo firewall-cmd --list-ports | grep -E '80|443'
```

If not open:
```bash
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --reload
```

---

## Step 2 — Install NGINX Ingress Controller

Install using the bare-metal manifest:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.1/deploy/static/provider/baremetal/deploy.yaml
```

---

## Step 3 — Patch to hostNetwork Mode

Patch the ingress controller deployment to bind directly to the node's ports 80 and 443:

```bash
kubectl patch deployment ingress-nginx-controller -n ingress-nginx --type=json -p='[
  {"op":"add","path":"/spec/template/spec/hostNetwork","value":true},
  {"op":"add","path":"/spec/template/spec/dnsPolicy","value":"ClusterFirstWithHostNet"}
]'
```

Change the service type from LoadBalancer/NodePort to ClusterIP since it's no longer needed for external access:

```bash
kubectl patch svc ingress-nginx-controller -n ingress-nginx \
  -p '{"spec":{"type":"ClusterIP"}}'
```

---

## Step 4 — Verify Installation

```bash
# Check controller is running and showing node IP
kubectl get pods -n ingress-nginx -o wide

# Check IngressClass
kubectl get ingressclass

# Test HTTP (expects 404 — no ingress rules yet, but NGINX is responding)
curl -I http://192.168.1.98

# Test HTTPS
curl -Ik https://192.168.1.98
```

Expected pod output — IP should match the node IP (192.168.1.98):
```
NAME                                      READY   STATUS    IP
ingress-nginx-controller-xxxx-xxxxx       1/1     Running   192.168.1.98
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
spec:
  ingressClassName: nginx
  rules:
  - host: my-app.cropseyit.com
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

Since there is no DNS server, add the hostname to `/etc/hosts` on any machine that needs to reach it:
```
192.168.1.98   my-app.cropseyit.com
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

# Restart NGINX controller (reloads config)
kubectl rollout restart deployment/ingress-nginx-controller -n ingress-nginx
```

---

## Notes

- **hostNetwork mode:** The NGINX pod uses the node's network stack directly. Its IP in `kubectl get pods -o wide` will show the node IP (192.168.1.98), not a pod IP.
- **IngressClass:** Always specify `ingressClassName: nginx` in Ingress resources — required in Kubernetes 1.18+.
- **No MetalLB needed:** hostNetwork gives standard port access without a floating VIP. If you add worker nodes in the future and want HA, consider adding MetalLB at that point.
- **DNS:** Since there is no DNS server, all hostnames must be added to `/etc/hosts` on every machine that needs to reach them — including all nodes, your Mac, and any integrations/sensors.
- **Admission webhook:** The two `Completed` jobs (admission-create/patch) are normal — they run once to set up the validating webhook and exit.
