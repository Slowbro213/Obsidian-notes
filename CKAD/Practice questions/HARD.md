
## Exercise 1 — Secure Microservice Platform with Canary Release

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


## Exercise 2 — Cluster-Wide Log Collector with Node Scheduling

**Scenario:** Your platform team needs a DaemonSet that collects logs from every node (including control-plane). It must run with elevated privileges, tolerate control-plane taints, and only run on nodes with a specific label. It reads its configuration from a ConfigMap mounted as a volume, writes to a hostPath, and must not be scheduled on nodes tainted `disk=slow:NoSchedule`. An init container must verify the target directory exists before the main container starts.

**Domains covered:** DaemonSets, init containers, tolerations, taints, node affinity, ConfigMap as volume, hostPath volume, security context (privileged), resource limits.

---

### Part A — Label a node and create the ConfigMap
```bash
❯ kubectl label node minikube log-collection=enabled
node/minikube labeled
❯ kubectl create configmap log-collector-config \
  --from-literal=OUTPUT_PATH=/var/log/collected \
  --from-literal=FLUSH_INTERVAL=30 \
  -n kube-system

configmap/log-collector-config created
```

### Part B — DaemonSet

Create a DaemonSet `log-collector` in `kube-system` with:
- Init container `dir-check` (image `busybox`): runs `mkdir -p /var/log/collected && echo ready`
- Main container `collector` (image `busybox`): runs `while true; do echo collecting; sleep 30; done`
- The ConfigMap `log-collector-config` mounted as a volume at `/etc/collector-config`
- A hostPath volume `host-logs` pointing to `/var/log` mounted at `/host-logs` in the main container
- A `nodeSelector`: `log-collection: enabled`
- Tolerations for:
  - `node-role.kubernetes.io/control-plane:NoSchedule` (operator `Exists`)
  - `disk=slow:NoSchedule` — **add this as a NoExecute toleration so it runs even on those nodes** ← actually, the scenario says to NOT run on those — so instead add a node **affinity** `requiredDuringSchedulingIgnoredDuringExecution` that requires `disk != slow` using a `NotIn` operator.
- Container `securityContext`: `privileged: true`
- Resource limits: `cpu: 50m`, `memory: 64Mi`
```bash
❯ kubectl create deployment log-collector -n kube-system --image=busybox --dry-run=client -o yaml > ds.yaml
❯ vim ds.yaml
❯ cat ds.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  creationTimestamp: null
  labels:
    app: log-collector
  name: log-collector
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: log-collector
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: disk
        operator: Equal
        value: slow
        effect: NoSchedule
      nodeSelector:
        log-collection: enabled
        affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: disk
                  operator: NotIn
                  values:
                  - slow
      initContainers:
      - name: dir-check
        image: busybox
        command: ['/bin/sh', '-c', 'mkdir -p /var/log/collected && echo ready']
        volumeMounts:
        - name: vol
          mountPath: /etc/collector-config
      containers:
      - image: busybox
        name: busybox
        command: ['/bin/sh', '-c', 'while true; do echo collecting; sleep 30; done']
        resources:
          limits:
            cpu: 50m
            memory: 64Mi
        securityContext:
          privileged: true
        volumeMounts:
        - name: vol
          mountPath: /etc/collector-config
        - name: host-logs
          mountPath: /host-logs
      volumes:
      - name: vol
        configMap:
          name: log-collector-config
      - name: host-logs
        hostPath:
          path: /var/log
status: {}
❯ kubectl create -f ds.yaml
daemonset.apps/log-collector created
```
---
## Exercise 3 — Multi-Tenant Web App with Full RBAC, Ingress, and HPA

**Scenario:** You are deploying a two-tier app in the `webapp` namespace: a frontend (`React` app served by nginx) and a backend API (`node-api`). The frontend calls the backend internally. External users reach the frontend via an Ingress on `app.example.com`. The backend gets `/api` traffic via a separate Ingress path. Each tier has its own ServiceAccount with minimal permissions. The backend auto-scales based on CPU. A LimitRange enforces per-container memory caps.

**Domains covered:** Namespaces, LimitRange, Deployments, Services, Ingress (multi-path + TLS), HPA, RBAC (two ServiceAccounts with different Roles), NetworkPolicy, init containers, labels/selectors.

---

### Part A — Namespace, LimitRange, and TLS secret
Create a namespace named `webapp`.
In the `webapp` namespace, create a `LimitRange` named `webapp-limits`.
The `LimitRange` must apply to containers.
Configure the `LimitRange` so that containers in the namespace have a maximum allowed CPU limit of `500m` and a maximum allowed memory limit of `512Mi`.
Configure the minimum allowed resource values so that containers must request at least `50m` CPU and `64Mi` memory.
Configure default limits so that containers without explicit limits automatically receive `100m` CPU and `128Mi` memory.
Configure default requests so that containers without explicit requests automatically receive `50m` CPU and `64Mi` memory.
Create a TLS secret named `webapp-tls` in the `webapp` namespace.
The TLS secret must be created from the files `tls.crt` and `tls.key`.
When practicing locally, generate a self-signed certificate for `app.example.com` if the certificate and key files are not already provided.
```bash
❯ kubectl create ns webapp
namespace/webapp created
❯ vim limitrange.yaml
❯ cat limitrange.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: webapp-limits
  namespace: webapp
spec:
  limits:
  - default:
      cpu: 100m
      memory: 128Mi
    defaultRequest:
      cpu: 50m
      memory: 128Mi
    max:
      cpu: 500m
      memory: 512Mi
    min:
      cpu: 50m
      memory: 128Mi
    type: Container
❯ kubectl create -f limitrange.yaml
limitrange/webapp-limits created
```

### Part B — RBAC for frontend

1. ServiceAccount `frontend-sa` in `webapp`
2. Role `frontend-role`: allows `get`, `list` on Services and Endpoints in `webapp`
3. RoleBinding `frontend-rb`
```bash
❯ kubectl create serviceaccount -n webapp frontend-sa
serviceaccount/frontend-sa created
❯ kubectl create role frontend-role --verb=get,list --resource=service,endpoints -n webapp
role.rbac.authorization.k8s.io/frontend-role created
❯ kubectl create rolebinding -n webapp frontend-rb --role=frontend-role --serviceaccount=webapp:frontend-sa
rolebinding.rbac.authorization.k8s.io/frontend-rb created
```
### Part C — RBAC for backend

1. ServiceAccount `backend-sa` in `webapp`
2. Role `backend-role`: allows `get`, `list`, `watch` on ConfigMaps only
3. RoleBinding `backend-rb`
```bash
❯ kubectl create serviceaccount backend-sa -n webapp
serviceaccount/backend-sa created
❯ kubectl create role -n webapp --verb=get,list,watch --resource=configmap backend-role
role.rbac.authorization.k8s.io/backend-role created
❯ kubectl create rolebinding -n webapp backend-rb --role=backend-role --serviceaccount=webapp:backend-sa
rolebinding.rbac.authorization.k8s.io/backend-rb created
```

### Part D — Backend deployment and service

Create a ConfigMap `backend-config` with `NODE_ENV=production` and `PORT=3000` in `webapp`.

Create Deployment `node-api` in `webapp`:
- 2 replicas, image `nginx:1.24` (simulating node), port 3000
- `serviceAccountName: backend-sa`
- All keys from `backend-config` loaded via `envFrom`
- Init container `schema-check` (image `busybox`): runs `echo "DB schema OK"` (simulates migration check)
- Label `app=node-api`, `tier=backend`
- readinessProbe: HTTP GET `/` port 80, `initialDelaySeconds: 3`
- livenessProbe: HTTP GET `/` port 80, `periodSeconds: 20`

Then create a ClusterIP Service `node-api-svc` on port 3000 → targetPort 80.
```bash
❯ kubectl create deploy node-api -n webapp --image=nginx:1.24 --replicas=2 --dry-run=client -o yaml > dep.yaml
❯ vim dep.yaml
❯ cat dep.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: node-api
    tier: backend
  name: node-api
  namespace: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: node-api
      tier: backend
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: node-api
        tier: backend
    spec:
      serviceAccountName: backend-sa
      initContainers:
      - name: schema-check
        image: busybox
        command: ['/bin/sh', '-c', 'echo "DB schema OK"']
      containers:
      - image: nginx:1.24
        name: nginx
        resources: {}
        ports:
        - containerPort: 80
        readinessProbe:
          initialDelaySeconds: 3
          httpGet:
            path: /
            port: 80
        livenessProbe:
          periodSeconds: 20
          httpGet:
            path: /
            port: 80
        envFrom:
        - configMapRef:
            name: backend-config
status: {}
❯ kubectl create -f dep.yaml
deployment.apps/node-api created
❯ kubectl expose deploy node-api -n webapp --port=3000 --target-port=80 --dry-run=client -o yaml > svc.yaml
❯ vim svc.yaml
❯ cat svc.yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: node-api
    tier: backend
  name: node-api-svc
  namespace: webapp
spec:
  ports:
  - port: 3000
    protocol: TCP
    targetPort: 80
  selector:
    app: node-api
    tier: backend
status:
  loadBalancer: {}
❯ kubectl create -f svc.yaml
service/node-api-svc created
```

### Part E — Frontend deployment and service

Create Deployment `frontend` in `webapp`:
- 2 replicas, image `nginx:1.24`, port 80
- `serviceAccountName: frontend-sa`
- Env var `BACKEND_URL=http://node-api-svc:3000` set directly
- Label `app=frontend`, `tier=frontend`
- startupProbe: HTTP GET `/` port 80, `failureThreshold: 10`, `periodSeconds: 5`
- readinessProbe: HTTP GET `/` port 80
- livenessProbe: HTTP GET `/` port 80, `periodSeconds: 20`

Then create ClusterIP Service `frontend-svc` on port 80 → targetPort 80.
```bash
❯ kubectl create deploy frontend -n webapp --replicas=2 --image=nginx:1.24 --port=80 --dry-run=client -o yaml > fe.yaml
❯ vim fe.yaml
❯ cat fe.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: frontend
    tier: frontend
  name: frontend
  namespace: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
      tier: frontend
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: frontend
        tier: frontend
    spec:
      serviceAccountName: frontend-sa
      containers:
      - image: nginx:1.24
        name: nginx
        ports:
        - containerPort: 80
        resources: {}
        startupProbe:
          failureThreshold: 10
          periodSeconds: 5
          httpGet:
            port: 80
            path: /
        readinessProbe:
          httpGet:
            port: 80
            path: /
        livenessProbe:
          periodSeconds: 20
          httpGet:
            port: 80
            path: /
        env:
        - name: BACKEND_URL
          value: http://node-api-svc:3000
status: {}
❯ kubectl create -f fe.yaml
deployment.apps/frontend created
❯ kubectl expose deploy -n webapp --port=80 --target-port=80 --dry-run=client -o yaml > fesvc.yaml
error: resource(s) were provided, but no name was specified
❯ kubectl expose deploy frontend -n webapp --port=80 --target-port=80 --dry-run=client -o yaml > fesvc.yaml
❯ vim fesvc.yaml
❯ cat fesvc.yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: frontend
    tier: frontend
  name: frontend-svc
  namespace: webapp
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: frontend
    tier: frontend
status:
  loadBalancer: {}
❯ kubectl create -f fesvc.yaml
service/frontend-svc created
```
### Part F — Ingress with TLS and multiple paths

Create an Ingress `webapp-ingress` in `webapp`:
- TLS using secret `webapp-tls` for host `app.example.com`
- Rule for `app.example.com`:
  - Path `/` (Prefix) → `frontend-svc:80`
  - Path `/api` (Prefix) → `node-api-svc:3000`
```bash
❯ vim ingress.yaml
❯ cat ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  namespace: webapp
spec:
  tls:
  - hosts:
      - app.example.com
    secretName: webapp-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: node-api-svc
            port:
              number: 3000
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80

❯ kubectl create -f ingress.yaml
ingress.networking.k8s.io/webapp-ingress created
```
### Part G — HPA for the backend

Create an HPA for `node-api` that keeps between 2 and 8 replicas targeting 60% CPU utilisation.
```bash
❯ kubectl autoscale -n webapp deploy node-api --cpu-percent=60 --min=2 --max=8
horizontalpodautoscaler.autoscaling/node-api autoscaled
```

### Part H — NetworkPolicy

In `webapp`, create NetworkPolicies that enforce:
1. Frontend pods (`tier=frontend`) may only receive ingress from the Ingress controller namespace (`ingress-nginx`) and only send egress to `tier=backend` pods on port 3000, plus DNS.
2. Backend pods (`tier=backend`) may only receive ingress from `tier=frontend` pods on port 80, plus DNS egress only.
```bash
❯ cat default-deny-ingress.yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: webapp
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  - Egress

❯ cat allow-ingress.yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: permit-from
  namespace: webapp
spec:
  podSelector:
    matchLabels:
      tier: frontend
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - port: 3000
      protocol: TCP
  - to:
    - podSelector:
        matchLabels:
          kube-app: kube-dns
      namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP

  policyTypes:
  - Ingress
  - Egress
    
❯ kubectl create -f allow-ingress.yaml
networkpolicy.networking.k8s.io/permit-from created
❯ kubectl create -f default-deny-ingress.yaml
networkpolicy.networking.k8s.io/default-deny-ingress created
```