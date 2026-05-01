## Core Concepts

### Helm

Helm is a package manager for k8s with template files for resources. Charts are a collection of template files for a set of related resources packages as a single unit. Releases are running instances of a Chart.

### Kustomize

Kustomize is a template free tool for customizing k8s manifests. Built in directly into kubectl via the `-k` flag. Works by applying patches and overlays to a common set of YAML files.

---

## Install Helm

We first need to install Helm:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

---

## Add a Helm Repository

As is with package managers, we need to add a repository:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

And then we update helm:

```bash
helm repo update
```

---

## Install a Chart

Now we install a chart:

```bash
helm install my-nginx bitnami/nginx --set service.type=NodePort
```

Lets break this command down:

```bash
helm install <release_name> <repo>/<package> --set service.type=NodePort
```

The NodePort service type allows this to be exposed publicly.

Now we can manage this package using helm!

```bash
helm upgrade my-nginx bitnami/nginx --set service.type=ClusterIP
helm rollback my-nginx 1
helm uninstall my-nginx
```

---

## Managing a Deployment with Kustomize

Now we are going to manage a deployment using Kustomize.

### First, create a deployment

```bash
mkdir -p my-app/base
cat <<EOF > my-app/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.25.0
EOF
```

### Create base Kustomization file (`my-app/base/kustomization.yaml`)

```bash
cat <<EOF > my-app/base/kustomization.yaml
resources:
- deployment.yaml
EOF
```

### Create production overlay and patch

```bash
mkdir -p my-app/overlays/production
cat <<EOF > my-app/overlays/production/patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
name: my-app
spec:
replicas: 3
EOF

cat <<EOF > my-app/overlays/production/kustomization.yaml
bases:
-../../base
patches:
- path: patch.yaml
EOF
```

### Apply the overlay

**Apply the overlay (note the** `-k` flag for kustomize):

```bash
kubectl apply -k my-app/overlays/production
```

---

## Verify the Change

```bash
vboxuser@k8s-init:~$ kubectl apply -k my-app/overlays/production
# Warning: 'bases' is deprecated. Please use 'resources' instead. Run 'kustomize edit fix' to update your Kustomization automatically.
deployment.apps/my-app created

vboxuser@k8s-init:~$ kubectl get deployments
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
my-app     0/3     3            0           9s
my-nginx   0/1     1            0           20m

vboxuser@k8s-init:~$ kubectl get deployment my-app
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
my-app   0/3     3            0           14s
```
