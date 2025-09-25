
# 🚀 Kubeadm Cluster Setup (CentOS)

## 📌 Project Description
This project demonstrates how to set up a self-hosted Kubernetes cluster from scratch using kubeadm on CentOS VMs.

The goal is to learn Kubernetes internals and cluster management without relying on cloud-managed services.

We’ll build a cluster with:
- 1 Master (Control Plane)
- 1 Worker Node

By the end, you’ll have a fully functional Kubernetes cluster capable of running containerized applications, with networking between nodes and a test Nginx deployment.

---

🛠️ Step 1: Prepare the Nodes

Update system packages
```bash
sudo yum update -y
```
Disable swap (Kubernetes requires swap to be off)
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```
Load required kernel modules
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```
```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Configure sysctl for networking
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```
```bash
sudo sysctl --system
```
---

🛠️ Step 2: Install Container Runtime (containerd)

Install containerd
```bash
sudo yum install -y containerd
```
Generate default config
```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```
Set systemd as cgroup driver
```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```
Restart & enable containerd
```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```
---

🛠️ Step 3: Install Kubernetes Tools

Install dependencies
```bash
sudo yum install -y yum-utils
```
Add Kubernetes repo
```bash
sudo yum-config-manager --add-repo https://pkgs.k8s.io/core:/stable:/v1.30/rpm/kubernetes.repo
```
Install tools
```bash
sudo yum install -y kubelet kubeadm kubectl
```
Enable kubelet service
```bash
sudo systemctl enable --now kubelet
```
Prevent auto-updates
```bash
sudo yum-mark hold kubelet kubeadm kubectl
```
---

🛠️ Step 4: Initialize the Control Plane (Master Node Only)

Initialize cluster
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
⚠️ Save the kubeadm join command printed at the end — you’ll need it for workers.

Configure kubectl for normal user
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Verify master status
```bash
kubectl get nodes
```
---

🛠️ Step 5: Install CNI Network Plugin (Flannel)
```bash
Install Flannel
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```
Verify networking
```bash
kubectl get pods -n kube-system
kubectl get nodes
```
---

🛠️ Step 6: Join Worker Node(s)

Run the kubeadm join command (from Step 4) on each worker:
sudo kubeadm join <MASTER_IP>:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>

Back on master, check node status
```bash
kubectl get nodes
```
---

🛠️ Step 7: Test the Cluster

Deploy Nginx
```bash
kubectl create deployment nginx --image=nginx
```
Expose Nginx as a service
```bash
kubectl expose deployment nginx --port=80 --type=NodePort
```
Get service info
```bash
kubectl get svc
```
➡️ Access Nginx via:
http://<NodeIP>:<NodePort>

---

✅ Step 8: Next Steps (For Production)

Add multiple master nodes (High Availability)
- Configure more than one control plane node.

Configure monitoring & logging
- Use Prometheus + Grafana for monitoring.
- Use EFK stack (Elasticsearch, Fluentd, Kibana) for logging.

Apply RBAC & security policies
- Define roles and permissions.
- Use PodSecurityPolicies or OPA Gatekeeper.

Setup backup strategy for etcd
- Regularly back up etcd database for disaster recovery.
