ConfigMaps and Secrets are ways to decouple apps from their configurations.  
- **ConfigMaps**: for non-sensitive configurations  
- **Secrets**: for sensitive data like API keys, passwords, etc.

---

## Creating a ConfigMap (Imperative)

```bash
kubectl create configmap app-config \
  --from-literal=app.color=blue \
  --from-literal=app.mode=production
```

This creates a ConfigMap called `app-config` with two key-value pairs:

```bash
kubectl describe configmap app-config
```

```text
Name:         app-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
app.color:
----
blue
app.mode:
----
production

BinaryData
====

Events:  <none>
```

---

## Creating a ConfigMap from a File

```bash
echo "retries = 3" > config.properties
kubectl create configmap app-config-file --from-file=config.properties
```

---

## Creating a ConfigMap (Declarative YAML)

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-declarative
data:
  database.url: "jdbc:mysql://db.example.com:3306/mydb"
  ui.theme: "dark"
```

```bash
kubectl apply -f configmap.yaml
# Output:
# configmap/app-config-declarative created
```

---

## Creating Secrets

> Note: These are base64-encoded, not encrypted.

### Imperative

```bash
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password='s3cr3t'
```

### Declarative YAML

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: api-key
type: Opaque
stringData:
  key: "my-super-secret-api-key"
```

```bash
kubectl apply -f secret.yaml
# Output:
# secret/api-key created
```

---

## Using ConfigMap and Secret in a Pod (Environment Variables)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-demo-pod
spec:
  containers:
    - name: demo-container
      image: busybox
      command: ["/bin/sh", "-c", "env && sleep 3600"]
      env:
        - name: THEME
          valueFrom:
            configMapKeyRef:
              name: app-config-declarative
              key: ui.theme
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
  restartPolicy: Never
```

---

## Mounting ConfigMap as a Volume

```yaml
# pod-volume.yaml (Assumes app-config-file ConfigMap exists)
apiVersion: v1
kind: Pod
metadata:
  name: volume-demo-pod
spec:
  containers:
    - name: demo-container
      image: busybox
      command: ["/bin/sh", "-c", "cat /etc/config/config.properties && sleep 3600"]
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: app-config-file
  restartPolicy: Never
```

```bash
kubectl apply -f pod-volume.yaml
```
