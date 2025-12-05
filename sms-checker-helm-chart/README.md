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

## Prometheus Monitoring (Local/Minikube)

The chart includes kube-prometheus-stack as a subchart for monitoring.

### First-time setup: Update Helm dependencies
```bash
cd sms-checker-helm-chart
helm dependency update
cd ..
```

### Install with monitoring enabled
```bash
helm install sms-checker ./sms-checker-helm-chart \
    -f sms-checker-helm-chart/values.yaml
sudo minikube tunnel
```

### Access Prometheus (via port-forward)
```bash
kubectl port-forward svc/sms-checker-kube-prometheus-prometheus 9090:9090
# Open http://localhost:9090
```

### Access Grafana (via port-forward)
```bash
kubectl port-forward svc/sms-checker-grafana 3000:80
# Open http://localhost:3000
# Default credentials: admin / admin
```

### Application Metrics

See [METRICS.md](./METRICS.md) for documentation of custom metrics.

### Disable monitoring
```yaml
# In values.yaml
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

```
