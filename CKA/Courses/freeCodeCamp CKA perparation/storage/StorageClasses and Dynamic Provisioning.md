#kubernetes #storage #dynamic-provisioning

> [!SUMMARY]  
> Manually creating [[PersistentVolume|PVs]] doesn't scale. Dynamic provisioning automates it.

An administrator defines one or more [[StorageClass]] objects.

A StorageClass describes a "class" of storage, specifying a provisioner — someone who provides storage, e.g., `ebs.csi.aws.com` — and parameters.

When a user creates a [[PersistentVolumeClaim|PVC]] referencing a `storageClassName`, the provisioner automatically creates a matching PV.

## Admin: StorageClass and provisioner setup

Let's practice the modern, automated way. The "Admin" can create a template and then the storage is created on demand. The template for creating storage is called a StorageClass. Most Kubernetes come with a default one already setup.

```bash
vboxuser@k8s-init:~$ kubectl get storageClass
No resources found
```

In our case, we don't have a default one setup since we created the cluster ourselves using kubeadm. Most Kubernetes providers like Google or AWS have a default one linked to their block storage provider, but since our cluster has no idea what our hardware is it can't make a default one yet. For a local environment, we need to install a "hostPath provisioner". This provisioner creates PVs by just creating directories on the disk of our worker nodes.

> [!IMPORTANT]  
> Never use this provisioner in a production environment, it ties all the data to a single node which is not durable or highly available.

## Install the Rancher local-path provisioner

Let's install the Rancher local-path provisioner:

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.26/deploy/local-path-storage.yaml
```

Now we can see that the `get storageClass` command returns a result!

```bash
vboxuser@k8s-init:~$ kubectl get storageClass
NAME         PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  22s
```

## Dev: Request storage with a PVC

Now as a Developer we should have the ability to request for storage and have that request met without an Admin needing to create PVs manually.

Let's define a PVC:

```yaml
# my-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-storage-claim
spec:
  # This tells the PVC to use the StorageClass we just created.
  storageClassName: local-path
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

So in the definition above, we are requesting for a PV which was created via a local-path provisioner, the accessMode of which is RWO and the storage it should have is at least 1Gi.

> [!NOTE]  
> It seems that when we created a PV manually, the `storageClassName` was `manual` because the provisioner was a manual one, so the only difference here is that static provisioning just has the provisioner class as `manual` while the automatic ones have other ones — in our case it's `local-path`.

## Create the PVC

Now let's create it:

```bash
kubectl apply -f my-pvc.yaml
```

And let's view it!

```bash
vboxuser@k8s-init:~$ kubectl get pvc my-storage-claim
NAME               STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
my-storage-claim   Pending                                      local-path     <unset>                 3m10s
```

It looks like our PVC is in the pending state, meaning no PV was bound to it yet. This is because the storage provisioner is waiting for a pod that uses this claim in order to actually create the PV.

The tutorial for storage ends here, but I'll go ahead and create a pod that uses this PVC in order to verify everything works as we think it should. So let's just get the pod YAML definition of the [[Static Provisioning]] note and modify it to use the PVC we just created!

## Create a Pod that uses the PVC

```yaml
# dynamic-pv-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: dynamic-storage-pod
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
      claimName: my-storage-claim
```

This pod definition is the same as the static provisioning one, but the name of the pod is `dynamic-storage-pod` and the `claimName` of the PVC is `my-storage-claim`, same as the PVC we created which asks for storage from the dynamic provisioner.

After creating the pod, we can inspect the PVC and see that it has a volume bound to it!

## Verify the dynamically created PV

```bash
vboxuser@k8s-init:~$ kubectl get pvc my-storage-claim
NAME               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
my-storage-claim   Bound    pvc-9987dbed-9c47-4ca6-8d1e-6daf1cd8f469   1Gi        RWO            local-path     <unset>                 31m
```

```bash
vboxuser@k8s-init:~$ kubectl get pv pvc-9987dbed-9c47-4ca6-8d1e-6daf1cd8f469
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                      STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-9987dbed-9c47-4ca6-8d1e-6daf1cd8f469   1Gi        RWO            Delete           Bound    default/my-storage-claim   local-path     <unset>                          28m
```