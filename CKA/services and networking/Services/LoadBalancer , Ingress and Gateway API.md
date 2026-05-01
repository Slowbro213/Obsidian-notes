## LoadBalancers

- Exposes the Service externally using a cloud provider’s LoadBalancer.
- The standard way to expose a service to the internet in a cloud environment.
- Automatically creates NodePort and ClusterIP services.
- The cloud provider provisions a load balancer and assigns an external IP.

---

# Ingress & Gateway API

## Overview

Ingress and Gateway API manage external access more efficiently than LoadBalancer services.

They provide **L7 (HTTP/HTTPS) routing**.

---

# Ingress

## What Ingress Does

Ingress manages external access and provides:

- SSL
- Load balancing
- Name-based virtual hosting

Ingress requires an **IngressController** to run in the cluster, such as:

- NGINX
- Traefik
- HAProxy

The controller watches for Ingress resources and configures the proxy accordingly.

---

## Creating an Ingress for Path-Based Routing

Now let’s create an Ingress for our services in order to make use of path-based routing.

The tutorial uses the popular but now deprecated NGINX Ingress Controller:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
```

---

## Creating Two Applications

Now we’ll create two applications.

This app is just an echo server:

```bash
kubectl create deployment app-one --image=k8s.gcr.io/echoserver:1.4
kubectl expose deployment app-one --port=8080

kubectl create deployment app-two --image=k8s.gcr.io/echoserver:1.4
kubectl expose deployment app-two --port=8080
```

---

## Creating the Ingress Resource

Now we’ll create an Ingress Resource where we will define our routing rules.

The Kubernetes docs provide the following YAML:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: app-one
            port:
              number: 8080
      - path: /app2
        pathType: Prefix
        backend:
          service:
            name: app-two
            port:
              number: 8080
```

The tutorial changes the name of the Resource, adds the annotations part, and changes the paths to work with our example.

The paths are changed by adding one more path, then setting the routing rule to be based on the path prefix, with the paths being `/app1` and `/app2`.

Then we set the rule that each path will route the request to `app-one` and the other to `app-two`.

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: app-one
            port:
              number: 8080
      - path: /app2
        pathType: Prefix
        backend:
          service:
            name: app-two
            port:
              number: 8080
```

Apply the Ingress resource:

```bash
kubectl apply -f ingress.yaml
```

---

## Testing the Ingress

Now we need to test it.

The Ingress has created a NodePort and ClusterIP service, as such we can test it just like we did before.

I’m not laying out the steps one by one again, but I’ll paste the entire flow.

---

# Troubleshooting: ImagePullBackOff

> [!IMPORTANT]
> It seems the deployment wasn’t working because, according to ChatGPT, the image format is outdated.
>
> I confirmed that there was a problem pulling the image myself without ChatGPT.

## 1. Notice They Aren’t Running

```bash
vboxuser@k8s-init:~$ kubectl get pods -o wide
NAME                       READY   STATUS             RESTARTS   AGE     IP                NODE          NOMINATED NODE   READINESS GATES
app-one-5d67bcb4f-b8qwn    0/1     ImagePullBackOff   0          14m     192.168.100.199   k8s-worker3   <none>           <none>
app-two-779c8786f5-fvm7w   0/1     ImagePullBackOff   0          13m     192.168.126.4     k8s-worker2   <none>           <none>
```

## 2. Read Logs From Both

```bash
vboxuser@k8s-init:~$ kubectl logs app-one-5d67bcb4f-b8qwn
Error from server (BadRequest): container "echoserver" in pod "app-one-5d67bcb4f-b8qwn" is waiting to start: trying and failing to pull image

vboxuser@k8s-init:~$ kubectl logs app-two-779c8786f5-fvm7w
Error from server (BadRequest): container "echoserver" in pod "app-two-779c8786f5-fvm7w" is waiting to start: trying and failing to pull image
```

## 3. Read the Description of One of Them

```bash
kubectl describe pod app-one-5d67bcb4f-b8qwn

Events:
  Type     Reason     Age                   From               Message
  ----     ------     ----                  ----               -------
  Normal   Scheduled  14m                   default-scheduler  Successfully assigned default/app-one-5d67bcb4f-b8qwn to k8s-worker3
  Warning  Failed     13m (x6 over 14m)     kubelet            Error: ImagePullBackOff
  Normal   Pulling    12m (x4 over 14m)     kubelet            Pulling image "k8s.gcr.io/echoserver:1.4"
  Warning  Failed     12m (x4 over 14m)     kubelet            Failed to pull image "k8s.gcr.io/echoserver:1.4": rpc error: code = InvalidArgument desc = failed to pull and unpack image "k8s.gcr.io/echoserver:1.4": schema 1 image manifests are no longer supported: invalid argument
  Warning  Failed     12m (x4 over 14m)     kubelet            Error: ErrImagePull
  Normal   BackOff    4m22s (x43 over 14m)  kubelet            Back-off pulling image "k8s.gcr.io/echoserver:1.4"
```

---

## Updating the Image

So now we need to do an update on both of them in order to update the image.

We have done this before when editing the metrics-server to add an additional argument, so we know how to do it again.

```bash
kubectl edit deployment app-one
```

And edit this:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: "2026-04-26T16:40:58Z"
  generation: 1
  labels:
    app: app-one
  name: app-one
  namespace: default
  resourceVersion: "27943"
  uid: d96d6e9d-7fd7-454b-bbf8-0dd406fae2c6
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: app-one
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: app-one
    spec:
      containers:
      - image: registry.k8s.io/echoserver:1.10 # changed the image source
        imagePullPolicy: IfNotPresent
```

Now we can see that this worked:

```bash
vboxuser@k8s-init:~$ kubectl get pods -o wide
NAME                       READY   STATUS             RESTARTS   AGE     IP               NODE          NOMINATED NODE   READINESS GATES
app-one-7494894cfc-m96w2   1/1     Running            0          5m7s    192.168.194.68   k8s-worker1   <none>           <none>
app-two-779c8786f5-fvm7w   0/1     ImagePullBackOff   0          24m     192.168.126.4    k8s-worker2   <none>           <none>
```

Now we’ll just do the same for the other deployment, but this time we’ll do it through a rolling update — the right way.

```bash
vboxuser@k8s-init:~$ kubectl set image deployment/app-two echoserver=registry.k8s.io/echoserver:1.10
deployment.apps/app-two image updated
vboxuser@k8s-init:~$ kubectl rollout status deployment/app-two
Waiting for deployment "app-two" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "app-two" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "app-two" rollout to finish: 1 old replicas are pending termination...
deployment "app-two" successfully rolled out
```

---

# Testing After Both Deployments Are Updated

Now that both deployments are updated, let’s run a test to see if our Ingress Controller worked:

```bash
vboxuser@k8s-init:~$ kubectl get svc app-one app-two -o wide
NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE   SELECTOR
app-one   ClusterIP   10.101.121.1    <none>        8080/TCP   31m   app=app-one
app-two   ClusterIP   10.105.199.14   <none>        8080/TCP   31m   app=app-two

vboxuser@k8s-init:~$ kubectl get service app-one app-two -o wide
NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE   SELECTOR
app-one   ClusterIP   10.101.121.1    <none>        8080/TCP   32m   app=app-one
app-two   ClusterIP   10.105.199.14   <none>        8080/TCP   31m   app=app-two

vboxuser@k8s-init:~$ kubectl get services app-one app-two -o wide
NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE   SELECTOR
app-one   ClusterIP   10.101.121.1    <none>        8080/TCP   32m   app=app-one
app-two   ClusterIP   10.105.199.14   <none>        8080/TCP   31m   app=app-two

vboxuser@k8s-init:~$ kubectl get services app-one app-two -o wide
NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE   SELECTOR
app-one   ClusterIP   10.101.121.1    <none>        8080/TCP   32m   app=app-one
app-two   ClusterIP   10.105.199.14   <none>        8080/TCP   31m   app=app-two

vboxuser@k8s-init:~$ kubectl get pod -l app=app-one
NAME                       READY   STATUS    RESTARTS   AGE
app-one-7494894cfc-m96w2   1/1     Running   0          12m

vboxuser@k8s-init:~$ kubectl get pod -l app=app-two
NAME                      READY   STATUS    RESTARTS   AGE
app-two-544df6df5-7g28v   1/1     Running   0          4m59s

vboxuser@k8s-init:~$ kubectl get services app-one app-two -o wide
NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE   SELECTOR
app-one   ClusterIP   10.101.121.1    <none>        8080/TCP   32m   app=app-one
app-two   ClusterIP   10.105.199.14   <none>        8080/TCP   32m   app=app-two

vboxuser@k8s-init:~$ kubectl get pod -l app=app-one -o wide
NAME                       READY   STATUS    RESTARTS   AGE   IP               NODE          NOMINATED NODE   READINESS GATES
app-one-7494894cfc-m96w2   1/1     Running   0          12m   192.168.194.68   k8s-worker1   <none>           <none>

vboxuser@k8s-init:~$ kubectl get pod -l app=app-two -o wide
NAME                      READY   STATUS    RESTARTS   AGE     IP                NODE          NOMINATED NODE   READINESS GATES
app-two-544df6df5-7g28v   1/1     Running   0          5m14s   192.168.100.200   k8s-worker3   <none>           <none>

vboxuser@k8s-init:~$ kubectl get node k8s-worker3 -o wide
NAME          STATUS   ROLES    AGE     VERSION    INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
k8s-worker3   Ready    <none>   4h48m   v1.29.15   192.168.1.35   <none>        Ubuntu 24.04.3 LTS   6.8.0-110-generic   containerd://2.2.1

vboxuser@k8s-init:~$ kubectl get node k8s-worker1 -o wide
NAME          STATUS   ROLES    AGE     VERSION    INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
k8s-worker1   Ready    <none>   4h49m   v1.29.15   192.168.1.33   <none>        Ubuntu 24.04.3 LTS   6.8.0-110-generic   containerd://2.2.1

vboxuser@k8s-init:~$ kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.96.1.211      <pending>     80:32542/TCP,443:31414/TCP   38m
ingress-nginx-controller-admission   ClusterIP      10.103.222.201   <none>        443/TCP                      38m

vboxuser@k8s-init:~$ curl 192.168.1.35:32542/app2


Hostname: app-two-544df6df5-7g28v

Pod Information:
        -no pod information available-

Server values:
        server_version=nginx: 1.13.3 - lua: 10008

Request Information:
        client_address=192.168.100.198
        method=GET
        real path=/
        query=
        request_version=1.1
        request_scheme=http
        request_uri=http://192.168.1.35:8080/

Request Headers:
        accept=*/*
        host=192.168.1.35:32542
        user-agent=curl/8.5.0
        x-forwarded-for=192.168.1.32
        x-forwarded-host=192.168.1.35:32542
        x-forwarded-port=80
        x-forwarded-proto=http
        x-forwarded-scheme=http
        x-real-ip=192.168.1.32
        x-request-id=9dc3b00ea71efa45c7ff6407ef2c6f3b
        x-scheme=http

Request Body:
        -no body in request-
```

---

# Troubleshooting: ExternalTrafficPolicy

Now I encountered an issue when testing:

```bash
curl http://192.168.1.33:32542
```

This is due to the fact that the Ingress Controller has its `ExternalTrafficPolicy` set to `Local`, which means it can only route to the same node it’s running on.

We need to change it to `Cluster` so that it can route to the other node as well:

```bash
kubectl patch svc ingress-nginx-controller -n ingress-nginx -p '{"spec":{"externalTrafficPolicy":"Cluster"}}'

service/ingress-nginx-controller patched
```

Now both of them work.

---

## Working Test Output

```bash
curl 192.168.1.33:32542/app1


Hostname: app-one-7494894cfc-m96w2

Pod Information:
        -no pod information available-

Server values:
        server_version=nginx: 1.13.3 - lua: 10008

Request Information:
        client_address=192.168.100.198
        method=GET
        real path=/
        query=
        request_version=1.1
        request_scheme=http
        request_uri=http://192.168.1.33:8080/

Request Headers:
        accept=*/*
        host=192.168.1.33:32542
        user-agent=curl/8.5.0
        x-forwarded-for=192.168.194.64
        x-forwarded-host=192.168.1.33:32542
        x-forwarded-port=80
        x-forwarded-proto=http
        x-forwarded-scheme=http
        x-real-ip=192.168.194.64
        x-request-id=94b9df455834586db971b745b805635e
        x-scheme=http

Request Body:
        -no body in request-



curl 192.168.1.35:32542/app2


Hostname: app-two-544df6df5-7g28v

Pod Information:
        -no pod information available-

Server values:
        server_version=nginx: 1.13.3 - lua: 10008

Request Information:
        client_address=192.168.100.198
        method=GET
        real path=/
        query=
        request_version=1.1
        request_scheme=http
        request_uri=http://192.168.1.35:8080/

Request Headers:
        accept=*/*
        host=192.168.1.35:32542
        user-agent=curl/8.5.0
        x-forwarded-for=192.168.1.35
        x-forwarded-host=192.168.1.35:32542
        x-forwarded-port=80
        x-forwarded-proto=http
        x-forwarded-scheme=http
        x-real-ip=192.168.1.35
        x-request-id=41bc629af0206dc143ee8f0fddf4b813
        x-scheme=http

Request Body:
        -no body in request-
```

---

# Gateway API

Gateway API is the next generation of Ingress: more flexible and role-oriented.

It decouples configuration into three resource types:

| Resource | Role | Description |
|---|---|---|
| GatewayClass | Admin | A template for a type of load-balancer. |
| Gateway | Operator | Defines where and how the load-balancer listens. |
| HTTPRoute | Developer | Defines protocol-specific routing rules. |

This separation of rules enhances security and modularity.
