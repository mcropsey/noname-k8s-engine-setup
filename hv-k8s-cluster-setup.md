# hv-k8s-cluster-setup.md

# Kubernetes Cluster Setup — Rocky Linux 9.8

## Reference Documentation

| Topic | URL |
|---|---|
| Install kubeadm, kubelet, kubectl | https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/ |
| Create cluster with kubeadm | https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/ |
| Container runtimes (containerd) | https://kubernetes.io/docs/setup/production-environment/container-runtimes/ |
| Flannel CNI | https://github.com/flannel-io/flannel |
| kubeadm join | https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/ |
| kubeadm token | https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-token/ |
| Firewall / ports required | https://kubernetes.io/docs/reference/networking/ports-and-protocols/ |

---

## Cluster Overview

| Role | Hostname | IP |
|---|---|---|
| Control Plane | hv-rocky-linux-1 | 192.168.1.98 |
| Worker | hv-rocky-linux-2 | 192.168.1.99 |
| Worker | hv-rocky-linux-3 | 192.168.1.100 |

- **Kubernetes version:** v1.31.14
- **Container runtime:** containerd 2.2.6
- **CNI:** Flannel (pod CIDR: 10.244.0.0/16)
- **OS:** Rocky Linux 9.8 (Blue Onyx)
- **SSH user:** mcropsey

---

## Step 1 — Prerequisites (All Nodes)

### Disable swap
```bash
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
```

### Add /etc/hosts entries
```bash
sudo tee -a /etc/hosts <<'EOF'
192.168.1.98  hv-rocky-linux-1
192.168.1.99  hv-rocky-linux-2
192.168.1.100 hv-rocky-linux-3
EOF
```

### Load kernel modules
```bash
sudo tee /etc/modules-load.d/k8s.conf <<'EOF'
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```

### Configure sysctl
```bash
sudo tee /etc/sysctl.d/k8s.conf <<'EOF'
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

---

## Step 2 — Install containerd (All Nodes)

> Reference: https://kubernetes.io/docs/setup/production-environment/container-runtimes/

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y containerd.io

# Configure containerd with SystemdCgroup enabled
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl enable --now containerd
```

---

## Step 3 — Install Kubernetes Tools (All Nodes)

> Reference: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

```bash
sudo tee /etc/yum.repos.d/kubernetes.repo <<'EOF'
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/repodata/repomd.xml.key
EOF

sudo dnf install -y kubelet kubeadm kubectl conntrack-tools
sudo systemctl enable kubelet
```

---

## Step 4 — Configure Firewall (All Nodes)

> Reference: https://kubernetes.io/docs/reference/networking/ports-and-protocols/

```bash
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=10251/tcp
sudo firewall-cmd --permanent --add-port=10252/tcp
sudo firewall-cmd --permanent --add-port=30000-32767/tcp
sudo firewall-cmd --permanent --add-port=8472/udp
sudo firewall-cmd --permanent --add-masquerade
sudo firewall-cmd --reload
```

---

## Step 5 — Initialize Control Plane (192.168.1.98 only)

> Reference: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

```bash
sudo kubeadm init \
  --control-plane-endpoint=192.168.1.98 \
  --apiserver-advertise-address=192.168.1.98 \
  --pod-network-cidr=10.244.0.0/16 \
  --cri-socket=unix:///run/containerd/containerd.sock

# Set up kubeconfig
mkdir -p ~/.kube
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
```

---

## Step 6 — Install Flannel CNI (Control Plane only)

> Reference: https://github.com/flannel-io/flannel

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

---

## Step 7 — Join Worker Nodes (192.168.1.99 and 192.168.1.100)

> Reference: https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/

```bash
sudo kubeadm join 192.168.1.98:6443 \
  --token jveovj.47yyr8e45saiytdo \
  --discovery-token-ca-cert-hash sha256:351588d0caec8e5b0ac6567c004c6f51d9685314b0186b0ef08697364201c69d \
  --cri-socket=unix:///run/containerd/containerd.sock
```

> **Note:** Tokens expire after 24 hours. To generate a new join command:
> ```bash
> kubeadm token create --print-join-command
> ```

---

## Verification

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

Expected output:
```
NAME               STATUS   ROLES           AGE   VERSION
hv-rocky-linux-1   Ready    control-plane   ...   v1.31.14
hv-rocky-linux-2   Ready    <none>          ...   v1.31.14
hv-rocky-linux-3   Ready    <none>          ...   v1.31.14
```

---

## Access the Cluster from Your Mac

```bash
scp mcropsey@192.168.1.98:~/.kube/config ~/.kube/hv-k8s-config
export KUBECONFIG=~/.kube/hv-k8s-config
kubectl get nodes
```

---

## Useful Commands

```bash
# Node status
kubectl get nodes -o wide

# All pods
kubectl get pods -A

# Cluster info
kubectl cluster-info

# Generate new worker join command (token expires after 24h)
kubeadm token create --print-join-command

# Check kubelet logs
journalctl -u kubelet -f

# Check containerd
sudo systemctl status containerd
```

---

## Notes

- SELinux was disabled on all nodes prior to cluster setup (`SELINUX=disabled` in `/etc/selinux/config`)
- Swap was disabled permanently (removed from `/etc/fstab`)
- `conntrack-tools` was required on Rocky Linux 9 — not installed by default and blocks `kubeadm init`
- The join token expires after **24 hours** — regenerate with `kubeadm token create --print-join-command` on the control plane if rejoining a node later
