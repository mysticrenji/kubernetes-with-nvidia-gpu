# Kubernetes Single-Node Cluster with NVIDIA GPU Support

This guide provides step-by-step instructions to set up a single-node Kubernetes cluster with NVIDIA GPU support on Ubuntu 25.

## Prerequisites

- Ubuntu 25 with NVIDIA GPU (tested on 4GB+ GPUs)
- Root or sudo access
- Internet connectivity for downloading packages

---

## Step 1: Host System Preparation

Prepare your Ubuntu system for Kubernetes and GPU workloads.

### Update System

```bash
sudo apt-get update && sudo apt-get upgrade -y
```

### Disable Swap

Kubernetes requires swap to be disabled:

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### Load Required Kernel Modules

Enable networking modules required by Kubernetes:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

### Configure Kubernetes Sysctl Parameters

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

---

## Step 2: Install NVIDIA Host Driver

The GPU Operator requires a pre-installed NVIDIA driver.

### Install Driver

```bash
sudo apt-get install -y nvidia-driver-535
```

**⚠️ Critical:** You must reboot after this step for the driver to load properly.

```bash
sudo reboot
```

### Verify Driver Installation

After rebooting, verify the driver is loaded:

```bash
nvidia-smi
```

You should see your GPU information displayed. **Do not proceed** until `nvidia-smi` works correctly.

---

## Step 3: Install Container Runtime (containerd)

Kubernetes requires a container runtime. We'll use containerd with NVIDIA support for GPU workloads.

### Install containerd and Dependencies

Install required packages and containerd:

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
sudo apt-get install -y containerd
```

### Enable containerd Service

```bash
sudo systemctl enable containerd
sudo systemctl start containerd
```

### Verify containerd Installation

```bash
containerd --version
```

---

## Step 4: Configure containerd & Install NVIDIA Container Toolkit

### Configure containerd for Kubernetes

```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

### Install NVIDIA Container Toolkit

Add the NVIDIA Container Toolkit repository and GPG key:

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

Install the toolkit:

```bash
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
```

### Configure containerd for NVIDIA Runtime

```bash
sudo nvidia-ctk runtime configure --runtime=containerd
```

### Restart containerd

```bash
sudo systemctl restart containerd
```

---

## Step 5: Install Kubernetes Components (kubeadm, kubelet, kubectl)

### Add Kubernetes Repository and GPG Key

```bash
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

### Add Kubernetes v1.34 Repository

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### Install Kubernetes Packages

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# Hold packages at current version to prevent automatic updates
sudo apt-mark hold kubelet kubeadm kubectl
```

### Enable kubelet Service

```bash
sudo systemctl enable kubelet
```

---

## Step 6: Initialize Single-Node Kubernetes Cluster

### Initialize the Control Plane

Initialize kubeadm with the Flannel CNI network CIDR:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

This command will:
- Pull Kubernetes v1.34 images
- Configure the control plane components
- Generate kubeconfig files

### Configure kubectl for Current User

Run these commands as your regular (non-root) user:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Verify Node Status

Check your node status (it will show NotReady until networking is installed):

```bash
kubectl get nodes
```
---

## Step 7: Install Container Network Interface (CNI) - Flannel

Install Flannel for pod-to-pod networking:

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Wait for Flannel pods to be ready:

```bash
kubectl get pods -n kube-flannel
```

Your node should now show as `Ready`:

```bash
kubectl get nodes
```

---

## Step 8: Install NVIDIA GPU Operator

The NVIDIA GPU Operator automates GPU support in Kubernetes. It will detect your pre-installed driver.

### Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Add NVIDIA Helm Repository

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update
```

### Install GPU Operator

Install the operator with `driver.enabled=false` since we already installed the driver in Step 2:

```bash
helm install gpu-operator nvidia/gpu-operator \
  -n gpu-operator --create-namespace \
  --set driver.enabled=false
```

---

## Step 9: Verify GPU Support

### Wait for GPU Operator to Deploy

Monitor the GPU Operator pods until all are running:

```bash
watch kubectl get pods -n gpu-operator
```

Press `Ctrl+C` once all pods show as `Running`.

### Check GPU Node Labels

Verify the node has GPU capability labels:

```bash
kubectl describe node | grep nvidia.com/gpu
```

You should see `nvidia.com/gpu: 1` under the **Allocatable** section.

### Run GPU Test Pod

Create and run a test pod to verify GPU access:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cuda-vector-add
spec:
  restartPolicy: OnFailure
  containers:
    - name: cuda-vector-add
      image: "nvidia/cuda:12.1.0-base-ubuntu22.04"
      command: [ "/bin/sh", "-c" ]
      args: [ "nvidia-smi" ]
      resources:
        limits:
          nvidia.com/gpu: 1 # Request 1 GPU
EOF
```

### Check Test Results

Wait for the pod to complete:

```bash
sleep 15
```

Check the pod logs:

```bash
kubectl logs cuda-vector-add
```

If you see the `nvidia-smi` output showing your GPU, your Kubernetes cluster is fully GPU-aware! ✅

---

## Troubleshooting

### GPU Not Detected

1. Verify driver: `nvidia-smi`
2. Check GPU Operator logs: `kubectl logs -n gpu-operator -l app=nvidia-device-plugin-daemonset`
3. Verify containerd configuration: `sudo grep -i nvidia /etc/containerd/config.toml`

### Node Still NotReady

1. Check pod status: `kubectl get pods -n kube-system`
2. Check Flannel: `kubectl get pods -n kube-flannel`
3. View node logs: `kubectl describe node`

### Pod Cannot Access GPU

1. Verify GPU resource is requested in spec: `nvidia.com/gpu: 1`
2. Check resource availability: `kubectl describe node | grep nvidia.com/gpu`
3. Verify GPU Operator is running: `kubectl get pods -n gpu-operator`

---

## Additional Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/overview.html)
- [Ubuntu 25 Release Notes](https://wiki.ubuntu.com/NobleNumbat)