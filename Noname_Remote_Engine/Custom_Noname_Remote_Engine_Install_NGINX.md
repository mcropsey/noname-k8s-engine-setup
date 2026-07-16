# Noname Remote Engine — Kubernetes/Helm Install (NGINX Ingress)

Engine URL: `https://engine.cropseyit.com/engine`

## Cluster Reference

| Node | IP | Role |
|---|---|---|
| hv-rocky-linux-1 | 192.168.1.98 | control-plane (run all kubectl/helm commands here) |

> **Single-node cluster:** This environment runs as a single-node cluster with the control plane taint removed. NGINX ingress runs in hostNetwork mode, binding directly to ports 80 and 443 on the node IP. No MetalLB or floating VIP is used.
>
> **Note:** Always run `kubectl` and `helm` commands from the control-plane node (192.168.1.98) unless stated otherwise. The IP above is specific to this environment — update this table if the cluster is rebuilt or reused elsewhere.

---

## Node Sizing Requirements

> **The engine pod has very high resource requirements. Undersized nodes will cause the engine pod to stay `Pending` indefinitely and the engine will show as Unregistered in the UI.**

| Environment | CPU | RAM | Hyper-V MB |
|---|---|---|---|
| **Lab (minimum)** | 16 vCPU | 48 GB | 49152 |
| **Production** | 16+ vCPU | 64 GB+ | 65536+ |

> **Why 48GB minimum for lab:** The engine (heavy-engine) pod alone requests 7 CPU and 40GB RAM. With OS overhead and other pods consuming memory on the same node, 40GB is not enough — 48GB provides the necessary headroom.
>
> **Lab vs Production:** Production sizing depends on multiple factors including traffic volume, number of workers, and API complexity. Always refer to the official Akamai sizing guide:
> - See the **Engine Sizing** section in `Akamai_Noname_K8s_Install_Guide.md` in this directory
> - Official online reference: [Akamai API Security Engine Sizing](https://docs.nonamesecurity.com/v368/docs/multiple-engine-datasheet#engine-sizing)

In this environment the node is a Hyper-V VM. If the engine pod shows `Pending` with `Insufficient memory` or `Insufficient cpu` in `kubectl describe pod`, shut down the VM and increase RAM and CPU in Hyper-V before proceeding.

---

## Prerequisites

- Single-node k8s cluster running (Rocky Linux 9.8, kubeadm) with Flannel CNI
- Control plane taint removed (`node-role.kubernetes.io/control-plane:NoSchedule-`)
- NGINX Ingress Controller installed in hostNetwork mode, listening on ports 80/443 on 192.168.1.98
- Firewall ports 80 and 443 open on the node
- Node meets minimum sizing requirements above
- Outbound TCP 443 open from the node to:
  - `mwc-lab.nonamesec.com`
  - `mwc-lab-mtls.nonamesec.com`
  - `pkg.nonamesec.com`
  - `us-central1-docker.pkg.dev`

---

## Section 0 — Pre-Install Setup

### Step 1 — Add /etc/hosts on the node and your Mac

Since there is no DNS server, `engine.cropseyit.com` must resolve via `/etc/hosts` on every machine that needs to reach the engine. Point it directly at the node IP.

> **Important:** This entry must be added to **every machine that sends traffic to the remote engine** — including all Akamai integrations (sensors, F5 iRules, API gateways, etc.) that are configured to forward traffic to `engine.cropseyit.com`. If a traffic source cannot resolve the hostname it will fail to connect to the engine regardless of whether the engine itself is running correctly.

Run on the node (192.168.1.98), your Mac, and any Akamai integration/sensor sending traffic to the engine:

```bash
sudo vi /etc/hosts
```

Add at the bottom:
```
192.168.1.98   engine.cropseyit.com
```

---

### Step 2 — Verify IngressClass name

> The rest of this doc assumes the IngressClass name is `nginx`. If your cluster returns a different name you must substitute it everywhere `ingressClassName: nginx` appears in this doc.

> **Note:** Always run `kubectl` commands from the control-plane node. The IP below is specific to this environment — update it if the cluster is rebuilt or reused in a different environment.

```bash
# 192.168.1.98 is the control-plane node in this environment
ssh mcropsey@192.168.1.98
kubectl get ingressclass
```

Expected output:
```
NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       ...
```

> **Important:** If the name is not `nginx`, note it and substitute it everywhere `ingressClassName: nginx` appears in this doc — specifically in Section 1 Step 3 and the reference section below. Using the wrong IngressClass means the NGINX controller will silently ignore the Ingress object and `engine.cropseyit.com` will never get an address.
>
> `ingress-nginx` and `ingress-nginx-controller` are the default names when NGINX ingress is installed using the standard bare-metal manifest. Verify with:
> ```bash
> kubectl get deploy -A | grep -i nginx
> ```

---

### Step 3 — Install StorageClass (local-path-provisioner)

> **Why:** Bare-metal and Hyper-V clusters have no built-in StorageClass. Without one the PVC will stay `Pending` indefinitely. Local-path-provisioner uses local disk on the node and sets itself as the default StorageClass automatically.
>
> **Reference:** Storage size requirements are in the Prerequisites section of `Akamai_Noname_K8s_Install_Guide.md` in this directory — **30 GB recommended, 15 GB minimum** per engine.

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml

# Wait for it to be ready
kubectl wait --namespace local-path-storage \
  --for=condition=ready pod \
  --selector=app=local-path-provisioner \
  --timeout=60s

# Set as default StorageClass
kubectl patch storageclass local-path \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Verify
kubectl get storageclass
# Should show: local-path (default)
```

---

### Step 4 — Create Namespace and PVC

```bash
kubectl create namespace akamai-api-security

kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: noname-engine-pvc
  namespace: akamai-api-security
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 30Gi
EOF

kubectl get pvc -n akamai-api-security
```

> **Note:** `Pending` here is normal — the PVC will not bind until the Helm install in Section 1 starts the engine pod. Once the pod runs, the PVC binds automatically. Only investigate if pods are running but the PVC is still `Pending`:
> ```bash
> kubectl describe pvc noname-engine-pvc -n akamai-api-security
> ```

> **Important:** The local-path-provisioner stores PVC data locally on the node where the pod first runs. If you rebuild the cluster or move the engine to a different node you must delete and recreate the PVC — the data is not portable between nodes.

---

### Step 5 — Install Git and Helm on the control-plane node

> **Do I need to install Helm?**
> - **Already included — nothing to do:** Rancher Desktop, K3s
> - **Need to enable it:** MicroK8s — run `microk8s enable helm3`
> - **Need to install it yourself:** kubeadm (this environment), AWS EKS, Azure AKS, Google GKE, Minikube
>
> **Rule of thumb:** For everything except Rancher Desktop and K3s, assume you need to install Helm.

> **Note:** This is your first real action step. Everything in Section 0 was either prereq setup or reference information. The Noname engine install begins here.

```bash
ssh mcropsey@192.168.1.98

# Install git first — required for Helm plugin support
sudo dnf install -y git

curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

---

## Section 1 — Deploy the Noname Remote Engine

### Step 1 — Create the Engine in the Noname UI

1. Go to **Settings → Engines → Add Engine**
2. Select **Kubernetes → Helm**
3. Enter a name for the engine
4. Set traffic URL to: `https://engine.cropseyit.com/engine`
   - Must include `https://`
   - Must end in `/engine`
5. Click **Submit**

---

### Step 2 — Get install commands from the Noname UI

Go to **Settings → Engines → your engine → Install**. The UI presents 4 steps:

- **UI Step 1** — `helm registry login` command with your token pre-filled
- **UI Step 2** — Download `custom_values.yaml`
- **UI Step 3** — `helm upgrade --install` command
- **UI Step 4** — Optional: download charts for offline review

**From UI Step 2**, download `custom_values.yaml` to your Mac, then copy it to the control-plane node:

```bash
scp custom_values.yaml mcropsey@192.168.1.98:~/custom_values.yaml
```

---

### Step 3 — Edit custom_values.yaml BEFORE installing

> **Do this before the registry login and Helm install** — if you install with the unmodified file it will use the wrong ingress class (`alb`) and the engine will not be reachable.

```bash
vi ~/custom_values.yaml
```

**Change 1 — Fix the ingress block.** The file as downloaded from the UI will have:

```yaml
ingress:
  enabled: true
  hosts:
  - host: "engine.cropseyit.com"
    paths:
    - path: /
      pathType: Prefix
```

Replace it with:

```yaml
ingress:
  enabled: true
  ingressClassName: nginx
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  hosts:
  - host: "engine.cropseyit.com"
    paths:
    - path: /
      pathType: Prefix
```

**Change 2 — Set the PVC name.** Find `existingPVCName: null` and change it to:

```yaml
      existingPVCName: "noname-engine-pvc"
```

**Verify both changes before moving on:**

```bash
grep -A8 'ingress:' ~/custom_values.yaml
grep 'existingPVCName' ~/custom_values.yaml
```

---

### Step 4 — Log in to the Akamai Helm registry

Use the exact command from **UI Step 1** — it has your real token pre-filled. Do not construct it manually.

> - `_json_key_base64` — literal fixed string, type it exactly as shown
> - The password is a base64-encoded GCP service account key generated by the UI
> - **Token expires after 7 days** — if you get a 401 error, download a fresh `custom_values.yaml` from the UI which will contain a new token

```bash
# Copy the exact command from UI Step 1 — it will look like this:
helm registry login \
  --username '_json_key_base64' \
  --password '<your-token-from-ui>' \
  us-central1-docker.pkg.dev/noname-artifacts/nns-docker
```

Expected output: `Login Succeeded`

---

### Step 5 — (Optional) Download charts for review

From **UI Step 4** — skip this if you want to go straight to the install:

```bash
helm pull oci://us-central1-docker.pkg.dev/noname-artifacts/nns-docker/helm/nonamesec --version 'v3.67.6'
```

---

### Step 6 — Install the Remote Engine

Use the command from **UI Step 3**:

```bash
helm upgrade --install akamai-api-security \
  oci://us-central1-docker.pkg.dev/noname-artifacts/nns-docker/helm/nonamesec \
  -n akamai-api-security --create-namespace \
  -f ~/custom_values.yaml \
  --version 'v3.67.6'
```

---

### Step 7 — Watch pods come up

```bash
kubectl get pods -n akamai-api-security -w
```

Wait until all pods show `Running`. Expected pods:

| Pod | Containers |
|---|---|
| engine-* | 1/1 |
| light-engine-* | 2/2 |
| router-* | 1/1 |
| nats-jetstream-* | 1/1 |
| nginx-* | 1/1 |
| nogate-* | 1/1 |
| integrations-adapter-* | 1/1 |

> If pods stay `Pending` check resources and events:
> ```bash
> kubectl describe pod <pod-name> -n akamai-api-security
> ```

---

### Step 8 — Verify ingress and connectivity

```bash
# Should show engine.cropseyit.com with ADDRESS 192.168.1.98 and CLASS nginx
kubectl get ingress -n akamai-api-security -o wide

# Confirm ingressClassName and backend service
kubectl describe ingress -n akamai-api-security

# Test the endpoint (from a machine with the /etc/hosts entry)
curl -vk https://engine.cropseyit.com/engine
# Expect: HTTP 200
```

**Confirm NGINX controller synced the Ingress** — run after pods are up:

```bash
# namespace: ingress-nginx  |  deployment: ingress-nginx-controller  (this environment)
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller | grep -i "akamai-api-security\|engine.cropseyit.com"

kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- \
  cat /etc/nginx/nginx.conf | grep -A 10 "server_name engine.cropseyit.com"
```

---

### Step 9 — Confirm Active in the Noname UI

Go to **Settings → Engines** and confirm the engine shows **Active**.

---

## FOR REFERENCE — What the Ingress object looks like after install

This is created automatically by Helm — you do not create this manually. After Step 6, compare against what was actually deployed:

```bash
kubectl get ingress -n akamai-api-security -o yaml
```

It should look like:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: akamai-api-security-nonamesec
  namespace: akamai-api-security
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: engine.cropseyit.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: akamai-api-security-nginx
                port:
                  number: 443
```

---

## Section 2 — Getting a Trusted TLS Cert (Optional — Future)

Since you own `cropseyit.com` you can get a free auto-renewing cert from Let's Encrypt using cert-manager with DNS-01 validation via Cloudflare. This avoids the self-signed cert warning. Ask when ready to set this up.

---

## Section 3 — Enabling mTLS at the Ingress (Optional — Future)

Authenticates callers (traffic sources/integrations) using client certificates. Ask when ready to set this up.

---

## Troubleshooting Quick Reference

| Symptom | Command | Likely Cause |
|---|---|---|
| Pods stuck `Pending` | `kubectl describe pod <pod> -n akamai-api-security` | Insufficient CPU/RAM, PVC not bound |
| `ImagePullBackOff` | `kubectl describe pod <pod> -n akamai-api-security` | Helm registry token expired (7-day expiry) |
| `CrashLoopBackOff` | `kubectl logs <pod> -n akamai-api-security --previous` | Engine can't reach `mwc-lab.nonamesec.com` |
| Ingress has no address | `kubectl describe ingress -n akamai-api-security` | Wrong `ingressClassName` |
| curl returns 404 | `kubectl get ingress -n akamai-api-security -o wide` | Host/path mismatch |
| curl returns 503 | `kubectl get pods -n akamai-api-security` | NATS or router pod not ready |
| curl returns 502/503 | `kubectl get endpoints -n akamai-api-security` | Backend pod not ready |
| Engine shows Offline in UI | `kubectl logs deployment/engine -n akamai-api-security` | Can't reach `mwc-lab.nonamesec.com:443` |
| Connection refused on 443 | `sudo firewall-cmd --list-ports` | Ports 80/443 not open on node firewall |

---

## Outbound Ports Required from K8s Node

| Destination | Port | Purpose |
|---|---|---|
| mwc-lab.nonamesec.com | TCP 443 | Engine → API Security Platform |
| mwc-lab-mtls.nonamesec.com | TCP 443 | mTLS fallback |
| pkg.nonamesec.com | TCP 443 | Engine updates |
| us-central1-docker.pkg.dev | TCP 443 | Image pulls |
