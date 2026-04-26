Im not going to execute this since its a small n step flow ( n is the number of nodes in the control plane ).

We first need to setup a load balancer. This isn't relevant to Kubernetes( for now at least ) but its required. After the load balancer is up and reachable, we will setup a kubernetes cluster via kubeadm. We are going to use kubeadm init but with one more flag to specify the IP and port of the load balancer:

```bash
 sudo kubeadm init --control-plane-endpoint "load-balancer.example.com:6443" --upload-certs
```

This will create the kubernetes cluster, and will provide us with 2 commands. One to have other nodes join the control plane, and one to have other nodes join as worker nodes. We have seen the worker nodes one before, so here is the control plane node one:

```bash
# EXAMPLE - Use the exact command from your `kubeadm init` output
sudo kubeadm join load-balancer.example.com:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> --control-plane --certificate-key <key>
```

