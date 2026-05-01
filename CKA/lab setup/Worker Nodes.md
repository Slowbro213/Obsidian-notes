## Worker Node Preparation

Worker nodes were clones of an early version of the control plane node,
this version had the kernel modifications, the kubectl, kubelet, kubeadm
and container runtime installed.

------------------------------------------------------------------------

## Initialize Control Plane

After setting up the kubernetes cluster via:

``` bash
sudo kubeadm init --pod-network-cidr=10.244.0.0 --apiserver-advertise-address=<control-plane-private-ip>
```

(in my case the control-plane-private-ip was 192.168.1.29)

------------------------------------------------------------------------

## Join Worker Nodes to Cluster

I ran this command to join each worker node to the cluster:

``` bash
kubeadm join 192.168.1.29:6443 --token pylzhp.mr7pmdl6ndpfkrbq \
        --discovery-token-ca-cert-hash sha256:d13a40f741793a4f8e3367ce7e28ba7cae7365d5d490bce4a66e8d3fec289baa
```
