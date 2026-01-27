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

A2 introduces a **self-hosted Kubernetes cluster** that was configured to serve as a platform to host our application and perform various features on the upcoming assignments. A2 specifically dealt with cluster provisioning without any application workloads or deployments, a bare-metal cluster. 

The focus was on establishing a stable, reproducible cluster with substantial idempotency, networking, ingress, observability, and Istio readiness. Provisioning of cluster is performed using `Vagrant` and `Ansible`.

### Key Design Principles 

| Concept                                                       | How it is implemented in our cluster and application                                                                                                                                                                                                                                                                                                 |
| ------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Idempotence**                                               | Idempotence is achieved at multiple layers. Cluster provisioning (Vagrant + Ansible + kubeadm) is designed so repeated runs do not re-initialize an existing cluster. At runtime, Kubernetes itself enforces idempotence: applying the same manifests or Helm chart multiple times converges the system to the same desired state. |
| **ConfigMaps + Secrets** | Non-sensitive and sensitive configuration are externalized from container images and injected at runtime using ConfigMaps and Secrets. This decouples configuration from code, avoids hardcoding credentials, and allows configuration changes or secret rotation without rebuilding or redeploying container images.                                |
| **Volume Mounting**                         | Volumes are used to provide runtime-accessible data (e.g., model artifacts) independent of container lifecycles. This avoids embedding data in images and enables reproducibility, data persistence, and independent updates of models and application code.                                                                                         |
| **Traffic Management**      | Istio routing resources externalize traffic behavior from application logic. VirtualServices define request routing and traffic splitting, while DestinationRules define subsets and consistency policies.                                                       |
| **Separation of Concerns**                                    | Responsibilities are clearly separated across layers: provisioning (Vagrant/Ansible) is independent of deployment (Helm), application logic is independent of traffic routing (Istio), and observability is independent of business functionality.         |
| 

### Services/Installations in Nodes

The control node runs all control-plane, ingress, mesh, and monitoring components, while worker nodes run application workloads, Envoy sidecars, and expose metrics.

| Category                  | Control Node (`ctrl`)                                                      | Worker Nodes (`worker-*`)                         |
| ------------------------- | -------------------------------------------------------------------------- | ------------------------------------------------- |
| **Container Runtime**     | Docker / containerd                                                        | Docker / containerd                               |
| **Kubernetes Core**       | kubeadm, kubelet, kubectl, API Server, Scheduler, Controller Manager, etcd | kubelet, kube-proxy                               |
| **Networking (CNI)**      | CNI plugin (Flannel / Calico)                                              | CNI plugin (Flannel / Calico)                     |
| **Ingress**               | Ingress Controller (NGINX)                                                 | —                                                 |
| **Service Mesh (Istio)**  | istiod, Istio Ingress Gateway                                              | Envoy sidecar (per pod)                           |
| **Application Workloads** | —                                                                          | app-service, model-service (stable & canary pods) |
| **Monitoring**            | Prometheus, Prometheus Operator, Alertmanager, Grafana                     | Metrics endpoints, scrape targets                 |
| **Storage (optional)**    | Helm-managed volumes / configs                                             | hostPath volumes (if used)                        |
| **Provisioning & Ops**    | Vagrant, Ansible, Helm                                                     | —                                                 |


### Traffic Flow

```
External Traffic
       |
       v
+-------------------+
| Istio Gateway     |   (or Kubernetes Ingress)
+-------------------+
       |
       v
+-------------------+
| VirtualService    |
| (traffic rules)   |
| - route by weight |
| - route by subset |
+-------------------+
       |
       v
+-------------------+
| DestinationRule   |
| (subsets)         |
| - stable          |
| - canary          |
| (map labels ->    |
|  pod versions)    |
+-------------------+
       |
       v
+-------------------------------+
| Service: sms-frontend         |
| - stable endpoint             |
| - load balancing              |
| - central hub                 |
+-------------------------------+
       |
       v
   +---------+   +---------+   +---------+
   | Pod v1  |   | Pod v2  |   | Pod v3  |
   | stable  |   | stable  |   | canary  |
   +---------+   +---------+   +---------+
        ^             ^             ^
        |             |             |
        +-------------+-------------+
                      |
        +-------------------------------+
        | ServiceMonitor (Prometheus)   |
        | - scrapes frontend metrics    |
        +-------------------------------+

Deployment
-----------
+-------------------------------+
| Deployment                    |
| - defines image & version     |
| - creates pods with labels    |
|   version=stable / canary     |
+-------------------------------+
```

Traffic flow through the k8 cluster:
1. Incoming traffic enters the cluster through an Istio Gateway (or Ingress).
2.  VirtualService applies routing rules (e.g., 90/10 split) and forwards traffic to subsets defined in the DestinationRule.
3. Subsets map to pod versions via labels managed by the Deployment.
4. Kubernetes Service provides a stable endpoint and load balancing, while Prometheus monitors the service via a ServiceMonitor.

## A3: Helm-based deployment and monitoring

(TODO) Describe Helm release structure, Prometheus/Grafana integration, and
service-level relations.

## A4: Istio traffic management and experimentation

### Deployment structure

A4 introduces istio service mesh for traffic management, enabling canary releases
with consistent routing and sticky sessions(is in fix). The deployment builds on A3 by adding Istio traffic management resources and extends monitoring with grafana dashboards.

#### Deployed Resources

**Application Layer (4 Deployments, 5 Services)**

| Resource | Type | Purpose |
|----------|------|---------|
| `app-stable` | Deployment | Stable version of the web application (90% traffic) |
| `app-canary` | Deployment | Canary version for experimentation (10% traffic) |
| `model-service-stable` | Deployment | ML model service for stable app |
| `model-service-canary` | Deployment | ML model service for canary app |
| `app` | Service | Main service endpoint for the application |
| `model-service` | Service | Generic model service (not used in A4) |
| `model-service-stable` | Service | Routes only to stable model pods |
| `model-service-canary` | Service | Routes only to canary model pods |

**Istio Traffic Management (4 Resources)**

| Resource | Type | Purpose |
|----------|------|---------|
| Istio Ingress Gateway | Pod (istio-system) | Entry point for external traffic, receives requests at `192.168.56.91:80`. Provisioned during cluster setup, not part of Helm chart. |
| `app-gateway` | Gateway | Accepts HTTP traffic for host `sms-istio.local` on port 80 and routes into the mesh |
| `app-vs` | VirtualService | Implements 90/10 traffic split and header-based canary routing |
| `app-destinationrule` | DestinationRule | Defines stable/canary subsets with sticky session configuration |
| `rate-limit-filter` | EnvoyFilter | Implements per-user rate limiting (additional use case) |

**Monitoring Stack** (Prometheus, Grafana, ServiceMonitors, Alertmanager)

**Configuration** (ConfigMaps for app settings, Secrets for credentials)

#### Consistent Routing (Old to old, new to new)

One requirement was to ensure version consistency between app and model-service.
This is achieved through dedicated Kubernetes Services with version specific selectors:

```
app-stable deployment                    model-service-stable deployment
  ENV: MODEL_HOST=model-service-stable ──────► selector: version=stable
  
app-canary deployment                    model-service-canary deployment  
  ENV: MODEL_HOST=model-service-canary ──────► selector: version=canary
```

The `MODEL_HOST` environment variable in each app deployment is hardcoded to call
the matching model service, guaranteeing routing at deployment time.

#### Ingress Gateway Configuration

The Istio Ingress Gateway is provisioned during cluster setup with label
`istio: ingressgateway`. The Helm chart makes this configurable via
`istio.ingressGateway.selector` in values.yaml for different clusters.

### How the data flows

1. User accesses `http://sms-istio.local/sms/` (requires `/etc/hosts` entry: `192.168.56.91 sms-istio.local`)
2. **Istio Ingress Gateway** → **Gateway** → **VirtualService** (canaryheader based routing)
3. **DestinationRule** applies sticky session via `sms-session` cookie (TTL: 1 hour)
4. **EnvoyFilter** checks rate limit (10 req/min per user, returns 429 if exceeded)
5. Request reaches `app-stable` or `app-canary` pod
6. App calls matching model service (`model-service-stable` or `model-service-canary`)
7. Model returns prediction, app renders result to user
8. **Prometheus** scrapes metrics, **Grafana** displays experiment dashboards

### Dynamic routing decisions

| Decision Point | Component | Configuration Location |
|----------------|-----------|------------------------|
| 90/10 traffic split | VirtualService (`app-vs`) | `istio.canary.weight` in values.yaml |
| Sticky sessions | DestinationRule (`app-destinationrule`) | `istio.stickySession.enabled` in values.yaml |
| Force canary (testing) | VirtualService (`app-vs`) | Header `x-canary: true` |
| Rate limiting | EnvoyFilter (`rate-limit-filter`) | `istio.rateLimiting` in values.yaml |
| App→Model routing | Kubernetes Service DNS | `MODEL_HOST` env var in app-deployment.yaml |

**Where is the 90/10 split configured?** In `values.yaml` under `istio.canary.weight`.

**Where is the routing decision taken?** The Envoy proxy in the Istio Ingress Gateway
evaluates VirtualService rules and reads the `sms-session` cookie for sticky sessions.

### Additional Use Case: Rate Limiting

Per user rate limiting using an EnvoyFilter with local rate limit configuration.
Users (by IP) are allowed 10 requests per minute; exceeding returns HTTP 429.
Configured via `istio.rateLimiting` in values.yaml (burst size, tokens per refill,
fill interval). The EnvoyFilter injects this into the Ingress Gateway's HTTP filter chain.

### Access points

| Endpoint | URL/ Command| Headers/Notes |
|----------|-------------|---------------|
| App (via Istio) | `http://sms-istio.local/sms/` | Requires `/etc/hosts`: `192.168.56.91 sms-istio.local` |
| Force canary | `curl -H "x-canary: true" http://sms-istio.local/sms/` | Always routes to canary version |
| Grafana | `kubectl port-forward svc/myprom-grafana 3000:80` | Credentials: admin/prom-operator |
| Prometheus | `kubectl port-forward svc/myprom-kube-prometheus-prometheus 9090:9090` | |
| App metrics | `http://sms-istio.local/actuator/prometheus` | Exposed via ServiceMonitor |

### Visualization

![A4 Istio deployment](./assets/deployment-a4-istio.png)


### Continuous Experimentation Support

The deployment supports A/B testing between stable and canary version.