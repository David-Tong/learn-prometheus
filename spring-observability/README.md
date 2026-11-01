# Spring Boot 可观测性示例

本目录用于将 Spring Boot 示例 API 接入已安装在 `prometheus` namespace 中的 Prometheus 和 Grafana。

## 文件说明

### `spring-boot-observability.yaml`

这是 Kubernetes 清单，使用 `kubectl apply` 部署 Spring Boot 示例 API。它会创建：

- `observability-demo` namespace，用来隔离示例资源。
- `spring-boot-api` Deployment，运行示例镜像，并配置启动、就绪和存活探针及资源请求/限制。
- `spring-boot-api` ClusterIP Service，为 Pod 提供集群内稳定地址。Service 上的 `prometheus.io/*` 注解告诉当前 Prometheus Chart 抓取 `8080` 端口的 `/actuator/prometheus`。

这个文件不会修改 Prometheus 或 Grafana 的配置。删除 API 示例时，可用 `kubectl delete -f spring-observability/spring-boot-observability.yaml` 删除清单中的资源。

### `prometheus-slo-rules-values.yaml`

这是传给 Helm 的 **values 覆盖文件**，不是可直接用 `kubectl apply` 部署的 Kubernetes 清单。它为现有 Prometheus Helm Release 增加两条告警规则：目标连续不可抓取告警，以及基于 99.9% 可用性目标的多窗口错误预算消耗速率告警。

使用文档中的 `helm upgrade ... --reuse-values -f ...` 命令，将这些规则合并到现有 Prometheus 配置中；该操作会更新 Prometheus Helm Release。应用规则后，去 Prometheus 的 **Alerts** 页面查看状态。若要移除规则，需要从 values 中删去这些规则并再次升级 Helm Release；仅删除 Spring Boot API 不会自动删除告警规则。

该示例镜像仅用于学习，最后一次发布是在 2020 年，不要用于生产环境。在实验环境之外使用前，请替换为你自己维护的应用镜像。

## 1. 部署 API

在仓库根目录执行以下命令应用配置：

```bash
kubectl apply -f spring-observability/spring-boot-observability.yaml
```

等待 Pod 就绪，并确认 Service 已有 Endpoint：

```bash
kubectl get pods,services -n observability-demo
kubectl get endpoints spring-boot-api -n observability-demo
```

Pod 应处于 `Running` 且就绪状态。Prometheus 通过 `prometheus.io/*` 注解发现该 Service；当前使用的 Prometheus Helm Chart 不需要 `ServiceMonitor`。

## 2. 验证 Prometheus 抓取

打开现有 Prometheus 页面，进入 **Status > Targets（状态 > 目标）**。找到 `kubernetes-service-endpoints` 任务，并确认 `spring-boot-api` 对应的 target 状态为 `UP`。

也可以在 Prometheus 中运行以下查询：

```promql
up{namespace="observability-demo", service="spring-boot-api"}
```

```promql
http_server_requests_seconds_count{namespace="observability-demo", service="spring-boot-api"}
```

抓取成功时，第一个查询应返回 `1`。有请求到达 API 后才会出现请求指标。健康检查和 Actuator 请求已从 SLO 计算中排除，因此错误预算告警需要有业务 API 流量后才能计算。

## 3. 安装 SLO 告警规则

此 values 文件用于现有的 `prometheus-community/prometheus` Helm Release。在仓库根目录运行：

```bash
helm upgrade prometheus prometheus-community/prometheus --namespace prometheus --reuse-values --version 27.45.0 -f spring-observability/prometheus-slo-rules-values.yaml
```

在 Prometheus 的 **Alerts（告警）** 页面查看规则和告警状态。规则如下：

- `SpringBootApiTargetDown`：Prometheus 连续 2 分钟无法抓取 API 时发出警告。
- `SpringBootApiSLOBurnRateHigh`：5 分钟和 1 小时两个窗口内的 5xx 比例均超过 1.44%，并持续 2 分钟时触发。对于 99.9% 可用性 SLO，这相当于 14.4 倍的错误预算消耗速率。

在 API 收到业务请求之前，消耗速率告警不会触发。规则排除了 `/health` 和 `/actuator` 路径；如果你的应用使用不同的健康检查路径或业务路由，请修改 values 文件中的路径过滤条件。

此 Helm 规则配置会让告警显示在 Prometheus 中。如需发送通知，还需单独为 Alertmanager 配置通知接收器。

## 4. 在 Grafana 中查看指标

使用 Grafana 中现有的 Prometheus 数据源。可尝试以下查询：

```promql
rate(http_server_requests_seconds_count{namespace="observability-demo", service="spring-boot-api"}[5m])
```

```promql
jvm_memory_used_bytes{namespace="observability-demo", service="spring-boot-api"}
```

可以使用这些查询创建监控仪表板面板，也可以导入 Spring Boot/Micrometer 仪表板，并选择现有的 Prometheus 数据源。

## 范围与限制

此示例只采集指标。Prometheus 不存储应用日志或 Kubernetes 事件。要构建更完整的可观测性方案，还需要日志采集（例如 Grafana Alloy 和 Loki）、Kubernetes 事件采集，以及分布式链路追踪（OpenTelemetry 和 Tempo）。这些组件不会由此处的配置安装。

此 SLO 规则仅用于学习，不是生产环境策略。生产告警应根据面向用户的服务可用性目标、请求量和约定的错误预算窗口制定。

## 删除示例

```bash
kubectl delete -f spring-observability/spring-boot-observability.yaml
```

告警规则会继续保留在 Prometheus Helm Release 中。要删除规则，需从 Helm values 中移除后再升级该 Release。