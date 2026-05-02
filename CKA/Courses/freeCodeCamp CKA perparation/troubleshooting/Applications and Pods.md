## Common Pod statuses and their meanings

| Status | Meaning |
|---|---|
| `Pending` | Waiting to be scheduled on a Node or downloading images. |
| `ContainerCreating` | Container Runtime is starting the container. |
| `ImagePullBackOff` / `ErrImagePull` | Unable to pull the container image. |
| `CrashLoopBackOff` | Container started but then exited with an error. |
| `OOMKilled` | Container was terminated because it exceeded its memory limit. |

## Debugging Pending Pods

### Cause

Usually a scheduling failure due to:

- Insufficient resources (Pods which ask for too many resources).
- Failing node affinity rules.
- Node taints without matching tolerations.
- Unbound PVC.

### Debug Steps

- `kubectl describe pod <pod-name>`.
- Look at the Events section for `FailedScheduling`. The message will tell you why.

## Debugging ImagePullBackOff

### Causes

- Incorrect Image or tag.
- Error authenticating with private registry.
- Registry is unreachable.

### Debug Steps

- `kubectl describe pod <pod-name>`.
- Check events for the `"Failed to pull image..."` error message.
- Verify the image name/tag and `imagePullSecrets` configuration.

## Debugging CrashLoopBackOff

### Cause

A problem with the application or its configuration:

- Error in application code.
- Misconfiguration (wrong password, bad file path).
- Failing liveness probe.

### Debug Steps

> [!IMPORTANT]
> **Check the logs!** This is the most critical step.

```bash
# Get logs from the current, crashing container
kubectl logs <pod-name>

# Get logs from the previous instance
kubectl logs <pod-name> --previous