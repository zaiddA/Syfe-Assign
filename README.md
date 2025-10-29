# WordPress Stack on Kind

Hey! I put this project together while prepping for the Syfe Infra assignment to prove that I can run a “real” WordPress stack on Kubernetes and keep an eye on it like an SRE. Everything lives in this repo so you can spin it up on any laptop with Docker.

## What’s Inside
- **OpenResty / Nginx (Lua build)** – fronts all traffic, talks FastCGI to PHP-FPM, and exposes `/nginx_status` so Prometheus can scrape useful counters.
- **WordPress PHP-FPM** – trimmed image that only exposes port `9000`; shares the web root with Nginx through a ReadWriteMany PVC.
- **MySQL 8** – custom image that bootstraps the `wordpress` database and uses another RWX volume.
- **Helm chart (`wordpress-stack/`)** – glues everything together: PVs, PVCs, Deployments, Services, ConfigMaps, ServiceMonitor, and a PrometheusRule for alerts.
- **Monitoring** – kube-prometheus-stack via Helm plus an extra exporter in the Nginx pod. Grafana dashboards and alert thresholds are already defined.
- **Docs & dashboards** – `docs/` contains runbooks and metrics notes, while `grafana/` holds ready-to-import JSON panels.

## Prerequisites
- Docker Desktop (or any Docker engine)
- `kind`, `kubectl`, `helm`
- Optional: `hey` or `ab` if you want to hammer the site to test alerts

Check versions quickly:
```bash
docker --version
kind version
kubectl version --client
helm version
```

## How I Spin It Up
1. **Create the kind cluster**
   ```bash
   kind create cluster --name syfe-cluster
   ```

2. **Build the custom images**
   ```bash
   docker build -t syfe-mysql:local mysql
   docker build -t syfe-wordpress:local wordpress
   docker build -t syfe-nginx:local nginx
   ```

3. **Load images into kind**
   ```bash
   kind load docker-image syfe-mysql:local --name syfe-cluster
   kind load docker-image syfe-wordpress:local --name syfe-cluster
   kind load docker-image syfe-nginx:local --name syfe-cluster
   ```

4. **Deploy WordPress**
   ```bash
   helm upgrade --install my-release ./wordpress-stack
   ```

5. **Drop in the monitoring stack**
   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update
   helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
     --namespace monitoring --create-namespace
   helm upgrade monitoring prometheus-community/kube-prometheus-stack \
     --namespace monitoring --reuse-values \
     --set kubelet.serviceMonitor.cAdvisor=true
   ```

## Quick Checks After Deploy
- `kubectl get pods -A` – every pod in `default` and `monitoring` should say `Running`.
- `kubectl get pv,pvc` – both PVs should be `Bound` (hostPath + RWX).
- `kubectl port-forward svc/my-release-wordpress-stack-nginx 8080:80` and open `http://localhost:8080` to finish the WordPress setup.
- `kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80` → log in with `admin / prom-operator`.
- `kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090` → check `Targets` to make sure the exporter is scraped.

## Screenshots
All proof-of-life shots live in `docs/images/`:
- `wordpress-home.png` – landing page after setup.
- `kubectl-pods.png` – cluster view showing all pods healthy.
- `grafana-wordpress-cpu.png` / `grafana-wordpress-memory.png` – PHP pod metrics.
- `grafana-nginx-requests.png` / `grafana-nginx-5xx.png` – Nginx throughput and error panels.

## Monitoring + Alerts
- Import any dashboard from `grafana/` (CPU, memory, Nginx requests, Nginx 5xx) through Grafana → **Dashboards → Import**.
- `docs/metrics.md` lists the PromQL I’m using and a few extra metrics I’d add if I had more time.
- Alert rules live in `wordpress-stack/templates/prometheus-rules.yaml`:
  - `WordPressHighCPU` – warns if a PHP pod keeps burning more CPU than the threshold in `values.yaml`.
  - `NginxHigh5xxRate` – catches spikes of 5xx responses.
- You can manage contact points from Grafana’s Alerting UI once the rules register in Prometheus.

## Handy Commands
- Watch everything: `kubectl get pods -A`
- Inspect Helm release: `helm status my-release`
- Check PrometheusRule: `kubectl get prometheusrule -n monitoring my-release-wordpress-stack-alerts -o yaml`
- Curl the exporter:  
  ```bash
  kubectl run curl --rm -it --image=curlimages/curl --restart=Never \
    -- curl http://my-release-wordpress-stack-nginx-metrics.default.svc.cluster.local:9113/metrics
  ```

## Troubles I Hit (and fixes)
- **PVC Pending forever** – delete the PVs (`kubectl delete pv my-release-wordpress-stack-{mysql,wordpress}`) and re-run the chart; hostPath will recreate them.
- **Port-forward refuses** – something else is on the port; use `lsof -i:8080` or just forward to `8081:80`.
- **No container metrics in Prometheus** – the kubelet cAdvisor scrape wasn’t enabled; rerun the monitoring upgrade command above.
- **Forgot Grafana password** – `kubectl get secret monitoring-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d`.
- **Alerts quiet even though I expect noise** – check `kubectl get prometheusrule -n monitoring` to make sure the CR loaded, then generate enough load/5xx to break the threshold.

## Cleaning Up
```bash
helm uninstall my-release
helm uninstall monitoring -n monitoring
kubectl delete namespace monitoring
kind delete cluster --name syfe-cluster
```

## Where To Read More
- `docs/INSTALL.md` – the full runbook if you want every command in order.
- `docs/metrics.md` – PromQL cheatsheet plus alert ideas.
- `docs/GRAFANA.md` – dashboard import steps and panel details.
- `grafana/*.json` – drop these into Grafana to get the exact dashboards I used during testing.
