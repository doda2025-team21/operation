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
- Vagrant builds one `ctrl` and `NUM_WORKERS` `node-*` hosts on a host-only network (192.168.56.100/101/...); CPU/memory/worker count come from `.env` (`NUM_WORKERS`, `CTRL_CPUS`, `CTRL_MEMORY`, `NODE_CPUS`, `NODE_MEMORY`).
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

## How to clean up
```bash
vagrant destroy
```


## Displaying Grafana Dashboards
Since a ConfigMap is provided (sms-checker-helm-chart/templates/grafana-a3.yaml), there is no need for the manual installation. 

Currently, since we have unresolved issues with the "Enable Monitoring" task, I created placeholder custom metrics, this file will be deleted in the end. You can check the file "sms-checker-helm-chart/templates/placeholder-metrics.yaml" for the implementation. 

Also, currently, it is normal that we display incorrect custom metrics, once the previous step is completed, it should successfully display everything. (our actual metrcis will be put instead of the placeholder metrics)

There are two separate dashboards, one for A3 and one for the decision process for A4. Their names in Grafana Dashboard list are: A3 Dashboard 1, and A4 Supporting the Decision Process. Once you navigate to the Dashboards in Grafana they should be at the top of the list. 

The steps to automatically display these dashboards are as follows:

(1) go into the sms-checker-helm-chart directory
(2) check if you have previous releases
```bash
helm list -A
```
(3) if you do, remove the previous releases
```bash
helm uninstall <release-name> || true
```
(4) start your minikube
```bash
minikube start
```
(5) Add repo
```bash
helm repo add prom-repo https://prometheus-community.github.io/helm-charts
helm repo update
```
(6) Install Prometheus and Grafana (yes it is prometheu, not prometheus)
```bash
helm upgrade --install sms-checker-kube-prometheu prom-repo/kube-prometheus-stack

kubectl get pods
```

(7) Install the sms-checker chart
```bash
helm upgrade --install sms-checker . 

kubectl get pods
kubectl get svc my-service
kubectl get servicemonitors
```

(8) If you want to check if Prometheus is scraping the placeholder metrics, you can run the following, but since the previous step is not fully implemented, currently, you will not see much
```bash
minikube service sms-checker-kube-prometheu-prometheus --url
```

(9) Find the password for Grafana
```bash
kubectl get secret --namespace default -l app.kubernetes.io/component=admin-secret \
-o jsonpath="{.items[0].data.admin-password}" | base64 --decode ; echo
```

(10) Now, run the following and go to the url that is generated, the name is: admin, and the password is the password generated by the previous command
```bash
minikube service sms-checker-kube-prometheu-grafana --url
```

(11) Go to the Dashboards from the side menu. And you chould be able to see our two dashboards at the top. (A3 Dashboard 1, and A4 Supporting the Decision Process)




