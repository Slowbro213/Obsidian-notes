CoreDNS is the default DNS server for Kubernetes. It is useful for providing service discovery.

When you create a service, CoreDNS automatically creates DNS records for it:

```text
<service>.<name>.<namespace>.svc.cluster.local
```

CoreDNS is configured via a ConfigMap in the `kube-system` namespace.

---

## Editing the CoreDNS ConfigMap

The tutorial edits this ConfigMap via:

```bash
kubectl edit configmap coredns -n kube-system
```

It edits the `Corefile` entry in there in order to add a rule which forwards queries to a custom domain in an internal IP.

> [!NOTE]
> I’m not going to perform this, since as a developer this seems a tad bit useless, but it’s good to at least have an understanding of this.
