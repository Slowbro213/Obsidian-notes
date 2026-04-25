# Kubernetes Rolling Update Notes (NGINX)

---

## Overview

The default strategy for deploying is a rolling update, where pods are incrementally updated with new ones to ensure your app has no downtime. fileciteturn0file0

---

## 1. Create Deployment

First we need to Have a deployment. Lets use NGINX. The tutorial gives us a deployment.yaml, but kubernetes docs seems to give almost the exact same yaml, so lets go with the docs version to get used to navigating and using k8s docs for k8s management ( a must during the CKAD test ):

### YAML Definition

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

---

## Create File via CLI

```bash
vboxuser@k8s-init:~$ cat <<EOF | tee deployment.yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: nginx-deployment
>   labels:
>     app: nginx
> spec:
>   replicas: 3
>   selector:
>     matchLabels:
>       app: nginx
>   template:
>     metadata:
>       labels:
>         app: nginx
>     spec:
>       containers:
>       - name: nginx
>         image: nginx:1.14.2
>         ports:
>         - containerPort: 80
> EOF
```

---

## Apply Deployment

```bash
kubectl apply -f deployment.yaml
```

---

## 2. Update Deployment (Rolling Update)

Now that we have a deployment called "nginx-deployment" we need to update our deployment, the way the tutorial does it is by updating the image version of the container withing the pod managed by the deployment. We do that by updating the nginx image for that deployment only. So:

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.25.0
```

### Command Pattern

```bash
kubectl set image <kind>/<name> <image_name>/<image:tag>
```

---

## 3. Monitor Rollout

```bash
kubectl rollout status deployment/nginx-deployment
```

Example:

```bash
vboxuser@k8s-init:~$ kubectl rollout status deployment/nginx-deployment
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated..
```

---

## 4. Issue Encountered

So , the rollout wasn't happening due to DNS issues on all of the pods.

---

## 5. Fix DNS

After fixing the DNS issues by uncommenting DNS and FallbackDNS in "/etc/systemd/resolved.conf" and setting them to:

```yaml
DNS=8.8.8.8 1.1.1.1
FallbackDNS=8.8.4.4 1.0.0.1
```

Restart service:

```bash
sudo systemctl restart systemd-resolved
```

---

## 6. Successful Rollout

The rollout worked just fine:

```bash
kubectl get pods -l app=nginx -w
```

(Full output preserved)

```bash
NAME                                READY   STATUS             RESTARTS   AGE
nginx-deployment-6d797fb658-8875r   0/1     ImagePullBackOff   0          17m
nginx-deployment-7b8ddcb488-wlb8l   0/1     ImagePullBackOff   0          23m
nginx-deployment-86dcfdf4c6-b2wgd   0/1     ImagePullBackOff   0          26m
nginx-deployment-86dcfdf4c6-xzh5b   1/1     Running            0          26m
nginx-deployment-6d797fb658-8875r   1/1     Running            0          17m
nginx-deployment-86dcfdf4c6-b2wgd   0/1     Terminating        0          26m
...
deployment "nginx-deployment" successfully rolled out
```

---

## 7. Notes on Flags

| Flag | Meaning |
|------|--------|
| `-l` | same as `--selector`, used for filtering |
| `-w` | watch (real-time updates) |

For some reason, the -l flag in "kubectl get pods" is the same as the --selector flag. its used for filtering. the -w flag is for watching ( makes sense ).

---

## 8. Rollback

Now if we want to do a rollback, we can do:

```bash
kubectl rollout undo deployment/nginx-deployment --to-revision=1
```

Example:

```bash
kubectl rollout undo deployment/nginx-deployment
kubectl get pods -w
```

Output:

```bash
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-6d797fb658-8875r   1/1     Running   0          25m
nginx-deployment-6d797fb658-j888v   1/1     Running   0          7m31s
nginx-deployment-6d797fb658-zpslf   1/1     Running   0          7m37s
```
