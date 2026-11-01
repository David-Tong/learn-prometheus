# Prometheus + Grafana Deployment Guide

Deploy Prometheus and Grafana on Kubernetes with Helm, expose them through NodePort, and use Prometheus as a Grafana data source.

## What Is Helm?

Helm is a package manager for Kubernetes applications. A Helm Chart contains templates and default configuration for Kubernetes resources. An installed chart instance is called a Release.

Installing the **Helm CLI** on your computer does not create Pods in the cluster. `helm repo add` and `helm repo update` only configure and update chart repository information on your computer. Running `helm install` deploys the resources defined by a chart to Kubernetes, such as Deployments, Services, and PVCs. Kubernetes controllers then create and maintain Pods. Helm therefore deploys more than just Pods; the exact resources depend on the chart and its configuration.

In this guide, `helm install prometheus ...` and `helm install grafana ...` deploy the applications. The later `kubectl expose ...` commands create separate NodePort Services using `kubectl`; those Services are not created by Helm.

Reference: [Deploy Prometheus and Grafana on Kubernetes using Helm](https://medium.com/@gayatripawar401/deploy-prometheus-and-grafana-on-kubernetes-using-helm-5aa9d4fbae66)

## Prerequisites

Configure `kubectl` and install Helm. Confirm that the cluster has an available StorageClass:

```bash
kubectl get nodes
kubectl get storageclass
```

The commands below use `nfs-client`. Replace it with your StorageClass name if it is different.

## Install Prometheus

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/prometheus --namespace prometheus --create-namespace --set server.persistentVolume.storageClass=nfs-client --set alertmanager.persistentVolume.storageClass=nfs-client
```

## Install Grafana

Grafana is a separate Helm Release; installing the Prometheus chart does not install Grafana. Add the Grafana chart repository, then install Grafana in the `prometheus` namespace:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install grafana grafana/grafana --namespace prometheus --set persistence.enabled=true --set persistence.storageClassName=nfs-client
```

On success, Helm should report `STATUS: deployed`. If no chart version is specified, Helm selects an available version from the repository. The example selected version `10.5.15` and reported that the chart is deprecated. This warning does not mean the installation failed, but for new or production deployments, verify and choose a Grafana chart that is still supported.

## Verify the Installation

First, confirm that both Helm Releases are installed:

```bash
helm list -n prometheus
```

### Prometheus

```bash
kubectl get pods -n prometheus -l app.kubernetes.io/instance=prometheus
kubectl get service prometheus-server -n prometheus
```

### Grafana

```bash
kubectl get pods -n prometheus -l app.kubernetes.io/instance=grafana
kubectl get service grafana -n prometheus
```

Check the PersistentVolumeClaims:

```bash
kubectl get pvc -n prometheus
```

Confirm that `helm list` shows both the `prometheus` and `grafana` Releases. Their Pods should be `Running`, both Services should exist, and the PVCs should be `Bound`.

## Access Prometheus and Grafana

```bash
kubectl expose service prometheus-server --type=NodePort --target-port=9090 --name=prometheus-server-ext -n prometheus
kubectl expose service grafana --type=NodePort --target-port=3000 --name=grafana-ext -n prometheus
```

Find the Prometheus NodePort:

```bash
kubectl get service prometheus-server-ext -n prometheus -o custom-columns=NAME:.metadata.name,NODEPORT:.spec.ports[0].nodePort
```

Find the Grafana NodePort:

```bash
kubectl get service grafana-ext -n prometheus -o custom-columns=NAME:.metadata.name,NODEPORT:.spec.ports[0].nodePort
```

Find a node IP address:

```bash
kubectl get nodes -o wide
```

If you use the regular `kubectl get services` output, look at the number after the colon in the `PORT(S)` column. For example, in `80:31870/TCP`, `31870` is the NodePort. Choose a node IP from the `INTERNAL-IP` column that is reachable from your network, then replace the placeholders in these URLs:

```text
Prometheus: http://<node-ip>:<prometheus-nodeport>/query
Grafana:    http://<node-ip>:<grafana-nodeport>
```

Grafana's in-cluster service address is `grafana.prometheus.svc.cluster.local`. Sign in to Grafana with the username `admin`. Run this command in a Bash terminal to retrieve the initial password:

```bash
kubectl get secret grafana -n prometheus -o jsonpath='{.data.admin-password}' | base64 --decode; echo
```

After signing in, add Prometheus as a data source using `http://prometheus-server.prometheus.svc.cluster.local` as the URL. To add a Kubernetes monitoring dashboard, go to **Dashboards > New > Import** in Grafana, import a dashboard from Grafana.com, and select this Prometheus data source.

## Spring Boot API Observability Example

For deployment, Prometheus scraping, SLO alert rules, and Grafana queries for the Spring Boot API, see the [Observability guide](observability/README.md).

## Uninstall

```bash
kubectl delete service prometheus-server-ext -n prometheus
kubectl delete service grafana-ext -n prometheus
helm uninstall grafana -n prometheus
helm uninstall prometheus -n prometheus
```

PVC data may remain after uninstalling the Helm releases. Delete PVCs separately only after confirming that the data is no longer needed.
