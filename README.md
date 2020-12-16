# helm-fest

helm-fest is about deploying a bunch of useful stuff on kubernetes using helm.

## Preliminary notes

This walkthrough was developed by using the kubernetes cluster provided by [https://github.com/deepshore/kubernetes-vagrant-for-dummies](https://github.com/deepshore/kubernetes-vagrant-for-dummies).

## Requirements

The following is required:
- a kubernetes cluster
- kubectl
- helm

## Kubernetes

Check if a cluster is available:

```bash
kubectl get nodes
```

## Install kube-prometheus-stack via chart repository

Add repo for kube-prometheus-stack:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

Add repos for subcharts:

```bash
helm repo add stable https://charts.helm.sh/stable
helm repo add grafana https://grafana.github.io/helm-charts
```

Update repos:

```bash
helm repo update
```

Create namespace:

```bash
kubectl create namespace monitoring
```

Install chart:

```bash
helm install -n monitoring kps prometheus-community/kube-prometheus-stack -f values/values-monitoring.yaml
```

Check deployment:

```bash
kubectl -n monitoring get all
```

Access Grafana:

```bash
export NODE_IP=$(kubectl get nodes -o jsonpath="{.items[0].status.addresses[0].address}")
export NODE_PORT=$(kubectl get --namespace monitoring -o jsonpath="{.spec.ports[0].nodePort}" services kps-grafana)
echo "http://${NODE_IP}:${NODE_PORT}"
```

## Install kubernetes-dashboard via unpacked chart directory

Get the code of the chart:

```bash
git clone https://github.com/kubernetes/dashboard
```
Add repos for subcharts:

```bash
helm repo add stable https://charts.helm.sh/stable
```

Create namespace:

```bash
kubectl create namespace dashboard
```

Create a service account with clusterrole cluster-admin:

```bash
kubectl -n dashboard create sa dashboard-admin
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=dashboard:dashboard-admin
```

Build dependencies:

```bash
helm dependency build dashboard/aio/deploy/helm-chart/kubernetes-dashboard/
```

Install chart:

```bash
helm -n dashboard install kubernetes-dashboard dashboard/aio/deploy/helm-chart/kubernetes-dashboard/ -f values/values-dashboard.yaml
```

Access dashboard:

```bash
export NODE_PORT=$(kubectl get -n dashboard -o jsonpath="{.spec.ports[0].nodePort}" services kubernetes-dashboard)
export NODE_IP=$(kubectl get nodes -o jsonpath="{.items[0].status.addresses[0].address}")
echo http://$NODE_IP:$NODE_PORT/
```

## Install postgres via local chart archive

Get the code of the chart:

```bash
git clone https://github.com/bitnami/charts.git
```

Add repos for subcharts:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Create namespace:

```bash
kubectl create namespace postgresql
```

Build dependencies:

```bash
helm dependency build charts/bitnami/postgresql
```

Package chart:

```bash
helm package charts/bitnami/postgresql
```

Install chart:

```bash
helm -n postgresql install postgresql 
```

Check if database is running:

```bash
kubectl -n postgresql get all
```

## Versions

For this tutorial the following chart versions were used:
- [https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack): 12.8.0
- [https://github.com/kubernetes/dashboard/tree/master/aio/deploy/helm-chart/kubernetes-dashboard](https://github.com/kubernetes/dashboard/tree/master/aio/deploy/helm-chart/kubernetes-dashboard): 3.0.2
- [https://github.com/bitnami/charts/tree/master/bitnami/postgresql](https://github.com/bitnami/charts/tree/master/bitnami/postgresql): 10.2.0

## Changes to the default values

Changes were made to the following values.

**kube-prometheus-stack**:
- grafana.service.type: NodePort
- grafana.service.nodePort: 30030
- prometheusOperator.tls.enabled: false
- prometheusOperator.admissionWebhooks.enabled: false

**kubernetes-dashboard**:
- extraEnv[0].name: enable-insecure-login
- extraEnv[0].value: 'true'
- protocolHttp: true
- service.type: NodePort
- service.nodePort: 30443
- rbac.create: false
- serviceAccount.create: false
- serviceAccount.name: dashboard-admin

**postgresql**:
- service.type: NodePort
- service.nodePort: 30432
- persistence.enabled: false
