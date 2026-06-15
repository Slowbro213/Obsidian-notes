### Create an nginx pod with a liveness probe that just runs the command 'ls'. Save its YAML in pod.yaml. Run it, check its probe status, delete it.
```bash
❯ kubectl run nginx --image=nginx --restart=Never -o yaml --dry-run=client > alive-nginx.yaml
❯ vim alive-nginx.yaml
❯ cat alive-nginx.yaml
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
    resources: {}
    livenessProbe: # Liveness probe
      exec: # here we tell it to execute
        command: ['ls']  # here we list commands
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
❯ kubectl create -f alive-nginx.yaml
pod/nginx created
❯ kubectl describe pod nginx | grep -i liveness
    Liveness:       exec [ls] delay=0s timeout=1s period=10s #success=1 #failure=3
❯ kubectl delete pod nginx
pod "nginx" deleted
```
>[!NOTE]
>You can add `livenessProbe`'s to containers by adding
>```yaml
>livenessProbe:
>	exec:
>		command: ['commands', 'here']
>```
>To the container spec

>[!NOTE]
>You can check a pods `livenessProbe` through the describe command. you can also grep the result via `-i liveness`

---
### Modify the pod.yaml file so that liveness probe starts kicking in after 5 seconds whereas the interval between probes would be 5 seconds. Run it, check the probe, delete it.
```bash
❯ kubectl explain pod.spec.containers.livenessProbe
# we care about these two fields
  initialDelaySeconds   <integer>
    Number of seconds after the container has started before liveness probes are
    initiated. More info:
    https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#container-probes

  periodSeconds <integer>
    How often (in seconds) to perform the probe. Default to 10 seconds. Minimum
    value is 1.
    
❯ vim alive-nginx.yaml
❯ cat alive-nginx.yaml
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
    resources: {}
    livenessProbe: # Liveness probe
      initialDelaySeconds: 5
      periodSeconds: 5
      exec: # here we tell it to execute
        command: ['ls']  # here we list commands
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
❯ kubectl create -f alive-nginx.yaml
pod/nginx created
❯ kubectl describe pod nginx | grep -i liveness
    Liveness:       exec [ls] delay=5s timeout=1s period=5s
❯ kubectl delete pod nginx
pod "nginx" deleted
```
>[!IMPORTANT]
>USE `kubectl explain <resource>.attributes.to.explain` TO GET FIELD NAMES AND EXPLANATIONS

---
### Create an nginx pod (that includes port 80) with an HTTP readinessProbe on path '/' on port 80. Again, run it, check the readinessProbe, delete it.
```bash
❯ vim ready-nginx.yaml
❯ cat ready-nginx.yaml
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
    resources: {}
    ports: # add the port
    - containerPort: 80 #specify the container port
    readinessProbe: # readiness probe
      httpGet: # as an http request
        path: / # on path /
        port: 80 # on port 80
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
❯ kubectl create -f ready-nginx.yaml
pod/nginx created
❯ kubectl describe pod nginx | grep -i readiness
    Readiness:      http-get http://:80/ delay=0s timeout=1s period=10s #success=1 #failure=3
❯ kubectl delete pod nginx
pod "nginx" deleted
```
>[!NOTE]
>You can add `readinessProbe`'s that use http requests via:
>```yaml
>readinessProbe:
>	httpGet:
>		path: /
>		port: 80
>```

---
### Create a busybox pod that runs `i=0; while true; do echo "$i: $(date)"; i=$((i+1)); sleep 1; done`. Check its logs
```bash
❯ kubectl run busybox --image=busybox --restart=Never -it --rm --command -- /bin/sh -c 'i=0; while true; do echo "$i: $(date)"; i=$((i+1)); sleep 1; done'
1: Mon Jun 15 19:25:25 UTC 2026
2: Mon Jun 15 19:25:26 UTC 2026
3: Mon Jun 15 19:25:27 UTC 2026
4: Mon Jun 15 19:25:28 UTC 2026
5: Mon Jun 15 19:25:29 UTC 2026
6: Mon Jun 15 19:25:30 UTC 2026
7: Mon Jun 15 19:25:31 UTC 2026
8: Mon Jun 15 19:25:32 UTC 2026
9: Mon Jun 15 19:25:33 UTC 2026
10: Mon Jun 15 19:25:34 UTC 2026
11: Mon Jun 15 19:25:35 UTC 2026
12: Mon Jun 15 19:25:36 UTC 2026
13: Mon Jun 15 19:25:37 UTC 2026
^C^C
❯ kubectl logs busybox
0: Mon Jun 15 19:25:24 UTC 2026
1: Mon Jun 15 19:25:25 UTC 2026
2: Mon Jun 15 19:25:26 UTC 2026
3: Mon Jun 15 19:25:27 UTC 2026
4: Mon Jun 15 19:25:28 UTC 2026
5: Mon Jun 15 19:25:29 UTC 2026
6: Mon Jun 15 19:25:30 UTC 2026
7: Mon Jun 15 19:25:31 UTC 2026
8: Mon Jun 15 19:25:32 UTC 2026
9: Mon Jun 15 19:25:33 UTC 2026
10: Mon Jun 15 19:25:34 UTC 2026
11: Mon Jun 15 19:25:35 UTC 2026
12: Mon Jun 15 19:25:36 UTC 2026
13: Mon Jun 15 19:25:37 UTC 2026
^C
```
---
### Create a busybox pod that runs 'ls /notexist'. Determine if there's an error (of course there is), see it. In the end, delete the pod
```bash
❯ kubectl run busybox --image=busybox --restart=Never --command -- /bin/sh -c 'ls /notexist'
❯ kubectl logs busybox -f
ls: /notexist: No such file or directory
❯ kubectl get pod busybox
NAME      READY   STATUS   RESTARTS   AGE
busybox   0/1     Error    0          40s
❯ kubectl delete pod busybox
pod "busybox" deleted
```
---
### Create a busybox pod that runs 'notexist'. Determine if there's an error (of course there is), see it. In the end, delete the pod forcefully with a 0 grace period
```bash
❯ kubectl run busybox --image=busybox --restart=Never --command -- /bin/sh -c 'notexist'
pod/busybox created
❯ kubectl get pod busybox
NAME      READY   STATUS              RESTARTS   AGE
busybox   0/1     ContainerCreating   0          4s
❯ kubectl get pod busybox -w
NAME      READY   STATUS   RESTARTS   AGE
busybox   0/1     Error    0          7s
busybox   0/1     Error    0          8s
^C
❯ kubectl logs busybox -f
/bin/sh: line 0: notexist: not found
❯ kubectl delete pod busybox --force=true --grace-period=0
```
>[!NOTE]
>You can forcefully delete a pod with a 0 grace period via the `--force` and `grace-period=0` flags

---
### Get CPU/memory utilization for nodes ([metrics-server](https://github.com/kubernetes-incubator/metrics-server) must be running)
```bash
kubectl top nodes
```
