# Application Metrics

Prometheus scrapes metrics from:
- `app` on port 9090
- `model-service` on port 9091

Both expose `/metrics` endpoint.

---

## App Service Metrics

### Counter: `sms_requests_total`
Total SMS classification requests. Shows usage patterns and error rates.

Labels:
- `endpoint` - which API endpoint (`/classify`, `/health`)
- `status` - response category (`2xx`, `4xx`, `5xx`)
- `classification` - result (`spam`, `ham`, `error`)

```python
from prometheus_client import Counter

sms_requests_total = Counter(
    'sms_requests_total',
    'Total SMS requests',
    ['endpoint', 'status', 'classification']
)

sms_requests_total.labels(endpoint='/classify', status='2xx', classification='spam').inc()
```

### Gauge: `sms_queue_size`
Current messages waiting to be processed. Shows system load.

Labels:
- `priority` - message priority (`high`, `normal`, `low`)

```python
from prometheus_client import Gauge

sms_queue_size = Gauge('sms_queue_size', 'Messages in queue', ['priority'])

sms_queue_size.labels(priority='normal').set(42)
sms_queue_size.labels(priority='high').inc()
```

### Histogram: `sms_classification_duration_seconds`
Time to classify messages. Shows latency distribution (P50, P95, P99).

Labels:
- `model_version` - ML model version
- `classification` - result (`spam`, `ham`)

```python
from prometheus_client import Histogram

sms_classification_duration = Histogram(
    'sms_classification_duration_seconds',
    'Classification latency',
    ['model_version', 'classification'],
    buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
)

with sms_classification_duration.labels(model_version='v1', classification='spam').time():
    result = classify(message)
```

---

## Model Service Metrics

### Counter: `model_predictions_total`
Total predictions made. Shows model usage and confidence distribution.

Labels:
- `model_name` - which model
- `prediction` - result (`spam`, `ham`)
- `confidence_bucket` - confidence level (`low`, `medium`, `high`)

```python
model_predictions_total = Counter(
    'model_predictions_total',
    'Total predictions',
    ['model_name', 'prediction', 'confidence_bucket']
)
```

### Gauge: `model_loaded`
Whether model is ready. Shows service health.

Labels:
- `model_name` - model identifier
- `version` - model version

```python
model_loaded = Gauge('model_loaded', 'Model ready (1=yes, 0=no)', ['model_name', 'version'])
model_loaded.labels(model_name='spam_classifier', version='v1').set(1)
```

### Histogram: `model_inference_duration_seconds`
Raw inference time. Shows model performance.

Labels:
- `model_name` - which model

```python
model_inference_duration = Histogram(
    'model_inference_duration_seconds',
    'Inference time',
    ['model_name'],
    buckets=[0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25]
)
```

---

## Setup

Add to app startup:

```python
from prometheus_client import start_http_server
import os

start_http_server(int(os.environ.get('METRICS_PORT', 9090)))
```

requirements.txt:
```
prometheus_client>=0.17.0
```

---

## Access

```bash
echo "192.168.56.90 prometheus.local grafana.local" | sudo tee -a /etc/hosts
```

- Prometheus: http://prometheus.local
- Grafana: http://grafana.local (admin/admin)

---

## Example Queries

```promql
# request rate
rate(sms_requests_total[5m])

# error rate
sum(rate(sms_requests_total{status="5xx"}[5m])) / sum(rate(sms_requests_total[5m]))

# P95 latency
histogram_quantile(0.95, rate(sms_classification_duration_seconds_bucket[5m]))

# spam ratio
sum(rate(sms_requests_total{classification="spam"}[5m])) / sum(rate(sms_requests_total[5m]))
```
