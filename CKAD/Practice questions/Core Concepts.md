### Create a namespace called `mynamespace` and a pod with image `nginx` called `nginx` on this namespace

```bash
kubectl create ns mynamespace
kubectl run nginx --image=nginx -n mynamespace
```

---

### Create the pod that was just described using YAML

```bash
kubectl run nginx --image=nginx --dry-run=client -n mynamespace -o yaml > nginx-pod.yaml

kubectl apply -f nginx-pod.yaml
```

---

### Create a BusyBox pod using `kubectl` command that runs the command `env`. Run it and see the output

```bash
kubectl run busybox --image=busybox -n mynamespace --command -it --rm -- env
```

> [!NOTE]  
> The `-it` flag lets you see the output immediately, but you can also remove it and inspect the container logs afterwards.
> 
> The `--rm` flag also needs to be removed since it only works for interactive (`-it`) containers.

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

---

### Get the YAML for a new namespace called `myns` without creating it

```bash
kubectl create ns myns --dry-run=client -o yaml
```

---

### Create the YAML for a new `ResourceQuota` called `myrq` with hard limits of 1 CPU, 1G memory and 2 pods without creating it

```bash
kubectl create quota myrq --hard=cpu=1,memory=1G,pods=2 --dry-run=client -o yaml
```

---

### Get pods on all namespaces

```bash
kubectl get pods -A
```

> [!TIP]  
> `-A` is shorthand for:
> 
> ```bash
> --all-namespaces
> ```

---

### Create a pod with image `nginx` called `nginx` and expose traffic on port 80

```bash
kubectl run nginx --image=nginx --restart=Never --port=80 -n myns
```

---

### Change pod's image to `nginx:1.24.0`. Observe that the container will be restarted as soon as the image gets pulled

```bash
kubectl set image pod/nginx nginx=nginx:1.24.0 -n myns

kubectl describe pod nginx -n myns
```

---

### Get nginx pod's IP created in previous step, use a temporary BusyBox image to `wget` its `/`

```bash
kubectl get pod nginx -n myns -o jsonpath={.status.podIP}
```

```bash
10.244.0.3%
```

```bash
kubectl run busybox -n myns --restart=Never --image=busybox -it --rm --command -- wget -O- 10.244.0.3
```

> [!NOTE]  
> If you start an interactive pod without `--restart=Never`, the pod may get stuck because Kubernetes will attempt to restart it after completion.

---

### Get pod's YAML

```bash
kubectl get pod nginx -n myns -o yaml
```

---

### Get information about the pod, including details about potential issues (e.g. pod hasn't started)

```bash
kubectl describe pod nginx -n myns
```

---

### Get pod logs

```bash
kubectl logs nginx -n myns
```

---

### If pod crashed and restarted, get logs about the previous instance

```bash
kubectl logs nginx -n myns -p
```

> [!NOTE]  
> The `-p` flag is used to get logs from the **previous** container instance.

---

### Execute a simple shell on the nginx pod

```bash
kubectl exec -it nginx -n myns -- sh
```

> [!NOTE]  
> Command explained:
> 
> - `exec` → Execute a command inside a container
>     
> - `-it` → Interactive terminal
>     
> - `nginx` → Pod name
>     
> - `-n myns` → Namespace
>     
> - `-- sh` → Start a shell inside the container
>     

---

### Create a BusyBox pod that echoes `hello world` and then exits

```bash
kubectl run busybox -n myns --image=busybox --restart=Never -it --command -- echo 'hello world'
```

---

### Do the same, but have the pod deleted automatically when it's completed

```bash
kubectl run busybox -n myns --image=busybox --restart=Never -it --rm --command -- echo 'hello world'
```

> [!NOTE]  
> The `--rm` flag automatically deletes the pod after the command finishes.

---

### Create an nginx pod and set an env value as `var1=val1`. Check the env value existence within the pod

```bash
kubectl run nginx --image=nginx --restart=Never -n myns --env=var1=val1

pod/nginx created

kubectl exec -it nginx -n myns -- /bin/bash -c 'echo $var1'
```

> [!NOTE]  
> Environment variables can be injected at pod creation time using:
> 
> ```bash
> --env=<key>=<value>
> ```
> 
> and verified from inside the container using:
> 
> ```bash
> echo $<variable>
> ```