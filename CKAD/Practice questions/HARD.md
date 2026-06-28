### Exercise 1 — Secure Microservice Platform with Canary Release

**Scenario:** Your team is releasing a new version of a payment API (`pay-api`) into the `payments` namespace. The service holds sensitive credentials and needs persistent audit logs. To reduce risk, you are deploying the new version as a canary (25% of traffic) alongside the stable version (75%). A logging sidecar must ship logs off-pod. RBAC must prevent the app from touching anything except its own ConfigMap. A NetworkPolicy must isolate the namespace.

**Domains covered:** Namespaces, Deployments, canary strategy, ConfigMaps, Secrets, volumes (emptyDir + PVC), sidecars, RBAC (ServiceAccount / Role / RoleBinding), Services, NetworkPolicy.

**Expected Time** 35–45 min

### Part A — Namespace and quota

Create namespace `payments`. Apply a ResourceQuota named `payments-quota` to it that caps: 10 pods, 4 CPUs (requests), 8Gi memory (requests).
```bash
❯ kubectl create ns payments

❯ kubectl create resourcequota payments-quota --hard=pods=10,cpu=4,memory=8Gi --dry-run=client -o yaml > rq.yaml
❯ vim rq.yaml
❯ cat rq.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  creationTimestamp: null
  name: payments-quota
  namespace: payments
spec:
  hard:
    cpu: "4"
    memory: 8Gi
    pods: "10"
status: {}
❯ kubectl create rq
error: Unexpected args: [rq]
See 'kubectl create -h' for help and examples
❯ kubectl create -f rq.yaml
resourcequota/payments-quota created
```

### Part B — Configuration

1. Create a ConfigMap `pay-api-config` in `payments` with keys:
   - `LOG_LEVEL=info`
   - `API_PORT=8080`

2. Create a Secret `pay-api-secret` in `payments` with keys:
   - `DB_PASSWORD=supersecret`
   - `API_KEY=abc123`
   
```bash
❯ kubectl create configmap -n payments pay-api-config --from-literal=LOG_LEVEL=info --from-literal=API_PORT=8080
configmap/pay-api-config created
❯ kubectl create secret generic -n payments pay-api-secret --from-literal=DB_PASSWORD=supersecret --from-literal=API_KEY=abc123
secret/pay-api-secret created
```

### Part C — Persistent storage for audit logs

Create a PersistentVolume `pay-audit-pv` (hostPath `/data/pay-audit`, 2Gi, `ReadWriteOnce`, storageClassName `manual`). Then create a PersistentVolumeClaim `pay-audit-pvc` (2Gi, `ReadWriteOnce`, storageClassName `manual`) in namespace `payments`.

```bash
❯ vim pv.yaml
❯ cat pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pay-audit-pv
spec:
  capacity:
    storage: 2Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  hostPath:
    path: /data/pay-audit

❯ kubectl create -f pv.yaml
persistentvolume/pay-audit-pv created
❯ vim pvc.yaml
❯ cat pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pay-audit-pvc
  namespace: payments
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 2Gi
  storageClassName: manual
❯ kubectl create -f pvc.yaml
persistentvolumeclaim/pay-audit-pvc created
```
### Part D — RBAC

1. Create a ServiceAccount `pay-api-sa` in `payments`.
2. Create a Role `pay-api-role` in `payments` that allows `get` and `watch` on ConfigMaps only.
3. Bind the role to the service account with a RoleBinding `pay-api-rb`.
```bash
❯ kubectl create serviceaccount pay-api-sa -n payments
serviceaccount/pay-api-sa created
❯ kubectl create role -n payments pay-api-role --verb=get,watch --resource=configmap
role.rbac.authorization.k8s.io/pay-api-role created
❯ kubectl create rolebinding -n payments pay-api-rb --role=pay-api-role --serviceaccount=payments:pay-api-sa
rolebinding.rbac.authorization.k8s.io/pay-api-rb created
```
### Part E — Stable deployment (v1, 75% of traffic)

Create a Deployment `pay-api-stable` in `payments` with:
- 3 replicas, image `nginx:1.24` (simulating the app)
- Label `app=pay-api`, `slot=stable`
- `serviceAccountName: pay-api-sa`
- Env vars from `pay-api-config` (all keys via `envFrom`) and individual keys `DB_PASSWORD` and `API_KEY` from `pay-api-secret`
- A sidecar container `log-shipper` (image `busybox`) running `tail -f /audit/app.log`
- An `emptyDir` volume `log-scratch` mounted at `/audit` in both containers
- The PVC `pay-audit-pvc` mounted at `/persistent-audit` in the main container only
- Resource requests: `cpu: 100m`, `memory: 128Mi`; limits: `cpu: 200m`, `memory: 256Mi`
- A `readinessProbe` on the main container: HTTP GET `/` on port 80, `initialDelaySeconds: 5`
- A `livenessProbe` on the main container: HTTP GET `/` on port 80, `periodSeconds: 15`
```bash
❯ kubectl create deploy pay-api-stable -n payments --image=nginx:1.24 --replicas=3 --dry-run=client -o yaml > dep.yaml
❯ vim dep.yaml
❯ cat dep.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: pay-api
    slot: stable
  name: pay-api-stable
  namespace: payments
spec:
  replicas: 3
  serviceAccountName: pay-api-sa
  selector:
    matchLabels:
      app: pay-api
      slot: stable
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: pay-api
        slot: stable
    spec:
      containers:
      - image: busybox
        name: log-shipper
        command: ['/bin/sh' , '-c', 'tail -f /audit/app.log']
        volumeMounts:
        - name: log-scratch
          mountPath: /audit
      - image: nginx:1.24
        name: nginx
        livenessProbe:
          periodSeconds: 15
          httpGet:
            path: /
            port: 80
        readinessProbe:
          initialDelaySeconds: 5
          httpGet:
            path: /
            port: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
        envFrom:
        - configMapRef:
            name: pay-api-config
        - secretRef:
            name: pay-api-secret
        volumeMounts:
        - name: log-scratch
          mountPath: /audit
        - name: pay-audit-pvc
          mountPath: /persistent-audit
      volumes:
      - name: log-scratch
        emptyDir: {}
      - name: pay-audit-pvc
        persistentVolumeClaim:
          claimName: pay-audit-pvc
status: {}
❯ kubectl create -f dep.yaml
Error from server (BadRequest): error when creating "dep.yaml": Deployment in version "v1" cannot be handled as a Deployment: strict decoding error: unknown field "spec.serviceAccountName"
❯ vim dep.yaml
❯ cat dep.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: pay-api
    slot: stable
  name: pay-api-stable
  namespace: payments
spec:
  replicas: 3
  serviceaccountName: pay-api-sa
  selector:
    matchLabels:
      app: pay-api
      slot: stable
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: pay-api
        slot: stable
    spec:
      containers:
      - image: busybox
        name: log-shipper
        command: ['/bin/sh' , '-c', 'tail -f /audit/app.log']
        volumeMounts:
        - name: log-scratch
          mountPath: /audit
      - image: nginx:1.24
        name: nginx
        livenessProbe:
          periodSeconds: 15
          httpGet:
            path: /
            port: 80
        readinessProbe:
          initialDelaySeconds: 5
          httpGet:
            path: /
            port: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
        envFrom:
        - configMapRef:
            name: pay-api-config
        - secretRef:
            name: pay-api-secret
        volumeMounts:
        - name: log-scratch
          mountPath: /audit
        - name: pay-audit-pvc
          mountPath: /persistent-audit
      volumes:
      - name: log-scratch
        emptyDir: {}
      - name: pay-audit-pvc
        persistentVolumeClaim:
          claimName: pay-audit-pvc
status: {}
❯ kubectl create -f dep.yaml
Error from server (BadRequest): error when creating "dep.yaml": Deployment in version "v1" cannot be handled as a Deployment: strict decoding error: unknown field "spec.serviceaccountName"
❯ vim dep.yaml
❯ cat dep.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: pay-api
    slot: stable
  name: pay-api-stable
  namespace: payments
spec:
  replicas: 3
  selector:
    matchLabels:
      app: pay-api
      slot: stable
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: pay-api
        slot: stable
    spec:
      serviceAccountName: pay-api-sa
      containers:
      - image: busybox
        name: log-shipper
        command: ['/bin/sh' , '-c', 'tail -f /audit/app.log']
        volumeMounts:
        - name: log-scratch
          mountPath: /audit
      - image: nginx:1.24
        name: nginx
        livenessProbe:
          periodSeconds: 15
          httpGet:
            path: /
            port: 80
        readinessProbe:
          initialDelaySeconds: 5
          httpGet:
            path: /
            port: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
        envFrom:
        - configMapRef:
            name: pay-api-config
        - secretRef:
            name: pay-api-secret
        volumeMounts:
        - name: log-scratch
          mountPath: /audit
        - name: pay-audit-pvc
          mountPath: /persistent-audit
      volumes:
      - name: log-scratch
        emptyDir: {}
      - name: pay-audit-pvc
        persistentVolumeClaim:
          claimName: pay-audit-pvc
status: {}
❯ kubectl create -f dep.yaml
deployment.apps/pay-api-stable created
```

### Part F — Canary deployment (v2, 25% of traffic)

Create a Deployment `pay-api-canary` in `payments` with:
- **1 replica** (1 out of 4 total pods = 25%), image `nginx:1.25`
- Same labels `app=pay-api`, but `slot=canary`
- Same ServiceAccount, env vars, volumes, and probes as stable
- Resource requests and limits identical to stable
```bash
❯ cp dep.yaml new-dep.yaml
❯ vim new-dep.yaml
❯ cat new-dep.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: pay-api
    slot: canary
  name: pay-api-canary
  namespace: payments
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pay-api
      slot: canary
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: pay-api
        slot: canary
    spec:
      serviceAccountName: pay-api-sa
      containers:
      - image: busybox
        name: log-shipper
        command: ['/bin/sh' , '-c', 'tail -f /audit/app.log']
        volumeMounts:
        - name: log-scratch
          mountPath: /audit
      - image: nginx:1.25
        name: nginx
        livenessProbe:
          periodSeconds: 15
          httpGet:
            path: /
            port: 80
        readinessProbe:
          initialDelaySeconds: 5
          httpGet:
            path: /
            port: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
        envFrom:
        - configMapRef:
            name: pay-api-config
        - secretRef:
            name: pay-api-secret
        volumeMounts:
        - name: log-scratch
          mountPath: /audit
        - name: pay-audit-pvc
          mountPath: /persistent-audit
      volumes:
      - name: log-scratch
        emptyDir: {}
      - name: pay-audit-pvc
        persistentVolumeClaim:
          claimName: pay-audit-pvc
status: {}
❯ kubectl create -f new-dep.yaml
deployment.apps/pay-api-canary created
```

### Part G — Service (routes to both stable + canary via shared label)

Create a ClusterIP Service `pay-api-svc` in `payments` that selects **only** on `app=pay-api` (not on `slot`), port 80 → targetPort 80. This naturally splits traffic 3:1 across all 4 pods.
```bash
❯ kubectl expose deploy -n payments pay-api-stable --port=80 --target-port=80 --dry-run=client -o yaml > svc.yaml
❯ vim svc.yaml
❯ cat svc.yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: pay-api
  name: pay-api-svc
  namespace: payments
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: pay-api
status:
  loadBalancer: {}
❯ kubectl create -f svc.yaml
service/pay-api-svc created
```

### Part H — NetworkPolicy

Create a NetworkPolicy `payments-isolation` in `payments` that:
1. Denies all ingress by default to all pods in the namespace
2. Allows ingress to pods with `app=pay-api` from pods with `role=gateway` in namespace `ingress-ns`, on port 80
3. Allows egress from `app=pay-api` pods only to DNS (port 53 UDP/TCP)

```bash
❯ vim netpol.yaml
❯ cat netpol.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payments-isolation
  namespace: payments
spec:
  podSelector:
    matchLabels:
      app: pay-api
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: gateway
      namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-ns
      ports:
      - protocol: TCP
        port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
      namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      ports:
      - protocol: UDP
        port: 53
      - protocol: TCP
        port: 53
  policyTypes:
  - Ingress
  - Egress
❯ kubectl create -f netpol.yaml
Error from server (BadRequest): error when creating "netpol.yaml": NetworkPolicy in version "v1" cannot be handled as a NetworkPolicy: json: cannot unmarshal object into Go struct field NetworkPolicyEgressRule.spec.egress.to of type []v1.NetworkPolicyPeer
❯ vim netpol.yaml
❯ cat netpol.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payments-isolation
  namespace: payments
spec:
  podSelector:
    matchLabels:
      app: pay-api
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: gateway
      namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-ns
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
      namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  policyTypes:
  - Ingress
  - Egress
❯ kubectl create -f netpol.yaml
networkpolicy.networking.k8s.io/payments-isolation created
```

---
## Exercise 9 — Finding and Fixing Deprecated API Versions

**Scenario:** A colleague committed manifests for a new `CronJob` and an `Ingress` years ago. The cluster has since been upgraded and those manifests now use removed `apiVersion` values. Your job is to identify the correct current versions, update the manifests, and verify the cluster accepts them. You also need to use `kubectl convert` (via the kubectl-convert plugin) on a third manifest.

---

### Part A — Know your tools

Before touching any manifest, run these commands and understand their output:

```bash
# List every API group and version the current cluster supports
kubectl api-versions

# Explain a field to confirm which version it lives in
kubectl explain cronjob --api-version=batch/v1
kubectl explain ingress --api-version=networking.k8s.io/v1
```
>[!EXPLANATION]
> kubectl api-versions lists the current api versions of all resources in the cluster
> `kubectl explain <resource> --api-version=<version>` allows you to understand the resource definition of the resource with the provided api-version


### Part B — Broken CronJob manifest (batch/v1beta1 → batch/v1)

`batch/v1beta1` was removed in Kubernetes 1.25. The following manifest will be rejected by any cluster running 1.25+.

**Broken manifest — `broken-cronjob.yaml`:**
```yaml
apiVersion: batch/v1beta1      # ← REMOVED in k8s 1.25
kind: CronJob
metadata:
  name: report-generator
  namespace: default
spec:
  schedule: "0 6 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: reporter
            image: busybox
            command: ["/bin/sh", "-c", "echo generating report"]
```

**Task:** Fix the `apiVersion` and apply it.
```bash
❯ vim broken-cronjob.yaml
❯ cat broken-cronjob.yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: report-generator
  namespace: default
spec:
  schedule: "0 6 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: reporter
            image: busybox
            command: ["/bin/sh", "-c", "echo generating report"]
❯ kubectl create -f broken-cronjob.yaml
error: resource mapping not found for name: "report-generator" namespace: "default" from "broken-cronjob.yaml": no matches for kind "CronJob" in version "batch/v1beta1"
ensure CRDs are installed first

❯ cp broken-cronjob.yaml working-cronjob.yaml
❯ vim working-cronjob.yaml
❯ cat working-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: report-generator
  namespace: default
spec:
  schedule: "0 6 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: reporter
            image: busybox
            command: ["/bin/sh", "-c", "echo generating report"]
❯ kubectl create -f working-cronjob.yaml
cronjob.batch/report-generator created
```

### Part C — Broken Ingress manifest (extensions/v1beta1 → networking.k8s.io/v1)

`extensions/v1beta1` for Ingress was removed in Kubernetes 1.22. Beyond the `apiVersion`, the spec structure itself changed — `backend` syntax is different in `networking.k8s.io/v1`.

**Broken manifest — `broken-ingress.yaml`:**
```yaml
apiVersion: extensions/v1beta1 
kind: Ingress
metadata:
  name: old-ingress
spec:
  rules:
  - host: old.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: web-svc
          servicePort: 80
```

**Task:** Convert this to the correct `networking.k8s.io/v1` format and apply it.
```bash
❯ vim broken-ingress.yaml
❯ cat broken-ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: old-ingress
spec:
  rules:
  - host: old.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: web-svc
          servicePort: 80
❯ kubectl create -f broken-ingress.yaml
error: resource mapping not found for name: "old-ingress" namespace: "" from "broken-ingress.yaml": no matches for kind "Ingress" in version "extensions/v1beta1"
ensure CRDs are installed first

❯ vim working-ingress.yaml
❯ cat working-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: old-ingress
spec:
  rules:
  - host: old.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port:
              number: 80


❯ kubectl create -f working-ingress.yaml
ingress.networking.k8s.io/old-ingress created
```

### Part D — Using kubectl-convert

`kubectl convert` is a standalone plugin (`kubectl-convert`) that rewrites a manifest from one API version to another. On the exam it is usually pre-installed.

```bash
# Check if it's available
kubectl convert --help

# Convert the broken CronJob manifest automatically
kubectl convert -f broken-cronjob.yaml --output-version batch/v1

# Convert and write output to a new file
kubectl convert -f broken-cronjob.yaml --output-version batch/v1 > converted-cronjob.yaml
cat converted-cronjob.yaml
kubectl apply -f converted-cronjob.yaml
```

---

