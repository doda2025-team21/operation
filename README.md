# SMS-Checker - Operation Registry
This repository is the main entry into this project. It introduces your high-level architecture and links to the other corresponding repositories, so visitors can easily understand your project and find all relevant information. This repository contains a single Docker Compose setup that orchestrates and runs the full application, including:
- `model-service:` Backend Model Service
- `app:` Frontend Web Application

## Repository Structure
The following are the repositories present in this organization:
- [app](https://github.com/doda2025-team21/app) : Frontend Java Spring Boot service providing the UI
- [model-service](https://github.com/doda2025-team21/model-service) : Backend Python microservice that loads and serves the spam classifier
- [lib-version](https://github.com/doda2025-team21/lib-version) : Library dependencies ?? //TODO

# A1: Version Release and Containerization
## Requirements
The following needs to be installed before running this project:
- Docker
- Docker Compose
Other dependencies such as Python or Java will be installed in respective docker containers.

## Environment Variables Setup
The docker compose uses `.env` file for setting up the environment variables:
```
APP_PORT = 8080
MODEL_PORT = 8081
APP_IMAGE = ghcr.io/doda2025-team21/frontend
MODEL_IMAGE = ghcr.io/doda2025-team21/backend
```
You can change the ports or image versions here.

## How to start the application?
1. Make sure to clone all the repositories using 
```
git clone ....
```
2. Navigate to the appropriate directory 
```
cd operation
```
3. Run the following command: 
```
docker compose up --pull always
```
The following command does the following:
- it will run the latest version of the image from `MODEL_IMAGE`
- Start the backend on [http://localhost:8081](http://localhost:8081) (or replace the port with one mentioned in MODEL_PORT). 
- Start the frontend on [http://localhost:8080/sms](http://localhost:8080/sms) (or replace port with one mentioned in APP_PORT).
An alternative to running the latest version of the image is specifying the version of the you'd like to run using the `TAG_VERSION=***` variable:
```
TAG_VERSION=0.0.1 docker compose up --pull always
```
4. To stop running everything, run the following command:
```
docker compose down
```

## Other handy docker commands

| Action                               | Command                      | Description                            |
| ------------------------------------ | ---------------------------- | -------------------------------------- |
| **Start everything**                 | `docker compose up`          | Starts all services (shows logs)       |
| **Start in background**              | `docker compose up -d`       | Runs services in detached mode         |
| **Stop all running services**        | `docker compose down`        | Stops and removes containers, networks |
| **Rebuild images**                   | `docker compose up --build`  | Rebuilds images before starting        |
| **View logs**                        | `docker compose logs`        | Shows combined logs from all services  |
| **View logs for a specific service** | `docker compose logs app`    | Shows logs only for the app            |
| **Restart one service**              | `docker compose restart app` | Restarts only the app service          |


# A2: Provisioning a Kubernetes Cluster
Goal: reproducibly provision the Kubernetes base platform described in the A2 brief (Vagrant + VirtualBox + Ansible, Flannel CNI, MetalLB, ingress, dashboard, Istio). The controller lives at 192.168.56.100 and workers start at 192.168.56.101; counts and resources are configurable.

## What the automation sets up
- Vagrant builds one `ctrl` and `NUM_WORKERS` `node-*` hosts on a host-only network (controller at 192.168.56.100, workers starting at 192.168.56.101/102/...); CPU/memory/worker count come from `.env` (`NUM_WORKERS`, `CTRL_CPUS`, `CTRL_MEMORY`, `NODE_CPUS`, `NODE_MEMORY`).
- `ansible/general.yml` (Steps 4-12): imports team SSH keys from `ansible/files/ssh-keys/*.pub`, disables swap and fstab entries, loads `overlay`/`br_netfilter`, enables IP forwarding sysctls, templates `/etc/hosts` with all nodes, adds the Kubernetes apt repo, installs containerd 1.7.24 + runc 1.1.12 + kubeadm/kubelet/kubectl 1.32.4, writes containerd config (pause 3.10, AppArmor off, `SystemdCgroup=true`), restarts containerd, enables kubelet.
- `ansible/ctrl.yml` (Steps 13-17): `kubeadm init` with advertise address 192.168.56.100 and pod CIDR 10.244.0.0/16 (idempotent via `/etc/kubernetes/admin.conf` check), copies kubeconfig to `~/.kube` for vagrant and fetches `./kubeconfig` for the host, installs Flannel with `--iface=eth1`, installs Helm and the `helm-diff` plugin.
- `ansible/node.yml` (Steps 18-19): delegates `kubeadm token create --print-join-command` to the controller and joins each worker if `kubelet.conf` is missing.
- `ansible/finalization.yml` (Steps 20-23): MetalLB with pool `192.168.56.90-192.168.56.99` and readiness waits; ingress-nginx with class `nginx` and load balancer IP 192.168.56.90; Kubernetes Dashboard via Helm with an ingress on `dashboard.local` (HTTPS backend) and admin ServiceAccount; Istio 1.25.2 with ingress gateway IP 192.168.56.91.

## Preparation
- Install VirtualBox, Vagrant, and Ansible on the host.
- Add your public key(s) to `ansible/files/ssh-keys/` so SSH works without passwords.
- Optional: adjust worker count/resources in `.env` (e.g., `NUM_WORKERS=2 NODE_MEMORY=6144`).

## Bring up the cluster
```bash
git pull

# Create VMs and run general/ctrl/node provisioning
vagrant up --no-provision            # rerun with `vagrant provision` if needed

# Rerun playbooks manually (useful after edits)
ansible-playbook -u vagrant -i inventory.cfg ansible/general.yml
ansible-playbook -u vagrant -i inventory.cfg ansible/ctrl.yml
ansible-playbook -u vagrant -i inventory.cfg ansible/node.yml

# Finish cluster features (MetalLB, ingress, dashboard, Istio)
ansible-playbook -u vagrant -i inventory.cfg ansible/finalization.yml
```

## Smoke tests (matches A2 expectations)
```bash
# Reach the VMs (if keys are missing, fallback user/password is vagrant/vagrant)
ssh vagrant@192.168.56.100
ssh vagrant@192.168.56.101
ssh vagrant@192.168.56.102

# Check cluster health from host
KUBECONFIG=./kubeconfig kubectl get nodes -o wide
KUBECONFIG=./kubeconfig kubectl get pods -A

# Verify load balancers / ingresses
KUBECONFIG=./kubeconfig kubectl get svc -n metallb-system
KUBECONFIG=./kubeconfig kubectl get svc -n ingress-nginx ingress-nginx-controller -o wide
KUBECONFIG=./kubeconfig kubectl get svc -n kubernetes-dashboard
KUBECONFIG=./kubeconfig kubectl get svc -n istio-system istio-ingressgateway -o wide
```

## Accessing the Kubernetes Dashboard (This doesn't work somehow)
```bash
# Add host entry on your laptop for ingress IP from MetalLB (default 192.168.56.90)
echo "192.168.56.90 dashboard.local" | sudo tee -a /etc/hosts

# Create a fresh admin token (run on host, talks to controller VM)
ssh vagrant@192.168.56.100 "kubectl -n kubernetes-dashboard create token admin-user"
```
Open `http://dashboard.local/` and paste the token. Traffic to the dashboard service is proxied via the nginx ingress with `nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"`.

## Using Istio (AI generated, Not sure it's runnable or not)
```bash
ssh vagrant@192.168.56.100 "istioctl version"
KUBECONFIG=./kubeconfig kubectl get svc -n istio-system istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}{"\n"}'
```
Deploy Istio-enabled workloads by labeling namespaces (`kubectl label ns <name> istio-injection=enabled`) and routing through the Istio ingress gateway IP (default 192.168.56.91).

## How to clean up
```bash
vagrant destroy
```

# A3: Operate and Monitor Kubernetes
## Deploy the stack with Helm
This will run very long time, might be few minutes or even more ........ :smiling_face_with_tear:
```bash
vagrant up --no-provision && \
    ansible-playbook -u vagrant -i inventory.cfg ansible/general.yml && \
    ansible-playbook -u vagrant -i inventory.cfg ansible/ctrl.yml && \
    ansible-playbook -u vagrant -i inventory.cfg ansible/node.yml && \
    ansible-playbook -u vagrant -i inventory.cfg ansible/finalization.yml && \
    helm upgrade --install sms-checker ./sms-checker-helm-chart \
    -f sms-checker-helm-chart/values.yaml \
    --set app.ingress.hosts.stable=sms.local \
    --set app.ingress.hosts.preview=sms-preview.local \
    --set app.ingress.className=nginx \
    --kubeconfig kubeconfig
```
Check resources and ingress:
```bash
KUBECONFIG=./kubeconfig kubectl get pods,svc,ingress
KUBECONFIG=./kubeconfig kubectl get svc -n ingress-nginx ingress-nginx-controller -o wide
```

## Expose ingress to access the app
MetalLB assigns the nginx ingress controller an IP (default 192.168.56.90 from `finalization.yml`). Point your host name to it:
```bash
# sms.local sms-preview.local are the --set host, change that if you used a different host name
echo "192.168.56.90 sms.local sms-preview.local" | sudo tee -a /etc/hosts
```
Then open `http://sms.local/` to reach the frontend; it talks to `model-service` via the cluster service. Open `sms.local/sms/` to open the SMS Checker web. (Note, the sms.local service might takes up 1 or 2 minutes to be ready and I don't know why :sweat_smile:).

### Ingress hosts (stable + preview)
- Stable host: `app.ingress.hosts.stable` (defaults to `sms.local`) drives the main ingress rule.
- Preview/experiment host: `app.ingress.hosts.preview` (defaults empty). If you set a value (e.g., `sms-preview.local`), the chart renders a second ingress rule to the same service—useful for A4 experiments.
- Ingress class: `app.ingress.className` defaults to `nginx`; override if your cluster uses a different controller.
- Point both hostnames to your ingress external IP (MetalLB or Minikube tunnel) via `/etc/hosts` so browsers resolve correctly.

### Minikube note
If using Minikube, enable ingress and get an IP via `minikube addons enable ingress` and `minikube tunnel`, add that IP to `/etc/hosts` for `sms.local`, and browse the same URL.

## Prometheus Monitoring

The Helm chart includes kube-prometheus-stack which installs:
- **Prometheus** → collects metrics
- **Grafana** → view dashboards
- **ServiceMonitors** → auto-discover app metrics

### Access Grafana
```bash
# Add to /etc/hosts
echo "192.168.56.90 grafana.local" | sudo tee -a /etc/hosts
```
Open http://grafana.local in your browser.
- Default credentials: `admin` / `admin`
- You'll be asked to change the password on first login.

### Access Prometheus
```bash
# Add to /etc/hosts
echo "192.168.56.90 prometheus.local" | sudo tee -a /etc/hosts
```
Open http://prometheus.local in your browser.

### Application Metrics

Both services expose Prometheus-compatible metrics:

**App Service** (port 9090, path `/actuator/prometheus`):

| Metric | Type | Description |
|--------|------|-------------|
| `sms_requests_total` | Counter | Total SMS classification requests (labels: endpoint) |
| `sms_queue_size` | Gauge | Current messages in processing queue (labels: priority) |
| `sms_classification_duration_seconds` | Histogram | Time to classify SMS messages (labels: model_version) |

**Model Service** (port 9091, path `/metrics`):

| Metric | Type | Description |
|--------|------|-------------|
| `model_predictions_total` | Counter | Total predictions made (labels: model_name, prediction, confidence_bucket) |
| `model_loaded` | Gauge | Whether model is ready (1=yes, 0=no) |
| `model_inference_duration_seconds` | Histogram | Model inference time (labels: model_name) |

See [METRICS.md](./sms-checker-helm-chart/METRICS.md) for detailed documentation.

### Verify Monitoring is Working
```bash
# Check ServiceMonitors
ssh vagrant@192.168.56.100 "kubectl get servicemonitors"

# Check Prometheus targets (should show UP)
# Open http://prometheus.local → Status → Targets
```

### Disable Monitoring
To deploy without Prometheus/Grafana, set in values.yaml:
```yaml
prometheus:
  enabled: false
```

## How to clean up
```bash
vagrant destroy
```

# A4: Traffic Management with Istio

This section documents the Istio based traffic management configuration for canary releases with sticky sessions. After the explanation, there will be a section where it could be verified.

## Istio Ingress Gateway Configuration

The Istio ingress gateway is provisioned during cluster setup (`ansible/finalization.yml`) and is not a part of the Helm chart, as specified in the assignment description. The gateway runs in the `istio-system` namespace with the following configuration:

### Gateway Labels and Selector

| Component | Label | Value | Description |
|-----------|-------|-------|-------------|
| Ingress Gateway Pod | `istio` | `ingressgateway` | Label used by Gateway resources to select the ingress gateway |
| Ingress Gateway Pod | `app` | `istio-ingressgateway` | Application identifier |
| Service | External IP | `192.168.56.91` | MetalLB-assigned load balancer IP |

### Configurable Gateway Selector

The ingress gateway selector is configurable in `values.yaml` to support different cluster configurations:

```yaml
istio:
  ingressGateway:
    # Default istio installations use 'ingressgateway'
    selector: ingressgateway
    # Below is the namespace where ingress gateway is deployed
    namespace: istio-system
```

If deploying to a cluster with a different ingress gateway configuration as mentioned in the assignment, you can override the selector using the following:
```bash
helm upgrade --install sms-checker ./sms-checker-helm-chart \
  --set istio.ingressGateway.selector=my-custom-gateway \
  --kubeconfig kubeconfig
```

## Deploy with Istio Traffic Management

### Full Deployment Command
```bash
# Deploy with Istio enabled, canary release, and monitoring
helm upgrade --install sms-checker ./sms-checker-helm-chart \
  -f sms-checker-helm-chart/values.yaml \
  --set istio.enabled=true \
  --set istio.host=sms-istio.local \
  --set istio.canary.enabled=true \
  --set istio.canary.weight=10 \
  --set istio.stickySession.enabled=true \
  --set prometheus.enabled=true \
  --kubeconfig kubeconfig
```

### Add Host Entry
```bash
# Point the Istio host to the ingress gateway IP
echo "192.168.56.91 sms-istio.local" | sudo tee -a /etc/hosts
```

## Istio Resources Created

The Helm chart creates the following Istio resources when `istio.enabled=true`.
The chart deploys three istio objects to be precise, to expose throught the provisioned 
istio ingressgateway.

### 1. Gateway (`app-gateway`)

It works as a front door for incoming http traffic. It tells the Istio ingress gateway to accept requests for sms-istio.local on port 80 and pass them into the mesh.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: app-gateway
spec:
  selector:
    istio: ingressgateway  # Configurable via values.yaml
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "sms-istio.local"
```

### 2. VirtualService (`app-vs`)


The virtualservice contains routing rules that supports two following ways of reaching the canary:
- **force canary (for testing)**: Requests with `x-canary: true` header go to canary 1005.
- **normal traffic split**: All requests follow the default traffic split (90% stable, 10% canary)

This matches the requirement to demonstrate small percentage of canary release and allowing special requests using special headers.

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: app-vs
spec:
  hosts: ["sms-istio.local"]
  gateways: ["app-gateway"]
  http:
    # Explicit canary testing via header
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: app
            subset: canary
          weight: 100
    # Default weighted routing
    - route:
        - destination:
            host: app
            subset: stable
          weight: 90
        - destination:
            host: app
            subset: canary
          weight: 10
```

### 3. DestinationRule (`app-destinationrule`)

It defines two subsets that istio can route to:

**stable** -> pods labelled stable version
**canary** -> pods labelled canary version

and also enables sticky sessions using http cookie ( so that the same user keeps hitting the same subset)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: app-destinationrule
spec:
  host: app
  trafficPolicy:
    loadBalancer:
      consistentHash:
        httpCookie:
          name: sms-session  # Sticky session cookie
          ttl: 3600s
  subsets:
    - name: stable
      labels:
        version: stable
    - name: canary
      labels:
        version: canary
```

## Sticky Sessions

Sticky sessions ensure that once a user is routed to a specific version (stable or canary), they continue to see that version on subsequent requests. This is implemented using consistent hashing with an HTTP cookie.

### How It Works
1. On first request, Istio routes based on the configured weights (90/10)
2. Istio sets a cookie (`sms-session`) that identifies the selected subset
3. Subsequent requests from the same user (with the cookie) are routed to the same subset
4. Cookie TTL is 3600 seconds (1 hour) by default but we can change this.

### Configuration
```yaml
istio:
  stickySession:
    enabled: true
    cookieName: "sms-session"
    ttl: 3600s
```

## Testing the Canary release

### Verify Istio Resources
```bash
# Check Gateway, VirtualService, and DestinationRule
KUBECONFIG=./kubeconfig kubectl get gateway,virtualservice,destinationrule

# View detailed configuration
KUBECONFIG=./kubeconfig kubectl get virtualservice app-vs -o yaml
KUBECONFIG=./kubeconfig kubectl get destinationrule app-destinationrule -o yaml
```

### Verify Deployments
```bash
# Check both stable and canary deployments are running
KUBECONFIG=./kubeconfig kubectl get pods -l app=app --show-labels

# Expected output shows pods with version=stable and version=canary labels
```

### Test Traffic Routing

#### Before testing, make sure you can access the cluster.

 Verify nodes are ready
KUBECONFIG=./kubeconfig kubectl get nodes

# Verify pods are running
KUBECONFIG=./kubeconfig kubectl get pods

# Verify Istio resources exist
KUBECONFIG=./kubeconfig kubectl get gateway,virtualservice,destinationrule


#### 1. Normal Request, this demonstrates that the app is accessible through the Istio gateway and virtual services
```bash
# Multiple requests will be distributed ~90% stable, ~10% canary
# First request sets the sticky session cookie
curl -v -H "Host: sms-istio.local" http://192.168.56.91/sms/

# It should return the http health and show that you are connected to the local app through Istio.
```

#### 2. Force Canary via Header
```bash
# Always routes to canary version
curl -v -H "Host: sms-istio.local" -H "x-canary: true" http://192.168.56.91/sms/
```

#### 3. Test Sticky Sessions
```bash
# First request - SetCookie header in response
curl -v -c cookies.txt -H "Host: sms-istio.local" http://192.168.56.91/sms/

# Subsequent requests with cookie - should go to same version
curl -v -b cookies.txt -H "Host: sms-istio.local" http://192.168.56.91/sms/
```
# > Cookie: sms-session="5cc84964593959a8"	Cookie is being sent with the request
< HTTP/1.1 200 OK	Request succeeded
< server: istio-envoy	Traffic routed through Istio proxy

Above is the example of sticky session working correctly


#### 4. Verify Version in Response
The app includes an `APP_VERSION` environment variable that can help identify which deployment served the request. Check application logs:
```bash
# View logs from stable deployment
KUBECONFIG=./kubeconfig kubectl logs -l app=app,version=stable

# View logs from canary deployment  
KUBECONFIG=./kubeconfig kubectl logs -l app=app,version=canary
```

## How we control the canary rollout

### Image Tags (two versions running)
The helm chart can deploy stable deployment using 0.0.1, and canary deployment using 0.0.2.

```yaml
app:
  tag: "0.0.1"       # Stable version
  canaryTag: "0.0.2" # Canary version 
```

### Traffic Weight
Adjust the percentage of traffic going to canary:
```yaml
istio:
  canary:
    enabled: true
    weight: 10  # 10% to canary, 90% to stable
```

### Progressive Rollout Example (how it can be gradually increased)
```bash
# Start with 5% canary
helm upgrade sms-checker ./sms-checker-helm-chart --set istio.canary.weight=5 --kubeconfig kubeconfig

# Monitor and increase to 25%
helm upgrade sms-checker ./sms-checker-helm-chart --set istio.canary.weight=25 --kubeconfig kubeconfig

# Full rollout  
helm upgrade sms-checker ./sms-checker-helm-chart --set app.tag=0.0.2 --set istio.canary.enabled=false --kubeconfig kubeconfig
```
