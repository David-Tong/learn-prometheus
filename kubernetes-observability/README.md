# 使用 Prometheus 和 Grafana 监控 Kubernetes 集群

本指南教你使用现有集群中的 Prometheus 和 Grafana 查看节点、Pod、Deployment 的健康状态和资源使用情况，并创建基础 Dashboard 与告警。Spring Boot 应用自身的指标配置见 [`spring-observability/README.md`](../spring-observability/README.md)。

## Prometheus 基本概念

Prometheus 会按配置定期访问一组 HTTP 地址抓取指标。每个要抓取的地址称为一个 **Target**；Target 表格中的字段可以这样理解：

| 字段 | 含义 | 示例 |
| --- | --- | --- |
| **Job** | 一组用途相同的 Targets 名称，表示 Prometheus 用哪项抓取配置发现它们。 | `kubernetes-service-endpoints` |
| **Endpoint** | Prometheus 实际访问的指标 URL，包含协议、地址、端口和路径。 | `http://<pod-ip>:<port>/metrics` |
| **Labels** | 附加在目标及其指标上的键值，用于识别来源和筛选数据。 | `namespace="prometheus"`、`service="prometheus-kube-state-metrics"`、`instance="10.0.25.22:8080"` |
| **State** | 最近一次抓取的结果。 | `UP` 或 `DOWN` |

例如，你的 kube-state-metrics Target 标签中，`namespace="prometheus"` 表示它位于 `prometheus` namespace，`service="prometheus-kube-state-metrics"` 表示对应 Service，`instance` 表示被抓取的地址。实际 Endpoint 和标签以集群中的 Targets 页面为准。

抓取成功后，Prometheus 保存的每条时间序列由**指标名和标签集合**共同确定。例如：

```promql
up{job="kubernetes-service-endpoints", namespace="prometheus", service="prometheus-kube-state-metrics"}
```

这里 `up` 是指标名，花括号内是标签筛选条件。此查询只选择指定 Job、namespace 和 Service 的抓取状态。标签也可以用来按来源分组，例如 `sum by (namespace) (...)`。不要把 request ID、用户 ID 等每次都变化的值作为 Prometheus 标签，否则会产生大量时间序列，增加存储和查询负担。

## 1. 确认监控组件已运行

当前 Prometheus 安装通常包含 Prometheus Server、Alertmanager、kube-state-metrics 和 node-exporter。先检查它们：

```bash
helm list -n prometheus
kubectl get pods,services -n prometheus
```

- **node-exporter** 提供节点操作系统指标，如 CPU、内存、磁盘和网络。
- **kube-state-metrics** 提供 Kubernetes 对象状态，如 Node Ready、Pod 阶段、容器重启次数和 Deployment 可用副本数。
- **Prometheus** 抓取并保存指标，可使用 PromQL 查询并评估告警规则。
- **Alertmanager** 对 Prometheus 告警进行分组、抑制和通知路由。未配置通知接收器时，告警仍可在 Prometheus 中查看，但不会发送到外部渠道。

## 2. 打开 Prometheus 和 Grafana

如果尚未创建 NodePort，先按仓库根目录 README 中的“访问 Prometheus 和 Grafana”步骤创建。查询端口和节点 IP：

```bash
kubectl get service prometheus-server-ext grafana-ext -n prometheus
kubectl get nodes -o wide
```

在 `PORT(S)` 列中读取冒号后的 NodePort，例如 `80:31870/TCP` 中的 `31870`。从 `INTERNAL-IP` 列选择当前网络可访问的节点 IP，然后打开：

```text
Prometheus: http://<node-ip>:<prometheus-nodeport>/query
Grafana:    http://<node-ip>:<grafana-nodeport>
```

### 配置 Grafana 的 Prometheus 数据源

在 Grafana 左侧菜单进入 **Connections > Data sources**，选择已有的 `prometheus` 数据源，或点击 **Add data source > Prometheus** 新建。在 **Connection > Prometheus server URL** 中输入：

```text
http://prometheus-server.prometheus.svc.cluster.local:80
```

这是 Prometheus Service 在 Kubernetes 集群内的 DNS 地址，`:80` 是该 Service 的端口。Grafana 运行在集群内时应使用这个地址。

不要使用 `http://localhost:9090`：Grafana 后端从自己的 Pod 发起连接，`localhost` 指的是 Grafana Pod 本身，不是 Prometheus。若输入框显示 `http://localhost:9090` 和 **Please enter a valid URL**，这可能只是空输入框的占位提示；点击输入框，填入上面的集群内地址。

填写后滚动到页面底部，点击 **Save & test**。成功时 Grafana 会提示数据源可用。若测试失败，先确认 Prometheus Service 和 Endpoint 存在：

```bash
kubectl get service prometheus-server -n prometheus
kubectl get endpoints prometheus-server -n prometheus
```

配置成功后，在 Grafana 的 **Explore** 中选择 `prometheus` 数据源，运行 `up` 测试是否能查询到 Prometheus 指标。

## 3. 确认 Prometheus 正在抓取数据

Prometheus 会定期从 Target（抓取目标）读取指标。先打开 **Status > Targets**，找到 `kubernetes-service-endpoints` 任务下的 `prometheus-kube-state-metrics` 目标。这个页面检查的是“Prometheus 能否读取指标”，不是直接检查 Kubernetes 节点是否健康。

按页面结果判断：

- **UP**：Prometheus 最近一次成功读取了该目标的指标。它不代表集群中的每个节点或应用都健康。
- **DOWN**：Prometheus 找到了目标，但读取失败。点击该目标查看 **Error**，再根据错误检查网络、端口、指标路径或访问权限。
- **没有这个目标**：Prometheus 没有发现该目标，或当前配置没有抓取它。检查 kube-state-metrics 的 Service、抓取注解及 Prometheus 的发现配置。

也可以在 **Graph** 页面直接检查 kube-state-metrics 的抓取状态。输入查询并点击 **Execute**：

```promql
up{namespace="prometheus", service="prometheus-kube-state-metrics"}
```

通常会看到一个或多个结果，每个结果对应一个抓取实例：值为 `1` 表示抓取成功，值为 `0` 表示抓取失败。没有结果表示这个查询没有匹配到目标；请先回到 **Status > Targets** 查看目标是否存在。

如果你要检查节点是否 Ready，应查询节点状态指标，而不是只看 `up`：`up=1` 仅说明 Prometheus 能抓取 kube-state-metrics。节点状态查询见下一节。

## 4. 在 Prometheus 中查询集群状态

先逐个运行查询确认指标存在，再将它们添加为 Grafana 面板。

### 节点 Ready 状态

查询不健康节点：

```promql
kube_node_status_condition{condition="Ready", status="false"} == 1
```

如果结果为空，通常表示目前没有 NotReady 节点，不代表查询失败。查询健康节点：

```promql
kube_node_status_condition{condition="Ready", status="true"} == 1
```

此指标来自 kube-state-metrics。如果健康和不健康查询都没有结果，检查 kube-state-metrics Pod、Service 和 Prometheus Target。

### 节点 CPU 使用率

```promql
100 * (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])))
```

`node_cpu_seconds_total` 是按 CPU 模式累计的时间计数器；`rate(...[5m])` 计算过去 5 分钟的增长速率。`mode="idle"` 代表空闲时间，`avg by (instance)` 对该节点各 CPU 核心的空闲比例取平均。`1 - 空闲比例` 得到忙碌比例，乘以 `100` 后单位为百分比。`instance` 通常显示为抓取地址，不一定是 Kubernetes Node 名称。

### 节点内存使用率

```promql
100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
```

结果单位是百分比。这里使用 Linux 的 `MemAvailable` 估算可供应用使用的内存，比单纯用 `MemFree` 更合适。

### 容器重启

`kube_pod_container_status_restarts_total` 由 **kube-state-metrics** 提供。它读取 Kubernetes API 中 Pod 的容器状态，把每个容器的累计重启次数暴露为指标；Prometheus 再从 `prometheus` namespace 中 kube-state-metrics 的 Service 定期抓取这些指标。因此这里有两个不同的检查对象：

- 检查 Prometheus 是否抓到了 kube-state-metrics：

	```promql
	up{namespace="prometheus", service="prometheus-kube-state-metrics"}
	```

	返回 `1` 表示 KSM 指标服务抓取成功。

- 查询 KSM 提供的容器重启指标：

	```promql
	kube_pod_container_status_restarts_total
	```

	结果按 `namespace`、`pod`、`container` 等标签区分容器。这个查询无结果时，先确认上面的 KSM 抓取状态，再检查目标是否发现该指标。

查询过去 15 分钟内发生过重启的容器：

```promql
sum by (namespace, pod, container) (increase(kube_pod_container_status_restarts_total[15m])) > 0
```

这里的 `increase(...[15m])` 计算每个容器的重启计数器在最近 15 分钟增加了多少；末尾的 `> 0` 只保留确实发生过重启的容器。因此显示 **Empty query result** 通常表示最近 15 分钟没有容器重启，是正常情况，不代表 Prometheus 抓取失败。更早发生的重启也不会被这个时间窗口显示。

如果原始指标有结果，但带 `> 0` 的查询为空，说明近期没有重启。也可以暂时移除过滤条件，查看每个容器最近 15 分钟的重启变化量：

```promql
sum by (namespace, pod, container) (increase(kube_pod_container_status_restarts_total[15m]))
```

如果原始指标也没有结果，到 Prometheus **Status > Targets** 检查 kube-state-metrics 是否为 `UP`，并确认 `kube_pod_container_status_restarts_total` 指标正在被采集。

### Deployment 副本不足

```promql
kube_deployment_spec_replicas - kube_deployment_status_replicas_available > 0
```

结果大于 `0` 表示可用副本少于期望副本。Kube-state-metrics 描述的是 Kubernetes 对象状态，不等同于应用请求是否成功；用户体验还需要 API 可用率、延迟和错误率等应用层 SLI。

## 5. 在 Grafana 创建集群 Dashboard

1. 登录 Grafana，确认 Prometheus 数据源可用。
2. 选择 **Dashboards > New > New dashboard > Add visualization**。
3. 如果出现数据源选择页面，选择名称为 `prometheus` 的数据源；如果直接进入面板编辑页，在查询区域顶部的 **Data source** 下拉框中选择 `prometheus`。
4. 在查询编辑器中切换到 **Code** 模式，粘贴上面的 PromQL。若列表中没有 `prometheus`，先按“打开 Prometheus 和 Grafana”一节配置数据源。
5. 点击 **Run queries**，确认图表有数据。
6. 在面板选项中设置标题、图例和单位。例如 CPU 和内存查询使用 **Percent (0-100)**；重启和副本差值可使用 **Short**。
7. 点击 **Apply**，再保存 Dashboard。

建议先创建这些面板：节点 CPU、节点内存、Ready 节点、Pod 重启、Deployment 可用副本，以及 Prometheus Targets 状态。若某面板没有数据，先把同一条 PromQL 放到 Prometheus Graph 中验证，再检查对应 exporter 和 Target。

### 实战：创建节点 CPU 使用率视图和告警

这个练习把 Prometheus 和 Grafana 串起来：先确认 Prometheus 收到 node-exporter 指标，再用同一条 PromQL 创建 Grafana 面板，最后设置一个持续高 CPU 的 warning 告警。

#### 第一步：确认 CPU 指标已采集

在 Prometheus **Graph** 页面运行：

```promql
count by (instance) (node_cpu_seconds_total{mode="idle"})
```

每个 `instance` 应有结果。如果没有结果，先到 **Status > Targets** 确认 node-exporter 对应的 Target 为 `UP`；Target 为 `DOWN` 时查看其错误信息。不要在指标还没有被抓取时先建 Grafana 面板。

#### 第二步：计算节点 CPU 使用率

在 Prometheus **Graph** 页面运行：

```promql
100 * (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])))
```

表达式先计算每个 CPU 核心过去 5 分钟的空闲比例，再按节点取平均，用 `1 - 空闲比例` 得到忙碌比例，最后乘以 `100` 转成百分比。结果按 `instance` 分组；`instance` 通常是抓取地址，不一定是 Kubernetes 节点名称。此查询用 5 分钟窗口平滑短暂波动。

#### 第三步：在 Grafana 创建面板

1. 登录 Grafana，确认 Prometheus 数据源的 URL 为 `http://prometheus-server.prometheus.svc.cluster.local:80`。
2. 选择 **Dashboards > New > New dashboard > Add visualization**，然后选择名为 `prometheus` 的数据源。
3. 在底部查询编辑区选择 **Code**，粘贴上面的 CPU 查询，点击 **Run queries**，确认图表显示每个节点的曲线。
4. 右侧 **Visualization** 选择 **Time series**。要自定义曲线图例，在查询编辑区展开 **Options**，找到当前显示 `Auto` 的 **Legend** 下拉框，选择 **Custom**，再输入 `{{instance}}`。如果保留 `Auto`，Grafana 会自动使用默认图例。
5. 在右侧 **Standard options > Unit** 选择 **Percent (0-100)**，并把 **Min** 设为 `0`、**Max** 设为 `100`。可在 **Thresholds** 中添加 `70` 和 `85` 作为参考阈值。
6. 面板修改完成后，点击右上角 **Save dashboard** 保存 Dashboard。首次保存时填写名称，例如 `Kubernetes - Node health`。

要重命名已有 Dashboard，回到 Dashboard 页面，打开 **Dashboard settings > General**，修改 **Title**，再点击右上角 **Save dashboard**。面板标题和 Dashboard 名称是两个不同的设置。

#### 第四步：添加持续高 CPU 告警

在 Grafana 左侧进入 **Alerting > Alert rules > New alert rule**。按页面从上到下填写：

1. **Enter alert rule name**：填写 `KubernetesNodeHighCPU`。
2. **Define query and alert condition**：数据源选择 `prometheus`。截图中的 PromQL 输入框为空，所以图表显示 **No data**；先在查询框粘贴以下表达式，再点 **Run queries**：

	```promql
	100 * (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])))
	```

	查询成功后应看到按节点区分的结果。若仍是 **No data**，先在 Prometheus Graph 单独运行同一条查询，并确认 node-exporter Target 为 `UP`；此时先不要设置告警阈值。

	在 **Alert condition** 中将查询选为 `A`（如果页面显示查询选择框），比较条件设为 **IS ABOVE**，把默认阈值 `0` 改为 `85`。表达式乘以了 `100`，因此 `85` 表示 CPU 使用率超过 85%。不要额外添加空的 B 表达式。

3. **Add folder and labels**：选择保存规则的 Folder。点击 **Add labels**，设置 `severity` = `warning`。
4. **Set evaluation behavior**：选择或创建 Evaluation group，将评估间隔设为 `1m`；将 **Pending period** 设为 `10m`。意思是每分钟检查一次，同一节点连续 10 分钟超过 85% 才触发。将 **Keep firing for** 设为 `5m`：当 CPU 已回落到阈值以下时，告警再保持 firing 5 分钟后才恢复，避免短暂回落导致告警反复切换。若希望条件一解除就立即恢复，选择 `None`。
5. **Configure notifications**：如果此处有 Contact point 选项，选择已配置并测试的接收渠道。若没有，在 **Alerting > Contact points** 配置接收渠道，再到 **Alerting > Notification policies** 建立匹配 `severity=warning` 的路由。
6. 点击 **Preview** 检查规则结果和节点标签，填写摘要与排查说明，例如摘要 `节点 {{ $labels.instance }} CPU 使用率持续偏高`，说明可提示检查节点负载、Pod 分布和 CPU throttling，然后保存规则。

保存后在 **Alerting > Alert rules** 查看规则状态。未配置 Contact point 或通知策略时，规则仍可能触发并显示在 Grafana 中，但不会发送外部通知。CPU 高属于容量/饱和度信号，不一定代表用户已受影响；应结合 API 错误率、延迟和 SLO burn-rate 决定是否升级为 paging 告警。无 CPU 数据时不能当作 CPU 为零，应另行监控 node-exporter 抓取状态。

## 6. 日志、Kubernetes 事件和链路追踪

Prometheus 主要存储指标，不是日志或 Kubernetes 事件数据库。更完整的可观测性通常还包括：

- **日志**：用 Grafana Alloy 等节点采集代理收集容器日志，发送到 Loki。避免把 request ID、用户 ID 等高基数字段作为标签。
- **Kubernetes 事件**：事件可能过期，可通过 event exporter 或 Alloy 转发到集中存储，作为故障上下文使用。
- **链路追踪**：应用通过 OpenTelemetry 生成 trace，经 Collector 发送到 Tempo 等后端，再用 trace ID 关联日志和指标。

这些后端不会由现有 Prometheus/Grafana Helm Release 自动安装。部署前先规划保留期限、存储容量、访问控制和敏感信息脱敏。

## 排查命令

```bash
kubectl get nodes
kubectl get pods -A
kubectl get pods,services -n prometheus
kubectl get events -A --sort-by=.lastTimestamp
```

查看异常 Pod 的详细状态和事件：

```bash
kubectl describe pod <pod-name> -n <namespace>
```

建议按这个顺序排查：先在 Prometheus Targets 确认数据是否进来，再在 Prometheus Graph 验证查询，接着在 Grafana 看趋势，最后结合 Pod 日志和 Kubernetes 事件定位原因。