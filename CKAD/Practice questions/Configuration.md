### Create a configmap named config with values foo=lala,foo2=lolo
```bash
kubectl create configmap config --from-literal=foo=lala --from-literal=foo2=lolo
```
>[!NOTE]
>You can specify the values of a configmap from the command line with `--from-literal=key=value`

---
### Display its values
```bash
❯ kubectl get configmap config -o yaml
# or
❯ kubectl get configmap config -o jsonpath={.data}
{"foo":"lala","foo2":"lolo"}
```
---
### Create and display a configmap from a file
```bash
❯ echo 'foo3=lala\nfoo4=lolo' > file-of-configmap
❯ kubectl create configmap conf-from-file --from-file=file-of-configmap
configmap/conf-from-file created
❯ kubectl get configmap conf-from-file -o jsonpath={.data}
{"file-of-configmap":"foo3=lala\nfoo4=lolo\n"}
```
---
### Create and display a configmap from a .env file
```bash
❯ kubectl create configmap conf-from-env-file --from-env-file=./file-of-configmap
configmap/conf-from-env-file created
❯ kubectl get configmap conf-from-env-file -o jsonpath={.data}
{"foo3":"lala","foo4":"lolo"}
```
---
### Create and display a configmap from a file, giving the key 'special'
```bash
❯ echo 'foo3=lala\nfoo4=lolo' > config-special.txt
❯ kubectl create configmap special-config --from-file=special=./config-special.txt
configmap/special-config created
❯ kubectl get cm special-config -o jsonpath={.data}
{"special":"foo3=lala\nfoo4=lolo\n"}
```
>[!NOTE]
>When using `--from-file` normally the key to the value is the name of the file, but you can override that by using it like `--from-file=<override>=<filepath>`

---
### Create a configMap called 'options' with the value var5=val5. Create a new nginx pod that loads the value from variable 'var5' in an env variable called 'option'
```bash
❯ kubectl run nginx --image=nginx --restart=Never --dry-run=client -o yaml > nginx-pod-option-cm.yaml
❯ vim nginx-pod-option-cm.yaml
❯ cat nginx-pod-option-cm.yaml
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
    env:
    - name: option # Name of the environment variable
      valueFrom:
        configMapKeyRef: # Here we specify the value of the `option` env var will be taken from a configMap ( secrets can provide values too )
          name: options # Name of configMap
          key: var5 # Name of key within configMap
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
❯ kubectl create -f nginx-pod-option-cm.yaml
pod/nginx created
```
>[!IMPORTANT]
> To load values from configMaps to containers, in the container manifest in a pod , specify `env.name` for the name of the key to be used as an env var, `env.valueFrom.configMapKeyRef.name` for the name of the configMap to be used and `env.valueFrom.configMapKeyRef.key` for the name of the key in that configMap the value of which will be used. This is essentially creating an env variable in the container, and mapping it to another variable in a configMap

---
### Create a configMap 'anotherone' with values 'var6=val6', 'var7=val7'. Load this configMap as env variables into a new nginx pod
```bash
❯ kubectl create configmap anotherone --from-literal=var6=val6 --from-literal=var7=val7
configmap/anotherone created
❯ kubectl run nginx --image=nginx --restart=Never --dry-run=client -o yaml > nginx-pod-anotherone-cm.yaml
❯ vim nginx-pod-anotherone-cm.yaml
❯ cat nginx-pod-anotherone-cm.yaml
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
    envFrom: # This gets all env variables from the following configMaps or secrets
    - configMapRef: # This specifies the reference to the configMap
        name: anotherone # This is the name of the configMap in use
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
❯ kubectl create -f nginx-pod-anotherone-cm.yaml
pod/nginx created
❯ kubectl exec -it nginx -- env | grep var
var6=val6
var7=val7
```
>[!NOTE]
>You can add to the env variables of a container by loading all keys and values of a configMap using `envFrom.configMapRef.name` ( remember , envFrom takes a list ) and name is the name of the configMap to be loaded

---
### Create a configMap 'cmvolume' with values 'var8=val8', 'var9=val9'. Load this as a volume inside an nginx pod on path '/etc/lala'. Create the pod and 'ls' into the '/etc/lala' directory.
```bash
❯ kubectl run nginx --image=nginx --restart=Never --dry-run=client -o yaml > nginx-pod-volume-cm.yaml
❯ vim nginx-pod-volume-cm.yaml
❯ cat nginx-pod-volume-cm.yaml
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
    volumeMounts:
    - name: cmvolume
      mountPath: "/etc/lala"
  dnsPolicy: ClusterFirst
  restartPolicy: Never
  volumes:
  - name: cmvolume
    configMap:
      name: cmvolume
status: {}
❯ kubectl create -f nginx-pod-volume-cm.yaml
pod/nginx created
```
>[!NOTE]
>You can create volumes from configMaps like so:
>```yaml
>volumes:
>- name: volName
>   configMap:
>    name: cmName
>```

---
### Create the YAML for an nginx pod that runs with the user ID 101. No need to create the pod
```bash
❯ kubectl run nginx --image=nginx --restart=Never --dry-run=client -o yaml > nginx-pod-userID.yaml
❯ vim nginx-pod-userID.yaml
 cat nginx-pod-userID.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  securityContext:
    runAsUser: 101
  containers:
  - image: nginx
    name: nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```
>[!NOTE]
>You can specify the userID the pod will run with via `securityContext.runAsUser`

---
### Create the YAML for an nginx pod that has the capabilities "NET_ADMIN", "SYS_TIME" added to its single container
```bash
❯ kubectl run nginx --image=nginx --restart=Never --dry-run=client -o yaml > capable-nginx.yaml
❯ vim capable-nginx.yaml
❯ cat capable-nginx.yaml
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
    securityContext: # Securitycontext for the container itself not just the pod
      capabilities: # capabilites , they can be added or dropped
        add: ["NET_ADMIN","SYS_TIME"] # specifying the capabilites to add
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```
>[!IMPORTANT]
>You can add or drop capabilites of a `container` by specifying its `securityContext.capabilites.add` or `.drop`. This is different from the pod securityContext

---
### Create an nginx pod with requests cpu=100m,memory=256Mi and limits cpu=200m,memory=512Mi

```bash
❯ kubectl run nginx --image=nginx --restart=Never --dry-run=client -o yaml > nginx-requests-limits.yaml
❯ vim nginx-requests-limits.yaml
❯ cat nginx-requests-limits.yaml
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
    resources:
      requests:
        memory: 256Mi
        cpu: 100m
      limits:
        memory: 512Mi
        cpu: 200m
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
❯ kubectl create -f nginx-requests-limits.yaml
pod/nginx created
```
>[!NOTE]
>In order to specify requests ( which is what resources the pod wants from the cluster ) and the limits ( which is what the cluster forces the pod to stay under ) you need to simply specify in the container of the pod the following:
>```yaml
>resources:
>	requests:
>		memory: 256Mi
>		cpu: 100m
>	limit:
>		memory: 512Mi
>		cpu: 200m
>```

---
### Create a namespace named limitrange with a LimitRange that limits pod memory to a max of 500Mi and min of 100Mi
```bash
❯ kubectl create ns limitrange
# There doesnt seem to be a way to create a LimitRange using the cli, so you have to use the kubernetes docs to get the yaml definition of a LimitRange
❯ vim memory-limit-range.yaml
❯ cat memory-limit-range.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: memory-limit-range
  namespace: limitrange
spec:
  limits:
  - max:
      memory: 500Mi
    min:
      memory: 100Mi
    type: Pod
❯ kubectl create -f memory-limit-range.yaml
limitrange/memory-limit-range created
```
>[!IMPORTANT]
>kubectl create command doesn't seem to be able to create LimitRanges , so you need to get the yaml defintion for an example of one from the kubernetes docs. there are 3 main parts of it ( at least in this excercise ). the `limits` which takes an array of limits, each limit in this example has a `max`, `min` and a `type`. The latter specifies what resource is constrained by these limits in this namespace

---
### Describe the namespace limitrange
```bash
❯ kubectl describe ns limitrange
Name:         limitrange
Labels:       kubernetes.io/metadata.name=limitrange
Annotations:  <none>
Status:       Active

No resource quota.

Resource Limits
 Type  Resource  Min    Max    Default Request  Default Limit  Max Limit/Request Ratio
 ----  --------  ---    ---    ---------------  -------------  -----------------------
 Pod   memory    100Mi  500Mi  -                -              -
```
---
### Create an nginx pod that requests 250Mi of memory in the limitrange namespace
```bash
❯ kubectl run nginx --image=nginx --restart=Never --dry-run=client -o yaml > nginx-requests-in-limitrange.yaml -n limitrange
❯ vim nginx-requests-in-limitrange.yaml
❯ cat nginx-requests-in-limitrange.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
  namespace: limitrange
spec:
  containers:
  - image: nginx
    name: nginx
    resources:
      requests:
        memory: 250Mi
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
❯ kubectl create -f nginx-requests-in-limitrange.yaml
Error from server (Forbidden): error when creating "nginx-requests-in-limitrange.yaml": pods "nginx" is forbidden: maximum memory usage per Pod is 500Mi.  No limit is specified
❯ vim nginx-requests-in-limitrange.yaml
❯ cat nginx-requests-in-limitrange.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
  namespace: limitrange
spec:
  containers:
  - image: nginx
    name: nginx
    resources:
      requests:
        memory: 250Mi
      limits:
        memory: 500Mi
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
❯ kubectl create -f nginx-requests-in-limitrange.yaml
pod/nginx created
```
---
### Create ResourceQuota in namespace `one` with hard requests `cpu=1`, `memory=1Gi` and hard limits `cpu=2`, `memory=2Gi`.
```bash
❯ kubectl create resourcequota one-rq -n one --hard=cpu=2,memory=2Gi --dry-run=client -o yaml > one-rq.yaml
❯ vim one-rq.yaml
❯ cat one-rq.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  creationTimestamp: null
  name: one-rq
  namespace: one
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
status: {}
❯ kubectl create -f one-rq.yaml
resourcequota/one-rq created
```
>[!NOTE]
>For some reason, in `ResourceQuota` resources dont behave like other resources, and use this flat `requests.<resource>` and `limits.<resource>` for each resource

---
### Attempt to create a pod with resource requests `cpu=2`, `memory=3Gi` and limits `cpu=3`, `memory=4Gi` in namespace `one`
```bash
❯ kubectl run nginx --image=nginx --restart=Never --dry-run=client -o yaml > nginx-requests-limits-one.yaml -n one
❯ vim nginx-requests-limits-one.yaml
❯ cat nginx-requests-limits-one.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
  namespace: one
spec:
  containers:
  - image: nginx
    name: nginx
    resources:
      requests:
        cpu: 2
        memory: 3Gi
      limits:
        cpu: 3
        memory: 4Gi
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
❯ kubectl create -f nginx-requests-limits-one.yaml
Error from server (Forbidden): error when creating "nginx-requests-limits-one.yaml": pods "nginx" is forbidden: exceeded quota: one-rq, requested: limits.cpu=3,limits.memory=4Gi,requests.cpu=2,requests.memory=3Gi, used: limits.cpu=0,limits.memory=0,requests.cpu=0,requests.memory=0, limited: limits.cpu=2,limits.memory=2Gi,requests.cpu=1,requests.memory=1Gi
```
---
### Create a pod with resource requests `cpu=0.5`, `memory=1Gi` and limits `cpu=1`, `memory=2Gi` in namespace `one`
```bash
❯ vim nginx-requests-limits-one.yaml
❯ cat nginx-requests-limits-one.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
  namespace: one
spec:
  containers:
  - image: nginx
    name: nginx
    resources:
      requests:
        cpu: 0.5
        memory: 1Gi
      limits:
        cpu: 1
        memory: 2Gi
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
❯ kubectl create -f nginx-requests-limits-one.yaml
pod/nginx created
```
---
### Create a secret called mysecret with the values password=mypass
```bash
❯ kubectl create secret generic mysecret --from-literal=password=mypass
secret/mysecret created
```
>[!NOTE]
>When creating secrets you need to specify the type. When using the command line the `--help` flag will tell you more info. in this case i used `generic`. At least `generic` secrets share much in common with ConfigMaps

---
### Create a secret called mysecret2 that gets key/value from a file
```bash
❯ echo "slowking213" > username
❯ kubectl create secret generic mysecret2 --from-file=username
❯ kubectl get secret mysecret2 -o jsonpath={.data}
{"username":"c2xvd2tpbmcyMTMK"}
```
---
### Get the value of mysecret2
```bash
❯ kubectl get secret mysecret2 -o jsonpath={.data}
{"username":"c2xvd2tpbmcyMTMK"}
```
---
### Create an nginx pod that mounts the secret mysecret2 in a volume on path /etc/foo
```bash
❯ kubectl run nginx --image=nginx --restart=Never --dry-run=client -o yaml > nginx-secret-volume.yaml
❯ vim nginx-secret-volume.yaml
❯ cat nginx-secret-volume.yaml
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
    volumeMounts:
    - name: secret-vol
      mountPath: /etc/foo
  dnsPolicy: ClusterFirst
  restartPolicy: Never
  volumes:
  - name: secret-vol
    secret:
      secretName: mysecret2
status: {}
❯ kubectl create -f nginx-secret-volume.yaml
pod/nginx created
```
>[!NOTE]
>This is almost identical to the ConfigMap one, but when defining the volume, you should use `secret.secretName` where with ConfigMap you use `configMap.name`

---
### Delete the pod you just created and mount the variable 'username' from secret mysecret2 onto a new nginx pod in env variable called 'USERNAME'
```bash
❯ kubectl run nginx --image=nginx --restart=Never --dry-run=client -o yaml > nginx-secret-env-mount.yaml
❯ vim nginx-secret-env-mount.yaml
❯ cat nginx-secret-env-mount.yaml
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
    env:
    - name: USERNAME
      valueFrom:
        secretKeyRef:
          name: mysecret2
          key: username
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
❯ kubectl create -f nginx-secret-env-mount.yaml
pod/nginx created
```
>[!NOTE]
>I expected that under secretKeyRef i'd be using `secretName` instead of `name` due to the use of `secretName` in the volume creation question

---
### Create a Secret named 'ext-service-secret' in the namespace 'secret-ops'. Then, provide the key-value pair API_KEY=LmLHbYhsgWZwNifiqaRorH8T as literal.
```bash
❯ kubectl create ns secret-ops
❯ kubectl create secret generic ext-service-secret --from-literal=API_KEY=LmLHbYhsgWZwNifiqaRorH8T -n secret-ops --dry-run=client -o yaml > ext-service-secret.yaml
❯ vim ext-service-secret.yaml
❯ cat ext-service-secret.yaml
apiVersion: v1
data:
  API_KEY: LmLHbYhsgWZwNifiqaRorH8T # Edit this line
kind: Secret
metadata:
  creationTimestamp: null
  name: ext-service-secret
  namespace: secret-ops
❯ kubectl create -f ext-service-secret.yaml
secret/ext-service-secret created
```
>[!NOTE] 
>Using `--from-literal` when creating a generic secret automatically encodes it with base64. if you want the secret to literally hold a key, you have to create the secret via a yaml file

---
### Consuming the Secret. Create a Pod named 'consumer' with the image 'nginx' in the namespace 'secret-ops' and consume the Secret as an environment variable. Then, open an interactive shell to the Pod, and print all environment variables.
```bash
❯ kubectl run consumer --image=nginx --restart=Never -n secret-ops --dry-run=client -o yaml > nginx-secret-consume.yaml
❯ vim nginx-secret-consume.yaml
❯ cat nginx-secret-consume.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: consumer
  name: consumer
  namespace: secret-ops
spec:
  containers:
  - image: nginx
    name: nginx
    resources: {}
    envFrom:
    - secretRef:
        name: ext-service-secret
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
❯ kubectl create -f nginx-secret-consume.yaml
pod/nginx created
❯ kubectl exec -it consumer -n secret-ops -- env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=nginx
TERM=xterm
API_KEY=.b�m�l�fp6'⩤h�
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_PORT=443
NGINX_VERSION=1.31.1
NJS_VERSION=0.9.9
NJS_RELEASE=1~trixie
ACME_VERSION=0.4.1
PKG_RELEASE=1~trixie
DYNPKG_RELEASE=1~trixie
HOME=/root
```
---
### Create a Secret named 'my-secret' of type 'kubernetes.io/ssh-auth' in the namespace 'secret-ops'. Define a single key named 'ssh-privatekey', and point it to the file 'id_rsa' in this directory.
```bash
ssh-keygen
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/slowking/.ssh/id_ed25519): id_rsa
...
❯ kubectl create secret generic my-secret --type='kubernetes.io/ssh-auth' -n secret-ops --from-file=ssh-privatekey=./id_rsa
secret/my-secret created
❯ kubectl get secret -n secret-ops my-secret -o jsonpath={.data}
{"ssh-privatekey":"LS0tLS1CRUdJTiBPUEVOU1NIIFBSSVZBVEUgS0VZLS0tLS0KYjNCbGJuTnphQzFyWlhrdGRqRUFBQUFBQkc1dmJtVUFBQUFFYm05dVpRQUFBQUFBQUFBQkFBQUFNd0FBQUF0emMyZ3RaVwpReU5UVXhPUUFBQUNDMC9TWE5QaDA0VW1DQVFoQkVOOEM3cmdGRHdMR1RtMzBNeTNybi82S2dTUUFBQUppMUVlbTl0UkhwCnZRQUFBQXR6YzJndFpXUXlOVFV4T1FBQUFDQzAvU1hOUGgwNFVtQ0FRaEJFTjhDN3JnRkR3TEdUbTMwTXkzcm4vNktnU1EKQUFBRUJIN3Z4Wi92cUlLYzlIZUcwN2pWSDZTUkhSdGpJeHRQMjcybElDTlk5RmZMVDlKYzArSFRoU1lJQkNFRVEzd0x1dQpBVVBBc1pPYmZRekxldWYvb3FCSkFBQUFEbk5zYjNkcmFXNW5RRzVwZUc5ekFRSURCQVVHQnc9PQotLS0tLUVORCBPUEVOU1NIIFBSSVZBVEUgS0VZLS0tLS0K"}
```
>[!NOTE]
>When asked to create a secret of a certain type, use the `--type` flag

---
### Create a Pod named 'consumer' with the image 'nginx' in the namespace 'secret-ops', and consume the Secret as Volume. Mount the Secret as Volume to the path /var/app with read-only access. Open an interactive shell to the Pod, and render the contents of the file.
```bash
❯ kubectl run consumer --image=nginx --restart=Never -n secret-ops --dry-run=client -o yaml > consumer-nginx.yaml
❯ vim consumer-nginx.yaml
❯ cat consumer-nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: consumer
  name: consumer
  namespace: secret-ops
spec:
  containers:
  - image: nginx
    name: consumer
    resources: {}
    volumeMounts:
    - name: svol
      mountPath: /var/app
      readOnly: true
  dnsPolicy: ClusterFirst
  restartPolicy: Never
  volumes:
  - name: svol
    secret:
      secretName: my-secret
status: {}
❯ kubectl create -f consumer-nginx.yaml
pod/consumer created
❯ kubectl exec -it consumer -n secret-ops -- /bin/bash
root@consumer:/# cat /var/app
cat: /var/app: Is a directory
root@consumer:/# cat /var/app/
..2026_06_13_20_01_09.2937077254/ ..data/                           ssh-privatekey
root@consumer:/# cat /var/app/ssh-privatekey
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACC0/SXNPh04UmCAQhBEN8C7rgFDwLGTm30My3rn/6KgSQAAAJi1Eem9tRHp
vQAAAAtzc2gtZWQyNTUxOQAAACC0/SXNPh04UmCAQhBEN8C7rgFDwLGTm30My3rn/6KgSQ
AAAEBH7vxZ/vqIKc9HeG07jVH6SRHRtjIxtP272lICNY9FfLT9Jc0+HThSYIBCEEQ3wLuu
AUPAsZObfQzLeuf/oqBJAAAADnNsb3draW5nQG5peG9zAQIDBAUGBw==
-----END OPENSSH PRIVATE KEY-----
root@consumer:/# exit
exit
```
>[!NOTE]
>For a volume mount to be read only, specify the `readOnly: true` attribute

---
### See all the service accounts of the cluster in all namespaces
```bash
❯ kubectl get serviceaccounts -A
NAMESPACE         NAME                                          SECRETS   AGE
default           default                                       0         5d2h
default           myuser                                        0         6s
```
---
### Create a new serviceaccount called 'myuser'
```bash
❯ kubectl create serviceaccount myuser
serviceaccount/myuser created
```
---
### Create an nginx pod that uses 'myuser' as a service account
```bash
❯ vim serviceaccount-nginx.yaml
❯ cat serviceaccount-nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  serviceAccountName: myuser
  containers:
  - image: nginx
    name: nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
❯ kubectl create -f serviceaccount-nginx.yaml
pod/nginx created
```
>[!NOTE]
>You can specify the ServiceAccount a pod will use via `.spec.serviceAccountName`

---
### Generate an API token for the service account 'myuser'
```bash
kubectl create token myuser
```
>[!NOTE]
>To create an API token for a given ServiceAccount, use the `create token <serviceaccount>` command

