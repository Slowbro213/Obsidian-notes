## Cluster-wide Application Failures

If applications are failing cluster-wide, the problem might be with the nodes or the control plane itself.

Nodes can be in several states. The most common problematic states are `NotReady` and `SchedulingDisabled`.

## NotReady Node

### Meaning

The kubelet on the node is not reporting a healthy status.

### Cause

- The kubelet process is not running.
- Network partition preventing communication with the API server.
- The underlying machine is down.

### Debug Steps

- SSH into the affected node.
- Check kubelet service status:

```bash
sudo systemctl status kubelet
```

- Examine kubelet logs:

```bash
sudo journalctl -u kubelet -f
```

## SchedulingDisabled Node

### Meaning

`SchedulingDisabled` might usually happen when an administrator has manually cordoned the node for maintenance purposes.

### Fix

The fix is to uncordone the node.

## Cluster Components

On a kubeadm initialized cluster, the control plane components run as static pods.

Their YAML manifests are found in:

```text
/etc/kubernetes/manifests
```

> [!QUESTION]
> Unsure how this would work with a HA control plane yet, the video doesn't say anything about it.

> [!TIP]
> If a `kubectl` command fails with `"connection refused"`, it’s likely the API server is down.

### Debug Steps

- SSH into the control plane node.
- Check the static pod container using:

```bash
# Im guessing crictl stands for "container runtime interface controller"
crictl ps
```

- Check its logs using:

```bash
crictl logs <container-id>
```

- Inspect its manifest for errors.

## Scheduler or Controller Manager Failure

### Symptoms

Symptoms include new pods remaining pending indefinitely in the case of a scheduler failure, or deployments not creating new pods in the case of a controller manager failure.

The process of debugging is the same as for the API server.

## Services and Networking

Let’s assume we have a pod that cannot connect to a service. A systematic approach to debugging this situation would be:

### Debug Steps

- Check CoreDNS resolution:

```bash
# Exec into the failing pod and execute nslookup for the service it can't connect to in order to verify if CoreDNS is the issue.
kubectl exec --it client-pod -- nslookup my-service
```

- Check Service Endpoints:

```bash
kubectl describe svc my-service
```

- Check Pod Connectivity: Try to curl the backend Pod's IP directly.
- Check Network Policies:

```bash
kubectl get netpol

# From here, you can temporarily delete network policies to isolate the issue.
```

> [!NOTE]
> I have ran into an issue myself when setting up the cluster for the first time using VMs and kubeadm where the nodes themselves had DNS issues. This caused several `ImagePullBackOff` errors.