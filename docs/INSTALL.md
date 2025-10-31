# Install Notes

These are the exact steps I follow on my laptop to bring the stack up, poke at it, and tear it back down. Copy/paste away.

## 1. Prep the tools
Make sure Docker, kind, kubectl, and Helm are installed.
```bash
docker --version
kind version
kubectl version --client
helm version
```

## 2. Launch a kind cluster
```bash
kind create cluster --name syfe-cluster
kubectl config use-context kind-syfe-cluster
kubectl get nodes
```

## 3. Build the images
```bash
docker build -t syfe-mysql:local mysql
docker build -t syfe-wordpress:local wordpress
docker build -t syfe-nginx:local nginx
```

## 4. Load the images into kind
```bash
kind load docker-image syfe-mysql:local --name syfe-cluster
kind load docker-image syfe-wordpress:local --name syfe-cluster
kind load docker-image syfe-nginx:local --name syfe-cluster
```

## 5. Deploy WordPress
```bash
helm upgrade --install my-release ./wordpress-stack
kubectl get pods -n default
kubectl get pv,pvc
```

## 6. Add the monitoring stack
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
helm upgrade monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --reuse-values \
  --set kubelet.serviceMonitor.cAdvisor=true
kubectl get pods -n monitoring
```

## 7. Sanity checks
- WordPress UI  
  ```bash
  kubectl port-forward svc/my-release-wordpress-stack-nginx 8080:80
  open http://localhost:8080
  ```
- Grafana (login: `admin / prom-operator`)  
  ```bash
  kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
  open http://localhost:3000
  ```
- Prometheus  
  ```bash
  kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090
  open http://localhost:9090
  ```
- Import dashboards: Grafana → Dashboards → Import → pick JSON from `grafana/`.
- Alert smoke test:
  - CPU spike: `hey -z 30s -c 10 http://localhost:8080/`
  - Force a 5xx: `curl -I http://localhost:8080/wp-admin/nonexistent`
  - Watch Grafana → Alerting → Alert rules.

## 8. Quick troubleshooting
- PVC stuck in `Pending` → `kubectl delete pv my-release-wordpress-stack-{mysql,wordpress}` and re-run Helm.
- Port-forward busy → switch ports, e.g. `kubectl port-forward svc/... 8081:80`.
- Missing Nginx metrics → make sure the metrics Service exists: `kubectl get svc my-release-wordpress-stack-nginx-metrics`.
- Lost Grafana password → `kubectl get secret monitoring-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d`.

## 9. Cleanup
```bash
helm uninstall my-release
helm uninstall monitoring -n monitoring
kubectl delete namespace monitoring
kind delete cluster --name syfe-cluster
```
