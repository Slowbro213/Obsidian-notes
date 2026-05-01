## Core Concepts

|Concept|Description|
|---|---|
|**Volumes**|Tied to the lifecycle of a Pod. Data is lost when the Pod is deleted. Suitable for ephemeral data.|
|**Persistent Volumes (PVs)**|A piece of storage in the cluster which is independent of any pod. Provisioned by admins.|
|**Persistent Volume Claims (PVCs)**|A request for storage made by a user. Acts as a claim to a PV resource.|

## Binding Process

1. A user creates a PVC requesting a certain size and access mode.
    
2. The k8s control plane looks for a suitable PV that satisfies the claim.
    
3. If a suitable PV is found, the PVC is bound to the PV in a one-to-one mapping.
    

> [!IMPORTANT]  
> Once bound no other PVC can be bound to that PV.

4. A Pod can then mount the storage using the PVC by name.
    

## Important PV/PVC Specifications

When defining PVs and PVCs two of the most important specifications are its access mode and reclaim policy.

- **Access modes** define how a volume can be mounted by nodes in a cluster.
    
- The chosen access mode must be supported by the underlying storage provider.
    

## Access Modes

|Access Mode|Description|
|---|---|
|**ReadWriteOnce (RWO)**|Read-write by a single node. Most common.|
|**ReadOnlyMany (ROX)**|Read-only by many nodes.|
|**ReadWriteMany (RWX)**|Read-write by many nodes. `(nfs)`|
|**ReadWriteOncePod (RWOP)**|Read-write by a single Pod. Most secure.|

## Reclaim Policies

What to do with a PV after its PVC is deleted.

|Reclaim Policy|Description|
|---|---|
|**Retain**|Safest. The PV remains. The underlying data is not deleted and requires manual cleanup.|
|**Delete**|The PV and the underlying storage asset are automatically deleted.|
|**Recycle**|Deprecated, replaced by dynamic provisioning. Performs a basic scrub (`rm -rf`) and makes the PV available again.|

> [!WARNING]  
> `Recycle` is deprecated and replaced by dynamic provisioning.