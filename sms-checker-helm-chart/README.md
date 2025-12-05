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
2. In browser, open `sms.local/sms/` and `sms-preview.local/sms/`(If you you'be `--set app.ingress.hosts.preview=sms-preview.local`). It takse probably half a minute to work.

## Prometheus Monitoring

The following shows how Prometheus monitoring is done via the kube-prometheus-stack subchart.

### Firsttime setup: Update Helm dependencies
```bash
cd sms-checker-helm-chart
helm dependency update
cd ..
```

### Install with monitoring enabled
```bash
helm install sms-checker ./sms-checker-helm-chart \
    -f sms-checker-helm-chart/values.yaml \
    --kubeconfig kubeconfig
```

### Access Grafana
```bash
# Add to /etc/hosts
echo "192.168.56.90 grafana.local" | sudo tee -a /etc/hosts

# Open http://grafana.local
# Default credentials: admin / admin
```

### Access Prometheus 
```bash
KUBECONFIG=./kubeconfig kubectl port-forward svc/sms-checker-kube-prometheus-prometheus 9090:9090
# Open http://localhost:9090
```

### Application Metrics

The app exposes these custom metrics at `/metrics`:

| Metric | Type | Description |
|--------|------|-------------|
| `sms_requests_total` | Counter | Total SMS classification requests (labels: endpoint, status, classification) |
| `sms_queue_size` | Gauge | Current messages in processing queue (labels: priority) |
| `sms_classification_duration_seconds` | Histogram | Time to classify SMS messages (labels: model_version, classification) |

See [METRICS.md](./METRICS.md) for documentation of custom metrics.

### Disable monitoring
```yaml
# put the following in values.yaml
prometheus:
  enabled: false
```

# How to delete everything
```bash
# delete helm repo
helm repo list -A
helm repo remove your_stuff


# delete helm 
helm list -A
helm uninstall your_stuff
# or
helm uninstall your_stuff -n your_namespace

# delete minikube all data
minikube delete --all -purge
```
