### Create 3 pods with names nginx1,nginx2,nginx3. All of them should have the label app=v1
```bash
❯ kubectl run nginx1 --image=nginx -n myns --restart=Never --labels=app=v1
pod/nginx1 created
❯ kubectl run nginx2 --image=nginx -n myns --restart=Never --labels=app=v1
pod/nginx2 created
❯ kubectl run nginx3 --image=nginx -n myns --restart=Never --labels=app=v1
pod/nginx3 created
```
### Show all labels of the pods
```bash
kuebctl get pods -n myns --show-labels
```
>[!NOTE]
>Use the flag `--show-labels` to get the labels of the pods 
### Change the labels of pod 'nginx2' to be app=v2
```
❯ kubectl label pod nginx2 -n myns app=v2 --overwrite
```
>[!NOTE]
>Use the `label pod <pod> key=value --overwrite` command to change the label of a pod. When in doubt you can always just `kubectl edit pod <pod>` and change the yaml directly as you wish
### Get the label 'app' for the pods (show a column with APP labels)
```bash
❯ kubectl get pods -n myns -l=app --show-labels
```
### Get only the 'app=v2' pods
```bash
❯ kubectl get pods -n myns -l=app=v2 --show-labels
```
### Get 'app=v2' and not 'tier=frontend' pods
```bash
❯ kubectl get pods -n myns -l=app=v2,tier!=frontend
```
>[!NOTE]
>You can use the `!=` operator with selector labels ( aka the `-l` flag )
### Add a new label tier=web to all pods having 'app=v2' or 'app=v1' labels
```bash
kubectl label pod -n myns -l "app in (v1,v2)" tier=web
```
>[!NOTE]
>You can use selector labels with conditions like `-l key in (val1,val2)`

