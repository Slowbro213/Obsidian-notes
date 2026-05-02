# Kubernetes Node Preparation Notes

## Load Required Kernel Modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

**Purpose:**
- Creates a configuration file
- Ensures required kernel modules load automatically on boot
- Provides both persistence and immediate activation support

### Manually Load Modules

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

**Explanation:**
- Activates the modules immediately without reboot

### Module Details

- **overlay**
  - Storage driver used by modern container runtimes (CRIs)
  - Uses OverlayFS:
    - Read-only base layer
    - Writable layers on top
  - Enables efficient disk usage

- **br_netfilter**
  - Enables network packet processing through Linux bridges
  - Required for Kubernetes networking

---

## Configure Network (sysctl)

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

**Purpose:**
- Configures networking for Kubernetes
- Required for kube-proxy to manage traffic via iptables

### Apply Settings

```bash
sudo sysctl --system
```

---

## Configure Containerd (CRI)

> kubelet and container runtime must use the same cgroup driver

### Create Config Directory

```bash
sudo mkdir -p /etc/containerd
```

### Generate Default Config

```bash
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

### Enable Systemd Cgroup Driver

```bash
sudo sed -i "s/SystemCgroup = false/SystemCgroup = true/" /etc/containerd/config.toml
```

**Explanation:**
- Switches containerd to use systemd cgroups
- Required for proper resource management

### Restart and Enable

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

## Disable Swap

```bash
sudo swapoff -a
```

### Make Persistent

```bash
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

**Reason:**
- Kubernetes requires swap to be disabled

---

## Install Kubernetes Components

### Install Dependencies

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

### Add Kubernetes Repository

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### Install Packages

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
```

### Prevent Auto Updates

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```
