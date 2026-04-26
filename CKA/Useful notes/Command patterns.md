Get resources

```bash
kubectl <action> <resource> 
```

sometimes with a namespace

```bash
kubectl <action> <resource> -n <namespace>
```

<\action>  here can be a get, create, describe etc. We can also apply whatever a yaml file contains like:

```bash
kubectl apply -f file.yaml
```

etcdctl commands

```bash
sudo env ETCDCTL_API=3 etcdctl snapshot save/restore/put/get <args> \
--endpoints=https://127.0.0.1:2379 --cacert=<trusted-ca-file> --cert=<cert-file> --key=<key-file>
```