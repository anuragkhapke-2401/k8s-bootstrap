# Kubernetes Cluster Setup on AWS EC2

> Ubuntu 24.04 · Kubernetes v1.35.3 · Calico CNI v3.27.0 · containerd v2.2.2

## Table of Contents

- [Environment](#environment)
- [Section 1 — AWS Security Groups](#section-1--aws-security-groups)
- [Section 2 — Both Nodes Setup](#section-2--both-nodes-setup)
- [Section 3 — Master Node Only](#section-3--master-node-only)
- [Section 4 — Worker Node Only](#section-4--worker-node-only)
- [Section 5 — Verify Cluster](#section-5--verify-cluster)
- [Section 6 — Complete Scripts](#section-6--complete-scripts)
- [Section 7 — Troubleshooting](#section-7--troubleshooting)
- [Section 8 — kubectl Cheatsheet](#section-8--kubectl-cheatsheet)

---

## Environment

| Component | Detail |
|---|---|
| OS | Ubuntu 24.04.4 LTS |
| Kernel | Linux 6.17.0-1007-aws x86-64 |
| Kubernetes | v1.35.3 |
| Container Runtime | containerd v2.2.2 |
| CNI Plugin | Calico v3.27.0 |
| Master Node | t3.small — 2 vCPU, 2 GB RAM |
| Worker Node | t3.micro — 1 vCPU, 1 GB RAM |

> **Note:** t3.micro has only ~497 MB available RAM. Suitable for learning — keep workloads light.

---

## Section 1 — AWS Security Groups

Create two separate security groups before launching instances.

### Master SG (`k8s-master-sg`) — Inbound Rules

| Port | Protocol | Source | Purpose |
|---|---|---|---|
| 22 | TCP | Your IP | SSH access |
| 6443 | TCP | k8s-worker-sg | kube-apiserver |
| 2379–2380 | TCP | k8s-master-sg | etcd (self only) |
| 10250 | TCP | k8s-worker-sg | kubelet API |
| 10257 | TCP | k8s-master-sg | kube-controller-manager |
| 10259 | TCP | k8s-master-sg | kube-scheduler |
| All traffic | All | k8s-worker-sg | Calico CNI pod traffic |

### Worker SG (`k8s-worker-sg`) — Inbound Rules

| Port | Protocol | Source | Purpose |
|---|---|---|---|
| 22 | TCP | Your IP | SSH access |
| 10250 | TCP | k8s-master-sg | kubelet API |
| 10256 | TCP | k8s-master-sg | kube-proxy health check |
| 30000–32767 | TCP | 0.0.0.0/0 | NodePort services |
| All traffic | All | k8s-master-sg | Calico CNI pod traffic |
| All traffic | All | k8s-worker-sg | Calico CNI pod traffic (self) |

> **Outbound:** Allow ALL traffic (`0.0.0.0/0`) on **both** security groups.  
> The CNI All-traffic rules are critical — without them pods cannot communicate across nodes.

---

## Section 2 — Both Nodes Setup

> **SSH into master-node AND worker-node. Run every command in this section on both before moving to Section 3.**

### Step 1 — Install containerd

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
sudo apt install -y containerd.io
# Do NOT install docker-ce — k8s only needs containerd
```

### Step 2 — Configure containerd for Kubernetes CRITICAL

> Docker's default containerd uses `cgroupfs`. Kubernetes requires **systemd** cgroup driver. Missing this step breaks `kubeadm init`.

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

# Switch to systemd cgroup driver
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Verify — must print: SystemdCgroup = true
grep SystemdCgroup /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

### Step 3 — Disable Swap

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Verify — Swap row must show 0B
free -h
```

### Step 4 — Load Required Kernel Modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Verify — both must appear
lsmod | grep -E 'overlay|br_netfilter'
```

### Step 5 — Configure sysctl Networking

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system

# Verify — both must return = 1
sysctl net.ipv4.ip_forward
sysctl net.bridge.bridge-nf-call-iptables
```

### Step 6 — Install kubeadm, kubelet, kubectl v1.35

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl   # prevent accidental upgrades
sudo systemctl enable --now kubelet
```

### Verification Checklist — Run on BOTH Nodes

```bash
sudo systemctl status containerd --no-pager      # active (running)
grep SystemdCgroup /etc/containerd/config.toml   # SystemdCgroup = true
free -h                                           # Swap: 0B
lsmod | grep -E 'overlay|br_netfilter'           # both modules present
sysctl net.ipv4.ip_forward                        # = 1
kubeadm version                                   # v1.35.x
```

---

## Section 3 — Master Node Only

### Step 7 — Get Master Private IP

> Simple `curl` to `169.254.169.254` returns empty on newer EC2. Always use the TOKEN method below (IMDSv2).

```bash
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

MASTER_PRIVATE_IP=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/local-ipv4)

echo "Master IP: $MASTER_PRIVATE_IP"
# Expected: Master IP: 172.31.xx.xx

# Fallback if above returns empty:
MASTER_PRIVATE_IP=$(hostname -I | awk '{print $1}')
echo $MASTER_PRIVATE_IP
```

### Step 8 — Initialize the Cluster

```bash
sudo kubeadm init \
  --apiserver-advertise-address=$MASTER_PRIVATE_IP \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket unix:///var/run/containerd/containerd.sock \
  --kubernetes-version=1.35.3
# Takes 2-3 minutes
```

> **IMPORTANT:** Save the `kubeadm join` command printed at the bottom!  
> `pod-network-cidr=192.168.0.0/16` is required for Calico — do not change this value.

### Step 9 — Set Up kubectl Access

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify (NotReady is normal here — CNI not installed yet)
kubectl get nodes
```

### Step 10 — Install Calico CNI

```bash
kubectl create -f \
  https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml

sleep 5

kubectl create -f \
  https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml

# Watch until master-node shows Ready (2-3 minutes)
watch kubectl get nodes
# Press Ctrl+C once Ready
```

**Expected output after Calico is ready:**

```
NAME          STATUS   ROLES           VERSION
master-node   Ready    control-plane   v1.35.3
```

---

## Section 4 — Worker Node Only

### Step 11 — Join the Cluster

> Always add `--cri-socket` to the join command. Without it, kubeadm gets confused when Docker is installed alongside containerd.

```bash
# Use the join command printed by kubeadm init on master
# Always add --cri-socket at the end
sudo kubeadm join <MASTER_PRIVATE_IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --cri-socket unix:///var/run/containerd/containerd.sock
```

**Token expired? Generate a fresh join command on master:**

```bash
# Run on MASTER to get a new join command
kubeadm token create --print-join-command
# Then append --cri-socket and run on worker with sudo
```

> Tokens expire after **24 hours**. Always regenerate if you see `token expired` errors on the worker.

---

## Section 5 — Verify Cluster (on Master)

```bash
# Both nodes must show Ready
kubectl get nodes -o wide

# All pods must show Running
kubectl get pods -A

# Cluster info
kubectl cluster-info
```

**Expected nodes output:**

```
NAME          STATUS   ROLES           VERSION   INTERNAL-IP     CONTAINER-RUNTIME
master-node   Ready    control-plane   v1.35.3   172.31.40.110   containerd://2.2.2
worker-node   Ready    <none>          v1.35.3   172.31.42.27    containerd://2.2.2
```

**Expected pods — all must show `Running`:**

| Namespace | Pod | Status |
|---|---|---|
| calico-apiserver | calico-apiserver-* (×2) | Running |
| calico-system | calico-kube-controllers-* | Running |
| calico-system | calico-node-* (one per node) | Running |
| calico-system | calico-typha-* | Running |
| calico-system | csi-node-driver-* (one per node) | Running |
| kube-system | coredns-* (×2) | Running |
| kube-system | etcd-master-node | Running |
| kube-system | kube-apiserver-master-node | Running |
| kube-system | kube-controller-manager-master-node | Running |
| kube-system | kube-proxy-* (one per node) | Running |
| kube-system | kube-scheduler-master-node | Running |
| tigera-operator | tigera-operator-* | Running |

---

## Section 6 — Complete Scripts

### Script 1 — `both-nodes.sh` (run on master AND worker)

```bash
#!/bin/bash
set -e  # stop on any error

# Step 1: containerd
sudo apt update && sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
sudo apt update && sudo apt install -y containerd.io

# Step 2: containerd config (CRITICAL)
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
grep SystemdCgroup /etc/containerd/config.toml  # must be true
sudo systemctl restart containerd && sudo systemctl enable containerd

# Step 3: disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Step 4: kernel modules
printf 'overlay\nbr_netfilter\n' | sudo tee /etc/modules-load.d/k8s.conf
sudo modprobe overlay && sudo modprobe br_netfilter

# Step 5: sysctl
printf 'net.bridge.bridge-nf-call-iptables=1\nnet.bridge.bridge-nf-call-ip6tables=1\nnet.ipv4.ip_forward=1\n' \
  | sudo tee /etc/sysctl.d/k8s.conf
sudo sysctl --system

# Step 6: kubeadm kubelet kubectl
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet

echo '===== Both-node setup complete ====='
```

### Script 2 — `master-init.sh` (run on master only)

```bash
#!/bin/bash
set -e

# Get private IP (IMDSv2)
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
MASTER_PRIVATE_IP=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/local-ipv4)
[ -z "$MASTER_PRIVATE_IP" ] && MASTER_PRIVATE_IP=$(hostname -I | awk '{print $1}')
echo "Master IP: $MASTER_PRIVATE_IP"

# kubeadm init
sudo kubeadm init \
  --apiserver-advertise-address=$MASTER_PRIVATE_IP \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket unix:///var/run/containerd/containerd.sock \
  --kubernetes-version=1.35.3

# kubectl config
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Calico CNI
kubectl create -f \
  https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
sleep 5
kubectl create -f \
  https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml

echo '===== Waiting for Ready — run: watch kubectl get nodes ====='
echo '===== Worker join command: ====='
kubeadm token create --print-join-command
```

### Script 3 — `worker-join.sh` (run on worker only)

> Copy the join command output from Script 2 and append `--cri-socket`.  
> Never reuse old tokens — regenerate with: `kubeadm token create --print-join-command`

```bash
#!/bin/bash
# Paste the join command from master here + add --cri-socket
sudo kubeadm join <MASTER_IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --cri-socket unix:///var/run/containerd/containerd.sock
```

---

## Section 7 — Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| Node stuck `NotReady` | CNI not installed | Install Calico (Step 10) |
| `kubeadm init` cgroup error | `SystemdCgroup = false` | Set `true` in `/etc/containerd/config.toml`, restart containerd |
| Worker join: not root | Missing `sudo` | Add `sudo` before `kubeadm join` |
| Worker can't reach 6443 | Security group missing rule | Add TCP 6443 from `k8s-worker-sg` to `k8s-master-sg` |
| Pods stuck `Pending` | Worker not joined yet | Run `kubeadm join` on worker |
| Token expired on join | Tokens expire after 24h | Run `kubeadm token create --print-join-command` on master |
| `MASTER_PRIVATE_IP` empty | IMDSv2 required | Use TOKEN method (Step 7) or `hostname -I \| awk '{print $1}'` |
| containerd not starting | Bad `config.toml` | Re-run: `containerd config default \| sudo tee /etc/containerd/config.toml` |
| Calico pods `CrashLoopBackOff` | Wrong pod CIDR | Must use `192.168.0.0/16` — reset and reinit if wrong |

### Debug Commands

```bash
# kubelet logs (most useful for init failures)
sudo journalctl -xeu kubelet | tail -50

# containerd status
sudo systemctl status containerd

# node details and conditions
kubectl describe node <node-name>

# pod logs
kubectl logs <pod-name> -n <namespace>

# calico pod status
kubectl get pods -n calico-system

# reset a node and start over
sudo kubeadm reset
sudo rm -rf /etc/cni/net.d $HOME/.kube
```

---

## Section 8 — kubectl Cheatsheet

| Task | Command |
|---|---|
| List all nodes with IPs | `kubectl get nodes -o wide` |
| List all pods | `kubectl get pods -A` |
| Deploy an app | `kubectl create deployment nginx --image=nginx --replicas=2` |
| Expose as NodePort | `kubectl expose deployment nginx --port=80 --type=NodePort` |
| Get service + port | `kubectl get svc` |
| Scale deployment | `kubectl scale deployment nginx --replicas=3` |
| Rolling update image | `kubectl set image deployment/nginx nginx=nginx:latest` |
| Watch rollout | `kubectl rollout status deployment/nginx` |
| Delete deployment | `kubectl delete deployment nginx` |
| Describe a node | `kubectl describe node worker-node` |
| Get pod logs | `kubectl logs <pod-name>` |
| Exec into pod | `kubectl exec -it <pod-name> -- /bin/bash` |
| Apply YAML | `kubectl apply -f file.yaml` |
| Delete YAML | `kubectl delete -f file.yaml` |
| Get all resources | `kubectl get all -A` |
| Regenerate join command | `kubeadm token create --print-join-command` |
| Reset a node | `sudo kubeadm reset` |
| Check component health | `kubectl get componentstatuses` |

---

>  Cluster setup complete. Both nodes Ready. Kubernetes v1.35.3 running on Ubuntu 24.04 with Calico CNI.
