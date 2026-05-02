A structured approach is essential:

1. **Identify the problem:** clearly define what is not working (e.g., kubelet get pods).
2. **Gather information:** Collect logs, events, definitions (`kubectl describe`).
3. **Analyze the data:** Form a hypothesis about the root cause.
4. **Implement a solution:** Apply a fix.
5. **Verify the solution:** Confirm the issue is resolved.

> [!NOTE]
> Work from the top down: Pod -> Service -> Node -> Cluster