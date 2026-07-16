# hv-k8s-metallb.md

# MetalLB Load Balancer Setup — Kubernetes on Rocky Linux 9.8

## Reference Documentation

| Topic | URL |
|---|---|
| MetalLB main docs | https://metallb.universe.tf/ |
| Installation guide | https://metallb.universe.tf/installation/ |
| Layer 2 mode | https://metallb.universe.tf/concepts/layer2/ |
| IP address pools | https://metallb.universe.tf/configuration/#defining-the-ips-to-assign-to-the-load-balancer-services |
| L2Advertisement | https://metallb.universe.tf/configuration/#layer-2-configuration |
| GitHub releases | https://github.com/metallb/metallb/releases |
| Kubernetes LoadBalancer services | https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer |

---

## Cluster Details

| Role | Hostname | IP |
|---|---|---|
| Control Plane | hv-rocky-linux-1 | 192.168.1.98 |
| Worker | hv-rocky-linux-2 | 192.168.1.99 |
| Worker | hv-rocky-linux-3 | 192.168.1.100 |

**MetalLB VIP:** `192.168.1.110` (floating across all 3 nodes in Layer 2 HA mode)

---

## How Layer 2 HA Works

In Layer 2 mode MetalLB elects one node as the **speaker** for the VIP. That node answers ARP requests for `192.168.1.110` so your LAN routes traffic to it. If that node goes down, another speaker takes over automatically within seconds.

```
Client → 192.168.1.110:80 or :443
              │
         MetalLB speaker (active node — any of the 3)
              │  ARP floats to next node on failure
         NGINX ingress controller
              │
         Your app pods
```

All three nodes run a MetalLB speaker pod (DaemonSet) — only one speaks at a time but all are ready to take over.

---

## Installation

> Reference: https://metallb.universe.tf/installation/

### Step 1 — Install MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
```

### Step 2 — Wait for pods to be ready

```bash
kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s

kubectl get pods -n metallb-system
```

Expected — controller + one speaker per node:
```
NAME                          READY   STATUS
controller-8694df9d9b-ddpdp   1/1     Running
speaker-hkw47                 1/1     Running   # hv-rocky-linux-1
speaker-pqltl                 1/1     Running   # hv-rocky-linux-2
speaker-xcd5x                 1/1     Running   # hv-rocky-linux-3
```

### Step 3 — Open firewall ports (All Nodes)

MetalLB requires pod/service CIDR traffic and its own ports to be allowed:

```bash
# Allow pod and service CIDR traffic
sudo firewall-cmd --permanent --add-rich-rule='rule family=ipv4 source address=10.244.0.0/16 accept'
sudo firewall-cmd --permanent --add-rich-rule='rule family=ipv4 source address=10.96.0.0/12 accept'

# MetalLB speaker ports
sudo firewall-cmd --permanent --add-port=7472/tcp
sudo firewall-cmd --permanent --add-port=7946/tcp
sudo firewall-cmd --permanent --add-port=7946/udp

# Trust Flannel interfaces (required for webhook connectivity)
sudo firewall-cmd --permanent --zone=trusted --add-interface=flannel.1
sudo firewall-cmd --permanent --zone=trusted --add-interface=cni0

sudo firewall-cmd --reload
```

### Step 4 — Configure IP Address Pool

> Reference: https://metallb.universe.tf/configuration/#layer-2-configuration

```bash
kubectl apply -f - <<'EOF'
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: hv-lab-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.110/32
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: hv-lab-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - hv-lab-pool
EOF
```

### Step 5 — Patch NGINX Ingress to LoadBalancer

```bash
kubectl patch svc ingress-nginx-controller -n ingress-nginx \
  -p '{"spec":{"type":"LoadBalancer"}}'
```

---

## Installed State

### MetalLB IP Pool

```
NAME          AUTO ASSIGN   ADDRESSES
hv-lab-pool   true          ["192.168.1.110/32"]
```

### NGINX Ingress Service

```
NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)
ingress-nginx-controller   LoadBalancer   10.96.86.159   192.168.1.110   80:30966/TCP, 443:32537/TCP
```

The NGINX ingress controller is now reachable at:

| Protocol | URL |
|---|---|
| HTTP | `http://192.168.1.110` |
| HTTPS | `https://192.168.1.110` |

---

## Verify

```bash
# Check MetalLB pods
kubectl get pods -n metallb-system

# Check IP pool
kubectl get ipaddresspool -n metallb-system

# Check NGINX service has external IP
kubectl get svc -n ingress-nginx

# Test HTTP (expects 404 — no ingress rules yet, but NGINX is responding)
curl -I http://192.168.1.110
```

---

## Useful Commands

```bash
# See which node is currently speaking for the VIP
kubectl get l2advertisement -n metallb-system -o wide

# Check MetalLB controller logs
kubectl logs -n metallb-system deployment/controller

# Check speaker logs (on active node)
kubectl logs -n metallb-system daemonset/speaker

# List all LoadBalancer services
kubectl get svc -A --field-selector spec.type=LoadBalancer
```

---

## Notes

- **Layer 2 is not true load balancing** — one node handles all inbound traffic for the VIP at a time. NGINX ingress then distributes across pods. For true load balancing across nodes use BGP mode (requires a BGP-capable router).
- **Failover time** is typically 10–30 seconds in Layer 2 mode — the time for ARP to re-announce on a new node after failure.
- **192.168.1.110 must be outside your DHCP pool** — if your DHCP server assigns it to another device you'll get an IP conflict.
- **Additional IPs**: To add more VIPs later, expand the `addresses` list in the `IPAddressPool` (e.g. `192.168.1.110-192.168.1.120`).
