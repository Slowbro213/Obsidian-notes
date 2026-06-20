### Create a deployment called `webapp` with image `nginx:1.20` and 3 replicas. Record the change so it appears in rollout history.
```bash
❯ kubectl create deployment webapp --image=nginx:1.20 --replicas=3
deployment.apps/webapp created
❯ kubectl rollout status deployment webapp
deployment "webapp" successfully rolled out
❯ kubectl rollout history deployment webapp
deployment.apps/webapp
REVISION  CHANGE-CAUSE
1         <none>

❯ kubectl annotate deployment webapp kubernetes.io/change-clause="change recorded"
deployment.apps/webapp annotated
```
>[!NOTE]
>Change clause annotations are done via the `kubernetes.io/change-clause` key

---
### Update the `webapp` deployment image to `nginx:1.21`. Then check the rollout status and history.
```bash
❯ kubectl set image deployment/webapp nginx=nginx:1.21
deployment.apps/webapp image updated
❯ kubectl rollout status deployment webapp
Waiting for deployment "webapp" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "webapp" rollout to finish: 1 old replicas are pending termination...
deployment "webapp" successfully rolled out
❯ kubectl rollout history deployment webapp
deployment.apps/webapp
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
```
>[!NOTE]
>I accidentally set the image as `nginx=nginx1.21` instead of `nginx=nginx:1.21` in a previous attempt due to a typo, which is why you see 3 revisions

---
### Roll back the `webapp` deployment to the previous version and confirm the image reverted.
```bash
❯ kubectl rollout undo deployment webapp --to-revision=1
deployment.apps/webapp rolled back
❯ kubectl rollout status deployment webapp
Waiting for deployment "webapp" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "webapp" rollout to finish: 1 old replicas are pending termination...
deployment "webapp" successfully rolled out
❯ kubectl rollout history deployment webapp
deployment.apps/webapp
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
4         <none>
```
---
### Inspect the details of revision 2 of the `webapp` deployment without rolling back to it.
```bash
❯ kubectl rollout history deployment webapp --revision=2
deployment.apps/webapp with revision #2
Pod Template:
  Labels:       app=webapp
        pod-template-hash=76cfccbc9b
  Containers:
   nginx:
    Image:      nginx1.21
    Port:       <none>
    Host Port:  <none>
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>
  Node-Selectors:       <none>
  Tolerations:  <none>
```
---
### Pause the `webapp` deployment, update its image to `nginx:1.22`, then resume and verify the rollout happened.
```bash
❯ kubectl rollout pause deployment webapp
deployment.apps/webapp paused
❯ kubectl set image deployment/webapp nginx=nginx:1.22
deployment.apps/webapp image updated
❯ kubectl rollout status deployment webapp
Waiting for deployment "webapp" rollout to finish: 0 out of 3 new replicas have been updated...
^C%                                                                                                                                                                                            ❯ kubectl rollout resume deployment webapp
deployment.apps/webapp resumed
❯ kubectl rollout status deployment webapp
Waiting for deployment "webapp" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "webapp" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "webapp" rollout to finish: 1 old replicas are pending termination...
deployment "webapp" successfully rolled out
```
---
### Change the update strategy of the `webapp` deployment to `Recreate` (instead of RollingUpdate). Explain when you would use this.
```bash
❯ kubectl edit deployment webapp
deployment.apps/webapp edited
❯ kubectl get deploy webapp -o yaml | grep -i strategy
  strategy:
❯ kubectl get deploy webapp -o yaml | grep -i strategy -A 3
  strategy:
    type: Recreate
  template:
    metadata:
```
>[!IMPORTANT]
>To change the strategy to Recreate, you must edit the yaml definition of the deployment, and specify the `type` to be `Recreate`

>[!NOTE]
>A recreate terminates all old pods before rolling out new ones. This causes  


---