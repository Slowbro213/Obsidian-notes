## Requests and Limits

### Requests

**Requests**: The amount of CPU/memory that is **guaranteed**. Used by the scheduler for placing Pods.

Assume this YAML:

```yaml
resources:
  requests:
    cpu: 200m
    memory: 128Mi
```

The scheduler will try to schedule this Pod on a Node that has at least:

- `200m` CPU (20% of a core)
- `128Mi` memory available

If **Requests is too high**, the Pod may become stuck in a `Pending` state (no node can satisfy it).

If **Requests is too low**, applications under heavy load may crash due to insufficient resources.

---

### Limits

**Limits**: The maximum amount of CPU/memory that a container is allowed to use.

- Enforced by the kubelet
- If CPU is exceeded → container is **throttled**
- If memory is exceeded → container is **OOMKilled**

---

# Advanced Scheduling

Scheduling is done through multiple mechanisms. We can influence how scheduling works to support specialized workflows.

We will explore:

- Node Affinity
- Taints and Tolerations

---

## Node Affinity

Node affinity attracts Pods to nodes based on labels.

### Types of Affinity Rules

- **requiredDuringSchedulingIgnoredDuringExecution**
  - Hard requirement
  - Pod will not be scheduled unless condition is met

- **preferredDuringScheduling**
  - Soft requirement
  - Scheduler will try, but not guarantee

---

## Labeling a Node

```bash
kubectl get nodes
```

Example:

```text
NAME          STATUS   ROLES           AGE    VERSION
k8s-init      Ready    control-plane   104m   v1.29.15
k8s-worker1   Ready    <none>          103m   v1.29.15
k8s-worker2   Ready    <none>          102m   v1.29.15
k8s-worker3   Ready    <none>          102m   v1.29.15
```

Add a label:

```bash
kubectl label node k8s-worker1 disktype=ssd
```

---

## Pod with Node Affinity

```yaml
# affinity-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
spec:
  containers:
    - name: nginx
      image: nginx
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
```

Apply:

```bash
kubectl apply -f affinity-pod.yaml
```

---

## Verify Scheduling

```bash
kubectl get pods -o wide
```

Example:

```text
NAME         READY   STATUS    RESTARTS   AGE    IP               NODE
probe-demo   1/1     Running   0          109m   192.168.126.1    k8s-worker2
ssd-pod      1/1     Running   0          66s    192.168.194.65   k8s-worker1
```

---

# Taints and Tolerations

These ensure Pods are not scheduled onto inappropriate nodes.

- **Taint** → repels Pods from a node
- **Toleration** → allows a Pod to ignore a taint

---

## Taint Effects

- **NoSchedule** → no new Pods scheduled
- **PreferNoSchedule** → avoid if possible
- **NoExecute** → evicts existing Pods without toleration

---

## Apply Taints

```bash
kubectl taint node k8s-worker1 app=gpu:NoSchedule
kubectl taint node k8s-worker2 app=gpu:NoSchedule
kubectl taint node k8s-worker3 app=gpu:NoSchedule
```

---

## Test with Non-Tolerating Pod

```bash
kubectl run non-tolerating-pod --image=nginx
```

Check scheduling:

```bash
kubectl get pods -o wide
```

Example:

```text
NAME                 READY   STATUS    NODE
non-tolerating-pod   0/1     Pending   <none>
probe-demo           1/1     Running   k8s-worker2
ssd-pod              1/1     Running   k8s-worker1
```

---

## Cleanup

```bash
kubectl taint node k8s-worker1 app=gpu:NoSchedule-
kubectl taint node k8s-worker2 app=gpu:NoSchedule-
kubectl taint node k8s-worker3 app=gpu:NoSchedule-
kubectl delete pod non-tolerating-pod
```
