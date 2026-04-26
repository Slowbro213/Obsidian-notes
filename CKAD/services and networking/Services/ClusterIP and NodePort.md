ClusterIP is the default Service type. Exposes the Service on a cluster-internal IP. Makes the Service reachable only from within the cluster. Standard for inter-microservice communication.

To better understand services, lets create one. the tutorial first creates a simple nginx deployment:

```bash
kubectl create deployment my-app --image=nginx --replicas=2
```

Now lets create a clusterip service declaretively:

```yaml
#clusterip-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  type: ClusterIP
  selector:
    app: my-app # This tells the service what pods its working with
  ports:
  - protocol: TCP
    # similar to 80:80 in docker, the port is the port on the service
    # the target port is the port on the container
    port: 80
    targetPort: 80
```

```bash
kubectl apply -f clusterip-service.yaml
```

Now we should test the connectivity to that service from within the cluster:

```bash
#After waiting for a bit it started working properly
vboxuser@k8s-init:~$ kubectl run tmp-shell -it --rm --image=busybox -- /bin/sh
If you don't see a command prompt, try pressing enter.
/ # wget -O- my-app-service
Connecting to my-app-service (10.105.118.221:80)
writing to stdout
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, nginx is successfully installed and working.
Further configuration is required for the web server, reverse proxy,
API gateway, load balancer, content cache, or other features.</p>

<p>For online documentation and support please refer to
<a href="https://nginx.org/">nginx.org</a>.<br/>
To engage with the community please visit
<a href="https://community.nginx.org/">community.nginx.org</a>.<br/>
For enterprise grade support, professional services, additional
security features and capabilities please refer to
<a href="https://f5.com/nginx">f5.com/nginx</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
-                    100% |***********************************************************************************************************************************************|   896  0:00:00 ETA
written to stdout
```



NodePort

A Nodeport service exposes a static port for an application on each node it runs on. It automatically routes traffic to a ClusterIP and is useful for when a loadbalancer isnt available. ports are chosen from the range 30000 - 32767. Lets create a nodeport service!

```yaml
#nodeport-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-nodeport
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

```bash
kubectl apply -f nodeport-service.yaml
```

Lets see the details of this service

```bash
~$ kubectl get service my-app-nodeport
NAME              TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
my-app-nodeport   NodePort   10.105.245.203   <none>        80:30192/TCP   62s
```

So our service seems to have exposed the deployment on port 30192. Given this information, we will get the ip address of the worker node one of these pods is running on, and well do a curl from the control plane using that IP and this port to check for connectivity:

```bash
vboxuser@k8s-init:~$ kubectl get pods -o wide
NAME                      READY   STATUS    RESTARTS   AGE     IP               NODE          NOMINATED NODE   READINESS GATES
my-app-5867cfb477-bfk7q   1/1     Running   0          20m     192.168.126.2    k8s-worker2   <none>           <none>
my-app-5867cfb477-wsw26   1/1     Running   0          20m     192.168.194.67   k8s-worker1   <none>           <none>
probe-demo                1/1     Running   0          3h43m   192.168.126.1    k8s-worker2   <none>           <none>
ssd-pod                   1/1     Running   0          115m    192.168.194.65   k8s-worker1   <none>           <none>
```
This tells us that one of the my-app pods is running on the worker1 node and the other on the worker2 node ( we have 2 replicas ), so either which IP should work here. for sake of testing we'll test for both. Now lets get the IPs of those two worker nodes:

```bash
vboxuser@k8s-init:~$ kubectl get nodes -o wide
NAME          STATUS   ROLES           AGE     VERSION    INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
k8s-init      Ready    control-plane   3h49m   v1.29.15   192.168.1.32   <none>        Ubuntu 24.04.3 LTS   6.8.0-110-generic   containerd://2.2.1
k8s-worker1   Ready    <none>          3h48m   v1.29.15   192.168.1.33   <none>        Ubuntu 24.04.3 LTS   6.8.0-110-generic   containerd://2.2.1
k8s-worker2   Ready    <none>          3h48m   v1.29.15   192.168.1.34   <none>        Ubuntu 24.04.3 LTS   6.8.0-110-generic   containerd://2.2.1
k8s-worker3   Ready    <none>          3h48m   v1.29.15   192.168.1.35   <none>        Ubuntu 24.04.3 LTS   6.8.0-110-generic   containerd://2.2.1
```

So the two IPs are 192.168.1.33-34 and the port is 30192. Now lets do a connectivity test:

```bash
vboxuser@k8s-init:~$ curl http://192.168.1.33:30192
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, nginx is successfully installed and working.
Further configuration is required for the web server, reverse proxy,
API gateway, load balancer, content cache, or other features.</p>

<p>For online documentation and support please refer to
<a href="https://nginx.org/">nginx.org</a>.<br/>
To engage with the community please visit
<a href="https://community.nginx.org/">community.nginx.org</a>.<br/>
For enterprise grade support, professional services, additional
security features and capabilities please refer to
<a href="https://f5.com/nginx">f5.com/nginx</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
vboxuser@k8s-init:~$ curl http://192.168.1.34:30192
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, nginx is successfully installed and working.
Further configuration is required for the web server, reverse proxy,
API gateway, load balancer, content cache, or other features.</p>

<p>For online documentation and support please refer to
<a href="https://nginx.org/">nginx.org</a>.<br/>
To engage with the community please visit
<a href="https://community.nginx.org/">community.nginx.org</a>.<br/>
For enterprise grade support, professional services, additional
security features and capabilities please refer to
<a href="https://f5.com/nginx">f5.com/nginx</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

So It worked on both!