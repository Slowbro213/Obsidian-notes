Firstly, on the control plane node, the following commands are run.

The first one unholds the kubeadm package, which will allow us to update the version:

```bash
sudo apt-mark unhold kubeadm
```

The following updates the current version of kubeadm:

```bash
 sudo apt-get update && sudo apt-get install -y kubeadm='1.29.1-1.1'
```

Then we go back to holding the package to prevent automatic updates:

```bash
sudo apt-mark hold kubeadm
```

After upgrading the package, we first run a command to plan the upgrade of the cluster:

```bash
 sudo kubeadm upgrade plan
```

And we Execute!

```bash
sudo kubeadm upgrade apply v1.29.1
```

Now we need to update the kubelet and the kube controller

```bash
 sudo apt-mark unhold kubelet kubectl
 sudo apt-get update && sudo apt-get install -y kubelet='1.29.1-1.1' kubectl='1.29.1-1.1'
 sudo apt-mark hold kubelet kubectl
 #Restart the daemon and kubelet services
 sudo systemctl daemon-reload
 sudo systemctl restart kubelet
```


Now we need to evict the worker nodes. Lets first get all worker nodes:

```bash
vboxuser@k8s-init:~$ kubectl get nodes -o wide
NAME          STATUS   ROLES           AGE   VERSION    INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
k8s-init      Ready    control-plane   78m   v1.29.15   192.168.1.29   <none>        Ubuntu 24.04.3 LTS   6.8.0-110-generic   containerd://2.2.1
k8s-worker1   Ready    <none>          50m   v1.29.15   192.168.1.30   <none>        Ubuntu 24.04.3 LTS   6.8.0-110-generic   containerd://2.2.1
k8s-worker2   Ready    <none>          50m   v1.29.15   192.168.1.31   <none>        Ubuntu 24.04.3 LTS   6.8.0-110-generic   containerd://2.2.1
k8s-worker3   Ready    <none>          50m   v1.29.15   192.168.1.32   <none>        Ubuntu 24.04.3 LTS   6.8.0-110-generic   containerd://2.2.1
```

Now we need to drain each node one by one ( draining a node just disables scheduling for that node ):

```bash
kubectl drain <node-name> --ignore-daemonsets
```

Concretely

```bash
vboxuser@k8s-init:~$ kubectl drain k8s-worker1 --ignore-daemonsets
node/k8s-worker1 cordoned
Warning: ignoring DaemonSet-managed Pods: kube-system/calico-node-w6lmb, kube-system/kube-proxy-7flz4
node/k8s-worker1 drained
vboxuser@k8s-init:~$ kubectl drain k8s-worker2 --ignore-daemonsets
node/k8s-worker2 cordoned
Warning: ignoring DaemonSet-managed Pods: kube-system/calico-node-w6vdv, kube-system/kube-proxy-25qwz
node/k8s-worker2 drained
vboxuser@k8s-init:~$ kubectl drain k8s-worker3 --ignore-daemonsets
node/k8s-worker3 cordoned
Warning: ignoring DaemonSet-managed Pods: kube-system/calico-node-cnkff, kube-system/kube-proxy-8vfsp
node/k8s-worker3 drained
```

Now, on each node we need to update and upgrade the kubeadm and kubelet packages just like on the control node.

```bash
 sudo apt-mark unhold kubeadm kubelet
 sudo apt-get update
 sudo apt-get install -y kubeadm='1.29.1-1.1' kubelet='1.29.1-1.1'
 sudo apt-mark hold kubeadm kubelet
```

Now we need to upgrade the node configuration for every worker node, we do this with kubeadm:

```bash
sudo kubeadm upgrade node
```

Now lets restart kubelet for every worker node:

```bash
sudo systemctl restart daemon-reload
sudo systemctl restart kubelet
```

Now lets make the worker nodes schedulable again. this is done through the kubectl uncordon <node_name> command:

```bash
vboxuser@k8s-init:~$ kubectl uncordon k8s-worker1
node/k8s-worker1 uncordoned
vboxuser@k8s-init:~$ kubectl uncordon k8s-worker2
node/k8s-worker2 uncordoned
vboxuser@k8s-init:~$ kubectl uncordon k8s-worker3
node/k8s-worker3 uncordoned
```