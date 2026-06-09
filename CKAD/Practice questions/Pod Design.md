# Kubernetes Labels, Annotations & Node Selectors

### Create 3 pods with names `nginx1`, `nginx2`, `nginx3`. All of them should have the label `app=v1`

```bash
❯ kubectl run nginx1 --image=nginx -n myns --restart=Never --labels=app=v1
pod/nginx1 created

❯ kubectl run nginx2 --image=nginx -n myns --restart=Never --labels=app=v1
pod/nginx2 created

❯ kubectl run nginx3 --image=nginx -n myns --restart=Never --labels=app=v1
pod/nginx3 created
```

### Show all labels of the pods

```bash
kubectl get pods -n myns --show-labels
```

> [!NOTE]  
> Use the flag `--show-labels` to display the labels of the pods.

---

### Change the labels of pod `nginx2` to be `app=v2`

```bash
❯ kubectl label pod nginx2 -n myns app=v2 --overwrite
```

> [!NOTE]  
> Use:
> 
> ```bash
> kubectl label pod <pod> key=value --overwrite
> ```
> 
> to change the label of a pod.
> 
> When in doubt, you can always use:
> 
> ```bash
> kubectl edit pod <pod>
> ```
> 
> and modify the YAML directly.

---

### Get the label `app` for the pods (show a column with APP labels)

```bash
❯ kubectl get pods -n myns -l=app --show-labels
```

### Get only the `app=v2` pods

```bash
❯ kubectl get pods -n myns -l=app=v2 --show-labels
```

### Get `app=v2` and not `tier=frontend` pods

```bash
❯ kubectl get pods -n myns -l=app=v2,tier!=frontend
```

> [!NOTE]  
> You can use the `!=` operator with label selectors (the `-l` flag).

---

### Add a new label `tier=web` to all pods having `app=v2` or `app=v1` labels

```bash
kubectl label pod -n myns -l "app in (v1,v2)" tier=web
```

> [!NOTE]  
> You can use selector labels with conditions such as:
> 
> ```text
> -l key in (val1,val2)
> ```

---

### Add an annotation `owner=marketing` to all pods having `app=v2` label

```bash
❯ kubectl annotate pod -n myns -l=app=v2 owner=marketing
```

> [!NOTE]  
> You can add annotations to pods just like labels.

---

### Remove the `app` label from the pods we created before

```bash
❯ kubectl label pod -n myns -l=app app-
```

> [!NOTE]  
> You can remove labels in the same way you add them, but instead of:
> 
> ```text
> key=value
> ```
> 
> use:
> 
> ```text
> key-
> ```

---

### Annotate pods `nginx1`, `nginx2`, `nginx3` with `description='my description'` value

```bash
❯ kubectl annotate pod nginx{1..3} -n myns description='my description'
```

### Check the annotations for pod `nginx1`

```bash
❯ kubectl get pod nginx1 -n myns -o jsonpath={.metadata.annotations}

# or

❯ kubectl annotate pod nginx1 -n myns --list
```

> [!NOTE]  
> With both annotations and labels, you can use:
> 
> ```bash
> kubectl annotate pod <pod> --list
> kubectl label pod <pod> --list
> ```
> 
> to display all annotations or labels.

---

### Remove the annotations for these three pods

```bash
❯ kubectl annotate pod nginx{1..3} -n myns description-
```

### Remove these pods to have a clean state in your cluster

```bash
❯ kubectl delete pod nginx{1..3} -n myns
```

---

### Create a pod that will be deployed to a Node that has the label `accelerator=nvidia-tesla-p100`

```bash
pod "nginx3" deleted

❯ kubectl get nodes
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   27h   v1.31.0

❯ kubectl label node minikube accelerator=nvidia-tesla-p100

❯ kubectl run labeledpod --image=busybox -n myns --restart=Never --dry-run=client -o yaml > labeledpod.yaml

❯ vim labeledpod.yaml # Change the pod yaml

❯ cat labeledpod.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: labeledpod
  name: labeledpod
  namespace: myns
spec:
  containers:
    - image: busybox
      name: labeledpod
      resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
  nodeSelector:
    accelerator: nvidia-tesla-p100
status: {}

❯ kubectl create -f labeledpod.yaml
```

> [!IMPORTANT]  
> Add a `nodeSelector` attribute to the pod `spec`, where each entry is a key-value pair.
> 
> In this example:
> 
> ```yaml
> nodeSelector:
>   accelerator: nvidia-tesla-p100
> ```
> 
> This ensures the pod is scheduled only on nodes matching that label.

