# NetworkPolicy

Act as a firewall for pods, controlling the traffic at the IP and port level.

- By default, all Pods can communicate with all other Pods.
- Requires a CNI plugin, such as Calico, Cilium, or Weave Net.
- Best practice is to deny all traffic and explicitly allow traffic to needed pods.

---

## Default Deny-All Policy

Now let's practice securing a network via network policies. Let's create a default deny-all policy to start.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {} # The empty selector means "select all pods"
  policyTypes:
    - Ingress
```

---

## Create an Nginx Server

Now let's create an nginx server really quick. We've done this a lot, so let's just paste the commands as before:

```bash
kubectl create deployment web-server --image=nginx
kubectl expose deployment web-server --port=80
```

Now, we have created and applied a policy which restricts all ingress traffic to all pods, and created an nginx server. Next we'll test to see if we can connect to the server to see the policy in action. The result should be a failed attempt to connect.

> [!QUESTION]
> So, what I'm seeing is that the request fails at the DNS level, which I'm unsure if that's supposed to happen since that's not what's happening on the tutorial, but for now we'll carry on.

```bash
vboxuser@k8s-init:~$ kubectl run tmp-shell --rm -it --image=busybox -- /bin/sh -c "wget -O- --timeout=2 web-server"
If you don't see a command prompt, try pressing enter.
wget: bad address 'web-server'
Session ended, resume using 'kubectl attach tmp-shell -c tmp-shell -i -t' command when the pod is running
pod "tmp-shell" deleted
```

---

## Allow Web Access Policy

Now let's create an `allow-web-access` policy:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-access
spec:
  podSelector:
    matchLabels:
      app: web-server
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              access: "true"
      ports:
        - protocol: TCP
          port: 80
```

```bash
vboxuser@k8s-init:~$ kubectl apply -f allow-web-access.yaml
networkpolicy.networking.k8s.io/allow-web-access created
```

The above policy allows ingress from all pods that have the label `access=true` to all pods that belong to the `web-server` service, which all have the label `app=web-server`.

So now if we run the test again, but this time give the temporary container the label `access=true`, it should, according to the tutorial, connect to the web server. I suspect this will not work and it will fail at the DNS level again, but it's worth a try:

```bash
vboxuser@k8s-init:~$ kubectl run tmp-shell --rm -it --image=busybox --labels=access=true -- /bin/sh -c "wget -O- --timeout=2 web-server"
If you don't see a command prompt, try pressing enter.
wget: download timed out
Session ended, resume using 'kubectl attach tmp-shell -c tmp-shell -i -t' command when the pod is running
pod "tmp-shell" deleted
```

It seems it did indeed fail, but the message is not what I expected. If my suspicion is right and the default policy is blocking ingress to the DNS service, then editing the deny policy to allow ingress to the CoreDNS service should resolve this problem.

---

## Troubleshooting

So, after trying that I was wrong. I now suspect I might have set up the cluster wrong and service communication between nodes might not be working. I'll try to remove the network policies just to confirm they're not what is causing the issue:

```bash
vboxuser@k8s-init:~$ kubectl delete netpol default-deny-ingress
networkpolicy.networking.k8s.io "default-deny-ingress" deleted
vboxuser@k8s-init:~$ kubectl delete netpol allow-web-access
networkpolicy.networking.k8s.io "allow-web-access" deleted
```

> [!IMPORTANT]
> Okay, after extensive retries, I just recreated the entire cluster again. I suspect the issue was due to not specifying the `--apiserver-advertise-address` flag when doing the `sudo kubeadm init` command. Either way, the test worked as in the tutorial!
