# Kubernetes Rolling Update Notes (NGINX)

---

## Overview

The default strategy for deploying is a rolling update, where pods are incrementally updated with new ones to ensure your app has no downtime.

---

## 1. Create Deployment

First we need to have a deployment. Let's use NGINX.

The tutorial gives us a `deployment.yaml`, but Kubernetes docs provide almost the exact same YAML, so we’ll use the docs version to get used to navigating and using Kubernetes documentation (important for CKAD).

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
cat <<EOF | tee deployment.yaml
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
EOF
```

---

## Apply Deployment

```bash
kubectl apply -f deployment.yaml
```

---

## 2. Update Deployment (Rolling Update)

Now that we have a deployment called `nginx-deployment`, we update it by changing the image version of the container.

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.25.0
```

### Command Pattern

```bash
kubectl set image <kind>/<name> <container>=<image:tag>
```

---

## 3. Monitor Rollout

```bash
kubectl rollout status deployment/nginx-deployment
```

Example:

```bash
kubectl rollout status deployment/nginx-deployment
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated..
```

---

## 4. Issue Encountered

The rollout was not happening due to DNS issues on all of the nodes.

---

## 5. Fix DNS

After fixing DNS by uncommenting `DNS` and `FallbackDNS` in `/etc/systemd/resolved.conf`:

```yaml
DNS=8.8.8.8 1.1.1.1
FallbackDNS=8.8.4.4 1.0.0.1
```

Restart the service:

```bash
sudo systemctl restart systemd-resolved
```

---

## 6. Successful Rollout

```bash
kubectl get pods -l app=nginx -w
```

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
| `-l` | Same as `--selector`, used for filtering |
| `-w` | Watch (real-time updates) |

---

## 8. Rollback

```bash
kubectl rollout undo deployment/nginx-deployment --to-revision=1
```

Example:

```bash
kubectl rollout undo deployment/nginx-deployment
kubectl get pods -w
```

```bash
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-6d797fb658-8875r   1/1     Running   0          25m
nginx-deployment-6d797fb658-j888v   1/1     Running   0          7m31s
nginx-deployment-6d797fb658-zpslf   1/1     Running   0          7m37s
```

---

# Self-Healing Deployments

In order to perform self-healing, Kubernetes needs to know if a container is alive and healthy. This is done through probes.

- **Liveness Probe**: Determines if the container is running. If it fails, Kubernetes restarts the container.
- **Readiness Probe**: Determines if the container is ready to serve traffic.
- **Startup Probe**: Determines if the container has started successfully.

---

## Probe Example

```yaml
# pod-probe.yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 10
      livenessProbe:
        tcpSocket:
          port: 80
        initialDelaySeconds: 15
        periodSeconds: 20
```

Apply:

```bash
kubectl apply -f pod-probe.yaml
```

---

## Inspect Probes

```bash
kubectl describe pod probe-demo
```

```bash
Containers:
  nginx:
    Container ID:   containerd://...
    Image:          nginx
    Port:           80/TCP
    State:          Running
    Ready:          True
    Restart Count:  0
    Liveness:       tcp-socket :80 delay=15s timeout=1s period=20s
    Readiness:      http-get http://:80/ delay=5s timeout=1s period=10s
```
