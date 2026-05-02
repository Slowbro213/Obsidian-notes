### Checking Resource Usage

- The `kubectl top <resource>` command is the primary tool for checking resource consumption.
- It relies on the **Metrics Server**.

- Check Node resource usage:

```bash
kubectl top nodes
```

- Check pod resource usage:

```bash
# Diagnose OOMKilled errors, tune HPA targets
kubectl top pods -n <namespace>
```