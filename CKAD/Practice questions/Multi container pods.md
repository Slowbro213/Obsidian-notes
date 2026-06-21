### Create a Pod with two containers, both with image `busybox` and command `echo hello; sleep 3600`. Connect to the second container and run `ls`

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
> When exec-ing into BusyBox containers, use:
> 
> ```bash
> /bin/sh
> ```
> 
> instead of:
> 
> ```bash
> /bin/bash
> ```
> 
> since BusyBox does not include Bash by default.

---

### Create a pod with an nginx container exposed on port 80. Add a busybox init container which downloads a page using `echo "Test" > /work-dir/index.html`. Make a volume of type `emptyDir` and mount it in both containers. For the nginx container, mount it on `/usr/share/nginx/html` and for the initContainer, mount it on `/work-dir`. When done, get the IP of the created pod and create a busybox pod and run `wget -O- IP`

```bash
❯ kubectl run nginx --image=nginx -n myns --restart=Never --port=80 --dry-run=client -o yaml > nginx-multic.yaml

❯ vim nginx-multic.yaml

# Next we make some changes to the yaml. Explanation below.

❯ kubectl create -f nginx-multic.yaml

❯ kubectl run busybox --image=busybox -n myns -it --rm --restart=Never --command -- wget -O- 10.244.0.29

Connecting to 10.244.0.29 (10.244.0.29:80)
writing to stdout

Test

-                    100% |********************************|     5  0:00:00 ETA

written to stdout

pod "busybox" deleted
```

> [!EXAMPLE]  
> Define a shared volume:
> 
> ```yaml
> volumes:
>   - name: <name>
>     emptyDir: {}
> ```

> [!EXAMPLE]  
> Mount the volume inside a container:
> 
> ```yaml
> volumeMounts:
>   - name: vol
>     mountPath: <path>
> ```

> [!EXAMPLE]  
> Define the BusyBox init container:
> 
> ```yaml
> initContainers:
>   - args:
>       - /bin/sh
>       - -c
>       - 'echo "Test" > /work-dir/index.html'
>     image: busybox
>     name: initbox
>     resources: {}
> ```
> 
> The init container writes the file into the shared volume before nginx starts.

> [!IMPORTANT]  
> The nginx container and the init container must mount the same `emptyDir` volume:
> 
> - nginx → `/usr/share/nginx/html`
>     
> - init container → `/work-dir`
>     
> 
> This allows nginx to serve the file created by the init container.

---
>[!IMPORTANT]
>The `ambassador` pattern has to do with a multi container scenario, such that one of the containers works a proxy for the others requests, so that the main app container doesnt need to worry about service discovery

>[!IMPORTANT]
> The `adapter` pattern has to do with the use of a container in order to format the output of another container.

