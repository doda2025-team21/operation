# sms-checker Helm chart

## What it does
Deploys the app and model services with a single Helm release.

## Install
### set up ingress
If we're running it in our PC, you need to set up ingress as LoadBalancer. So [`minikube tunnel` can work as expected.](https://minikube.sigs.k8s.io/docs/commands/tunnel/)
```bash
helm list -A
#make sure no ingress-nginx is running
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# make sure service.type = LoadBalancer
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer
```

```bash
helm install sms-checker ./sms-checker-helm-chart \
    -f sms-checker-helm-chart/values.yaml \
    --set app.ingress.hosts.preview=sms-preview.local 
sudo minikube tunnel
```

## Point hosts to your ingress IP

1. `127.0.0.1 sms.local sms-preview.local` put this in /etc/hosts.
2. In browser, open `sms.local/sms/` and `sms-preview.local/sms/`(If you you'be `--set app.ingress.hosts.preview=sms-preview.local`). It takes probably half a minute to work.

## Prometheus Monitoring

We are going to use kube-prometheus-stack to install:

Prometheus → collects metrics

Grafana → lets you view dashboards

Alertmanager → handles alerts

### Firsttime setup: Update Helm dependencies

The monitoring stack is included as a subchart, so we need to pull its dependencies once.

```bash
cd sms-checker-helm-chart
helm dependency update
cd ..
```
This makes sure Helm has all required charts before installing.

### Install with monitoring enabled
```bash
helm install sms-checker ./sms-checker-helm-chart \
    -f sms-checker-helm-chart/values.yaml \
    --kubeconfig kubeconfig
```
This allows SMS Checker app, Prometheus, Grafana and Service monitors to run inside the Kubernetes cluster.

### Access Grafana
```bash
# Add to /etc/hosts
echo "192.168.56.90 grafana.local" | sudo tee -a /etc/hosts

after this, you can open Grafana easily in your browser:http://grafana.local
# Default credentials: admin / admin

You’ll be asked to change the password on first login.
```

### Access Prometheus 
```bash
KUBECONFIG=./kubeconfig kubectl port-forward svc/sms-checker-kube-prometheus-prometheus 9090:9090
# Open http://localhost:9090
```

### Application Metrics

The app exposes these Prometheus-compatible metrics at `/metrics`:

| Metric | Type | Description |
|--------|------|-------------|
| `sms_requests_total` | Counter | Total SMS classification requests (labels: endpoint, status, classification) |
| `sms_queue_size` | Gauge | Current messages in processing queue (labels: priority) |
| `sms_classification_duration_seconds` | Histogram | Time to classify SMS messages (labels: model_version, classification) |

See [METRICS.md](./METRICS.md) for documentation of custom metrics.

### Disable monitoring
If you don’t want Prometheus and Grafana installed with your chart, you can turn them off in your values.yaml:

```yaml
# put the following in values.yaml
prometheus:
  enabled: false
```

This keeps the deployment lightweight, only the application will be installed.

# How to delete everything
```bash
# delete helm repo
helm repo list -A
helm repo remove your_stuff

# delete helm 
helm list -A
helm uninstall your_stuff

```
