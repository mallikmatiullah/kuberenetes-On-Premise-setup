#                                                      Setting Up a Kubernetes On-Premise Cluster with kubeadm
**Note:** This guide is intended for development and staging environments only and is not appropriate for production use.


## 1. **Prerequisites**

### **Hardware Requirements**
- **Master Node:**
  - CPU: Minimum 2 cores
  - Memory: Minimum 2 GB RAM (4 GB recommended)
  - Disk: 30 GB or more

- **Worker Node:**
  - CPU: Minimum 1 core
  - Memory: Minimum 1 GB RAM (2 GB recommended)
  - Disk: 30 GB or more

### **OS Requirements**
- **Operating System:**
  - Ubuntu 20.04 or 22.04 LTS
  - Ensure both nodes have the same OS version and kernel version.

- **Network Requirements:**
  - Each node should have a static IP address.
  - Both nodes should be able to communicate with each other over the network.

### **Dependencies:**
  - Disable swap on both nodes (Kubernetes does not support swap).
  - Enable required kernel modules.
  - Firewall should allow necessary ports for Kubernetes (6443, 10250, etc.).

---

## 2. **Step-by-Step Instructions**

### **Step 1: Change Hostnames on Both Nodes**
Changing hostnames ensures that both the master and worker nodes are easily identifiable by their roles.

- **Master Node (example hostname: `master-node`)**
  ```bash
  sudo hostnamectl set-hostname master-node
  ```

- **Worker Node (example hostname: `worker-node`)**
  ```bash
  sudo hostnamectl set-hostname worker-node
  ```

- **Verify hostname changes** (on both nodes):
  ```bash
  hostnamectl
  ```

### **Step 2: Update `/etc/hosts` File on Both Nodes**
This step ensures that the nodes can resolve each other's IP addresses and hostnames.

On both the **Master** and **Worker** nodes, add the following lines to `/etc/hosts`:

```bash
sudo vi /etc/hosts
```

Add these lines, replacing with actual IP addresses:
```
<Master_Node_IP> master-node
<Worker_Node_IP> worker-node
```

### **Step 3: Disable Swap and Load Kernel Modules**
Disabling swap is essential for Kubernetes to function.

- **Disable Swap**:
  ```bash
  sudo sed -i '/ swap / s/^/#/' /etc/fstab
  sudo swapoff -a
  ```
  Confirm if the setting is correct

sudo swapoff -a
sudo mount -a
free -h

- **Enable Kernel Modules**:
  ```bash
  sudo modprobe overlay
  sudo modprobe br_netfilter
  ```

- **Add Kernel Settings**:
  ```bash
  sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
  net.bridge.bridge-nf-call-ip6tables = 1
  net.bridge.bridge-nf-call-iptables = 1
  net.ipv4.ip_forward = 1
  EOF
  ```

- **Apply sysctl settings**:
  ```bash
  sudo sysctl --system
  ```

### **Step 4: Install `containerd` on Both Nodes**

1. **Install prerequisites**:
   ```bash
   sudo apt update
   sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
   ```

2. **Install `containerd`**:
   ```bash
   sudo mkdir -p /etc/containerd
   curl -LO https://github.com/containerd/containerd/releases/download/v1.7.2/containerd-1.7.22-linux-amd64.tar.gz
   sudo tar Cxzvf containerd-1.7.22-linux-amd64.tar.gz -C /usr/local
   ```

3. **Configure `containerd`**:
   ```bash
   containerd config default | sudo tee /etc/containerd/config.toml
   sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
   ```

4. **Set up `containerd` as a service**:
   ```bash
   sudo curl -L https://raw.githubusercontent.com/containerd/containerd/main/containerd.service -o /etc/systemd/system/containerd.service
   sudo systemctl daemon-reload
   sudo systemctl enable --now containerd
   ```

5. **Verify containerd is running**:
   ```bash
   sudo systemctl status containerd
   ```

### **Step 5: Install Kubernetes Components on Both Nodes**

1. **Add Kubernetes apt repository**:
   ```bash
   sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p /etc/apt/keyrings/
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

   ```

2. **Install `kubeadm`, `kubelet`, and `kubectl`**:
   ```bash
   sudo apt update
   sudo apt install -y kubelet kubeadm kubectl
   sudo apt-mark hold kubelet kubeadm kubectl
   ```

3. **Enable and start kubelet**:
   ```bash
   sudo systemctl enable kubelet
   ```

### **Step 6: Initialize the Master Node**

1. **Initialize the Kubernetes Control Plane**:
   ```bash
   sudo kubeadm init --pod-network-cidr=192.168.0.0/16
   ```

2. **Set up `kubectl` for the root user**:
   ```bash
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```

3. **Verify the Master Node is ready**:
   ```bash
   kubectl get nodes
   ```

### **Step 7: Install a Pod Network Add-on (Calico)**

1. **Install Calico**:
   ```bash
   kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
   ```

2. **Verify the network is set up**:
   ```bash
   kubectl get pods -n kube-system
   ```

### **Step 8: Join the Worker Node to the Cluster**

1. **On the Master Node, get the join command**:
   ```bash
   kubeadm token create --print-join-command
   ```

2. **On the Worker Node, run the join command**:
   ```bash
   sudo kubeadm join <master-node-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
   ```

3. **Verify the Worker Node has joined the cluster (on the Master Node)**:
   ```bash
   kubectl get nodes
   ```

---

## 3. **Deprecated Features**
- **Swap usage is deprecated**: Ensure swap is disabled on all nodes.
- **Docker support is deprecated**: Use `containerd` as the container runtime instead of Docker.

---

## 4. **Kubernetes Stable Release**
This guide is compatible with Kubernetes version **1.29** and later stable releases. Ensure to check the Kubernetes release page for any updates.

