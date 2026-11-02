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

## 1. 在 Spring Boot 中生成指标

Prometheus 采用拉取（pull）模式：它会按照抓取周期请求应用的指标端点，并保存返回的指标样本。首先要让 Spring Boot 提供可抓取的指标端点。

添加 Spring Boot Actuator 和 Micrometer Prometheus registry 依赖。以下 Maven 示例的版本由 Spring Boot dependency management 管理：

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

在 `application.yml` 中将 `health` 和 `prometheus` 加入 HTTP 暴露列表：

```yaml
server:
  port: 8080

management:
  endpoints:
    web:
      exposure:
        include:
          - health
          - prometheus
```

应用包含这些依赖和配置并启动后，Actuator 会提供 `/actuator/prometheus` 端点，Micrometer Prometheus registry 以 Prometheus 格式输出指标。Spring Boot/Micrometer 会自动记录标准 HTTP 请求的次数、状态码和耗时，通常不需要为这些指标另写 Controller。

若要衡量业务操作（例如成功完成了多少个订单），可以在业务代码中使用 Micrometer `Counter` 计数。下面的例子在订单数据库事务成功提交后递增计数器，因此事务回滚的订单不会被统计为已完成。将 `OrderRepository` 和 `Order` 换成应用自己的类型：

```java
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;

    public OrderService(
            OrderRepository orderRepository,
            ApplicationEventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.eventPublisher = eventPublisher;
    }

    @Transactional
    public Order completeOrder(long orderId) {
        Order order = orderRepository.markCompleted(orderId);
        eventPublisher.publishEvent(new OrderCompletedEvent());
        return order;
    }
}
```

将下面的事件和监听器分别放在各自的 Java 文件中。监听器只在事务提交后递增 Counter：

```java
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import org.springframework.stereotype.Component;
import org.springframework.transaction.event.TransactionPhase;
import org.springframework.transaction.event.TransactionalEventListener;

public class OrderCompletedEvent {
}

@Component
public class OrderMetrics {
    private final Counter completedOrders;

    public OrderMetrics(MeterRegistry meterRegistry) {
        this.completedOrders = Counter.builder("business.orders.completed")
                .description("Number of successfully completed orders")
                .register(meterRegistry);
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onOrderCompleted(OrderCompletedEvent event) {
        completedOrders.increment();
    }
}
```

这三部分通过 Spring 应用事件连接起来，而不是由 `OrderService` 直接调用 `OrderMetrics`：

1. `completeOrder()` 更新订单后，`eventPublisher.publishEvent(new OrderCompletedEvent())` 将一个 `OrderCompletedEvent` 发布到 Spring 的应用事件机制中。
2. `OrderCompletedEvent` 是事件类型，表示订单完成这一事实。Spring 根据事件类型找到 `OrderMetrics` 中参数类型匹配的监听方法 `onOrderCompleted(OrderCompletedEvent event)`。
3. 因为监听方法使用 `@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)`，Spring 会等当前事务成功提交后才调用它。监听方法随后执行 `completedOrders.increment()`，将已完成订单数加一；如果事务回滚，监听器不会执行。

因此，`publishEvent(...)` 的作用是发布事件，Spring 负责把事件交给监听器，并遵守事务阶段。当前事件不带订单 ID，因为 Counter 只需记录发生次数；不要把订单 ID 加成指标标签。

应用加入 `micrometer-registry-prometheus` 后，Spring Boot 会自动配置 Prometheus registry。`Counter.builder("business.orders.completed")` 注册 Counter，Actuator 的 `/actuator/prometheus` 端点会调用 registry，将当前指标序列化为 Prometheus 文本格式。Micrometer 会将指标名中的点号转换为下划线，并为 Counter 加上 `_total` 后缀。假设进程启动后成功完成了 3 个订单，端点中会看到类似内容：

```text
# HELP business_orders_completed_total Number of successfully completed orders
# TYPE business_orders_completed_total counter
business_orders_completed_total 3.0
```

每次 `completedOrders.increment()` 后，这个值增加 1。Prometheus 按抓取周期请求该端点，并把每次读到的值保存为带时间戳的样本；在 Prometheus 中可直接查询 `business_orders_completed_total`，查看每个实例上报的累计值。目标的 `namespace`、`service`、`job`、`instance` 等标签会用于区分时间序列，具体标签取决于抓取配置。例如，可以用 `sum(business_orders_completed_total)` 查看各实例当前累计值之和，或用 `sum(increase(business_orders_completed_total[1h]))` 估算所有实例最近一小时完成的订单数。

Counter 是每个应用进程启动以来单调增加的累计数；应用重启后会从零开始。Prometheus 的 `increase` 会处理抓取到的计数器重置，因此统计时间窗口内的增量时应使用 `increase`，而不是把当前累计值当作某个时间段内的完成数。

Prometheus 不会自动获知未被应用记录的业务结果。为指标添加标签时，应使用有限且稳定的取值（例如业务类型），避免用户 ID、订单 ID 等高基数值，否则会产生大量时间序列并增加 Prometheus 的内存和存储开销。

## 2. 在 Kubernetes 中部署并暴露指标

本示例的 [spring-boot-observability.yaml](./spring-boot-observability.yaml) 创建 Deployment 和 Service。以下分别摘录其中的容器端口配置和完整 Service 配置：

```yaml
# Container configuration excerpt from the Deployment
containers:
  - name: api
    image: pierreroudier/micrometer-prometheus@sha256:76327fa67839d67dd20b8b8480524aba2f6e3a7b7939701a2f56ea998483c488
    ports:
      - name: http
        containerPort: 8080
    readinessProbe:
      httpGet:
        path: /actuator/health
        port: http
```

```yaml
# spring-boot-api Service
apiVersion: v1
kind: Service
metadata:
  name: spring-boot-api
  namespace: observability-demo
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/path: /actuator/prometheus
    prometheus.io/port: "8080"
spec:
  selector:
    app.kubernetes.io/name: spring-boot-api
  ports:
    - name: http
      port: 8080
      targetPort: http
```

**本示例使用已构建好的镜像**，不是在此仓库中编译 Spring Boot 源码。该镜像必须已经包含前面所述的依赖和 Actuator 配置；将自己的应用接入时，应将 `image` 替换为包含这些配置的应用镜像，并确认应用实际监听 `8080`、指标路径为 `/actuator/prometheus`。`containerPort` 只是声明端口，不会让应用开始监听。

各项 Kubernetes 配置的作用如下：

- `containerPort: 8080` 声明名为 `http` 的容器端口；应用进程本身也必须监听该端口。`readinessProbe` 用 `/actuator/health` 判断 Pod 是否就绪，未就绪的 Pod 不会作为可用 Service Endpoint。
- Service 的 `selector` 选择 API Pod；`port: 8080` 是集群内 Service 端口，`targetPort: http` 将请求转发到 Pod 中名为 `http` 的端口，也就是应用的 `8080` 端口。
- **注解本身不会启动抓取。** Prometheus 必须先配置一个 Kubernetes service-discovery 抓取任务，并拥有查询 Kubernetes API 所需的权限。当前使用的 Prometheus Helm Chart 配置了 `kubernetes-service-endpoints` 任务：Prometheus 通过 Kubernetes API 发现 Service/Endpoint，再用该任务的 relabel 规则筛选出 `prometheus.io/scrape: "true"` 的目标。因此它可以发现其权限及任务配置范围内的 Service/Endpoint，但不会无差别地抓取每一个 Service；未带该注解（或未通过筛选）的目标不会被此任务抓取。
- 对通过筛选的目标，`prometheus.io/path` 用来设置抓取 URL 路径，`prometheus.io/port` 用来设置抓取端口。Prometheus 随后从对应 Endpoint（Pod IP 和端口）请求指标，例如 `http://<pod-ip>:8080/actuator/prometheus`。只有目标可达且端点返回有效指标时，抓取才成功。其他 Prometheus 安装方式不一定配置了相同的发现任务和 relabel 规则；它们可能需要配置自己的 scrape job，或使用由 Prometheus Operator 管理的 `ServiceMonitor`。

完整链路是：Spring Boot/Micrometer 记录指标 → `/actuator/prometheus` 导出指标 → Service selector 产生指向就绪 Pod 的 Endpoint → Prometheus 的 Kubernetes 发现任务从 API 获取候选目标，并通过 relabel 规则保留带抓取注解的目标 → Prometheus 请求 `http://<pod-ip>:8080/actuator/prometheus` 并保存抓取到的指标。应用监听端口、`containerPort`、Service `targetPort`、注解中的端口和指标路径必须相互匹配。

在仓库根目录应用 Kubernetes 清单：

```bash
kubectl apply -f spring-observability/spring-boot-observability.yaml
```

等待 Pod 就绪，并确认 Service 已有 Endpoint：

```bash
kubectl get pods,services -n observability-demo
kubectl get endpoints spring-boot-api -n observability-demo
```

Pod 应处于 `Running` 且就绪状态。当前 Prometheus Helm Chart 通过 Service 注解发现目标，不需要 `ServiceMonitor`。

## 3. 验证 Prometheus 抓取

确认 Pod 已就绪，且 Service Endpoint 指向该 Pod。然后在 Prometheus 的 **Targets** 页面找到 `kubernetes-service-endpoints` 任务，展开目标列表并查找 namespace 为 `observability-demo`、Service 为 `spring-boot-api` 的目标：

- `UP`：最近一次抓取成功。点击目标可查看抓取地址、最近抓取时间和耗时。
- `DOWN`：Prometheus 找到了目标，但抓取失败。点击目标查看 **Error** 信息，通常可据此判断是连接、端口、路径或 HTTP 错误。
- 找不到目标：Prometheus 尚未发现这个 Service，或抓取配置未选中它。检查 Service 注解和 Endpoint。

`DOWN` 或找不到目标时，检查 Service 注解、端口映射、Pod 就绪状态及 Endpoint：

```bash
kubectl get service spring-boot-api -n observability-demo -o yaml
kubectl get endpoints spring-boot-api -n observability-demo
kubectl get pods -n observability-demo
```

Service 应带有 `prometheus.io/scrape: "true"`、`prometheus.io/path: /actuator/prometheus` 和 `prometheus.io/port: "8080"` 注解，且 Endpoint 应指向就绪的 Pod。

在 Prometheus 的 **Graph** 页面运行以下 PromQL，确认抓取状态和应用请求指标：

```promql
up{namespace="observability-demo", service="spring-boot-api"}
```

```promql
http_server_requests_seconds_count{namespace="observability-demo", service="spring-boot-api"}
```

第二个查询展示的是 Micrometer HTTP 请求计时器的 `_count`：它是**应用启动以来累计处理的请求数**，不是当前在线人数，也不是 Prometheus 发现的用户数。应用重启后计数器可能归零；结果还可能按 `uri`、`method`、`status` 等标签拆成多条时间序列。这里的 `namespace` 和 `service` 是 Prometheus 给被抓取目标附加的标签，用来标识指标来自哪个 Kubernetes Service，**不表示这些请求是由该 Service 或真实用户发起的**。

即使没有人手动访问 API，Kubernetes 也会定期调用健康检查。在本示例中，`readinessProbe` 每 10 秒请求一次 `/actuator/health`，`livenessProbe` 每 20 秒请求一次；应用启动期间，`startupProbe` 还会每 5 秒检查一次，直到启动成功。因此，仅就这两种持续运行的探针而言，单个 Pod 每分钟大约会收到 9 次健康检查请求，启动探针会额外产生短暂请求。它们由 Kubernetes 节点上的 kubelet 发起，不是业务用户请求。Prometheus 也会定期请求 `/actuator/prometheus` 来抓取指标；该管理端点请求是否出现在 HTTP 请求计数中，取决于应用的 HTTP 指标埋点行为。

可以按路径和状态码查看最近 15 分钟各类请求的新增数量，区分健康检查、Actuator 和业务 API 流量：

```promql
sum by (uri, status) (
  increase(http_server_requests_seconds_count{namespace="observability-demo", service="spring-boot-api"}[15m])
)
```

查看结果中的 `uri` 标签即可识别 `/actuator/health` 等探针流量。示例 SLO 规则通过 `uri!~"/health|/actuator.*"` 排除健康检查和 Actuator 路径，只用业务 HTTP 请求计算错误率。第一个 `up` 查询返回 `1` 表示 Prometheus 成功抓取指标端点；它不代表业务请求成功。后续 SLO 规则根据业务请求指标计算错误率，因此没有业务流量时不会有可计算的业务 SLI。生产环境应限制 Actuator 管理端点的网络访问，只向 Prometheus 和必要的健康检查调用方开放。

## 4. 本示例中的 SLI、SLO 与 SLA

### SLI：Service Level Indicator（服务等级指标）

SLI 是衡量服务表现的指标。本示例用业务 HTTP 请求的成功比例作为一个简化的可用性 SLI：从请求计数中排除 `/health` 和 `/actuator`，再计算非 `5xx` 请求数占业务请求总数的比例。

这里把 `5xx` 视为失败；`4xx` 是否算成功要根据 API 语义决定，不能不加判断地套用到所有服务。`up` 是抓取目标是否可达的指标，不是这个业务 SLI。

### SLO：Service Level Objective（服务等级目标）

SLO 是 SLI 要达到的目标及其统计周期。例如，“滚动 30 天内，至少 99.9% 的业务请求成功”是一个 SLO。按此目标计算，允许失败的错误预算是 0.1%。

这只是说明目标的示例；当前示例规则没有实现滚动 30 天达标统计。

### SLA：Service Level Agreement（服务等级协议）

SLA 是服务提供方与用户之间的正式服务承诺，通常会约定服务水平、统计口径，以及未达标时的补救措施（例如服务抵扣）。它可能以 SLO 为依据，但 SLO 是内部服务目标，SLA 则是对用户的约定。本示例只是演示如何采集指标和配置告警，不定义服务承诺、赔偿条款或 SLA。

`SpringBootApiSLOBurnRateHigh` 是一个**风险告警**，不是 SLO 或 SLA 判定：当业务请求的 `5xx` 比例在最近 5 分钟和 1 小时窗口都超过 1.44%，并持续 2 分钟时，Prometheus 会触发告警。这里的 1.44% 是基于示例中的 99.9% 目标（即 0.1% 错误预算）设置的高错误预算消耗速率阈值，用来提示可能需要处理的问题；它不是服务目标，也不能单独说明某个统计周期最终是否达标。

实际使用时，应先按业务定义 SLI（哪些请求算成功）、SLO（目标和统计周期），再根据错误预算设计告警。若需要对外承诺服务水平，还需另行制定 SLA，并明确其统计口径、适用范围和未达标处理方式。

## 5. 可选：将 SLO 告警规则应用到 Prometheus

这一步**不是 Prometheus 抓取指标或在 Grafana 查询指标的前提**。它的作用是让 Prometheus 定期评估预先写好的 PromQL 规则，并在条件持续满足时生成告警：

- `SpringBootApiTargetDown`：连续 2 分钟抓取不到 API 指标端点时，提示抓取目标不可用。
- `SpringBootApiSLOBurnRateHigh`：业务请求的 5xx 比例在 5 分钟和 1 小时窗口内都超过阈值，并持续 2 分钟时，提示错误预算消耗过快。

因此，只有希望 Prometheus 自动判断这些状况并显示告警时，才需要应用这组示例规则；若只想抓取和查看指标，可以跳过本节。规则仅对本示例的 namespace、Service 标签以及请求指标名称生效，实际使用时应按应用的 SLI/SLO 和流量特征调整。没有业务请求时，错误率规则不会触发。

### 为什么在 Prometheus 中配置，而不是 Grafana？

Grafana 同样可以基于 Prometheus 指标创建告警。两种方式都能执行 PromQL 并判断条件是否满足，选择哪一种取决于团队希望由哪个组件负责告警规则和通知流程：

- **Prometheus 告警规则**由 Prometheus Server 定期评估。本示例把规则放在 Prometheus 的 Helm values 中，是为了和指标采集及 PromQL 规则一起配置、版本管理；规则评估不依赖 Grafana 是否运行。告警通知通常由 Alertmanager 负责分组、路由和发送。
- **Grafana 告警规则**由 Grafana 的告警功能评估和管理，适合希望在 Grafana 中集中管理仪表板与告警，或统一查看多个数据源告警的场景。Grafana 需要运行并配置相应的数据源和通知渠道。

本示例选择 Prometheus 规则只是配置方式的演示，并不表示 Grafana 告警不合适，也不是 Prometheus 抓取指标所必需的。若改在 Grafana 中创建相同条件的告警，就不必再应用本节的 Prometheus SLO 规则；避免两边同时启用相同规则，否则可能收到重复告警。

### Helm 命令和 values 文件分别做什么？

`prometheus-slo-rules-values.yaml` 不是可单独安装的程序或 Kubernetes 清单，而是传给 Helm Chart 的配置文件；它在 `serverFiles.alerting_rules.yml` 中定义 Prometheus 要加载的告警规则。下面的命令不会创建第二个 Release，但会对现有 Release 执行 Helm upgrade，因此不应把它理解成“只上传一个规则文件、其他内容绝不会变”。

在仓库根目录执行以下命令，将规则配置应用到现有的 Prometheus Helm Release：

```bash
helm upgrade prometheus prometheus-community/prometheus --namespace prometheus --reuse-values --version 27.45.0 -f spring-observability/prometheus-slo-rules-values.yaml
```

各部分含义如下：

- `helm upgrade`：升级/更新一个已存在的 Helm Release；这里不是创建另一套 Prometheus。
- `prometheus`：Release 名称，必须与现有安装的 Release 名相符。
- `prometheus-community/prometheus`：使用的 Chart 名称。
- `--namespace prometheus`：Release 所在的 Kubernetes namespace。
- `--reuse-values`：保留该 Release 上次使用的 values，避免本次只提供规则文件时覆盖原有配置。
- `--version 27.45.0`：指定这次使用的 Chart 版本。若现有 Release 使用的 Chart 版本不同，命令会将它切换到 `27.45.0`；若当前已经是该版本，则不会发生 Chart 版本切换。
- `-f spring-observability/prometheus-slo-rules-values.yaml`：把本地 values 文件传给 Chart。Chart 根据其中 `serverFiles.alerting_rules.yml` 的内容更新 Prometheus 规则配置。

因此，命令一定会尝试更新已有 Helm Release；是否升级了 **Chart 版本**取决于当前版本与 `27.45.0` 是否相同。Chart 版本变化时，Chart 模板或默认值也可能改变，所以 Helm 可能更新 Prometheus 相关 Kubernetes 资源；Prometheus Server 的镜像版本是否改变，则取决于新 Chart 的默认值、保留的旧 values 以及本次 values 合并后的结果，不能仅凭这条命令断定“Prometheus 程序一定升级”或“一定不会升级”。执行前可用 `helm list -n prometheus` 查看当前 Release 的 Chart 版本；生产环境应先审阅目标 Chart 版本和渲染后的变更，再决定是否执行。

本示例 values 文件的结构是：

```yaml
serverFiles:
  alerting_rules.yml:
    groups:
      - name: spring-boot-api-slo
        interval: 30s
        rules:
          # SpringBootApiTargetDown and SpringBootApiSLOBurnRateHigh are defined here.
```

- `serverFiles` 是该 Helm Chart 用来生成 Prometheus Server 配置文件的 values 项。
- `alerting_rules.yml` 是规则文件名；下面的 `groups` 和 `rules` 是 Prometheus 规则文件的内容。Helm Chart 会据此更新 Prometheus 的规则配置，Prometheus 再按规则组的 `interval` 周期评估 PromQL 表达式。
- 这些规则表达式引用前面已抓取的 `up` 和 HTTP 请求计数指标。条件满足并达到 `for` 指定的持续时间后，Prometheus 将规则标记为告警状态；规则本身不会产生业务请求，也不会代替 Service 注解或抓取任务。

### 文件中的告警规则如何工作？

`groups` 定义一组相关规则。`name: spring-boot-api-slo` 是规则组名称；`interval: 30s` 表示 Prometheus 每 30 秒评估一次组内规则。`rules` 下定义了两条告警：

#### `SpringBootApiTargetDown`

```promql
up{namespace="observability-demo", service="spring-boot-api"} == 0
```

`up` 是 Prometheus 为每个抓取目标生成的指标：最近一次抓取成功时为 `1`，失败时为 `0`。此表达式选中本示例的 API 目标，并在抓取失败时满足条件。`for: 2m` 要求条件持续成立 2 分钟，才进入告警状态，避免短暂失败立即告警。`severity: warning` 是告警级别标签；`summary` 和 `description` 是供告警页面或通知展示的说明文字。

#### `SpringBootApiSLOBurnRateHigh`

这条规则用 HTTP 请求计数器计算业务请求中的 `5xx` 比例，并检查两个时间窗口：

```promql
sum(rate(http_server_requests_seconds_count{namespace="observability-demo", service="spring-boot-api", status=~"5..", uri!~"/health|/actuator.*"}[5m]))
/
sum(rate(http_server_requests_seconds_count{namespace="observability-demo", service="spring-boot-api", uri!~"/health|/actuator.*"}[5m]))
```

分子是最近 5 分钟内 `5xx` 业务请求的每秒平均速率；分母是同一时间窗口内所有纳入统计的业务请求速率。`rate(...[5m])` 根据计数器在 5 分钟窗口的变化计算速率，`sum` 汇总不同 URI、状态码等标签对应的时间序列。`status=~"5.."` 选择所有 `5xx` 状态码；`uri!~"/health|/actuator.*"` 排除健康检查和 Actuator 路径。规则对 1 小时窗口使用相同的计算方式，只将 `[5m]` 换为 `[1h]`。

告警要求 **5 分钟和 1 小时的 `5xx` 比例都大于 `0.0144`（1.44%）**，并额外要求最近 5 分钟有业务请求，避免无流量时计算空比例。`for: 2m` 表示这组条件还需连续成立 2 分钟才触发。按示例假设的 99.9% 成功率目标，错误预算为 0.1%；1.44% 是其 14.4 倍，因此该规则用于发现错误预算消耗过快，不用于判断滚动 30 天 SLO 是否达标。`severity: critical`、`summary` 和 `description` 分别提供告警级别和展示文本。

这两条告警规则只在对应指标存在时才有意义：目标 Down 告警依赖 `up`，错误预算告警依赖 HTTP 请求指标和业务流量。若修改 namespace、Service 名、健康检查 URI 或业务成功定义，应同步调整文件中的 PromQL。当前 URI 过滤假设健康检查为 `/health` 或 Actuator 路径；若应用使用其他路径，应改为匹配实际 URI。

若名为 `prometheus` 的 Release 尚不存在，`helm upgrade` 不会创建它；应先按环境实际情况安装并配置 Prometheus。应用配置后可在 Prometheus 的 **Alerts（告警）** 页面查看规则和告警状态。Prometheus 中显示告警状态不等于已发送通知；若要发送邮件、聊天消息等，还需配置 Alertmanager 的接收器和路由。

规则排除了 `/health` 和 `/actuator` 路径；如果你的应用使用不同的健康检查路径或业务路由，请修改 values 文件中的路径过滤条件。

## 6. 在 Grafana 中查看指标

使用 Grafana 中现有的 Prometheus 数据源创建一个名为 **Spring Boot API Observability** 的 Dashboard，并添加以下两个 Panel：

1. **Business Request Rate (req/s)** — 查看最近 5 分钟的 HTTP 请求速率：

```promql
rate(http_server_requests_seconds_count{namespace="observability-demo", service="spring-boot-api"}[5m])
```

2. **JVM Memory Used (MB)** — 查看应用当前使用的 JVM 内存，查询将字节数换算为 MB：

```promql
jvm_memory_used_bytes{namespace="observability-demo", service="spring-boot-api"} / 1000000
```

以上英文名称可直接用作 Dashboard 和 Panel 标题。也可以导入 Spring Boot/Micrometer 仪表板，并选择现有的 Prometheus 数据源。

## 范围与限制

此示例只采集指标。Prometheus 不存储应用日志或 Kubernetes 事件。要构建更完整的可观测性方案，还需要日志采集（例如 Grafana Alloy 和 Loki）、Kubernetes 事件采集，以及分布式链路追踪（OpenTelemetry 和 Tempo）。这些组件不会由此处的配置安装。

此 SLO 规则仅用于学习，不是生产环境策略。生产告警应根据面向用户的服务可用性目标、请求量和约定的错误预算窗口制定。

## 删除示例

```bash
kubectl delete -f spring-observability/spring-boot-observability.yaml
```

告警规则会继续保留在 Prometheus Helm Release 中。要删除规则，需从 Helm values 中移除后再升级该 Release。