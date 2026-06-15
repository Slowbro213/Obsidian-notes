### Create a pod with image nginx called nginx and expose its port 80
```bash
❯ kubectl run nginx --image=nginx --restart=Never --port=80 --expose
service/nginx created
pod/nginx created
```
>[!NOTE]
>Do not confuse with `kubectl run nginx --image=nginx --port=80` , that command doesn't create a service, it only adds the 
>```yaml
>ports:
>- containerPort: 80
>   protocol: TCP
>```
>yaml to its manifest

---
### Confirm that ClusterIP has been created. Also check endpoints
```bash
❯ kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP   75m
nginx        ClusterIP   10.106.110.236   <none>        80/TCP    2m9s
❯ kubectl run busybox --image=busybox --restart=Never -it --rm --command -- /bin/sh -c 'wget -O- nginx'
Connecting to nginx (10.106.110.236:80)
writing to stdout
<!DOCTYPE html>
...
```
---
### Get service's ClusterIP, create a temp busybox pod and 'hit' that IP with wget
```bash
❯ kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP   78m
nginx        ClusterIP   10.106.110.236   <none>        80/TCP    4m56s
❯ kubectl run busybox --image=busybox --restart=Never -it --rm --command -- /bin/sh -c 'wget -qO- 10.106.110.236'
<!DOCTYPE html>
<html>
```
---
### Convert the ClusterIP to NodePort for the same service and find the NodePort port. Hit service using Node's IP. Delete the service and the pod at the end.
```bash
❯ kubectl edit svc nginx
service/nginx edited
❯ kubectl get svc -o yaml nginx
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2026-06-15T19:44:38Z"
  name: nginx
  namespace: default
  resourceVersion: "4492"
  uid: d780e998-682d-4328-919c-125183d9dbd8
spec:
  clusterIP: 10.106.110.236
  clusterIPs:
  - 10.106.110.236
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - nodePort: 31445
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: nginx
  sessionAffinity: None
  type: NodePort # i edited here
status:
  loadBalancer: {}
❯ kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP        80m
nginx        NodePort    10.106.110.236   <none>        80:31445/TCP   7m19s
❯ kubectl get nodes -o wide
NAME       STATUS   ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
minikube   Ready    control-plane   84m   v1.31.0   192.168.49.2   <none>        Ubuntu 22.04.4 LTS   6.18.2           docker://27.2.0
❯ wget -O- 192.168.49.2:31445
Prepended http:// to '192.168.49.2:31445'
--2026-06-15 21:55:24--  http://192.168.49.2:31445/
Connecting to 192.168.49.2:31445... connected.
HTTP request sent, awaiting response... 200 OK
Length: 896 [text/html]
Saving to: ‘STDOUT’
...
❯ kubectl delete svc nginx
service "nginx" deleted
❯ kubectl delete pod nginx
pod "nginx" deleted
```
>[!NOTE]
>When sending request to NodePort services, make sure to use the IP of the node the service is on, and to get that ip via `kubectl get nodes -o wide`

---
### Create a deployment called foo using image 'dgkanatsios/simpleapp' (a simple server that returns hostname) and 3 replicas. Label it as 'app=foo'. Declare that containers in this pod will accept traffic on port 8080 (do NOT create a service yet)
```bash
❯ kubectl create deployment foo --image=dgkanatsios/simpleapp --replicas=3 --port=8080
```
>[!NOTE]
>The flag `--labels` doesn't work on deployments. if you wish to edit the label you must do so after the fact

---
### Get the pod IPs. Create a temp busybox pod and try hitting them on port 8080
```bash
❯ kubectl describe deploy foo
Name:                   foo
Namespace:              default
CreationTimestamp:      Mon, 15 Jun 2026 22:01:01 +0200
Labels:                 app=foo
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=foo
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=foo
  Containers:
   simpleapp:
    Image:         dgkanatsios/simpleapp
    Port:          8080/TCP
    Host Port:     0/TCP
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   foo-6b747fc57f (3/3 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  78s   deployment-controller  Scaled up replica set foo-6b747fc57f to 3
❯ kubectl describe rs foo-6b747fc57f
Name:           foo-6b747fc57f
Namespace:      default
Selector:       app=foo,pod-template-hash=6b747fc57f
Labels:         app=foo
                pod-template-hash=6b747fc57f
Annotations:    deployment.kubernetes.io/desired-replicas: 3
                deployment.kubernetes.io/max-replicas: 4
                deployment.kubernetes.io/revision: 1
Controlled By:  Deployment/foo
Replicas:       3 current / 3 desired
Pods Status:    3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=foo
           pod-template-hash=6b747fc57f
  Containers:
   simpleapp:
    Image:         dgkanatsios/simpleapp
    Port:          8080/TCP
    Host Port:     0/TCP
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
  Normal  SuccessfulCreate  93s   replicaset-controller  Created pod: foo-6b747fc57f-lw57n
  Normal  SuccessfulCreate  93s   replicaset-controller  Created pod: foo-6b747fc57f-6bpvh
  Normal  SuccessfulCreate  93s   replicaset-controller  Created pod: foo-6b747fc57f-nm24n
❯ kubectl get pod foo-6b747fc57f-lw57n -o wide
NAME                   READY   STATUS    RESTARTS   AGE    IP            NODE       NOMINATED NODE   READINESS GATES
foo-6b747fc57f-lw57n   1/1     Running   0          107s   10.244.0.21   minikube   <none>           <none>
❯ kubectl run busybox --image=busybox --restart=Never -it --rm --command -- /bin/sh -c 'wget -qO- 10.244.0.21:8080'
Hello world from foo-6b747fc57f-lw57n and version 2.0
pod "busybox" deleted
```
---
### Create a service that exposes the deployment on port 6262. Verify its existence, check the endpoints
```bash
❯ kubectl expose deployment foo --port=6262 --target-port=8080
service/foo exposed
❯ kubectl get svc
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
foo          ClusterIP   10.97.19.73   <none>        6262/TCP   5s
❯ kubectl get svc foo
NAME   TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
foo    ClusterIP   10.97.19.73   <none>        6262/TCP   29s
❯ kubectl get endpoints foo
NAME   ENDPOINTS                                            AGE
foo    10.244.0.19:6262,10.244.0.20:6262,10.244.0.21:6262   36s
```
>[!NOTE]
>Use the `--target-port` flag to specify how the service port will forward traffic to the pod ports

>[!NOTE]
>You can get the `endpoints` for a service via `kubectl get endpoints <svc-name>`

---
### Create a temp busybox pod and connect via wget to foo service. Verify that each time there's a different hostname returned. Delete deployment and services to cleanup the cluster
```bash
service/foo edited
❯ kubectl run busybox --image=busybox --restart=Never -it --rm --command -- /bin/sh -c 'while true; do wget -qO- foo:6262; sleep 1; done'
If you don't see a command prompt, try pressing enter.
Hello world from foo-6b747fc57f-nm24n and version 2.0
Hello world from foo-6b747fc57f-nm24n and version 2.0
Hello world from foo-6b747fc57f-lw57n and version 2.0
Hello world from foo-6b747fc57f-lw57n and version 2.0
Hello world from foo-6b747fc57f-lw57n and version 2.0
Hello world from foo-6b747fc57f-nm24n and version 2.0
Hello world from foo-6b747fc57f-lw57n and version 2.0
Hello world from foo-6b747fc57f-lw57n and version 2.0
Hello world from foo-6b747fc57f-nm24n and version 2.0
^C^C%                                                                                                                                                                                          ❯ kubectl delete pod busybox
pod "busybox" deleted
❯ kubectl delete svc foo
service "foo" deleted
❯ kubectl delete deploy foo
deployment.apps "foo" deleted
```
---
### Create an nginx deployment of 2 replicas, expose it via a ClusterIP service on port 80. Create a NetworkPolicy so that only pods with labels 'access: granted' can access the pods in this deployment and apply it
```bash
❯ kubectl create deploy final-deploy --image=nginx --replicas=2 --port=80
deployment.apps/final-deploy created
❯ kubectl expose deploy final-deploy --port=80 --target-port=80
service/final-deploy exposed
❯ vim netpol.yaml
❯ cat netpol.yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: access-final-deploy # pick a name
spec:
  podSelector:
    matchLabels:
      app: final-deploy # selector for the pods
  ingress: # allow ingress traffic
  - from:
    - podSelector: # from pods
        matchLabels: # with this label
          access: granted

❯ kubectl create -f netpol.yaml
networkpolicy.networking.k8s.io/access-final-deploy created
```