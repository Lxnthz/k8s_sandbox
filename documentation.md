# On-Prem Kubernetes Simulation

# Environments
- Master =  Ubuntu Desktop LTS
  - 25GB Disk
  - 4GB RAM
  - 1 CPU 2 CORES
- Worker-1 = Ubuntu Server
  - 20GB Disk
  - 2GB RAM
  - 1 CPU 1 CORES (LIGHTWEIGHT WORKLOAD) 
- Worker-2 = Ubuntu Server
  - 20GB Disk
  - 2GB RAM
  - 1 CPU 1 CORES (LIGHTWEIGHT WORKLOAD)

# Test Cases
### Control Plane & Node Health
1. Cluster bring-up → verify master and workers join, kubectl get nodes.
2. Node health → stop/start a worker VM → see pods reschedule.
3. kubectl behavior if API server restarts → restart kube-apiserver on master.

### Pod Scheduling & Workloads
4. Simple Deployment (e.g., nginx with 2 replicas) → check distribution across nodes.
5. ReplicaSet rescheduling → delete one pod, kubelet respawns it.
6. NodeSelector / affinity (very light) → schedule pod only on worker-node1.
7. Resource limits → create pod with requests/limits, ensure scheduling respects resources.

### Networking (CNI & Services)
8. Pod-to-Pod same node → ping/curl between 2 pods on worker-node1.
9. Pod-to-Pod cross node → verify CNI routes traffic worker1 ↔ worker2.
10. ClusterIP Service → expose nginx → curl from another pod.
11. NodePort Service → access from your Windows host browser.
12. Pod down test → kill Pod1 on worker1, Pod2 should still reach replica on worker2.
13. NetworkPolicy (optional) → block traffic between two test pods.

### Storage (Keep Simple, No Heavy PVCs)
14. EmptyDir → test temporary storage inside a pod.
15. HostPath → write a file on host via pod, check persistence.

### Scaling & Updates (Lightweight Only)
16. Rolling update → update nginx image → ensure pods roll one by one.
17. Rollback → roll back to old image.
(HPA is excluded — metrics-server is too heavy for your resources.)

### Resiliency
18. Worker node failure → shut down worker-node1 → pods reschedule to worker-node2.
19. Pod disruption budget (PDB) (optional) → prevent all pods from being deleted at once.

### Security & RBAC (Low Impact)
20. RBAC → create namespace + restricted service account.
21. Secrets → mount a secret into a pod and read it.

### Monitoring & Logging (Very Light)
22. kubectl logs → check logs of a pod.
23. kubectl top pods/nodes (requires metrics-server, might be borderline on RAM — only install if you want).

# Setup Master Node
## 1. Prepare Host OS
- Make sure the system is up-to-date
```bash
sudo apt update && sudo apt upgrade -y
```
- Set hostname to match /etc/hosts
```bash
sudo hostnamectl set-hostname master-node
# On each worker, run with appropriate name:
# sudo hostnamectl set-hostname worker-node1 
# sudo hostnamectl set-hostname worker-node2
```
- Configure /etc/hosts with Static IPs (or DHCP reservation) for each node
```bash
cat << EOF | sudo tee -a /etc/hosts
192.168.10.100 master-node
192.168.10.101 worker-node1
192.168.10.102 worker-node2
EOF
```
- Disable swap
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/' /etc/fstab
```

## 2. Configure Kernel Modules & Sysctl
Load modules:
```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```
Persist modules:
```bash
cat << EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```
Configure sysctl:
```bash
cat << EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```
Apply:
```bash
sudo sysctl --system
```

## 3. Install containerd
```bash
sudo apt install -y containerd

# Generate default config
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Switch to systemd cgroup driver
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Restart and enable containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
```

## 4. Import Kubernetes Binaries
Update the apt package index and install packages needed to use the Kubernetes apt repository:
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
```
Download the public signing key for the Kubernetes package repositories:
```bash
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg # allow unprivileged APT programs to read this keyring
```
Add the appropriate Kubernetes apt repository:
```bash
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list   # helps tools such as command-not-found to work correctly
```
Update apt package index, then install:
```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl # Disable auto-update to prevent bug
```

## 5. Initialize Control Plane (Master Node Only)
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
Set up kubeconfig for kubectl:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 6. Install a Pod Network Add-on (CNI)
- Flannel via kubectl (example):
```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

- Flannel via Helm:
  1. Install Helm Packages Manager:
  ```bash
  curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
  echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
  sudo apt-get update
  sudo apt-get install helm
  ```
  2. Manually Install Flannel:
  ```bash
  # Needs manual creation of namespace to avoid helm error
  kubectl create ns kube-flannel
  kubectl label --overwrite ns kube-flannel pod-security.kubernetes.io/enforce=privileged
  
  helm repo add flannel https://flannel-io.github.io/flannel/
  helm install flannel --set podCidr="10.244.0.0/16" --namespace kube-flannel flannel/flannel
  ```
> **Note:** If you use a different CNI (such as Calico), refer to its official documentation and adjust `--pod-network-cidr` as required.

## 7. (On Master) Get the Join Command for Worker Nodes
```bash
kubeadm token create --print-join-command
```
Copy and run the output command on each worker node.

# Setup Worker Nodes
- Repeat **steps 1-4** (prepare OS, kernel modules, install containerd, import Kubernetes binaries) on each worker node.
- Disable swap and set the correct hostname for each worker.
- Join the cluster by running the `kubeadm join ...` command obtained from the master.

# After All Nodes Are Joined
- On master, verify:
```bash
kubectl get nodes
kubectl get pods -A
```
All nodes should be in Ready state.

# Troubleshooting
- If nodes do not join or remain NotReady, ensure:
  - Networking is correct and all nodes can reach each other
  - CNI plugin is installed and functioning
  - Hostnames and /etc/hosts entries are correct
  - Swap remains disabled
