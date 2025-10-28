# WordPress Stack on Kind

This repo contains everything needed to run a production-style WordPress stack on a local `kind` cluster along with observability tooling.

## Components
- **MySQL** – custom image with the `wordpress` database and default credentials for local testing.
- **WordPress (PHP-FPM)** – minimal Dockerfile exposing port 9000 so it can sit behind Nginx.
- **OpenResty / Nginx** – compiled with Lua, `stub_status`, Postgres, and iconv modules, serving WordPress via FastCGI and exposing `/nginx_status` for scraping.
- **Helm chart (`wordpress-stack/`)** – deploys MySQL, WordPress, and Nginx, creates hostPath-backed RWX PVCs, and wires Services/ConfigMaps.
- **Monitoring** – `kube-prometheus-stack` installed via Helm with cAdvisor scraping enabled for pod-level metrics.

## Quick Start
```bash
# Build and load images
docker build -t syfe-mysql:local mysql
docker build -t syfe-wordpress:local wordpress
docker build -t syfe-nginx:local nginx
kind load docker-image syfe-mysql:local --name syfe-cluster
kind load docker-image syfe-wordpress:local --name syfe-cluster
kind load docker-image syfe-nginx:local --name syfe-cluster

# Deploy WordPress stack
helm install my-release ./wordpress-stack

# Deploy monitoring stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace
helm upgrade monitoring prometheus-community/kube-prometheus-stack --namespace monitoring --reuse-values --set kubelet.serviceMonitor.cAdvisor=true
```

## Useful Commands
- Check workloads: `kubectl get pods -A`
- Port-forward WordPress ingress: `kubectl port-forward svc/my-release-wordpress-stack-nginx 8080:80`
- Port-forward Grafana: `kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80`
- Port-forward Prometheus: `kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090`

## Cleanup
```bash
helm uninstall my-release
helm uninstall monitoring -n monitoring
kubectl delete namespace monitoring
```
