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
### Create a pod with an init container that puts custom content into a shared volume, and a main nginx container that serves the cloned content.
```bash
❯ kubectl run nginx --image=nginx --restart=Never --port=80 --dry-run=client -o yaml > nginx.yaml
❯ vim nginx.yaml
❯ cat nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
    resources: {}
    volumeMounts:
    - name: vol
      mountPath: /usr/share/nginx/html
  initContainers:
  - image: nginx
    name: init
    command: ['/bin/sh' , '-c' , 'echo "Hello World!" > /work-dir/index.html']
    volumeMounts:
    - name: vol
      mountPath: /work-dir
  volumes:
  - name: vol
    emptyDir: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
❯ kubectl create -f nginx.yaml
pod/nginx created
❯ kubectl get pod nginx -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP             NODE       NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          10s   10.244.0.106   minikube   <none>           <none>
❯ kubectl run busybox --image=busybox --restart=Never -it --rm --command -- /bin/sh -c 'wget -O- 10.244.0.106'
Connecting to 10.244.0.106 (10.244.0.106:80)
writing to stdout
Hello World!
-                    100% |********************************|    13  0:00:00 ETA
written to stdout
pod "busybox" deleted
```
---
### Create an nginx pod with a `startupProbe` that checks `GET /` on port 80, with a `failureThreshold` of 30 and `periodSeconds` of 10 (giving the container up to 5 minutes to start). Also add a `livenessProbe` on the same endpoint.
```bash
❯ kubectl run nginx --image=nginx --restart=Never --port=80 --dry-run=client -o yaml > nginx.yaml
❯ vim nginx.yaml
❯ cat nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
    resources: {}
    startupProbe:
      failureThreshold: 30
      periodSeconds: 10
      httpGet:
        port: 80
        path: /
    livenessProbe:
      httpGet:
        port: 80
        path: /
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
❯ kubectl create -f nginx.yaml
```
>[!NOTE]
>You can specify the number of failures allowed for the startup of a container via `startupProbe.failureThreshold`

>[!NOTE]
>`periodSeconds` on a `startupProbe` specifies the amount of time between each `startupProbe`

---
### Create a busybox pod that intentionally delays its readiness (using `sleep 20` in an init container). Add a `startupProbe` that runs `ls /tmp/ready` and a `readinessProbe` that does the same. Create the file `/tmp/ready` after 20 seconds via an init container. Observe the pod lifecycle.
```bash
❯ kubectl run nginx --image=nginx --restart=Never --port=80 --dry-run=client -o yaml > nginx.yaml
❯ vim nginx.yaml
❯ cat nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  initContainers:
  - image: busybox
    name: init
    command: ['/bin/sh', '-c' , 'sleep 20 && touch /tmp/ready']
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
    resources: {}
    startupProbe:
      exec:
        command: ['ls', '/tmp/ready']
    readinessProbe:
      exec:
        command: ['ls', '/tmp/ready']
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
❯ kubectl create -f nginx.yaml
	pod/nginx created`
❯ kubectl get pod nginx
NAME    READY   STATUS     RESTARTS   AGE
nginx   0/1     Init:0/1   0          8s
❯ kubectl get pod nginx
NAME    READY   STATUS            RESTARTS   AGE
nginx   0/1     PodInitializing   0          23s
❯ kubectl get pod nginx
NAME    READY   STATUS    RESTARTS   AGE
nginx   0/1     Running   0          26s
❯ kubectl get pod nginx
NAME    READY   STATUS    RESTARTS   AGE
nginx   0/1     Running   0          28s
```
---
### Create a deployment `web` with image `nginx` (3 replicas) and expose it as a ClusterIP service on port 80. Then create an Ingress resource that routes HTTP traffic for the host `web.example.com` at path `/` to that service.
```bash
❯ kubectl create deploy web --image=nginx --replicas=3 --port=80
deployment.apps/web created
❯ kubectl expose deploy web --port=80 --target-port=80 --type=ClusterIP
error: unknown flag: --targetPort
See 'kubectl expose --help' for usage.
❯ kubectl expose deploy web --port=80 --type=ClusterIP
service/web exposed
❯ vim nginx-ingress.yaml
❯ cat nginx-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  ingressClassName: nginx-example
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
    host: 'web.example.com'
❯ kubectl create -f nginx-ingress.yaml
ingress.networking.k8s.io/nginx-ingress created
```

>[!NOTE]
>its `--target-port` not `--targetPort`

---
### Create an Ingress that routes to two different services based on the URL path: `/api` → service `api-svc:8080` and `/web` → service `web-svc:80`. Use the host `app.example.com`.
```bash
❯ kubectl create deploy api-svc --image=nginx --dry-run=client -o yaml > api-nginx.yaml
❯ vim api-nginx.yaml
❯ cat api-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: api-svc
  name: api-svc
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api-svc
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: api-svc
    spec:
      initContainers:
      - image: busybox
        name: init
        command: ['/bin/sh', '-c', 'echo "API" > /work-dir/index.html']
        volumeMounts:
        - name: vol
          mountPath: /work-dir
      containers:
      - image: nginx
        name: nginx
        resources: {}
        volumeMounts:
        - name: vol
          mountPath: /usr/share/nginx/html
      volumes:
      - name: vol
        emptyDir: {}
status: {}
❯ kubectl create -f api-nginx.yaml
deployment.apps/api-svc created
❯ kubectl expose deploy api-svc --port=8080 --target-port=80
service/api-svc exposed
❯ cp api-nginx.yaml web-nginx.yaml
❯ vim web-nginx.yaml
❯ cat webapp-svc.yaml
cat: webapp-svc.yaml: No such file or directory
❯ cat web-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: web-svc
  name: web-svc
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-svc
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: web-svc
    spec:
      initContainers:
      - image: busybox
        name: init
        command: ['/bin/sh', '-c', 'echo "Web" > /work-dir/index.html']
        volumeMounts:
        - name: vol
          mountPath: /work-dir
      containers:
      - image: nginx
        name: nginx
        resources: {}
        volumeMounts:
        - name: vol
          mountPath: /usr/share/nginx/html
      volumes:
      - name: vol
        emptyDir: {}
status: {}
❯ kubectl create -f web-nginx.yaml
deployment.apps/web-svc created
❯ kubectl expose deploy web-svc --port=80 --target-port=80
service/web-svc exposed
❯ vim ingress.yaml
❯ cat ingress.yaml
❯ cat ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress
spec:
  ingressClassName: nginx-example
  rules:
  - http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 8080
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port:
              number: 80
    host: app.example.com

❯ kubectl create -f ingress.yaml
ingress.networking.k8s.io/ingress created
```
---
### Create an Ingress that uses a TLS secret. First create a TLS secret called `tls-secret` with a self-signed cert, then reference it in an Ingress for host `secure.example.com`.
```bash
❯ ls tls.*
tls.crt  tls.key
❯ kubectl create secret tls tls-secret --key=tls.key --cert=tls.crt
	secret/tls-secret created
❯ kubectl create secret tls tls-secret --key=tls.key --cert=tls.crt
secret/tls-secret created
❯ vim ingress.yaml
❯ cat ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-example-ingress
spec:
  tls:
  - hosts:
      - secure.example.com
    secretName: tls-secret
  rules:
  - host: https-example.foo.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 80

❯ kubectl create -f ingress.yaml
ingress.networking.k8s.io/tls-example-ingress created
```
>[!IMPORTANT]
>Ingress yaml definitions are readily available and easily findable in the kubernetes docs

---
### Update an existing Ingress `web-ingress` to add a new path rule `/v2` → `web-v2-svc:80` without recreating it.
```bash
❯ kubectl edit ingress tls-example-ingress
ingress.networking.k8s.io/tls-example-ingress edited
❯ kubectl get ingress tls-example-ingress -o yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  creationTimestamp: "2026-06-22T19:09:42Z"
  generation: 2
  name: tls-example-ingress
  namespace: default
  resourceVersion: "76681"
  uid: bf06ab4c-19e1-4212-8db4-6a1dd3585b82
spec:
  rules:
  - host: https-example.foo.com
    http:
      paths:
      - backend:
          service:
            name: service1
            port:
              number: 80
        path: /
        pathType: Prefix
      - backend:
          service:
            name: web-v2-svc
            port:
              number: 80
        path: /v2
        pathType: Prefix
  tls:
  - hosts:
    - secure.example.com
    secretName: tls-secret
status:
  loadBalancer: {}
```
---
### Create a NetworkPolicy in namespace `production` that denies all ingress traffic to all pods in the namespace (a default-deny ingress policy).
```bash
❯ vim default-deny-netpol.yaml
❯ cat default-deny-netpol.yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
❯ kubectl create -f default-deny-netpol.yaml
networkpolicy.networking.k8s.io/default-deny-ingress created
```
>[!IMPORTANT]
>Network Policies yaml definitions are easy to find in the kubernetes docs

---
### In namespace `production`, allow ingress to pods labelled `app=backend` only from pods labelled `app=frontend` in the same namespace, on port 8080.
```bash
❯ vim netpol.yaml
❯ cat netpol.yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: from-fe-to-be
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  policyTypes:
  - Ingress

❯ kubectl create -f netpol.yaml
networkpolicy.networking.k8s.io/from-fe-to-be created
```
>[!EXPLANATION]
>Here we define that all pods in the `production` namespace , which match the `app=backend` label are subject to an ingress rule, where ingress is only allowed from pods that match the `app=frontend` label and the port must be a TCP port on 8080. so, we are applying to:
>```yaml
>podSelector:
>	matchLabels:
>		app: backend
>``` 	
>the following ingress rule:
>```yaml
>ingress:
> - from:
>	- podSelector:
>			matchLabels:
>				app: frontend
>   ports:
>    - protocol: TCP
>	  port: 8080	
>```

---
### Allow ingress to pods labelled `app=database` in namespace `production` from pods in namespace `monitoring` (matched by `purpose=monitoring` label), on port 5432.
```bash
❯ vim netpol.yaml
❯ cat netpol.yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring-to-database
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring
      podSelector:
        matchLabels:
          purpose: monitoring
    ports:
    - protocol: TCP
      port: 5432

❯ kubectl create -f netpol.yaml
networkpolicy.networking.k8s.io/default-deny-ingress created
```
---
### Create a NetworkPolicy that denies all egress from pods labelled `app=restricted` in namespace `default`, except DNS (port 53 UDP/TCP).
```bash
❯ vim egress.yaml
❯ cat egress.yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
spec:
  podSelector:
    matchLabels:
      app: restricted
  egress:
  - ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  policyTypes:
  - Egress

❯ kubectl create -f egress.yaml
networkpolicy.networking.k8s.io/default-deny-egress created
```
---
### Create a combined NetworkPolicy for pods labelled `app=api` that: (1) allows ingress from pods with `role=frontend` on port 3000, and (2) allows egress only to pods with `app=database` on port 5432 and to DNS. Deny everything else.
```bash
❯ vim egress-ingress-netpol.yaml
❯ cat egress-ingress-netpol.yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: combined
spec:
  podSelector:
    matchLabels:
      app: api
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 3000
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
      namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53

  policyTypes:
  - Ingress
  - Egress
❯ kubectl create -f egress-ingress-netpol.yaml
networkpolicy.networking.k8s.io/combined created
```
---
### HARD EXERCISE FOR LEARNING
Create the following in namespace **`production`**.

You have four groups of pods:

| Pod group     | Labels         |
| ------------- | -------------- |
| API pods      | `app=api`      |
| Frontend pods | `app=frontend` |
| Database pods | `app=database` |
| Metrics pods  | `app=metrics`  |

There is also a namespace called **`monitoring`** with pods labelled:

```
purpose: monitoring
```


Task

Create **one or more NetworkPolicies** that enforce all of the following:

 1. API ingress

Pods labelled `app=api` must accept ingress **only** from:

- pods labelled `app=frontend` in the same namespace
- on TCP port `8080`

No other ingress to API pods should be allowed.

2. API egress

Pods labelled `app=api` must be allowed to send egress **only** to:

- pods labelled `app=database` in the same namespace on TCP port `5432`
- DNS on UDP/TCP port `53`

No other egress from API pods should be allowed.

3. Database ingress

Pods labelled `app=database` must accept ingress **only** from:

- pods labelled `app=api` in the same namespace
- on TCP port `5432`

No other ingress to database pods should be allowed.

4. Metrics ingress

Pods labelled `app=metrics` must accept ingress **only** from:

- pods labelled `purpose=monitoring`
- in namespace `monitoring`
- on TCP port `9090`

No other ingress to metrics pods should be allowed.

 5. Frontend egress

Pods labelled `app=frontend` must be allowed to send egress **only** to:

- pods labelled `app=api` in the same namespace
- on TCP port `8080`
- DNS on UDP/TCP port `53`

No other egress from frontend pods should be allowed.

```bash
❯ vim exercise.yaml
❯ cat exercise.yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: all
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  policyTypes:
  - Ingress
  - Egress

❯ kubectl create -f exercise.yaml
networkpolicy.networking.k8s.io/all created
❯ cp exercise.yaml 2-exercise.yaml
❯ vim 2-exercise.yaml
❯ kubectl create -f 2-exercise.yaml
networkpolicy.networking.k8s.io/all-database created
❯ cp exercise.yaml 3-exercise.yaml
❯ vim 3-exercise.yaml
❯ kubectl create -f 3-exercise.yaml
networkpolicy.networking.k8s.io/all-monitoring created
❯ cp exercise.yaml 4-exercise.yaml
❯ vim 4-exercise.yaml
❯ kubectl create -f 4-exercise.yaml
networkpolicy.networking.k8s.io/all-frontend created
❯ cat 2-exercise.yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: all-database
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: database
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 5432
  policyTypes:
  - Ingress

❯ cat 3-exercise.yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: all-monitoring
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: metrics
  ingress:
  - from:
    - podSelector:
        matchLabels:
          purpose: monitoring
      namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring
    ports:
    - protocol: TCP
      port: 9090
  policyTypes:
  - Ingress

❯ cat 4-exercise.yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: all-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: frontend
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 8080
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  policyTypes:
  - Egress
```
>[!NOTE]
>I did this all on my own in 8 minutes!

>[!IMPORTANT]
>When writing `netpol`'s , keep in mind the following. Lets say you need to allow ingress from either pods with `app=frontend`, or pods in the namespace `monitoring`, an OR yaml definition would be:
>```yaml
>ingress:
>- from:
>   - podSelector:
> 	   matchLabels:
> 		   app: frontend
>   - namespaceSelector:
> 	   matchLabels:
> 		   kubernetes.io/metadata.name: monitoring
>```
>while an AND definition would be
>```yaml
>ingress:
>- from:
>   - podSelector:
> 	  matchLabels:
> 		  app: frontend
>     namespaceSelector:
> 	   matchLabels:
> 		   kubernetes.io/metadata.name: monitoring
>```

---
