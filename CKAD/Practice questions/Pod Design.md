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

---
### Create a pod that will be placed on node `node01` using `nodeName`
```bash
❯ kubectl run nginx --image=nginx -n myns --restart=Never --dry-run=client -o yaml > nginx-nodeName.yaml
❯ vim nginx-nodeName.yaml # Edit the pod to include nodeName
❯ cat nginx-nodeName.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
  namespace: myns
spec:
  containers:
  - image: nginx
    name: nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
  nodeName: minikube
status: {}
❯ kubectl create -f nginx-nodeName.yaml
```

---
### Taint a node with key `tier` and value `frontend` with the effect `NoSchedule`. Then, create a pod that tolerates this taint.
```bash
kubectl taint node minikube tier=frontend:NoSchedule
❯ kubectl run nginx --image=nginx -n myns --restart=Never --dry-run=client -o yaml > nginx-tolerant.yaml
vim nginx-tolerant.yaml # edit the yaml to include the toleration
cat nginx-tolerant.yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:
  - key: "tier"
    operator: "Equal"
    value: "frontend"
    effect: "NoSchedule"
    
status: {}
kubectl create nginx-tolerant.yaml
```
>[!NOTE]
>You write tolerations like so:
>```yaml
>tolerations:
>- key: "tier"
>   operator: "Equal"
>   value: "frontend"
>   effect: "NoSchedule"
>```

---
### Create a pod that will be placed on node `controlplane`. Use nodeSelector and tolerations.
```bash
❯ kubectl run nginx --image=nginx -n myns --restart=Never --dry-run=client -o yaml > nginx-nodeSelector-taint.yaml
❯ vim nginx-nodeSelector-taint.yaml # edit the yaml to add nodeSelector and toleration
❯ cat nginx-nodeSelector-taint.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
  namespace: myns
spec:
  containers:
  - image: nginx
    name: nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
  nodeSelector:
    kubernetes.io/hostname: controlplane
  tolerations:
    - key: "node-role.kubernetes.io/control-plane"
      operator: "Exists"
      effect: "NoSchedule"
status: {}
❯ kubectl create -f nginx-nodeSelector-taint.yaml
```
>[!NOTE]
>nodeSelecting on hostnames is done by using the `kubernetes.io/hostname` key, and providing tolerations for the controlplane taint is done by ( this can be found in the docs ) `node-role.kubernetes.io/control-plane` and the operator `Exists`

---
### Create a deployment with image nginx:1.18.0, called nginx, having 2 replicas, defining port 80 as the port that this container exposes (don't create a service for this deployment)
```bash
❯ kubectl create deployment nginx --image=nginx:1.18.0 --replicas=2 --dry-run=client -o yaml > deploy-nginx.yaml
❯ vim deploy-nginx.yaml
❯ cat deploy-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.18.0
        name: nginx
        resources: {}
        ports:
        - containerPort: 80
status: {}
❯ kubectl apply -f deploy-nginx.yaml
deployment.apps/nginx created
```
>[!NOTE]
>You can specify the port of a container via:
>```yaml
>ports:
>- containerPort: <port>
>```
>This can be easily found on the kubernetes docs

---
### View the YAML of this deployment
```bash
kubectl get deploy nginx -o yaml
```
---
### View the YAML of the replica set that was created by this deployment
```bash
kubectl describe deploy nginx
❯ kubectl get rs nginx-7b4cd9f47b -o yaml
```
>[!NOTE]
>You can view the replicaset a deployment has created by the `Events` or `NewReplicaSet` section when using `describe`

---
### Get the YAML for one of the pods
```bash
❯ kubectl describe rs nginx-7b4cd9f47b
...
Events:
  Type    Reason            Age    From                   Message
  ----    ------            ----   ----                   -------
  Normal  SuccessfulCreate  7m46s  replicaset-controller  Created pod: nginx-7b4cd9f47b-x5lxp
  Normal  SuccessfulCreate  7m46s  replicaset-controller  Created pod: nginx-7b4cd9f47b-6c9ck
❯ kubectl get pod nginx-7b4cd9f47b-x5lxp -o yaml
```
---
### Check how the deployment rollout is going
```bash
❯ kubectl rollout status deployment nginx
deployment "nginx" successfully rolled out
```
---
### Update the nginx image to nginx:1.19.8
```bash
❯ kubectl set image deploy/nginx nginx=nginx1.19.8
deployment.apps/nginx image updated
```
---
### Check the rollout history and confirm that the replicas are OK

```bash
❯ kubectl rollout status deployment nginx
Waiting for deployment "nginx" rollout to finish: 1 out of 2 new replicas have been updated... # wait for both replicas to be updated , then check the history and the replicas and the pods
kubectl rollout history deployment nginx
kubectl get rs
kubectl get pod
```
---
### Undo the latest rollout and verify that new pods have the old image (nginx:1.18.0)
```bash
❯ kubectl describe deploy nginx
❯ kubectl describe rs nginx-7b4cd9f47b
❯ kubectl get pod nginx-7b4cd9f47b-x5lxp -o yaml
...
spec:
  containers:
  - image: nginx:1.18.0
```
---
### Do an on-purpose update of the deployment with a wrong image nginx:1.91
```bash
kubectl set image deploy/nginx nginx=nginx:1.91
```
---
### Verify that something's wrong with the rollout
```bash
kubectl rollout status deploy nginx
kubectl logs <pod>
```
---
### Return the deployment to the second revision (number 2) and verify the image is nginx:1.19.8
```bash
kubectl rollout undo deployment nginx --to-revision=2
kubectl describe deploy nginx | grep Image:
kubectl rollout status deploy nginx
```
>[!NOTE]
>With the command `kubectl rollout undo deployment <deploy>` you can add the `--to-revision=x` flag to specify the revision of the rollout

---
### Check the details of the fourth revision (number 4)
```bash
kubectl rollout history deployment nginx --revision=4
```
>[!NOTE]
>`kubectl rollout history` gives you a list of revisions. You can specify which revision you want the details of with the flag `--revision=x`

---
### Scale the deployment to 5 replicas
```bash
kubectl edit deployment nginx # edit the replicas to five
# or
❯ kubectl scale --replicas=5 deployment nginx
```
>[!NOTE]
>Use the `kubectl scale deployment <deploy> --replicas=x` to scale up or down the deployment

---
### Autoscale the deployment, pods between 5 and 10, targeting CPU utilization at 80%
```bash
❯ kubectl autoscale deployment nginx --cpu-percent=80 --min=5 --max=10
```
>[!NOTE]
>I didnt remember how to do this but the kubernetes docs and `kubectl autoscale --help` was really helpful

>[!IMPORTANT]
>USE `kubectl <command> --help` WHEN IN DOUBT!!!

---
### Pause the rollout of the deployment
```bash
❯ kubectl rollout pause deployment nginx
```
---
### Update the image to nginx:1.19.9 and check that there's nothing going on, since we paused the rollout
```bash
❯ kubectl set image deployment/nginx nginx=nginx:1.19.9
❯ kubectl rollout status deployment nginx
❯ kubectl rollout history deploy nginx
```
---
### Resume the rollout and check that the nginx:1.19.9 image has been applied
```bash
❯ kubectl rollout resume deploy nginx
deployment.apps/nginx resumed
❯ kubectl rollout status deployment nginx
Waiting for deployment "nginx" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 2 old replicas are pending termination...
Waiting for deployment "nginx" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx" rollout to finish: 1 old replicas are pending termination...
deployment "nginx" successfully rolled out
❯ kubectl rollout history deploy nginx
deployment.apps/nginx
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
4         <none>
5         <none>
```
---
### Delete the deployment and the horizontal pod autoscaler you created
```bash
❯ kubectl delete deployment nginx
deployment.apps "nginx" deleted
❯ kubectl delete hpa nginx
horizontalpodautoscaler.autoscaling "nginx" deleted
```
>[!NOTE]
>HPAs are their own resource, you need to specifically delete them

---
### Implement canary deployment by running two instances of nginx marked as version=v1 and version=v2 so that the load is balanced at 75%-25% ratio
