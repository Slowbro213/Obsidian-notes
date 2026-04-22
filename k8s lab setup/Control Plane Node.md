# Kubernetes Setup Notes

## Initialize Cluster

``` bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

This creates a Kubernetes Cluster and sets this node as the control
plane

------------------------------------------------------------------------

## Configure kubectl (Regular User)

``` bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

------------------------------------------------------------------------

## Notes on VM Setup

I created 4 total vms to simulate worker nodes, but the tutorial is
running on a single node, so ill follow this part. we are going to
untaint the control plane node so it can run application pods, by
default the control plane is tained which means the scheduler wont
schedule pods to it.

``` bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-node/ubuntu untained
```

(Note, although this was shown by the tutorial, it did not work)

------------------------------------------------------------------------

## Install CNI Plugin

``` bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

This is the container network interface which will allow pods to
communicate with each other. I should note the tutorial video used
flannel, the commands page is using calico.

------------------------------------------------------------------------

## Join Nodes to Cluster

This CNI command should have been ran before the following command, but i ran
this command as it was provided by kubeadm to have other nodes join the
cluster:

``` bash
kubeadm join 192.168.1.29:6443 --token pylzhp.mr7pmdl6ndpfkrbq \
        --discovery-token-ca-cert-hash sha256:d13a40f741793a4f8e3367ce7e28ba7cae7365d5d490bce4a66e8d3fec289baa
```

This allowed my nodes to join, but for some reason they were NotReady.
Re-running the CNI installation command fixed this issue.

------------------------------------------------------------------------

## Verify Nodes

``` bash
kubectl get nodes
```

Output:

``` bash
vboxuser@k8s-init:~$ kubectl get nodes
NAME          STATUS   ROLES           AGE    VERSION
k8s-init      Ready    control-plane   29m    v1.29.15
k8s-worker1   Ready    <none>          102s   v1.29.15
k8s-worker2   Ready    <none>          93s    v1.29.15
k8s-worker3   Ready    <none>          80s    v1.29.15
```

------------------------------------------------------------------------

## Verify System Pods

``` bash
kubectl get pods -n kube-system
```

Output:

``` bash
vboxuser@k8s-init:~$ kubectl get pods -n kube-system
NAME                                      READY   STATUS    RESTARTS   AGE
calico-kube-controllers-8d76c5f9b-77bb8   1/1     Running   0          16m
calico-node-cnkff                         1/1     Running   0          2m34s
calico-node-s7rl2                         1/1     Running   0          16m
calico-node-w6lmb                         1/1     Running   0          2m56s
calico-node-w6vdv                         1/1     Running   0          2m47s
coredns-76f75df574-9gr65                  1/1     Running   0          30m
coredns-76f75df574-vwsvm                  1/1     Running   0          30m
etcd-k8s-init                             1/1     Running   0          30m
kube-apiserver-k8s-init                   1/1     Running   0          30m
kube-controller-manager-k8s-init          1/1     Running   0          30m
kube-proxy-9lnxq                          1/1     Running   0          2m34s
kube-proxy-cjpf2                          1/1     Running   0          2m56s
kube-proxy-k6mps                          1/1     Running   0          30m
kube-proxy-ptckw                          1/1     Running   0          2m47s
kube-scheduler-k8s-init                   1/1     Running   0          30m
```

------------------------------------------------------------------------

## Namespace Note

note, after typing kubectl get `<resource>`{=html} , which in this case
is "pods", i need to specify a namespace via " -n kube-system"
