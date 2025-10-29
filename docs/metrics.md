# Observability Metrics

This stack exposes application and infrastructure telemetry through Prometheus. The tables below outline the core metrics to watch along with PromQL examples you can use in Grafana dashboards or alert rules.

## WordPress / PHP-FPM

| Metric | Purpose | PromQL |
| --- | --- | --- |
| Pod CPU usage | Detect PHP worker saturation | `sum by (pod)(rate(container_cpu_usage_seconds_total{namespace="default",pod=~"my-release-wordpress-stack-wordpress.*",container!="",container!="POD"}[5m]))` |
| Pod memory usage | Ensure PHP pods stay within limits | `sum by (pod)(container_memory_working_set_bytes{namespace="default",pod=~"my-release-wordpress-stack-wordpress.*",container!="",container!="POD"})` |
| Pod restarts | Catch crash loops | `increase(kube_pod_container_status_restarts_total{namespace="default",pod=~"my-release-wordpress-stack-wordpress.*"}[1h])` |
| Request latency (PHP-FPM) | Optional: track upstream latency from Nginx logs | Collect via Nginx log exporter or OpenTelemetry pipeline |

## MySQL

| Metric | Purpose | PromQL |
| --- | --- | --- |
| Uptime | Ensure DB availability | `mysql_global_status_uptime{pod=~"my-release-wordpress-stack-mysql.*"}` (requires mysqld exporter) |
| Connections | Track client saturation | `mysql_global_status_threads_connected{pod=~"my-release-wordpress-stack-mysql.*"}` |
| InnoDB buffer pool usage | Identify memory pressure | `mysql_global_status_buffer_pool_bytes_data{pod=~"my-release-wordpress-stack-mysql.*"}` |

> **Note:** Add `prom/mysqld-exporter` as a sidecar or standalone deployment to collect the above metrics.

## Nginx / OpenResty

| Metric | Purpose | PromQL |
| --- | --- | --- |
| Total requests per second | Baseline traffic | `sum(rate(nginx_http_requests_total{pod=~"my-release-wordpress-stack-nginx.*"}[5m]))` |
| HTTP 5xx rate | Detect upstream or app errors | `sum(rate(nginx_http_requests_total{pod=~"my-release-wordpress-stack-nginx.*",code=~"5.."}[5m]))` |
| Active connections | Gauge load on the proxy | `sum(nginx_connections_active{pod=~"my-release-wordpress-stack-nginx.*"})` |
| Upstream latency | Identify slow PHP responses | `histogram_quantile(0.95, sum by (pod,le)(rate(nginx_upstream_response_mu_seconds_bucket{pod=~"my-release-wordpress-stack-nginx.*"}[5m])))` (requires exporter histogram support) |

## Alerting Checklist

1. **WordPressHighCPU** (already defined in `wordpress-stack/templates/prometheus-rules.yaml`): triggers when PHP-FPM CPU > 0.8 cores for 5 minutes.
2. **NginxHigh5xxRate**: triggers when Nginx serves more than 5 HTTP 5xx responses per second over 5 minutes.
3. Add MySQL health alerts once mysqld exporter is deployed (e.g., `Threads_connected` above threshold, uptime drop).

Use Grafana dashboards to visualise these metrics and refine thresholds based on baseline traffic. Adjust the PromQL examples with your release name or namespace if you deploy multiple environments.
