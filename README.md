# Prometheus + Grafana 部署指南

使用 Helm 在 Kubernetes 集群中部署 Prometheus 和 Grafana，通过 NodePort 访问，并将 Prometheus 配置为 Grafana 的数据源。

## 什么是 Helm？

Helm 是 Kubernetes 应用的包管理器。Helm Chart 包含 Kubernetes 资源模板和默认配置；安装 Chart 后生成的应用实例称为 Release。

在电脑上**安装 Helm 命令行工具（Helm CLI）**不会在集群中创建 Pod。`helm repo add` 和 `helm repo update` 只会配置和更新本机的 Chart 仓库信息。运行 `helm install` 后，Helm 才会把 Chart 定义的资源部署到 Kubernetes，例如 Deployment、Service 和 PVC。随后 Kubernetes 控制器会创建并维护 Pod。因此 Helm 部署的不只是 Pod，实际创建哪些资源取决于 Chart 及其配置。

在本指南中，`helm install prometheus ...` 和 `helm install grafana ...` 会部署相应应用。后续的 `kubectl expose ...` 命令则使用 `kubectl` 单独创建 NodePort Service，这些 Service 不是由 Helm 创建的。

参考文章：[使用 Helm 在 Kubernetes 上部署 Prometheus 和 Grafana](https://medium.com/@gayatripawar401/deploy-prometheus-and-grafana-on-kubernetes-using-helm-5aa9d4fbae66)

## 准备工作

确保已配置 `kubectl` 并安装 Helm。确认集群中存在可用的 StorageClass：

```bash
kubectl get nodes
kubectl get storageclass
```

以下命令使用 `nfs-client`。如果你的 StorageClass 名称不同，请替换为实际名称。

## 安装 Prometheus

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/prometheus --namespace prometheus --create-namespace --set server.persistentVolume.storageClass=nfs-client --set alertmanager.persistentVolume.storageClass=nfs-client
```

## 安装 Grafana

Grafana 是独立的 Helm Release；安装 Prometheus Chart 不会自动安装 Grafana。先添加 Grafana Chart 仓库，再将 Grafana 安装到 `prometheus` namespace：

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install grafana grafana/grafana --namespace prometheus --set persistence.enabled=true --set persistence.storageClassName=nfs-client
```

安装成功时，Helm 输出中的状态应为 `STATUS: deployed`。如果没有指定 Chart 版本，Helm 会从仓库中选择一个可用版本。示例选择了 `10.5.15`，并提示该 Chart 已弃用。弃用警告不代表安装失败；但新部署或生产环境应先确认并选择仍受支持的 Grafana Chart。

## 检查安装状态

先确认两个 Helm Release 都已安装：

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

检查持久卷声明（PVC）：

```bash
kubectl get pvc -n prometheus
```

确认 `helm list` 中有 `prometheus` 和 `grafana` 两个 Release。对应 Pod 应为 `Running`，两个 Service 应存在，PVC 状态应为 `Bound`。

## 访问 Prometheus 和 Grafana

```bash
kubectl expose service prometheus-server --type=NodePort --target-port=9090 --name=prometheus-server-ext -n prometheus
kubectl expose service grafana --type=NodePort --target-port=3000 --name=grafana-ext -n prometheus
```

查询 Prometheus 的 NodePort：

```bash
kubectl get service prometheus-server-ext -n prometheus -o custom-columns=NAME:.metadata.name,NODEPORT:.spec.ports[0].nodePort
```

查询 Grafana 的 NodePort：

```bash
kubectl get service grafana-ext -n prometheus -o custom-columns=NAME:.metadata.name,NODEPORT:.spec.ports[0].nodePort
```

查询节点 IP：

```bash
kubectl get nodes -o wide
```

如果使用普通的 `kubectl get services` 输出，请查看 `PORT(S)` 列中冒号后的数字。例如 `80:31870/TCP` 中的 `31870` 就是 NodePort。选择 `INTERNAL-IP` 列中从你当前网络可以访问的节点 IP，然后替换以下 URL 中的占位符：

```text
Prometheus: http://<node-ip>:<prometheus-nodeport>/query
Grafana:    http://<node-ip>:<grafana-nodeport>
```

Grafana 在集群内的服务地址是 `grafana.prometheus.svc.cluster.local`。登录 Grafana 时，用户名使用 `admin`。在 Bash 终端运行以下命令获取初始密码：

```bash
kubectl get secret grafana -n prometheus -o jsonpath='{.data.admin-password}' | base64 --decode; echo
```

登录后添加 Prometheus 数据源，URL 填写 `http://prometheus-server.prometheus.svc.cluster.local`。如需添加 Kubernetes 监控面板，在 Grafana 中选择 **Dashboards > New > Import**，导入 Grafana.com 上的 Dashboard，并选择该 Prometheus 数据源。

## Spring Boot API 可观测性示例

Spring Boot API 的部署、Prometheus 抓取、SLO 告警规则及 Grafana 查询方法，请参阅 [Spring Boot 可观测性指南](spring-observability/README.md)。

## Kubernetes 集群可观测性

如何利用现有 Prometheus 和 Grafana 监控 Kubernetes 节点、工作负载、告警、日志及事件，请参阅 [Kubernetes 可观测性指南](kubernetes-observability/README.md)。

## 卸载

```bash
kubectl delete service prometheus-server-ext -n prometheus
kubectl delete service grafana-ext -n prometheus
helm uninstall grafana -n prometheus
helm uninstall prometheus -n prometheus
```

卸载 Helm Release 后，PVC 中的数据可能仍会保留。确认不再需要这些数据后，再单独删除 PVC。
