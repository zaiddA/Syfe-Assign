# Grafana Dashboards

I exported the panels I used while testing so you can import them without rebuilding everything by hand.

## Import steps
1. Port-forward Grafana:
   ```bash
   kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
   ```
2. Open [http://localhost:3000](http://localhost:3000) and log in (default `admin / prom-operator`).
3. Go to **Dashboards → Import**.
4. Upload any JSON from the `grafana/` folder (or copy/paste the contents).
5. Pick the Prometheus data source (UID `prometheus`) when Grafana asks.

## What each JSON contains
| File | Panel | Query |
| --- | --- | --- |
| `grafana/wordpress-cpu-dashboard.json` | WordPress CPU per pod | `sum by (pod)(rate(container_cpu_usage_seconds_total{...}[5m]))` |
| `grafana/wordpress-memory-dashboard.json` | WordPress memory per pod | `sum by (pod)(container_memory_working_set_bytes{...})` |
| `grafana/nginx-requests-dashboard.json` | Total requests per second | `sum(rate(nginx_http_requests_total{...}[5m]))` |
| `grafana/nginx-5xx-dashboard.json` | 5xx requests per second (with zero fallback) | `sum(rate(nginx_http_requests_total{code=~"5.."}[5m])) or on() vector(0)` |


## Extra polish ideas
- Add units/transformations (e.g. MiB for memory).
- Draw horizontal threshold lines to match the alert values in `values.yaml`.
- Hook alert rules to Grafana’s Contact points if you want alert routing managed there instead of PrometheusRule.

