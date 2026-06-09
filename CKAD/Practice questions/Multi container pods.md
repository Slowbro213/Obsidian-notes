### Create a Pod with two containers, both with image busybox and command "echo hello; sleep 3600". Connect to the second container and run 'ls'
```bash
❯ kubectl run busybox2 --image=busybox -n myns --restart=Never --dry-run=client -o yaml --command -- /bin/sh -c "echo hello;sleep 3600" > busybox2.yaml
❯ vim busybox2.yaml
# Here we edit the yaml to create 2 containers instead of one
❯ cat busybox2.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: busybox2
  name: busybox2
  namespace: myns
spec:
  containers:
  - command:
    - /bin/sh
    - -c
    - echo hello;sleep 3600
    image: busybox
    name: busybox
    resources: {}
  - command:
    - /bin/sh
    - -c
    - echo hello;sleep 3600
    image: busybox
    name: busybox2
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
❯ kubectl apply -f busybox2.yaml
❯ kubectl exec -it busybox2 -n myns -c busybox2 -- ls
bin    etc    lib    proc   sys    usr
dev    home   lib64  root   tmp    var
```

> [!NOTE]
> When exec-ing in busybox containers, use '/bin/sh' instead of '/bin/bash' 

### Create a pod with an nginx container exposed on port 80. Add a busybox init container which downloads a page using 'echo "Test" > /work-dir/index.html'. Make a volume of type emptyDir and mount it in both containers. For the nginx container, mount it on "/usr/share/nginx/html" and for the initcontainer, mount it on "/work-dir". When done, get the IP of the created pod and create a busybox pod and run "wget -O- IP"

```bash
❯ kubectl run nginx --image=nginx -n myns --restart=Never --port=80 --dry-run=client -o yaml > nginx-multic.yaml
❯ vim nginx-multic.yaml
# next we make some changes to the yaml. explanation below
❯ kubectl create -f nginx-multic.yaml
❯ kubectl run busybox --image=busybox -n myns -it --rm --restart=Never --command -- wget -O- 10.244.0.29
Connecting to 10.244.0.29 (10.244.0.29:80)
writing to stdout
Test
-                    100% |********************************|     5  0:00:00 ETA
written to stdout
pod "busybox" deleted
```

>[!EXPLANATION]
>We first create a simple nginx container by writing the yaml for it. then we create an initContainer, a keyword that can be defined in the spec for containers. Under the specs section, we define the volumes. volumes here are defines as such:
>```yaml
>volumes:
>- name: <name>
>   emptyDir: {}
>```
>And under each container we mount that volume like so:
>```yaml
>volumeMounts:
>- name: vol
>   mountPath: <path>
>```
>The initContainer needs to be a busybox image which executes the ' echo "Test" > /work-dir/index.html' command. in order to do this we define the initContainer like so:
>```yaml
>initContainer:
>- args:
>	- /bin/sh
>	- -c
>	- 'echo "Test" > /work-dir/index.html'
>   image: busybox
>   name: initbox
>   resources: {}
>```











