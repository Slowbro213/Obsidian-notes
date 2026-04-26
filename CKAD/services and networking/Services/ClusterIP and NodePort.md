## ClusterIP

ClusterIP is the default Service type. Exposes the Service on a cluster-internal IP. Makes the Service reachable only from within the cluster. Standard for inter-microservice communication.

To better understand services, lets create one. the tutorial first creates a simple nginx deployment:

```bash
kubectl create deployment my-app --image=nginx --replicas=2
```

Now lets create a clusterip service declaratively:

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
# After waiting for a bit it started working properly
vboxuser@k8s-init:~$ kubectl run tmp-shell -it --rm --image=busybox -- /bin/sh
If you don't see a command prompt, try pressing enter.
/ # wget -O- my-app-service
Connecting to my-app-service (10.105.118.221:80)
writing to stdout
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</html>
```

---

## NodePort

A NodePort service exposes a static port for an application on each node it runs on. It automatically routes traffic to a ClusterIP and is useful for when a loadbalancer isnt available. ports are chosen from the range 30000 - 32767. Lets create a nodeport service!

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

Lets see the details of this service:

```bash
~$ kubectl get service my-app-nodeport
NAME              TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
my-app-nodeport   NodePort   10.105.245.203   <none>        80:30192/TCP   62s
```

So our service seems to have exposed the deployment on port 30192. Given this information, we will get the IP address of the worker node one of these pods is running on, and well do a curl from the control plane using that IP and this port to check for connectivity:

```bash
vboxuser@k8s-init:~$ kubectl get pods -o wide
NAME                      READY   STATUS    RESTARTS   AGE     IP               NODE
my-app-5867cfb477-bfk7q   1/1     Running   0          20m     192.168.126.2    k8s-worker2
my-app-5867cfb477-wsw26   1/1     Running   0          20m     192.168.194.67   k8s-worker1
```

This tells us that one of the my-app pods is running on the worker1 node and the other on the worker2 node (we have 2 replicas), so either IP should work here. for sake of testing we'll test for both. Now lets get the IPs of those two worker nodes:

```bash
vboxuser@k8s-init:~$ kubectl get nodes -o wide
NAME          STATUS   ROLES           AGE     VERSION    INTERNAL-IP
k8s-worker1   Ready    <none>          3h48m   v1.29.15   192.168.1.33
k8s-worker2   Ready    <none>          3h48m   v1.29.15   192.168.1.34
```

So the two IPs are 192.168.1.33–34 and the port is 30192. Now lets do a connectivity test:

```bash
vboxuser@k8s-init:~$ curl http://192.168.1.33:30192
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</html>

vboxuser@k8s-init:~$ curl http://192.168.1.34:30192
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</html>
```

So it worked on both!