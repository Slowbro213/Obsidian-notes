
### Create a namespace called 'mynamespace' and a pod with image nginx called nginx on this namespace

```bash
kubectl create ns mynamespace
kubectl run nginx --image=nginx -n mynamespace
```

### Create the pod that was just described using YAML

```bash
kubectl run nginx --image=nginx --dry-run=client -n mynamespace -o yaml > nginx-pod.yaml
kubectl apply -f nginx-pod.yaml
```

### Create a busybox pod (using kubectl command) that runs the command "env". Run it and see the output

```bash
kubectl run busybox --image=busybox -n mynamespace --command -it --rm -- env
```

The -it flag lets you see the output right away, but you can also remove it and check the logs of the container afterwards ( --rm also needs to be removed since that only works for -it containers):
```bash
laborant@dev-machine:~$ kubectl logs busybox
error: error from server (NotFound): pods "busybox" not found in namespace "default"
laborant@dev-machine:~$ kubectl logs busybox -n mynamespace
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=busybox
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
HOME=/root
```


### Get the YAML for a new namespace called 'myns' without creating it


```bash
kubectl create ns myns --dry-run=client -o yaml
```

### Create the YAML for a new ResourceQuota called 'myrq' with hard limits of 1 CPU, 1G memory and 2 pods without creating it
```bash
kubectl create quota myrq --hard=cpu=1,memory=1G,pods=2 --dry-run=client -o yaml
```
### Get pods on all namespaces
```bash
kubectl get pods -A
```

### Create a pod with image nginx called nginx and expose traffic on port 80

```bash
kubectl run nginx --image=nginx --restart=Never --port=80
```

### Change pod's image to nginx:1.24.0. Observe that the container will be restarted as soon as the image gets pulled
