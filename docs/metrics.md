# Observability Cheat Sheet

Here’s the stuff I actually look at once the stack is running. Everything is Prometheus-based so you can drop these queries into Grafana or straight into the Prometheus UI.

## WordPress / PHP-FPM

| Metric | Why I care | PromQL |
| --- | --- | --- |
| Pod CPU usage | Detects when PHP workers are maxing out | `sum by (pod)(rate(container_cpu_usage_seconds_total{namespace="default",pod=~"my-release-wordpress-stack-wordpress.*",container!="",container!="POD"}[5m]))` |
| Pod memory usage | Keeps an eye on memory leaks/uploads | `sum by (pod)(container_memory_working_set_bytes{namespace="default",pod=~"my-release-wordpress-stack-wordpress.*",container!="",container!="POD"})` |
| Pod restarts | Flags crash loops or OOMKills | `increase(kube_pod_container_status_restarts_total{namespace="default",pod=~"my-release-wordpress-stack-wordpress.*"}[1h])` |
| Request latency (needs log exporter) | Lets me spot slow PHP responses | Capture via nginx log exporter or an OpenTelemetry pipeline |

## MySQL (if you add `mysql-exporter`)

| Metric | Why I care | PromQL |
| --- | --- | --- |
| Uptime | Basic liveness check | `mysql_global_status_uptime{pod=~"my-release-wordpress-stack-mysql.*"}` |
| Active connections | Shows if WordPress is overwhelming MySQL | `mysql_global_status_threads_connected{pod=~"my-release-wordpress-stack-mysql.*"}` |
| Buffer pool usage | Highlights memory pressure | `mysql_global_status_buffer_pool_bytes_data{pod=~"my-release-wordpress-stack-mysql.*"}` |

## Nginx / OpenResty

| Metric | Why I care | PromQL |
| --- | --- | --- |
| Requests per second | Quick traffic view | `sum(rate(nginx_http_requests_total{pod=~"my-release-wordpress-stack-nginx.*"}[5m]))` |
| 5xx rate | Detects upstream/app meltdowns | `sum(rate(nginx_http_requests_total{pod=~"my-release-wordpress-stack-nginx.*",code=~"5.."}[5m]))` |
| Active connections | How busy Nginx is right now | `sum(nginx_connections_active{pod=~"my-release-wordpress-stack-nginx.*"})` |
| Upstream latency (if exporter has buckets) | Useful for 95th percentile latency | `histogram_quantile(0.95, sum by (pod,le)(rate(nginx_upstream_response_mu_seconds_bucket{pod=~"my-release-wordpress-stack-nginx.*"}[5m])))` |

## Alerts in this repo
Defined in `wordpress-stack/templates/prometheus-rules.yaml`:

1. **WordPressHighCPU** – fires when a PHP pod burns more CPU than the `values.yaml` threshold for 5 minutes.
2. **NginxHigh5xxRate** – alerts if Nginx responds with too many 5xx codes (default >5 req/s over 5 minutes).

Future ideas once MySQL exporter is wired up:
- Alert if `Threads_connected` crosses a high watermark.
- Alert if uptime resets (MySQL restarts).

Tune the queries to match your release name or namespace if you deploy multiple copies of the chart. I kept everything hardcoded to `default` and the Helm release from this repo for clarity.
