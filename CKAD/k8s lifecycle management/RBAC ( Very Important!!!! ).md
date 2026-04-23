# Kubernetes RBAC Notes

------------------------------------------------------------------------

## Core RBAC Objects

There are 4 main objects RBAC is built on:

-   **Role**: set of permissions within a namespace\
-   **ClusterRole**: set of permissions to cluster resources or for all
    namespaces\
-   **RoleBinding**: Grants a user/group/ServiceAccount a Role\
-   **ClusterRoleBinding**: Grants a user/group/ServiceAccount a
    ClusterRole

------------------------------------------------------------------------

## Goal

Create a **ServiceAccount with read-only permissions**.

High-level steps: 1. Create a namespace
2. Create a ServiceAccount
3. Create a Role
4. Bind Role to ServiceAccount

------------------------------------------------------------------------

## Step 1: Create Namespace

``` bash
kubectl create namespace rbac-test
```

------------------------------------------------------------------------

## Step 2: Create ServiceAccount

``` bash
kubectl create serviceaccount dev-user -n rbac-test
```

------------------------------------------------------------------------

## Step 3: Create Role

YAML definition:

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: rbac-test
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

Create file:

``` bash
cat <<EOF | tee simple-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: rbac-test
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
EOF
```

Apply role:

``` bash
kubectl apply -f simple-role.yaml
```

------------------------------------------------------------------------

## Step 4: Create RoleBinding

Modified from Kubernetes docs:

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: rbac-test
subjects:
- kind: ServiceAccount
  name: dev-user
  namespace: rbac-test
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Create file:

``` bash
cat <<EOF | tee rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: rbac-test
subjects:
- kind: ServiceAccount
  name: dev-user
  namespace: rbac-test
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF
```

Apply:

``` bash
kubectl apply -f rolebinding.yaml
```

------------------------------------------------------------------------

## Step 5: Verify Permissions

Use `kubectl auth can-i`:

### Examples from docs

``` bash
kubectl auth can-i create pods --all-namespaces
kubectl auth can-i list deployments.apps
kubectl auth can-i list pods --as=system:serviceaccount:dev:foo -n prod
kubectl auth can-i '*' '*'
kubectl auth can-i list jobs.batch/bar -n foo
kubectl auth can-i get pods --subresource=log
kubectl auth can-i get /logs/
kubectl auth can-i approve certificates.k8s.io
kubectl auth can-i --list --namespace=foo
```

### Our Test

``` bash
kubectl auth can-i list pods --as=system:serviceaccount:rbac-test:dev-user -n rbac-test
```

Result:

``` bash
yes
```

### Our other Test
``` bash
kubectl auth can-i delete pods --as=system:serviceaccount:rbac-test:dev-user -n rbac-test
```

Result:

``` bash
no
```

------------------------------------------------------------------------

## Result

Read-only access to pods confirmed.

**great success!**
