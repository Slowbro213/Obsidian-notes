Get resources

```bash
kubectl get <resource> 
```

sometimes with a namespace

```bash
kubectl get <resource> -n <namespace>
```

etcdctl commands

```bash
sudo env ETCDCTL_API=3 etcdctl snapshot save/restore/put/get <args> \
--endpoints=https://127.0.0.1:2379 --cacert=<trusted-ca-file> --cert=<cert-file> --key=<key-file>
```