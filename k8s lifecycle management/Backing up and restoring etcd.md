First we install the etcd-client

```bash
sudo apt install -y etcd-client
```

Lets create a directory where we will store the etcd backup

```bash
sudo mkdir -p /var/lib/etcd-backup
```

Next we paste this command to create a snapshot of etcd and store it. The kubernetes docs give this command template:

```shell
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=<trusted-ca-file> --cert=<cert-file> --key=<key-file> \
  snapshot save <backup-file-location>
```

The tutorial gives us this:

```bash
 sudo ETCDCTL_API=3 etcdctl snapshot save /var/lib/etcd-backup/snapshot.db \
     --endpoints=https://127.0.0.1:2379 \
     --cacert=/etc/kubernetes/pki/etcd/ca.crt \
     --cert=/etc/kubernetes/pki/etcd/server.crt \
     --key=/etc/kubernetes/pki/etcd/server.key
```

Running that command we get:

```bash
vboxuser@k8s-init:~$  sudo ETCDCTL_API=3 etcdctl snapshot save /var/lib/etcd-backup/snapshot.db \
>      --endpoints=https://127.0.0.1:2379 \
>      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
>      --cert=/etc/kubernetes/pki/etcd/server.crt \
>      --key=/etc/kubernetes/pki/etcd/server.key
{"level":"info","ts":1776882224.3606982,"caller":"snapshot/v3_snapshot.go:119","msg":"created temporary db file","path":"/var/lib/etcd-backup/snapshot.db.part"}
{"level":"info","ts":"2026-04-22T18:23:44.364485Z","caller":"clientv3/maintenance.go:212","msg":"opened snapshot stream; downloading"}
{"level":"info","ts":1776882224.3645093,"caller":"snapshot/v3_snapshot.go:127","msg":"fetching snapshot","endpoint":"https://127.0.0.1:2379"}
{"level":"info","ts":"2026-04-22T18:23:44.388604Z","caller":"clientv3/maintenance.go:220","msg":"completed snapshot read; closing"}
{"level":"info","ts":1776882224.3934631,"caller":"snapshot/v3_snapshot.go:142","msg":"fetched snapshot","endpoint":"https://127.0.0.1:2379","size":"2.0 MB","took":0.032333965}
{"level":"info","ts":1776882224.393515,"caller":"snapshot/v3_snapshot.go:152","msg":"saved","path":"/var/lib/etcd-backup/snapshot.db"}
Snapshot saved at /var/lib/etcd-backup/snapshot.db
```
Lets remember the files, the template is again:

```bash
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=<trusted-ca-file> --cert=<cert-file> --key=<key-file> \
  snapshot save <backup-file-location>
```

the files are all in "/etc/kubernetes/pki/etcd\<filename>", so for cacert we get "/etc/kubernetes/pki/etcd/ca.crt", for the two others for filename we get "server.<arg_name>". So that makes --cert become "/etc/kubernetes/pki/etcd/server.crt" and for --key that becomes "/etc/kubernetes/pki/etcd/server.key". putting it all together directly from the docs, that creates the command.

```bash
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save <backup-file-location>
```
Which is identical ( almost, we need sudo and we need a path for the backup, but that path can be anything ) to the tutorial!

Now for restoring. We first need to stop the kubelet:

```bash
sudo systemctl stop kubelet
```

Looking at the config file for etcd which can be found in "/etc/kubernetes/manifests/etcd.yaml", we can find the following part:

  volumes:
  - hostPath:
      path: /etc/kubernetes/pki/etcd
      type: DirectoryOrCreate
    name: etcd-certs
  - hostPath:
      path: /var/lib/etcd
      type: DirectoryOrCreate
    name: etcd-data
status: {}


in here we can find that the hostPath -> path config holds the path of the file used by etcd store the values. Therefore, we need to restore the backup into a usable file, then change the etcd manifest to point to this restored file then restart etcd. So here are the steps:

1: store a key value pair in etcd and confirm it was saved
2: create a backup file
3: store the same key in etcd but with a different value, effectively overriding it and verifying it was overwritten
4: restore the backup file
5: make etcd use the restored backup file
6: verify the old key value pair is present to prove etcd is using the backed up values

step 1:
```bash
sudo env ETCDCTL_API=3 etcdctl \
	--endpoints=https://127.0.0.1:2379 \
	--cacert=/etc/kubernetes/pki/etcd/ca.crt \
	--cert=/etc/kubernetes/pki/etcd/server.crt \
	--key=/etc/kubernetes/pki/etcd/server.key \
	put key1 value1
OK
sudo env ETCDCTL_API=3 etcdctl \
   --endpoints=https://127.0.0.1:2379 \
   --cacert=/etc/kubernetes/pki/etcd/ca.crt \
   --cert=/etc/kubernetes/pki/etcd/server.crt \
   --key=/etc/kubernetes/pki/etcd/server.key \
   get key1
key1
value1
```

step 2:
```bash
sudo ETCDCTL_API=3 etcdctl snapshot save /var/lib/etcd-backup/snapshot.db \
>      --endpoints=https://127.0.0.1:2379 \
>      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
>      --cert=/etc/kubernetes/pki/etcd/server.crt \
>      --key=/etc/kubernetes/pki/etcd/server.key
{"level":"info","ts":1776883909.6419735,"caller":"snapshot/v3_snapshot.go:119","msg":"created temporary db file","path":"/var/lib/etcd-backup/snapshot.db.part"}
{"level":"info","ts":"2026-04-22T18:51:49.645606Z","caller":"clientv3/maintenance.go:212","msg":"opened snapshot stream; downloading"}
{"level":"info","ts":1776883909.645933,"caller":"snapshot/v3_snapshot.go:127","msg":"fetching snapshot","endpoint":"https://127.0.0.1:2379"}
{"level":"info","ts":"2026-04-22T18:51:49.665509Z","caller":"clientv3/maintenance.go:220","msg":"completed snapshot read; closing"}
{"level":"info","ts":1776883909.6702693,"caller":"snapshot/v3_snapshot.go:142","msg":"fetched snapshot","endpoint":"https://127.0.0.1:2379","size":"2.0 MB","took":0.028241657}
{"level":"info","ts":1776883909.6709158,"caller":"snapshot/v3_snapshot.go:152","msg":"saved","path":"/var/lib/etcd-backup/snapshot.db"}
Snapshot saved at /var/lib/etcd-backup/snapshot.db
```

step3:

```bash
sudo env ETCDCTL_API=3 etcdctl \
	--endpoints=https://127.0.0.1:2379 \
	--cacert=/etc/kubernetes/pki/etcd/ca.crt \
	--cert=/etc/kubernetes/pki/etcd/server.crt \
	--key=/etc/kubernetes/pki/etcd/server.key \
	put key1 value2
OK
sudo env ETCDCTL_API=3 etcdctl \
   --endpoints=https://127.0.0.1:2379 \
   --cacert=/etc/kubernetes/pki/etcd/ca.crt \
   --cert=/etc/kubernetes/pki/etcd/server.crt \
   --key=/etc/kubernetes/pki/etcd/server.key \
   get key1
key1
value2
```

step 4:
```bash
sudo systemctl stop kubelet
sudo ETCDCTL_API=3 etcdctl snapshot restore /var/lib/etcd-backup/snapshot.db \
     --data-dir /var/lib/etcd-restored
{"level":"info","ts":1776884124.6527627,"caller":"snapshot/v3_snapshot.go:306","msg":"restoring snapshot","path":"/var/lib/etcd-backup/snapshot.db","wal-dir":"/var/lib/etcd-restored/member/wal","data-dir":"/var/lib/etcd-restored","snap-dir":"/var/lib/etcd-restored/member/snap"}
{"level":"info","ts":1776884124.6625369,"caller":"mvcc/kvstore.go:388","msg":"restored last compact revision","meta-bucket-name":"meta","meta-bucket-name-key":"finishedCompactRev","restored-compact-revision":4779}
{"level":"info","ts":1776884124.6681104,"caller":"membership/cluster.go:392","msg":"added member","cluster-id":"cdf818194e3a8c32","local-member-id":"0","added-peer-id":"8e9e05c52164694d","added-peer-peer-urls":["http://localhost:2380"]}
{"level":"info","ts":1776884124.6726894,"caller":"snapshot/v3_snapshot.go:326","msg":"restored snapshot","path":"/var/lib/etcd-backup/snapshot.db","wal-dir":"/var/lib/etcd-restored/member/wal","data-dir":"/var/lib/etcd-restored","snap-dir":"/var/lib/etcd-restored/member/snap"}
#After editing the file
sudo systemctl start kubelet
```

step 5:
```bash
sudo env ETCDCTL_API=3 etcdctl \
   --endpoints=https://127.0.0.1:2379 \
   --cacert=/etc/kubernetes/pki/etcd/ca.crt \
   --cert=/etc/kubernetes/pki/etcd/server.crt \
   --key=/etc/kubernetes/pki/etcd/server.key \
   get key1
key1
value1
```

As we can see, the test confirms the old key value pair is present and the new one was discarded by the restore!