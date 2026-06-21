### Create a deployment called `webapp` with image `nginx:1.20` and 3 replicas. Record the change so it appears in rollout history.
```bash
❯ kubectl create deployment webapp --image=nginx:1.20 --replicas=3
deployment.apps/webapp created
❯ kubectl rollout status deployment webapp
deployment "webapp" successfully rolled out
❯ kubectl rollout history deployment webapp
deployment.apps/webapp
REVISION  CHANGE-CAUSE
1         <none>

❯ kubectl annotate deployment webapp kubernetes.io/change-clause="change recorded"
deployment.apps/webapp annotated
```
>[!NOTE]
>Change clause annotations are done via the `kubernetes.io/change-clause` key

---
### Update the `webapp` deployment image to `nginx:1.21`. Then check the rollout status and history.
```bash
❯ kubectl set image deployment/webapp nginx=nginx:1.21
deployment.apps/webapp image updated
❯ kubectl rollout status deployment webapp
Waiting for deployment "webapp" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "webapp" rollout to finish: 1 old replicas are pending termination...
deployment "webapp" successfully rolled out
❯ kubectl rollout history deployment webapp
deployment.apps/webapp
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
```
>[!NOTE]
>I accidentally set the image as `nginx=nginx1.21` instead of `nginx=nginx:1.21` in a previous attempt due to a typo, which is why you see 3 revisions

---
### Roll back the `webapp` deployment to the previous version and confirm the image reverted.
```bash
❯ kubectl rollout undo deployment webapp --to-revision=1
deployment.apps/webapp rolled back
❯ kubectl rollout status deployment webapp
Waiting for deployment "webapp" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "webapp" rollout to finish: 1 old replicas are pending termination...
deployment "webapp" successfully rolled out
❯ kubectl rollout history deployment webapp
deployment.apps/webapp
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
4         <none>
```
---
### Inspect the details of revision 2 of the `webapp` deployment without rolling back to it.
```bash
❯ kubectl rollout history deployment webapp --revision=2
deployment.apps/webapp with revision #2
Pod Template:
  Labels:       app=webapp
        pod-template-hash=76cfccbc9b
  Containers:
   nginx:
    Image:      nginx1.21
    Port:       <none>
    Host Port:  <none>
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>
  Node-Selectors:       <none>
  Tolerations:  <none>
```
---
### Pause the `webapp` deployment, update its image to `nginx:1.22`, then resume and verify the rollout happened.
```bash
❯ kubectl rollout pause deployment webapp
deployment.apps/webapp paused
❯ kubectl set image deployment/webapp nginx=nginx:1.22
deployment.apps/webapp image updated
❯ kubectl rollout status deployment webapp
Waiting for deployment "webapp" rollout to finish: 0 out of 3 new replicas have been updated...
^C%                                                                                                                                                                                            ❯ kubectl rollout resume deployment webapp
deployment.apps/webapp resumed
❯ kubectl rollout status deployment webapp
Waiting for deployment "webapp" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "webapp" rollout to finish: 1 old replicas are pending termination...
deployment "webapp" successfully rolled out
```
---
### Change the update strategy of the `webapp` deployment to `Recreate` (instead of RollingUpdate). Explain when you would use this.
```bash
❯ kubectl edit deployment webapp
deployment.apps/webapp edited
❯ kubectl get deploy webapp -o yaml | grep -i strategy
  strategy:
❯ kubectl get deploy webapp -o yaml | grep -i strategy -A 3
  strategy:
    type: Recreate
  template:
    metadata:
```
>[!IMPORTANT]
>To change the strategy to Recreate, you must edit the yaml definition of the deployment, and specify the `type` to be `Recreate`

>[!NOTE]
>A recreate terminates all old pods before rolling out new ones. This causes  downtime but might be needed in cases when your application holds database locks ( in which case your new versions wouldn't work with the old version present )


---
### Scale the `webapp` deployment to 6 replicas, then set up a HorizontalPodAutoscaler that keeps replicas between 3 and 10, targeting 70% CPU.
```bash
❯ kubectl scale deploy webapp --replicas=6
deployment.apps/webapp scaled
❯ kubectl autoscale deployment webapp --cpu-percent=70 --min=3 --max=10
horizontalpodautoscaler.autoscaling/webapp autoscaled
```
---
### Implement a blue/green deployment. Deploy `nginx:1.20` as the `blue` version with 3 replicas, and expose it via a service called `webapp-svc` using a selector of `slot: blue`. Then deploy `nginx:1.21` as the `green` version with 3 replicas. Switch all traffic to green by updating the service selector.
```bash
❯ kubectl create deployment blue --image=nginx:1.20 --replicas=3 --dry-run=client -o yaml > blue.yaml
❯ vim blue.yaml
❯ cat blue.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: blue
    slot: blue
  name: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: blue
      slot: blue
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: blue
        slot: blue
    spec:
      containers:
      - image: nginx:1.20
        name: nginx
        resources: {}
status: {}
❯ kubectl apply -f blue.yaml
deployment.apps/blue created

❯ kubectl expose deploy blue --port=80 --dry-run=client -o yaml > webapp-svc.yaml
❯ vim webapp-svc.yaml
❯ cat webapp-svc.yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: blue
    slot: blue
  name: webapp-svc
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: blue
    slot: blue
status:
  loadBalancer: {}
❯ kubectl create -f webapp-svc.yaml
service/webapp-svc created

❯ kubectl create deployment green --image=nginx:1.21 --replicas=3 --dry-run=client -o yaml > green.yaml
❯ vim green.yaml
❯ cat green.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: green
    slot: green
  name: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: green
      slot: green
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: green
        slot: green
    spec:
      containers:
      - image: nginx:1.21
        name: nginx
        resources: {}
status: {}
❯ kubectl create -f green.yaml
deployment.apps/green created

❯ kubectl edit svc webapp-svc
service/webapp-svc edited
❯ kubectl get svc webapp-svc -o yaml | grep -i slot -A 2
    slot: green
  name: webapp-svc
  namespace: default
--
    slot: green
  sessionAffinity: None
  type: ClusterIP
```
---
### Create a DaemonSet called `monitor` using image `busybox` with the command `sleep 3600`. Verify that a pod is running on every node.
```bash
❯ kubectl create deployment monitor --image=busybox --dry-run=client -o yaml -- /bin/sh -c 'sleep 3600' > ds.yaml
❯ vim ds.yaml # Here we need to change the kind, and remove deployment
# specific fields
❯ cat ds.yaml
apiVersion: apps/v1
kind: DaemonSet # change kind from Deployment to DaemonSet
metadata:
  creationTimestamp: null
  labels:
    app: monitor
  name: monitor
spec:
  # removed replicas and strategy
  selector:
    matchLabels:
      app: monitor
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: monitor
    spec:
      containers:
      - command:
        - /bin/sh
        - -c
        - sleep 3600
        image: busybox
        name: busybox
        resources: {}
status: {}
❯ kubectl create -f ds.yaml
daemonset.apps/monitor created
❯ kubectl describe daemonset monitor
...
Events:
  Type    Reason            Age    From                  Message
  ----    ------            ----   ----                  -------
  Normal  SuccessfulCreate  3m15s  daemonset-controller  Created pod: monitor-57gfg
❯ kubectl get pod monitor-57gfg -o wide
NAME            READY   STATUS    RESTARTS   AGE     IP            NODE       NOMINATED NODE   READINESS GATES
monitor-57gfg   1/1     Running   0          3m26s   10.244.0.96   minikube   <none>           <none>
❯ kubectl get nodes
NAME       STATUS   ROLES           AGE     VERSION
minikube   Ready    control-plane   5d20h   v1.31.0
```
>[!IMPORTANT]
>In order to quickly create DaemonSets, create a deployment with `kubectl create deployment` and then edit the kind to be a `DaemonSet` and remove the `Deployment` specific fields like `replicas` and `strategy`

---
### The `monitor` DaemonSet should also run on the control-plane node, which has a taint. Update the DaemonSet to tolerate the `node-role.kubernetes.io/control-plane:NoSchedule` taint.
```bash
❯ kubectl delete daemonset monitor
daemonset.apps "monitor" deleted
❯ vim ds.yaml
❯ cat ds.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  creationTimestamp: null
  labels:
    app: monitor
  name: monitor
spec:
  selector:
    matchLabels:
      app: monitor
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: monitor
    spec:
      containers:
      - command:
        - /bin/sh
        - -c
        - sleep 3600
        image: busybox
        name: busybox
        resources: {}
      tolerations:
      - key: 'node-role.kubernetes.io/control-plane'
        operator: 'Exists'
        effect: 'NoSchedule'
status: {}
❯ kubectl create -f ds.yaml
daemonset.apps/monitor created
```
---
### Select which nodes the `monitor` DaemonSet runs on by adding a `nodeSelector` so it only runs on nodes labelled `monitoring=enabled`. Label one node and verify.
```bash
❯ kubectl delete daemonset monitor
daemonset.apps "monitor" deleted
❯ vim ds.yaml
❯ cat ds.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  creationTimestamp: null
  labels:
    app: monitor
  name: monitor
spec:
  selector:
    matchLabels:
      app: monitor
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: monitor
    spec:
      containers:
      - command:
        - /bin/sh
        - -c
        - sleep 3600
        image: busybox
        name: busybox
        resources: {}
      nodeSelector:
        monitoring: enabled
      tolerations:
      - key: 'node-role.kubernetes.io/control-plane'
        operator: 'Exists'
        effect: 'NoSchedule'
status: {}
❯ kubectl create -f ds.yaml
daemonset.apps/monitor created
❯ kubectl label node minikube monitoring=enabled
node/minikube labeled
❯ kubectl get pod monitor-rhwdm -o wide
NAME            READY   STATUS    RESTARTS   AGE   IP            NODE       NOMINATED NODE   READINESS GATES
monitor-rhwdm   1/1     Running   0          26s   10.244.0.98   minikube   <none>           <none>
```
---
### Create a pod called `multi` with two containers: a main `nginx` container, and a sidecar `busybox` container that runs `tail -f /dev/null`. Exec into the sidecar and confirm you can see the nginx process namespace.
```bash
❯ kubectl run multi --image=nginx --dry-run=client -o yaml > multi.yaml
❯ kubectl explain pod.spec | grep -i share
    spec.shareProcessNamespace - spec.securityContext.runAsUser -
  shareProcessNamespace <boolean>
    Share a single process namespace between all of the containers in a pod.
    will not be assigned PID 1. HostPID and ShareProcessNamespace cannot both be
❯ vim multi.yaml
❯ cat multi.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: multi
  name: multi
spec:
  shareProcessNamespace: true
  containers:
  - image: nginx
    name: multi
    resources: {}
  - image: busybox
    name: sidecar
    command: ['tail', '-f', '/dev/null']
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
❯ kubectl create -f multi.yaml
pod/multi created
❯ kubectl exec multi -c sidecar -it -- /bin/sh
/ # ps aux
PID   USER     TIME  COMMAND
    1 65535     0:00 /pause
    7 root      0:00 nginx: master process nginx -g daemon off;
   35 101       0:00 nginx: worker process
   36 101       0:00 nginx: worker process
   37 101       0:00 nginx: worker process
   38 101       0:00 nginx: worker process
   39 101       0:00 nginx: worker process
   40 101       0:00 nginx: worker process
   41 101       0:00 nginx: worker process
   42 101       0:00 nginx: worker process
   43 101       0:00 nginx: worker process
   44 101       0:00 nginx: worker process
   45 101       0:00 nginx: worker process
   46 101       0:00 nginx: worker process
   47 101       0:00 nginx: worker process
   48 101       0:00 nginx: worker process
   49 101       0:00 nginx: worker process
   50 101       0:00 nginx: worker process
   51 root      0:00 tail -f /dev/null
   57 root      0:00 /bin/sh
   63 root      0:00 ps aux
```
>[!NOTE]
>By default , containers within the same pod have different process namespaces, as such we need to set the `spec.shareProcessNamespace`  to true in order for one to see the other

---
### Create a pod with a main `nginx` container and a logging sidecar. The sidecar uses `busybox` to continuously print the nginx access log to stdout. Use a shared `emptyDir` volume — nginx writes logs there, the sidecar reads them.
```bash
❯ kubectl run multi --image=nginx --dry-run=client -o yaml > multi.yaml
❯ vim multi.yaml
❯ cat multi.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: multi
  name: multi
spec:
  containers:
  - image: nginx
    name: nginx
    resources: {}
    volumeMounts:
    - name: vol
      mountPath: /var/log/nginx
  - image: busybox
    name: logger
    command: ['/bin/sh', '-c' , 'tail -f /var/log/nginx/access.log']
    resources: {}
    volumeMounts:
    - name: vol
      mountPath: /var/log/nginx
  volumes:
  - name: vol
    emptyDir: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
❯ kubectl create -f multi.yaml
pod/multi created
```
>[!NOTE]
>`nginx` writes its access logs to `/var/log/nginx/access.log`

---
### Create a pod with an `ambassador` container pattern. The main `busybox` container connects to `localhost:6379`. An ambassador container (`haproxy` or a simple `socat` busybox) forwards that traffic to an external service. Demonstrate the concept using a socat ambassador that forwards `localhost:6379` to `google.com:80`.
