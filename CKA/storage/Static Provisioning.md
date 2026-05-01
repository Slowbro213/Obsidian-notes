#kubernetes #storage #persistent-volumes #provisioning

We'll simulate two roles here. An Admin who will create a [[PersistentVolume|PV]], and the Dev who will use the PV.

## Admin: Define and create a PV

Let's define a PV and create it.

```yaml
# pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv-volume
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: "/mnt/data"
```

```bash
kubectl apply -f pv.yaml
```

> [!WARNING]  
> This would never be used in production because the data can only be used in a single node, but it's good enough for local testing.

## Inspect the PV

Let's look at the PV we created.

```bash
persistentvolume/task-pv-volume created
vboxuser@k8s-init:~$ kubectl get pv task-pv-volume
NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
task-pv-volume   5Gi        RWO            Retain           Available           manual         <unset>                          2m35s
```

## Dev: Define and create a PVC

Now we'll act like the developer. We should make a request for a PV by creating a [[PersistentVolumeClaim|PVC]] and then using it in a [[Pod]] by referring to it by name.

Let's define a PVC and create one.

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

```bash
kubectl apply -f pvc.yaml
```

Here we are defining a PVC that is asking for a PV which has been created manually, has the RWO access mode, and can satisfy 2Gi of storage. A claim fulfilled by the PV we defined and created previously.

## Verify the PVC and corresponding PV

Now let's have a look at it and the corresponding PV we are trying to claim:

```bash
vboxuser@k8s-init:~$ kubectl get pvc task-pv-claim
NAME            STATUS   VOLUME           CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
task-pv-claim   Bound    task-pv-volume   5Gi        RWO            manual         <unset>                 40s
vboxuser@k8s-init:~$ kubectl get pv task-pv-volume
NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
task-pv-volume   5Gi        RWO            Retain           Bound    default/task-pv-claim   manual         <unset>                          7m4s
```

As we can see, the PVC resource says it has claimed the `task-pv-volume`, and the PV says it has been claimed by the `task-pv-claim`, which means the PVC made by the "Developer" has successfully claimed the PV made statically by the "Admin".

## Use the PVC in a Pod

Now that we have access to the PV via the PVC, let's use it on a pod.

Let's define the following pod:

```yaml
# pod-storage.yaml
apiVersion: v1
kind: Pod
metadata:
  name: storage-pod
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - mountPath: "/usr/share/nginx/html"
      name: my-storage
  volumes:
  - name: my-storage
    persistentVolumeClaim:
      claimName: task-pv-claim
```

Here we are defining a pod which uses nginx, has a volume mounted to it called `my-storage`, and the volume is taken from the persistent volume the "Admin" created using the name of its claim.