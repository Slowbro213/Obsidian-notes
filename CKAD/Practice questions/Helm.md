### Creating a basic Helm chart
```bash
helm create chart-test ## this would create a helm
```
>[!IMPORTANT]
>Create a basic helm chart via `helm create <name>`

---
### Running a Helm chart
```bash
helm install -f myvalues.yaml myredis ./redis
```
>[!IMPORTANT]
>To run a helm chart, you can use the following syntax
>`helm install -f <values-file> <name> <chart-directory>

---
### Find pending Helm deployments on all namespaces
```bash
❯ helm list --pending -A
```
>[!IMPORTANT]
>Get pending helm deployments via `helm list --pending`

>[!IMPORTANT]
>For any kind of helm listing or finding, you can use `helm list --help`

---
### Uninstall a Helm release
```bash
helm uninstall -n namespace name
```
>[!NOTE]
>Uninstall a helm release via `helm uninstall -n namespace name`

>[!NOTE]
>`helm` commands use namespaces just like `kubectl` commands

---
### Upgrading a Helm chart
```bash
helm upgrade -f ./myvalues.yaml -f ./override.yaml redis ./redis
```
>[!NOTE]
>For upgrading a helm chart, the command is just like installing but it also accepts an additional `-f ./override.yaml` argument with new values to override the old

---
### Using Helm repo
```bash
❯ helm repo --help

This command consists of multiple subcommands to interact with chart repositories.

It can be used to add, remove, list, and index chart repositories.

Usage:
  helm repo [command]

Available Commands:
  add         add a chart repository
  index       generate an index file given a directory containing packaged charts
  list        list chart repositories
  remove      remove one or more chart repositories
  update      update information of available charts locally from chart repositories

Flags:
  -h, --help   help for repo

Global Flags:
      --burst-limit int                 client-side default throttling limit (default 100)
      --debug                           enable verbose output
      --kube-apiserver string           the address and the port for the Kubernetes API server
      --kube-as-group stringArray       group to impersonate for the operation, this flag can be repeated to specify multiple groups.
      --kube-as-user string             username to impersonate for the operation
      --kube-ca-file string             the certificate authority file for the Kubernetes API server connection
      --kube-context string             name of the kubeconfig context to use
      --kube-insecure-skip-tls-verify   if true, the Kubernetes API server's certificate will not be checked for validity. This will make your HTTPS connections insecure
      --kube-tls-server-name string     server name to use for Kubernetes API server certificate validation. If it is not provided, the hostname used to contact the server is used
      --kube-token string               bearer token used for authentication
      --kubeconfig string               path to the kubeconfig file
  -n, --namespace string                namespace scope for this request
      --qps float32                     queries per second used when communicating with the Kubernetes API, not including bursting
      --registry-config string          path to the registry config file (default "/home/slowking/.config/helm/registry/config.json")
      --repository-cache string         path to the directory containing cached repository indexes (default "/home/slowking/.cache/helm/repository")
      --repository-config string        path to the file containing repository names and URLs (default "/home/slowking/.config/helm/repositories.yaml")

Use "helm repo [command] --help" for more information about a command.
```
>[!NOTE]
>There are various `helm repo` commands

---
### Download a Helm chart from a repository 
```bash
helm pull [chart URL | repo/chartname] [...] [flags]
```
>[!NOTE]
>`helm pull --help` can be useful, but generally to download a helm chart you use `helm pull`

---
### Add the Bitnami repo at https://charts.bitnami.com/bitnami to Helm
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```
---
### Write the contents of the values.yaml file of the `bitnami/node` chart to standard output
```bash
helm show values bitnami/node
```
>[!IMPORTANT]
>`helm show` can be used to get information about a helm chart. Use `helm show --help` for more information

---
### Install the `bitnami/node` chart setting the number of replicas to 5
```bash
❯ helm show values bitnami/node | grep -A 10 replicas
## @param replicaCount Specify the number of replicas for the application
##
replicaCount: 1
## @param updateStrategy.type Strategy to use to replace existing pods.
## ref: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy
## Example:
## updateStrategy:
##  type: RollingUpdate
##  rollingUpdate:
##    maxSurge: 25%
##    maxUnavailable: 25%
❯ vim myvalues.yaml
❯ cat myvalues.yaml
replicaCount: 5
❯ helm install -f ./myvalues.yaml node bitnami/node
WARNING: This chart is deprecated
NAME: node
LAST DEPLOYED: Tue Jun 16 22:03:15 2026
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
This Helm chart is deprecated

CHART NAME: node
CHART VERSION: 19.1.7
APP VERSION: 16.18.0

** Please be patient while the chart is being deployed **

1. Get the URL of your Node app  by running:

  kubectl port-forward --namespace default svc/node 80:80
  echo "Node app URL: http://127.0.0.1:80/"
❯ kubectl get deploy
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
node           0/5     5            0           13s
node-mongodb   0/1     1            0           13s
```
