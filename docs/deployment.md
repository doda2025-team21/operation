# Deployment Documentation

This document explains the final deployment at an abstract level and is split
by assignment phase (A1-A4). It focuses on entities, connections, and request
flow, and avoids low-level YAML details. Links are provided when exact config
matters.

## A1: Versioned containerized deployment (Docker Compose)

### Deployment structure
A1 deploys two containers in a single Docker Compose network and relies on a
shared library during the app build:

- **app**: Spring Boot UI, exposed to the host at `APP_PORT` (default 8080).
- **model-service**: Python REST API, exposed to the host at `MODEL_PORT`
  (default 8081).
- **lib-version**: Maven library used by the app at build time to align
  versioning across releases (no runtime container).
- **shared model volume**: host path `./model` mounted into the model-service
  container at `/app/output` to hold model artifacts.

The app container depends on the model-service and calls it through the Compose
network using the service name `model-service`.

References:
- `operation/docker-compose.yml`
- `operation/README.md`
- `app/README.md`
- `lib-version/README.md`

### Request/data flow
1. A user opens the UI at `http://localhost:8080/sms`.
2. The app container forwards classification to
   `http://model-service:8081/predict` using the Compose service DNS
   (internal network name, not reachable directly from the host).
3. The model-service loads the model from `/app/output` and returns a prediction.
4. The app returns the result to the user.

There is no dynamic routing in A1 (all traffic is direct).
Inside the Compose network the DNS name is `model-service`, while the host
accesses the same service via `http://localhost:8081`.

### Access points
- UI: `http://localhost:8080/sms`
- Model API: `http://localhost:8081/predict`

### How to run (A1)
From the `operation` repo root:

```bash
cd operation
docker compose up --pull always
```


Then you can go to `http://localhost:8080/sms` and test out prediction.


Optional variables (via `.env`):
- `APP_PORT`, `MODEL_PORT`, `APP_IMAGE`, `MODEL_IMAGE`

### Visualization

![A1 Compose deployment](./assets/deployment-a1-compose.svg)

## A2: Kubernetes cluster provisioning

(TODO) Describe the abstract cluster setup and core components introduced in A2.

## A3: Helm-based deployment and monitoring

(TODO) Describe Helm release structure, Prometheus/Grafana integration, and
service-level relations.

## A4: Istio traffic management and experimentation

(TODO) Describe traffic routing (stable/canary), dynamic routing decisions, and
additional Istio use case (rate limiting).
