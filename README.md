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

```
## Inventory generation
- Our Vagrantfile generates a valid `inventory.cfg` for Ansible that contains all (and only) active nodes. 
- To avoid potential race conditions, first create VMs as stated in the previous step, and then run provisioning. 
- Inspect the generated inventory (`inventory.cfg`) by running:
```bash
vagrant provision
```
## Testing inventory generation
- To test `inventory.cfg` contains only active nodes:
  - halt one of the nodes, for instance, in the case that number of workers are adjusted to 3 in the Preparation step (e.g.,`NUM_WORKERS=3`), run:

```bash
vagrant halt node-3
vagrant provision
cat inventory.cfg
```
- node-3 should not be visible under [workers].
- Afterwards:

```bash
vagrant up node-3
vagrant provision
cat inventory.cfg
```
- node-3 should now be visible under [workers].

The folowing commands can be used to rerun the playbooks manually. 

```bash 
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

## Displaying Grafana Dashboards
Since a ConfigMap is provided (sms-checker-helm-chart/templates/grafana-a3.yaml), there is no need for the manual installation. 

There are two separate dashboards, one for A3 and one for the decision process for A4. Their names in Grafana Dashboard list are: A3 Dashboard 1, and A4 Supporting the Decision Process. Once you navigate to the Dashboards in Grafana they should be at the top of the list. 

Open http://grafana.local in your browser.

Go to the Dashboards from the side menu. And you chould be able to see our two dashboards at the top. (A3 Dashboard 1, and A4 Supporting the Decision Process)

## How to clean up
```bash
vagrant destroy
```

## Email Alerting using Prometheus

### Installation Pre-requisites
- Minikube (running with application deployed)
- Helm
- istioctl (install istio to your cluster if not installed and enable sidecar injection)
    ```yaml
    istioctl install --set profile=demo -y
    kubectl label namespace default istio-injection=enabled
    ```
- Docker (Minikube driver)

### Alert Details
#### HighRequestRate

**Condition:** More than 15 requests per minute, aggregated across all pods

**Expression:** ``` rate(<request_counter_metric>[1m]) * 60 > 15 ```

**Duration:** Must hold for 2 consecutive minutes

**Severity:** warning

**Notification:** Email (with resolve notification enabled)

### Install/Upgrade the Helm Chart
```yaml
helm install sms-checker . \
  --set alertmanager.smtp.user="YOUR_GMAIL@gmail.com" \
  --set alertmanager.smtp.password="YOUR_APP_PASSWORD" \
  --set alertmanager.recipient="YOUR_EMAIL@example.com"
```
run this command from the `sms-checker-helm-chart` folder. Replace YOUR_EMAIL@example.com with your actual email address.

**Note:** Gmail requires an App Password, not your normal account password.

### Verify Setup
1. Check pods: Ensure the following pods are running:
    - Prometheus
    - AlertManager
    - Grafana
    - Application pods (app & model-service)
    ```yaml
    kubectl get pods
    ```

2. Verify existence of alert rule in kubernetes; you're expected to see `high-traffic-alert` rule in the list of rules.
    ```yaml
    kubectl get prometheusrules
    ```

3. Verify Alertmanager Config Mount: Look for the following file in the directory: alertmanager.yaml.gz
    ```yaml
    kubectl exec -it alertmanager-<pod-name> -- ls /etc/alertmanager/config
    ```

4. Verify Alert Rule in Prometheus: 
    ```yaml
    kubectl port-forward svc/prometheus-operated 9090:9090
    ```
    Open: [http://localhost:9090](http://localhost:9090)\
    Navigate to Status -> Alerts, you should see:
      ```bash
      traffic-alerts
      └── HighRequestRate
      ```
      The initial state should be **INACTIVE** (green).
      ![alt text](docs\sc_traffic_alert.png)

### Testing the alert end-to-end
1. View the Rule in Prometheus by port-forwarding:
    ```yaml
    kubectl port-forward svc/prometheus-operated 9090:9090
    ```
    Now you can open prometheus on [http://localhost:9090](http://localhost:9090). Verify under Status -> Rule Health to see `high traffic alert` rule with the status as OK. Additionally under Alerts, you can find the rule with status INACTIVE.

    Keep this terminal running.

2. In a separate terminal window, to view the metrics and dashboard in Grafana, use a different port to port-forward to:
    ```yaml
    kubectl port-forward svc/sms-checker-grafana 3000:80
    ```
    Go to [http://localhost:3000](http://localhost:3000) and use username: `admin` and password: `admin` to login. Open dashboard -> Application Metrics.

3. Generate traffic to observe metrics & trigger alert. Port-forward istio ingress: 
    ```yaml
    kubectl port-forward -n istio-system svc/istio-ingressgateway 8080:80
    ```
    Keep this terminal running.

    Open a new terminal and generate high traffic for about 3-4 minutes:
    ```bash
    while true; do
      curl -X POST http://localhost:8080/sms/ \
        -H "Host: sms-istio.local" \
        -H "Content-Type: application/json" \
        -d '{"sms":"alert test"}' >/dev/null
      sleep 0.05
    done
    ```

4. Observe Alert Lifecycle: In Prometheus (Alerts page);
  
    | Time    | State   |
    | ------- | ------- |
    | < 2 min | Pending |
    | ≥ 2 min | Firing  |

5. Verify Alertmanager: 
    ```yaml
    kubectl port-forward svc/alertmanager-operated 9093:9093
    ```
    Open: [http://localhost:9093](http://localhost:9093)\
    You should see **HighRequestRate** in Firing state.

6. Verify Email Delivery
    - Alert email is sent when the alert fires
    - A resolve email is sent after traffic stops

    Check spam folder if not visible.



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

## Additional Use Case
### Rate Limiting

```yaml
name: envoy.filters.http.local_ratelimit
 typed_config:
  "@type": type.googleapis.com/udpa.type.v1.TypedStruct
  type_url: type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
  value:
    stat_prefix: http_local_rate_limiter
    # token_bucket: This sections has 3 attributes
    token_bucket:
      # max_tokens: Specifies the maximum number of tokens in the bucket, 
      # representing the maximum allowed requests within a certain time frame.
      max_tokens: {{ .Values.istio.rateLimiting.burstSize }}
      # tokens_per_fill: Indicates the number of tokens added to the bucket with each fill, 
      # essentially the rate at which tokens are replenished.
      tokens_per_fill: {{ .Values.istio.rateLimiting.tokensPerFill }}
      # fill_interval: Defines the time interval at which the bucket is replenished with tokens.
      fill_interval: {{ .Values.istio.rateLimiting.fillIntervalSeconds }}s
```

This configuration allows a burst that is up to 5 requests, and it refills 1 token every 60 seconds.  

When there are more requests, the receive HTTP 429 Too Many Requests. 

We also implemented a custom response header for visibility purposes.

```yaml
response_headers_to_add: 
  - append: false
    header:
      key: {{ quote .Values.istio.rateLimiting.headerName }}
      value: {{ quote .Values.istio.rateLimiting.headerValue }}
```
Also, our rate limiting is applied to both app and model-service by attaching the filter to the Istio ingress gateway. 

```yaml
workloadSelector:
  labels: 
    istio: {{ .Values.istio.ingressGateway.selector }}
```

### Check if everything is running
```bash
KUBECONFIG=./kubeconfig kubectl -n istio-system get envoyfilter
```

You should see "local-rate-limit" in the list. 

### Make Istio host resolve to the Istio gateway IP

```bash
echo "192.168.56.91 sms-istio.local" | sudo tee -a /etc/hosts
```

### Testing Rate Limiting

User-based rate-limiting is implemented, the users that we know have the user IDs: 001, 002, and 003. 

In the case that the user ID does not match, we fall back to global rate-limiting. 

For demonstration purposes, we have kept the "tokens_per_fill" relatively low. 

```bash
echo "waiting just in case"
sleep 65

echo User 1 with user ID: 001
for i in {1..5}; do
  code=$(curl -s -o /dev/null -w "001 $i %{http_code}\n" \
  -H "Host: sms-istio.local" \
  -H "x-instance-id: 001" \
  http://sms-istio.local/sms/)
  echo "Request $i: $code"
done

echo "waiting for refill"
sleep 65

echo User 2 with user ID: 002
for i in {1..7}; do
  code=$(curl -s -o /dev/null -w "002 $i %{http_code}\n" \
  -H "Host: sms-istio.local" \
  -H "x-instance-id: 002" \
  http://sms-istio.local/sms/)
  echo "Request $i: $code"
done

echo "waiting for refill"
sleep 65

echo User 3 with user ID: 003
for i in {1..9}; do
  code=$(curl -s -o /dev/null -w "003 $i %{http_code}\n" \
  -H "Host: sms-istio.local" \
  -H "x-instance-id: 003" \
  http://sms-istio.local/sms/)
  echo "Request $i: $code"
done

echo "waiting for refill"
sleep 65

echo Unknown user with user ID: 777
for i in {1..15}; do
  code=$(curl -s -o /dev/null -w "777 $i %{http_code}\n" \
  -H "Host: sms-istio.local" \
  -H "x-instance-id: 777" \
  http://sms-istio.local/sms/)
  echo "Request $i: $code"
done

echo "waiting for refill"
sleep 65

echo User with no header
for i in {1..15}; do
  code=$(curl -s -o /dev/null -w "NOHEADER $i %{http_code}\n" \
  -H "Host: sms-istio.local" \
  http://sms-istio.local/sms/)
  echo "Request $i: $code"
done

```
For user 1:
- You should be seeing HTTP 200 for the first ~3 requests
- Afterwards, you should be seeing HTTP 429

For user 2:
- You should be seeing HTTP 200 for the first ~5 requests
- Afterwards, you should be seeing HTTP 429

For user 3:
- You should be seeing HTTP 200 for the first ~7 requests
- Afterwards, you should be seeing HTTP 429

For unknown user and no header:
- You should be seeing HTTP 200 until the global limit is reached
- Afterwards, you should start seeing HTTP 429

It is expected that you will start seeing HTTP 200 after some HTTP 429 responses. The fill interval is 60s, meaning that every minute, the specified amount of tokens (tokens per fill) is added to the token bucket. Since we are sending low numbers of requests with the first three users, the 60s limit is not reached. However, as you can see in the last two users, since we send more requests and more time passes, we start seeing HTTP 200 responses as the tokens are getting refilled. 


In order to inspect,
```bash
curl -i http://sms-istio.local/sms/
```

That should give you:
```http
HTTP/1.1 429 Too Many Requests
x-local-rate-limit: TOO_MANY_REQUESTS
```

