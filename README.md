
🚀 Kubernetes Cluster with kubeadm (CentOS)
Step 0: Prerequisites

2+ Linux servers (VMs or bare-metal)

1 Master (Control Plane) + 1 Worker

At least 2 CPUs & 2GB RAM per node

Root or sudo access

Nodes must communicate via private IPs

Basic Linux/Docker knowledge

Step 1: Prepare the Nodes (Run on all nodes)
Update system
sudo yum update -y

Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

Load required kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

Configure sysctl
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system

Step 2: Install Container Runtime (containerd)
sudo yum install -y containerd

sudo mkdir -p /etc/containerd

containerd config default | sudo tee /etc/containerd/config.toml

sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd

Step 3: Install Kubernetes Tools
sudo yum install -y yum-utils

sudo yum-config-manager --add-repo https://pkgs.k8s.io/core:/stable:/v1.30/rpm/kubernetes.repo

sudo yum install -y kubelet kubeadm kubectl

sudo systemctl enable --now kubelet

sudo yum-mark hold kubelet kubeadm kubectl

Step 4: Initialize the Control Plane (Master only)
sudo kubeadm init --pod-network-cidr=10.244.0.0/16


⚠️ Save the kubeadm join ... command shown at the end — you’ll need it on the worker nodes.

Configure kubectl:

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


Check nodes:

kubectl get nodes

Step 5: Install CNI Network Plugin (Flannel)
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml


Verify:

kubectl get pods -n kube-system
kubectl get nodes

Step 6: Join Worker Node(s)

On each worker node (use the command copied from Step 4):

sudo kubeadm join <MASTER_IP>:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>


Back on master, verify:

kubectl get nodes

Step 7: Test the Cluster
kubectl create deployment nginx --image=nginx

kubectl expose deployment nginx --port=80 --type=NodePort

kubectl get svc


➡️ Access Nginx via http://<NodeIP>:<NodePort>

Step 8: Next Steps

✅ Great for labs, testing, and demos.
For production:

Use multiple HA master nodes

Add monitoring & logging

Apply security hardening

Backup etcd regularly
