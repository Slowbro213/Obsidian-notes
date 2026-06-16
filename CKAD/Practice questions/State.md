### Create busybox pod with two containers, each one will have the image busybox and will run the 'sleep 3600' command. Make both containers mount an emptyDir at '/etc/foo'. Connect to the second busybox, write the first column of '/etc/passwd' file to '/etc/foo/passwd'. Connect to the first busybox and write '/etc/foo/passwd' file to standard output. Delete pod.
```bash
❯ kubectl run busybox-2-containers --image=busybox --restart=Never --dry-run=client -o yaml --command -- /bin/sh -c 'sleep 3600' > busybox-2-containers.yaml
❯ vim busybox-2-containers.yaml
❯ cat busybox-2-containers.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: busybox-2-containers
  name: busybox-2-containers
spec:
  containers:
  - command:
    - /bin/sh
    - -c
    - sleep 3600
    image: busybox
    name: busybox
    resources: {}
    volumeMounts:
    - name: vol
      mountPath: /etc/foo
  - command:
    - /bin/sh
    - -c
    - sleep 3600
    image: busybox
    name: busybox-2
    resources: {}
    volumeMounts:
    - name: vol
      mountPath: /etc/foo
  dnsPolicy: ClusterFirst
  restartPolicy: Never
  volumes:
  - name: vol
    emptyDir: {}
status: {}
❯ kubectl create -f busybox-2-containers.yaml
pod/busybox-2-containers created
❯ kubectl exec busybox-2-containers -c busybox-2 -it -- /bin/sh -c 'cut -d: -f1 /etc/passwd > /etc/foo/passwd'
❯ kubectl exec busybox-2-containers -c busybox -it -- /bin/sh -c 'cat /etc/foo/passwd'
root
daemon
bin
sys
sync
mail
www-data
operator
nobody
❯ kubectl delete pod busybox-2-containers
pod "busybox-2-containers" deleted
```
>[!NOTE]
>This is more of a general linux tip, but you can use the command `cut -d: -f1 <path>` to separate each new line with a given `-d` ( delimiter ) and then get the x column like `-f<x>`

---
### Create a PersistentVolume of 10Gi, called 'myvolume'. Make it have accessMode of 'ReadWriteOnce' and 'ReadWriteMany', storageClassName 'normal', mounted on hostPath '/etc/foo'. Save it on pv.yaml, add it to the cluster. Show the PersistentVolumes that exist on the cluster
```bash
❯ vim pv.yaml
❯ cat pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: myvolume
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
    - ReadWriteMany
  storageClassName: normal
  hostPath:
    path: /etc/foo
❯ kubectl create -f pv.yaml
persistentvolume/myvolume created
❯ kubectl get pv -A
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
myvolume   10Gi       RWO,RWX        Retain           Available           normal         <unset>                          7s
```
>[!NOTE]
>The yaml for a pv can be found on the kubernetes docs. I had to edit the storage from 5Gi to 10Gi, add an `accessMode`, edit the `storageClassName`, the name of the pv itself, and the kind of volume from nfs to `hostPath`


---
### Create a PersistentVolumeClaim for this PersistentVolume, called 'mypvc', a request of 4Gi and an accessMode of ReadWriteOnce, with the storageClassName of normal, and save it on pvc.yaml. Create it on the cluster. Show the PersistentVolumeClaims of the cluster. Show the PersistentVolumes of the cluster
```bash
❯ vim mypvc.yaml
❯ cat mypvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 4Gi
  storageClassName: normal
❯ kubectl create -f mypvc.yaml
persistentvolumeclaim/myvc created
❯ kubectl get pvc
NAME   STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
myvc   Bound    myvolume   10Gi       RWO,RWX        normal         <unset>                 4s
```
>[!NOTE]
>You can find the yaml file for creating a `pvc` in the kubernetes docs

---
### Create a busybox pod with command 'sleep 3600', save it on pod.yaml. Mount the PersistentVolumeClaim to '/etc/foo'. Connect to the 'busybox' pod, and copy the '/etc/passwd' file to '/etc/foo/passwd'
```bash
❯ kubectl run busybox --image=busybox --restart=Never --command --dry-run=client -o yaml -- /bin/sh -c 'sleep 3600' > pod.yaml
❯ vim pod.yaml
❯ cat pod.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: busybox
  name: busybox
spec:
  containers:
  - command:
    - /bin/sh
    - -c
    - sleep 3600
    image: busybox
    name: busybox
    resources: {}
    volumeMounts:
    - name: mypd
      mountPath: /etc/foo
  dnsPolicy: ClusterFirst
  restartPolicy: Never
  volumes:
  - name: mypd
    persistentVolumeClaim:
      claimName: myvc
status: {}
❯ kubectl create -f pod.yaml
pod/busybox created
❯ kubectl exec busybox -- /bin/sh -c 'cp /etc/passwd /etc/foo/passwd'
```
>[!NOTE]
>You can create `volumes` out of `persistentVolumeClaim`'s via
>```yaml
>volumes:
>- name: vol
>   persistentVolumeClaim:
> 	  claimName: name
>```

---
### Create a second pod which is identical with the one you just created (you can easily do it by changing the 'name' property on pod.yaml). Connect to it and verify that '/etc/foo' contains the 'passwd' file. Delete pods to cleanup. Note: If you can't see the file from the second pod, can you figure out why? What would you do to fix that?
```bash
❯ cp pod.yaml pod-copy.yaml
❯ vim pod-copy.yaml
❯ cat pod-copy.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: busybox-copy
  name: busybox-copy # Changed the name of the pod here
  # the rest is identical
spec:
  containers:
  - command:
    - /bin/sh
    - -c
    - sleep 3600
    image: busybox
    name: busybox-copy
    resources: {}
    volumeMounts:
    - name: mypd
      mountPath: /etc/foo
  dnsPolicy: ClusterFirst
  restartPolicy: Never
  volumes:
  - name: mypd
    persistentVolumeClaim:
      claimName: myvc
status: {}
❯ kubectl create -f pod-copy.yaml
pod/busybox-copy created
❯ kubectl exec busybox-copy -it -- /bin/sh -c 'ls /etc/foo'
passwd
```
---
### Create a busybox pod with 'sleep 3600' as arguments. Copy '/etc/passwd' from the pod to your local folder
```bash
kubectl cp busybox:/etc/foo/passwd ./local-folder/passwd
```
>[!IMPORTANT]
>Use `kubectl cp <pod>:<pod-file> <local-file>` to copy files from pods to your local machine

