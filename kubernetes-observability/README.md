# Kubernetes 集群可观测性

本指南介绍如何用现有 Kubernetes 集群和 `prometheus` namespace 中的 Prometheus、Grafana 监控集群本身。重点是节点、控制面、Pod、Deployment、告警、日志和 Kubernetes 事件，不重复 Spring Boot API 的应用指标配置。

## 1. 盘点现有监控组件

先确认 Prometheus 已部署哪些组件：

```bash
helm list -n prometheus
kubectl get pods,services -n prometheus
```

当前使用的 `prometheus-community/prometheus` Chart 通常会部署 Prometheus Server、Alertmanager、kube-state-metrics 和 node-exporter。它们提供不同类型的数据：

- **node-exporter**：节点操作系统指标，例如 CPU、内存、磁盘和网络。
- **kube-state-metrics**：Kubernetes 对象状态，例如节点 Ready 状态、Pod 阶段、容器重启次数和 Deployment 可用副本数。
- **Prometheus**：定期抓取并保存指标，执行 PromQL 查询和告警规则。
- **Alertmanager**：接收 Prometheus 告警并负责分组、抑制及发送通知。若未配置接收器，告警可以在 Prometheus 中看到，但不会发到外部渠道。

不要只看 Pod 是否 `Running`；还要确认 Prometheus 确实抓取到了目标。

## 2. 验证 Prometheus Targets

在 Prometheus 中打开 **Status > Targets**，检查相关任务及其状态。常见任务包括：

- `kubernetes-service-endpoints`：通过 Service 注解发现的服务指标，包括 kube-state-metrics。
- `kubernetes-nodes`：节点 kubelet 指标。
- `kubernetes-nodes-cadvisor`：容器资源使用情况。
- `kubernetes-apiservers`：Kubernetes API Server 指标。

任务名称和可抓取目标取决于 Chart 配置及集群权限。`DOWN`、`Missing` 或没有目标时，检查 Prometheus 配置、ServiceAccount/RBAC、网络可达性，以及托管 Kubernetes 是否允许访问控制面指标。不要为了让目标变绿而关闭 TLS 校验或扩大权限；先确认集群安全边界和必要的只读权限。

也可以查询目标状态：

```promql
up
```

`up` 值为 `1` 表示最近一次抓取成功，`0` 表示抓取失败。没有对应时间序列通常表示目标没有被发现或没有被配置抓取。

## 3. 建立集群健康视图

先在 Prometheus 或 Grafana Explore 中运行这些查询，确认指标存在，再用它们创建 Dashboard 面板。

### 节点

节点是否处于 Ready 状态：

```promql
kube_node_status_condition{condition="Ready", status="false"} == 1
```

节点 CPU 使用率（按 Prometheus target 的 `instance` 分组）：

```promql
1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))
```

节点内存使用率：

```promql
1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes
```

### 工作负载

最近 15 分钟发生过重启的容器：

```promql
sum by (namespace, pod, container) (increase(kube_pod_container_status_restarts_total[15m])) > 0
```

期望副本数大于可用副本数的 Deployment：

```promql
kube_deployment_spec_replicas - kube_deployment_status_replicas_available > 0
```

Pod 当前处于 Pending 的数量：

```promql
sum by (namespace) (kube_pod_status_phase{phase="Pending"})
```

Kube-state-metrics 提供的是 Kubernetes 对象状态，不等同于应用请求是否成功。判断用户体验还需要应用层 SLI，例如 API 可用率、延迟和错误率。

## 4. 规划告警和 SLO

优先添加可采取行动的告警，而不是对每个指标设置固定阈值。可从这些情况开始：

- 节点 NotReady 持续数分钟。
- Deployment 可用副本低于期望值并持续一段时间。
- 容器短时间内反复重启。
- 节点磁盘可用空间过低或预计很快耗尽。
- Prometheus 抓取失败，或 Prometheus 自身存储/容量接近限制。
- 面向用户的 API 错误率或延迟持续超出服务 SLO。

集群健康告警适合帮助定位故障；面向用户的可用性 SLO 应以真实业务请求为准。为每项告警写清楚影响、排查入口和操作建议，并设置 `for` 时长来减少瞬时抖动。使用 Alertmanager 配置通知接收器、分组和抑制规则。先在 Prometheus **Alerts** 中确认规则触发，再验证通知链路。

## 5. 补齐日志、事件和链路追踪

Prometheus 主要存储指标，不是日志或 Kubernetes 事件数据库。更完整的可观测性通常包含：

- **Metrics**：Prometheus 抓取节点、Kubernetes 对象及应用指标；Grafana 展示趋势和告警。
- **Logs**：通过 Grafana Alloy 等节点采集代理收集容器日志，再写入 Loki。使用 namespace、pod、container 等低基数标签；不要把 request ID、用户 ID 等高基数字段设为标签。
- **Kubernetes events**：事件可能过期，适合通过 event exporter 或 Alloy 等组件转发并集中保存。将事件作为故障上下文，不要把它当作长期指标存储。
- **Traces**：应用使用 OpenTelemetry SDK/Agent 生成 trace，经 OpenTelemetry Collector 发送到 Tempo 等后端。通过 trace ID 在日志、指标和链路之间关联问题。

这些日志、事件和追踪后端不会由当前 Prometheus/Grafana Helm releases 自动安装。部署前先确定数据保留时间、存储容量、访问控制和敏感信息脱敏策略。

## 6. 建议实施顺序

1. 检查 Prometheus、node-exporter、kube-state-metrics 和 Alertmanager 是否已运行。
2. 在 **Status > Targets** 确认节点、Kubernetes 对象和 API Server 目标的抓取状态；记录因集群类型或权限限制而不可用的目标。
3. 在 Grafana 建立节点、Pod、Deployment 和 Prometheus 自身的基础 Dashboard。
4. 添加少量高价值告警，在 Prometheus 中验证触发条件，并测试 Alertmanager 通知。
5. 再按需要部署日志、事件和追踪采集组件，并定义数据保留及访问策略。

## 排查入口

```bash
kubectl get nodes
kubectl get pods -A
kubectl get pods,services -n prometheus
kubectl get events -A --sort-by=.lastTimestamp
```

针对某个异常 Pod 查看详细状态和事件：

```bash
kubectl describe pod <pod-name> -n <namespace>
```

先从 Prometheus Targets 确认数据是否进来，再从 Grafana Dashboard 观察趋势，最后结合 Pod 日志和 Kubernetes events 定位原因。