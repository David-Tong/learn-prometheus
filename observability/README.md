# Spring Boot Observability Demo

This folder connects a demo Spring Boot API to the existing Prometheus and Grafana installation in the `prometheus` namespace.

## Files

- `spring-boot-observability.yaml` creates the `observability-demo` namespace, Spring Boot Deployment, and Service. The Service annotations enable scraping at `/actuator/prometheus`.
- `prometheus-slo-rules-values.yaml` adds a target-down alert and a multi-window error-budget burn-rate alert for a 99.9% availability SLO.

The demo image is for learning only and was last published in 2020. Do not use it in production. Replace the image with an application image you maintain before using this setup outside a lab.

## 1. Deploy the API

From the repository root, apply the manifest:

```bash
kubectl apply -f observability/spring-boot-observability.yaml
```

Wait for the Pod to become ready and confirm the Service has an endpoint:

```bash
kubectl get pods,services -n observability-demo
kubectl get endpoints spring-boot-api -n observability-demo
```

The Pod should be `Running` and ready. Prometheus discovers the Service through its `prometheus.io/*` annotations; no `ServiceMonitor` is required for the existing Prometheus Helm chart.

## 2. Verify Prometheus scraping

Open the existing Prometheus UI and go to **Status > Targets**. Find the `kubernetes-service-endpoints` job and confirm the target for `spring-boot-api` is `UP`.

You can also run these queries in Prometheus:

```promql
up{namespace="observability-demo", service="spring-boot-api"}
```

```promql
http_server_requests_seconds_count{namespace="observability-demo", service="spring-boot-api"}
```

The first query should return `1` when scraping succeeds. The request metric appears after requests reach the API. Health and Actuator requests are excluded from the SLO calculation, so the error-budget alert needs business API traffic to evaluate.

## 3. Install the SLO alert rules

The values file is intended for the existing `prometheus-community/prometheus` Helm release. From the repository root, run:

```bash
helm upgrade prometheus prometheus-community/prometheus --namespace prometheus --reuse-values --version 27.45.0 -f observability/prometheus-slo-rules-values.yaml
```

Check the rule and alert state in Prometheus under **Alerts**. The rules are:

- `SpringBootApiTargetDown`: warns when Prometheus cannot scrape the API for 2 minutes.
- `SpringBootApiSLOBurnRateHigh`: fires when the 5xx ratio is above 1.44% over both 5-minute and 1-hour windows for 2 minutes. This corresponds to a 14.4x burn rate against a 99.9% availability SLO.

The burn-rate alert is intentionally inactive until the API has business requests. It excludes `/health` and `/actuator` paths; update the route filter in the values file if your application uses different health paths or business routes.

This Helm rule configuration makes alerts visible in Prometheus. To send notifications, configure a receiver in Alertmanager separately.

## 4. View metrics in Grafana

Use the existing Prometheus data source in Grafana. Useful queries include:

```promql
rate(http_server_requests_seconds_count{namespace="observability-demo", service="spring-boot-api"}[5m])
```

```promql
jvm_memory_used_bytes{namespace="observability-demo", service="spring-boot-api"}
```

You can create a dashboard panel from these queries or import a Spring Boot/Micrometer dashboard and select the existing Prometheus data source.

## Scope and limitations

This demo collects metrics only. Prometheus does not store application logs or Kubernetes events. A broader observability stack also needs log collection (for example Grafana Alloy and Loki), Kubernetes event collection, and distributed tracing (OpenTelemetry with Tempo). Those components are not installed by these manifests.

The SLO rule is a learning example, not a production SLO policy. Production alerting should be based on the service's user-facing availability contract, request volume, and agreed error-budget windows.

## Remove the demo

```bash
kubectl delete -f observability/spring-boot-observability.yaml
```

The alert rules remain part of the Prometheus Helm release until you remove them from its values and upgrade the release.