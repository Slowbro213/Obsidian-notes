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
  nodeName: minikube # placed minikube instead of node01 here since i dont have
  # a node named node01
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
```bash
# This is a bit complicated
kubectl create deployment my-app-v1 --image=nginx --replicas=3 --dry-run=client -o yaml > version1.yaml
vim version1.yaml # Here we edit the version1.yaml to change the label of the deployment. the label becomes my-app and the deployment is edited to print 'Test' when a request is sent  
cp version1.yaml version2.yaml
vim version2.yaml # Same thing here but the request prints 'Test2' and replicas is set to 1
kubectl create -f version1.yaml
kubectl create -f version2.yaml
kubectl expose deployment my-app-v2 --port=80 --dry-run=client -o yaml > myapp-service.yaml
vim myapp-service.yaml # here we edit the service to use the my-app label
kubectl create -f myapp-service.yaml
kubectl run busybox --image=busybox --restart=Never -it --rm --command -- /bin/sh -c 'while true; do wget -qO- my-app; sleep 1; done'
Test2
Test
Test
Test2
Test
Test
Test
Test
Test
# Here we can see the requests are being load balanced roughly in a 75%-25% manner
kubectl scale deployment my-app-v2 --replicas=4
kubectl scale deployment my-app-v1 --replicas=0
# Watch the deployments finish then confirm with the busybox command traffic is only being served by version 2
kubectl delete deployment my-app-v1
```
Yamls:
```yaml
# service
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: my-app
  name: my-app
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: my-app
status:
  loadBalancer: {}
```

```yaml
# version 1
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: my-app
    version: v1
  name: my-app-v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: v2
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: my-app
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
        volumeMounts:
        - name: vol
          mountPath: "/usr/share/nginx/html"
      initContainers:
      - image: busybox
        name: init-box
        resources: {}
        command: ["/bin/sh" , "-c" , 'echo "Test" > /work-dir/index.html']
        volumeMounts:
        - name: vol
          mountPath: "/work-dir"
      volumes:
      - name: vol
        emptyDir: {}
status: {}
```

```yaml
# version 2
❯ cat version2.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: my-app
    version: v2
  name: my-app-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
      version: v2
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: my-app
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
        volumeMounts:
        - name: vol
          mountPath: "/usr/share/nginx/html"
      initContainers:
      - image: busybox
        name: init-box
        resources: {}
        command: ["/bin/sh" , "-c" , 'echo "Test2" > /work-dir/index.html']
        volumeMounts:
        - name: vol
          mountPath: "/work-dir"
      volumes:
      - name: vol
        emptyDir: {}
status: {}
```

>[!NOTE]
>REMEMBER TO PLACE THE VERSION OF THE DEPLOYMENTS UNDER METADATA AND TEMPLATE!!


>[!EXPLANATION]
>So we first create a simple deployment with 3 replicas and treat that as version 1. then we edit that deployment to do the following:
>- Change the label to one it will share with version 2
>- Add an emptyDir volume and an initContainer. the volume will be shared between the nginx container and the initContainer. The initContainer will serve to create the index.html file with "Test" in it that the nginx container will then serve. the same volume will then be mounted to the nginx container with the path `/usr/share/nginx/html` so that the index.html file created previously will be somewhere nginx can find and serve on request
>- Copy everything we did to version 1 and do that with version 2, but set the replicas to 1 , change the name of the deployment ( but not the label ) and then change "Test" to "Test2" so that responses to  requests can be identified by the version that served them.
>- Expose one of the deployments in order to create a service that will route to the label shared by both deployments
>- Create a busybox container with a shell that will send requests to the newly created service and will show the output in order to confirm the load balancing is as expected.
>- Scale up version 2 to 4 replicas
>- Scale down version 1 to 0 replicas
>- Delete version 1
>- Confirm the version deployment occured as expected

---
### Create a job named pi with image perl:5.34 that runs the command with arguments "perl -Mbignum=bpi -wle 'print bpi(2000)'"
```bash
#This can be found verbatim in the kubernetes docs, but im going to avoid copying and pasting in order to learn how to do it from the command line from memory since thats faster
❯ kubectl create job pi --image=perl:5.34 -- /bin/bash -c "perl -Mbignum=bpi -wle 'print bpi(2000)'"
# Somehow i managed to guess this correctly! Yay me!
```
---
### Wait till it's done, get the output
```bash
❯ kubectl get job pi -w # Wait until its done ( this one i had to look up, i thought maybe rollout status would work since its a workflow and it works for deployments )
NAME   STATUS     COMPLETIONS   DURATION   AGE
pi     Complete   1/1           15s        112s
❯ kubectl describe job pi # Find out the pod it created
❯ kubectl logs pi-cfsnm # Get the logs of that pod
3.14159265358979323846264338327950288419716939937510582097494459230781640628620899862803...
❯ kubectl delete job pi # Unsure if this step is needed, the previous i did myself but this one was in the answers
```
---
### Create a job with the image busybox that executes the command 'echo hello;sleep 30;echo world'
```bash
kubectl create job busyjob --image=busybox -- /bin/sh -c 'echo hello;sleep 30;echo world'
```
---
### Follow the logs for the pod (you'll wait for 30 seconds)
```bash
❯ kubectl describe job busyjob
❯ kubectl logs busyjob-vm7gq -f
hello
world
```
---
### See the status of the job, describe it and see the logs
```bash
❯ kubectl get job busyjob -o jsonpath={.status}
{"completionTime":"2026-06-11T20:14:42Z","conditions":[{"lastProbeTime":"2026-06-11T20:14:42Z","lastTransitionTime":"2026-06-11T20:14:42Z","message":"Reached expected number of succeeded pods","reason":"CompletionsReached","status":"True","type":"SuccessCriteriaMet"},{"lastProbeTime":"2026-06-11T20:14:42Z","lastTransitionTime":"2026-06-11T20:14:42Z","message":"Reached expected number of succeeded pods","reason":"CompletionsReached","status":"True","type":"Complete"}],"ready":0,"startTime":"2026-06-11T20:14:02Z","succeeded":1,"terminating":0,"uncountedTerminatedPods":{}}
❯ kubectl describe job busyjob
❯ kubectl logs busyjob-vm7gq -f
```
---
### Delete the job
```bash
❯ kubectl delete job busyjob
job.batch "busyjob" deleted
```
---
### Create the same job, make it run 5 times, one after the other. Verify its status and delete it
```bash
❯ kubectl create job busyjob --image=busybox --dry-run=client -o yaml -- /bin/sh -c 'echo hello;sleep 30; echo world' > busyjob.yaml
❯ vim busyjob.yaml
❯ cat busyjob.yaml
apiVersion: batch/v1
kind: Job
metadata:
  creationTimestamp: null
  name: busyjob
spec:
  completions: 5
  template:
    metadata:
      creationTimestamp: null
    spec:
      containers:
      - command:
        - /bin/sh
        - -c
        - echo hello;sleep 30; echo world
        image: busybox
        name: busyjob
        resources: {}
      restartPolicy: Never
❯ kubectl get job busyjob -w
NAME      STATUS    COMPLETIONS   DURATION   AGE
busyjob   Running   0/5           15s        15s
busyjob   Running   0/5           34s        34s
busyjob   Running   0/5           35s        35s
busyjob   Running   1/5           35s        35s
busyjob   Running   1/5           38s        38s
...
busyjob   Running   4/5           2m54s      2m54s
busyjob   Running   4/5           2m55s      2m55s
busyjob   Complete   5/5           2m55s      2m55s

❯ kubectl delete job busyjob
```
>[!NOTE]
>You can modify the completions of a job by adding it under the `spec` part of the *resource*

---
### Create the same job, but make it run 5 parallel times
```bash
❯ kubectl create job busyjob --image=busybox --dry-run=client -o yaml -- /bin/sh -c 'echo hello;sleep 30; echo world' > busyjob.yaml
❯ vim busyjob.yaml
❯ cat busyjob.yaml
apiVersion: batch/v1
kind: Job
metadata:
  creationTimestamp: null
  name: busyjob
spec:
  parallelism: 5
  template:
    metadata:
      creationTimestamp: null
    spec:
      containers:
      - command:
        - /bin/sh
        - -c
        - echo hello;sleep 30; echo world
        image: busybox
        name: busyjob
        resources: {}
      restartPolicy: Never
❯ kubectl create -f busyjob.yaml
job.batch/busyjob created
❯ kubectl get job busyjob -w
NAME      STATUS    COMPLETIONS   DURATION   AGE
busyjob   Running   0/1 of 5      17s        17s
busyjob   Running   0/1 of 5      34s        34s
busyjob   Running   0/1 of 5      35s        35s
busyjob   Running   1/1 of 5      35s        35s
busyjob   Running   1/1 of 5      36s        36s
busyjob   Running   2/1 of 5      36s        36s
busyjob   Running   2/1 of 5      37s        37s
busyjob   Running   3/1 of 5      37s        37s
busyjob   Running   3/1 of 5      38s        38s
busyjob   Running   3/1 of 5      39s        39s
busyjob   Running   4/1 of 5      39s        39s
busyjob   Running   4/1 of 5      40s        40s
busyjob   Complete   5/1 of 5      40s        40s
```
>[!NOTE]
>Similarly to `completions`, you can specify `parallelism` the same way, by putting it under the `spec` of the resource

---
### Create a job but ensure that it will be automatically terminated by kubernetes if it takes more than 30 seconds to execute
```bash
# This can easily be found in the kubernetes docs
❯ kubectl create job busyjob --image=busybox --dry-run=client -o yaml -- /bin/sh -c 'echo hello;sleep 30; echo world' > busyjob.yaml
❯ vim busyjob.yaml # Edit the manifest to add .spec.activeDeadlineSeconds
❯ cat busyjob.yaml
apiVersion: batch/v1
kind: Job
metadata:
  creationTimestamp: null
  name: busyjob
spec:
  activeDeadlineSeconds: 30
  template:
    metadata:
      creationTimestamp: null
    spec:
      containers:
      - command:
        - /bin/sh
        - -c
        - echo hello;sleep 30; echo world
        image: busybox
        name: busyjob
        resources: {}
      restartPolicy: Never
status: {}
❯ kubectl create -f busyjob.yaml
job.batch/busyjob created
```
---
### Create a cron job with image busybox that runs on a schedule of "*/1 * * * *" and writes 'date; echo Hello from the Kubernetes cluster' to standard output
```bash
❯ kubectl create cronjob busycronjob --image=busybox --schedule='*/1 * * * *' -- /bin/sh -c 'date;echo "Hello from the Kubernetes cluster"'
❯ kubectl describe cronjob busycronjob
❯ kubectl logs busycronjob-29686851-6k9qf
Thu Jun 11 20:51:01 UTC 2026
Hello from the Kubernetes cluster
```
>[!NOTE]
>Just like you can create `jobs`, you can also create `cronjobs`, which can be run on a schedule

---
### See its logs and delete it
```bash
❯ kubectl describe cronjob busycronjob
❯ kubectl logs busycronjob-29686851-6k9qf
Thu Jun 11 20:51:01 UTC 2026
Hello from the Kubernetes cluster
❯ kubectl delete cronjob busycronjob
cronjob.batch "busycronjob" deleted
```
---
### Create the same cron job again, and watch the status. Once it ran, check which job ran by the created cron job. Check the log, and delete the cron job
```bash
❯ kubectl create cronjob busycronjob --image=busybox --schedule='*/1 * * * *' -- /bin/sh -c 'date;echo "Hello from the Kubernetes cluster"'
cronjob.batch/busycronjob created
❯ kubectl get cronjob busycronjob -w
NAME          SCHEDULE      TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
busycronjob   */1 * * * *   <none>     False     0        <none>          7s
busycronjob   */1 * * * *   <none>     False     1        0s              27s
busycronjob   */1 * * * *   <none>     False     0        8s              35s
busycronjob   */1 * * * *   <none>     False     1        0s              87s
❯ kubectl describe cronjob busycronjob
...
Events:
  Type    Reason            Age   From                Message
  ----    ------            ----  ----                -------
  Normal  SuccessfulCreate  63s   cronjob-controller  Created job busycronjob-29686856
  Normal  SawCompletedJob   54s   cronjob-controller  Saw completed job: busycronjob-29686856, condition: Complete
  Normal  SuccessfulCreate  3s    cronjob-controller  Created job busycronjob-29686857
❯ kubectl delete cronjob busycronjob
cronjob.batch "busycronjob" deleted
```
---
### Create a cron job with image busybox that runs every minute and writes 'date; echo Hello from the Kubernetes cluster' to standard output. The cron job should be terminated if it takes more than 17 seconds to start execution after its scheduled time (i.e. the job missed its scheduled time).
```bash
❯ kubectl create cronjob busycronjob --image=busybox --dry-run=client -o yaml --schedule='* * * * *' -- /bin/sh -c 'date;echo "Hello from the Kubernetes cluster"' > busyjob.yaml
❯ vim busyjob.yaml
❯ cat busyjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  creationTimestamp: null
  name: busycronjob
spec:
  startingDeadlineSeconds: 17
  jobTemplate:
    metadata:
      creationTimestamp: null
      name: busycronjob
    spec:
      template:
        metadata:
          creationTimestamp: null
        spec:
          containers:
          - command:
            - /bin/sh
            - -c
            - date;echo "Hello from the Kubernetes cluster"
            image: busybox
            name: busycronjob
            resources: {}
          restartPolicy: OnFailure
  schedule: '* * * * *'
status: {}
❯ kubectl create -f busyjob.yaml
cronjob.batch/busycronjob created
❯ kubectl describe cronjob busycronjob
Events:
  Type    Reason            Age                  From                Message
  ----    ------            ----                 ----                -------
  Normal  SuccessfulCreate  16m                  cronjob-controller  Created job busycronjob-29686862
  Normal  SawCompletedJob   16m                  cronjob-controller  Saw completed job: busycronjob-29686862, condition: Complete
  Normal  SuccessfulCreate  15m                  cronjob-controller  Created job busycronjob-29686863
  Normal  SawCompletedJob   15m                  cronjob-controller  Saw completed job: busycronjob-29686863, condition: Complete
  Normal  SuccessfulCreate  14m                  cronjob-controller  Created job busycronjob-29686864
  Normal  SawCompletedJob   14m                  cronjob-controller  Saw completed job: busycronjob-29686864, condition: Complete
  Normal  SuccessfulCreate  13m                  cronjob-controller  Created job busycronjob-29686865
  Normal  SuccessfulDelete  13m                  cronjob-controller  Deleted job busycronjob-29686862
  Normal  SawCompletedJob   13m                  cronjob-controller  Saw completed job: busycronjob-29686865, condition: Complete
  Normal  SuccessfulCreate  12m                  cronjob-controller  Created job busycronjob-29686866
```
---
### Create a cron job with image busybox that runs every minute and writes 'date; echo Hello from the Kubernetes cluster' to standard output. The cron job should be terminated if it successfully starts but takes more than 12 seconds to complete execution.
```bash
❯ kubectl create cronjob busycronjob --image=busybox --dry-run=client -o yaml --schedule='* * * * *' -- /bin/sh -c 'date;echo "Hello from the Kubernetes cluster"' > busyjob.yaml
❯ vim busyjob.yaml # add the activeDeadlineSeconds to the jobTemplate
❯ cat busyjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  creationTimestamp: null
  name: busycronjob
spec:
  jobTemplate:
    metadata:
      creationTimestamp: null
      name: busycronjob
    spec:
      activeDeadlineSeconds: 12
      template:
        metadata:
          creationTimestamp: null
        spec:
          containers:
          - command:
            - /bin/sh
            - -c
            - date;echo "Hello from the Kubernetes cluster"
            image: busybox
            name: busycronjob
            resources: {}
          restartPolicy: OnFailure
  schedule: '* * * * *'
status: {}
❯ kubectl create -f busyjob.yaml
❯ kubectl describe cronjob busycronjob
Events:
  Type    Reason            Age   From                Message
  ----    ------            ----  ----                -------
  Normal  SuccessfulCreate  22s   cronjob-controller  Created job busycronjob-29686884
  Normal  SawCompletedJob   18s   cronjob-controller  Saw completed job: busycronjob-29686884, condition: Complete
❯ kubectl describe job busycronjob-29686884
Events:
  Type    Reason            Age   From            Message
  ----    ------            ----  ----            -------
  Normal  SuccessfulCreate  47s   job-controller  Created pod: busycronjob-29686884-4wqlt
  Normal  Completed         43s   job-controller  Job completed
❯ kubectl logs busycronjob-29686884-4wqlt
Thu Jun 11 21:24:02 UTC 2026
Hello from the Kubernetes cluster
```
---
### Create a job from cronjob.
```bash
❯ kubectl create job --from=cronjob/busycronjob busyjob
job.batch/busyjob created
```
>[!NOTE]
>You can create a job from a cronjob using the `--from=cronjob/<cronjob>` flag

