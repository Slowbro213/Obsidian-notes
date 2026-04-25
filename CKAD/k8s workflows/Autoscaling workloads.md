Kubernetes can autoscale Pods based on resource usage. More accurately, it changes the `replicas` attribute of a Deployment based on the load of an application.

In order for Horizontal Pod Autoscalers (HPA) to know when they should scale the Pods, they need to pull metrics from the kubelet. This requires:

- A Metrics Server addon
- Containers with resource requests defined

We'll be installing the following Metrics Server like the tutorial does:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

## Fixing Metrics Server in a Local Cluster

Because we are in a locally running Kubernetes cluster, the API server cannot verify certificates and such, so the Metrics Server appears unavailable.

We need to tell the Metrics Server to skip this verification for testing purposes. For this, we need to edit the Deployment YAML for the `metrics-server` Deployment to include one more argument: `--kubelet-insecure-tls`.

```bash
kubectl edit deployment metrics-server -n kube-system
```

This command opens up a Vim TUI for editing the Deployment YAML of the `metrics-server` Deployment.

```yaml
...
  template:
    metadata:
      creationTimestamp: null
      labels:
        k8s-app: metrics-server
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=10250
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls # This arg
        image: registry.k8s.io/metrics-server/metrics-server:v0.8.1
...
```

Now it works:

```bash
vboxuser@k8s-init:~$ kubectl top nodes
NAME          CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
k8s-init      39m          1%     1778Mi          37%
k8s-worker1   7m           0%     758Mi           16%
k8s-worker2   9m           0%     744Mi           15%
k8s-worker3   9m           0%     736Mi           15%

vboxuser@k8s-init:~$ kubectl top pods -A
NAMESPACE     NAME                                      CPU(cores)   MEMORY(bytes)
default       nginx-deployment-6d797fb658-8875r         0m           2Mi
default       nginx-deployment-6d797fb658-j888v         0m           2Mi
default       nginx-deployment-6d797fb658-zpslf         0m           2Mi
kube-system   calico-kube-controllers-8d76c5f9b-qlcm4   1m           54Mi
kube-system   coredns-76f75df574-hfsdn                  1m           56Mi
kube-system   coredns-76f75df574-wkb5x                  2m           13Mi
kube-system   etcd-k8s-init                             10m          46Mi
kube-system   kube-apiserver-k8s-init                   25m          257Mi
kube-system   kube-controller-manager-k8s-init          7m           46Mi
kube-system   kube-proxy-9c6lh                          1m           12Mi
kube-system   kube-proxy-ftdtz                          1m           12Mi
kube-system   kube-proxy-h87zd                          1m           12Mi
kube-system   kube-proxy-w9d9q                          1m           12Mi
kube-system   kube-scheduler-k8s-init                   2m           18Mi
kube-system   metrics-server-6db6cd6674-ts9l2           2m           19Mi
```

---

## Creating the Demo Deployment

Now let's actually test this on a Deployment. The tutorial creates a `php-apache` Deployment like so:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  replicas: 1
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
        - name: php-apache
          # Image specifically made to consume CPU
          image: registry.k8s.io/hpa-example
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 200m # 20% of one CPU core
```

Now apply the Deployment:

```bash
kubectl apply -f hpa-demo-deployment.yaml
```

---

## Exposing the Deployment as a Service

Now that we have the Deployment running, we need to expose it as a **Service**.

```bash
kubectl expose deployment php-apache --port=80
```

Although the word `service` isn't found anywhere here, this command exposes the Deployment as a Service:

```bash
kubectl get services
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP   9h
php-apache   ClusterIP   10.97.207.97   <none>        80/TCP    2m25s
```

---

## Creating the Horizontal Pod Autoscaler

Now we need to tell Kubernetes to autoscale this Deployment when CPU utilization reaches `50%`.

The previous `200m` we configured for the Pod means that the Pod can at most consume `20%` of the CPU of the node, and the `50%` of the HPA is `50%` of the `20%`.

So when the Pod consumes `10%` of the node's CPU, it will be horizontally scaled to other nodes.

In this command:

- `--cpu-percent` is the metric
- `--min` is the minimum number of replicas
- `--max` is the maximum number of replicas

```bash
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```

---

## Observing Autoscaling in Action

Now we can observe the autoscaling in action:

```bash
vboxuser@k8s-init:~$ kubectl run load-generator --image=busybox -- /bin/sh -c "while true; do wget -q -O- http://php-apache; done"
pod/load-generator created

vboxuser@k8s-init:~$ kubectl get hpa -w
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/50%    1         10        6          7m14s
php-apache   Deployment/php-apache   21%/50%   1         10        6          7m30s
php-apache   Deployment/php-apache   65%/50%   1         10        6          7m45s
php-apache   Deployment/php-apache   82%/50%   1         10        8          8m
php-apache   Deployment/php-apache   35%/50%   1         10        10         8m15s
```
