# 🚀 Kubernetes Cluster Setup on CentOS with Kubeadm

## 📌 Prerequisites
* I used CentOS VMs
1. 2 or more Linux servers (VMs or bare-metal).
2. 1 Control plane (master) node, 1 or more Worker nodes.
3. At least 2 CPUs and 2 GB RAM per node (for testing).
4. Root or sudo access.
5. Ensure all nodes can talk to each other via private IPs.

---

## ⚙️ Step 1: Prepare the Nodes (On all nodes)
```bash
# Update packages:
sudo yum update -y

# Disable swap (Kubernetes does not support swap):
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Load kernel modules:
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Set sysctl params:
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```

---

## 🐳 Step 2: Install Container Runtime
```bash
sudo yum install -y yum-utils device-mapper-persistent-data lvm2

# Add Docker repo (includes containerd)
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# Install containerd
sudo yum install -y containerd.io

# Configure containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Set Cgroup driver = systemd
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

## ⬇️ Step 3: Install Kubernetes Tools
```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
EOF

sudo yum install -y kubelet kubeadm kubectl
sudo systemctl enable kubelet
```

---

## 🖥️ Step 4: Initialize the Control Plane (Master Node)
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

The `--pod-network-cidr` specifies the subnet for Pods. (Needed by most CNI plugins like Flannel.)

After a few minutes, you should see a success message with a `kubeadm join` command. Save it — you’ll use it on workers.  
Example:
```bash
kubeadm join 192.168.182.131:6443 --token <your-token>   --discovery-token-ca-cert-hash sha256:<your-hash>
```

### Configure kubectl for your user
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify:
```bash
kubectl get nodes
```
You’ll see the master node, but it will be in **NotReady** state until networking is installed.

---

## 🌐 Step 5: Install a Network Plugin (CNI)
Install **Flannel**:
```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

Stop firewall if there is a problem:
```bash
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

Wait a few seconds, then check:
```bash
kubectl get pods -n kube-system
kubectl get nodes
```

Your control plane should now be **Ready**.

---

## 👷 Step 6: Join Worker Nodes
On each worker node, use the join command from step 4. Example:
```bash
kubeadm join 192.168.182.131:6443 --token <your-token>   --discovery-token-ca-cert-hash sha256:<your-hash>
```

---

## 📤 Upload this Guide to GitHub

### 1️⃣ Initialize Git
```bash
git init
```

### 2️⃣ Add README.md
```bash
git add README.md
git commit -m "Kubernetes setup guide with kubeadm on CentOS"
```

### 3️⃣ Connect to GitHub
```bash
git branch -M main
git remote add origin https://github.com/<YourUsername>/<YourRepoName>.git
```

### 4️⃣ Push to GitHub
```bash
git push -u origin main
```
