# kubeadm-cluster

This project demonstrates how to set up a self-hosted Kubernetes cluster from scratch using **kubeadm** on two CentOS (or Linux) VMs — one as a master (control plane) and the other as a worker. The goal is to understand Kubernetes internals and cluster management without relying on managed services. By the end, you’ll have a fully functional cluster capable of running containerized applications, complete with networking between nodes and a test deployment.

---

## Step 0: Prerequisites

- 2 or more Linux servers (VMs or bare-metal)  
  - 1 Master (Control Plane)  
  - 1 Worker  
- At least 2 CPUs & 2GB RAM per node  
- Root or sudo access  
- Nodes can communicate over private IPs  
- Basic Linux/Docker knowledge  

---

## Step 1: Prepare the Nodes

Update packages:

```bash
sudo yum update -y
Disable swap:

bash
Copy code
sudo swapoff -a
bash
Copy code
sudo sed -i '/ swap / s/^/#/' /etc/fstab
Load required kernel modules:

bash
Copy code
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
bash
Copy code
sudo modprobe overlay
bash
Copy code
sudo modprobe br_netfilter
Configure sysctl:

bash
Copy code
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
bash
Copy code
sudo sysctl --system
Step 2: Install Container Runtime (containerd)
Install containerd:

bash
Copy code
sudo yum install -y containerd
Generate default config:

bash
Copy code
sudo mkdir -p /etc/containerd
bash
Copy code
containerd config default | sudo tee /etc/containerd/config.toml
Set systemd cgroup driver:

bash
Copy code
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
Restart and enable containerd:

bash
Copy code
sudo systemctl restart containerd
bash
Copy code
sudo systemctl enable containerd
Step 3: Install Kubernetes Tools
Install yum-utils:

bash
Copy code
sudo yum install -y yum-utils
Add Kubernetes repo:

bash
Copy code
sudo yum-config-manager --add-repo https://pkgs.k8s.io/core:/stable:/v1.30/rpm/kubernetes.repo
Install kubelet, kubeadm, kubectl:

bash
Copy code
sudo yum install -y kubelet kubeadm kubectl
Enable kubelet:

bash
Copy code
sudo systemctl enable --now kubelet
Prevent auto-updates:

bash
Copy code
sudo yum-mark hold kubelet kubeadm kubectl
Step 4: Initialize the Control Plane (Master)
bash
Copy code
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
Configure kubectl for your user:

bash
Copy code
mkdir -p $HOME/.kube
bash
Copy code
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
bash
Copy code
sudo chown $(id -u):$(id -g) $HOME/.kube/config
Check nodes:

bash
Copy code
kubectl get nodes
Master will be NotReady until networking is installed.

Step 5: Install CNI Network Plugin (Flannel)
bash
Copy code
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
Verify:

bash
Copy code
kubectl get pods -n kube-system
bash
Copy code
kubectl get nodes
Master should now be Ready.

Step 6: Join Worker Node(s)
On each worker node:

bash
Copy code
sudo kubeadm join <MASTER_IP>:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
Verify from master:

bash
Copy code
kubectl get nodes
Step 7: Test the Cluster
Deploy Nginx:

bash
Copy code
kubectl create deployment nginx --image=nginx
Expose service:

bash
Copy code
kubectl expose deployment nginx --port=80 --type=NodePort
Check service:

bash
Copy code
kubectl get svc
Access via <NodeIP>:<NodePort>.

Step 8: Next Steps / Notes
Ideal for learning, testing, and demos.

For production:

Consider HA master nodes

Setup monitoring & logging

Implement security hardening

Backup etcd regularly

Notes for Users
✅ Key Points
Use normal Markdown for explanations.

Use separate triple-backtick blocks for each command.

Use bash after the triple backticks so GitHub shows syntax highlighting and the copy button.  so i have to copy this and put this in file called readme and puh it from my centos to repo

