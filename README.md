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
## TODO, every group member
put your ssh public key into ansible/files/ssh-keys/, so that you can ssh into virtual machines easily.
## How to run
```bash
git pull # Pull latest content

# In operation folder.
vagrant up # If failed, try do it again
```
## Test whether it worked
Make sure you've addedd your ssh public key. Otherwise, you'll need to input username(vagrant) and password(vagrant).
```bash
# Try these commands
ssh vagrant@192.168.56.100
ssh vagrant@192.168.56.101
ssh vagrant@192.168.56.102
```

## Step 21: Install Nginx Ingress Controller
An IngressController exposes cluster services publicly by running a LoadBalancer service that requests a public IP from MetalLB and configures routing based on Ingress resources. Add the Helm repository and install the chart (this will also introduce required CRDs):

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx && helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx
```

To request a specific IP from MetalLB, override controller.service.loadBalancerIP when installing the chart, for example:

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --set controller.service.loadBalancerIP=192.168.56.95
```

If using a values file, set controller.service.loadBalancerIP in your values.yml and install with:

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx -f values.yml
```

## Step 22: Install Kubernetes Dashboard
The Kubernetes Dashboard provides a web UI to explore cluster resources. Install with Helm and create an admin ServiceAccount and ClusterRoleBinding to obtain an admin token for login (creating the token manually via kubectl -n kubernetes-dashboard create token admin-user is acceptable):

```bash
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/ && helm repo update
helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard

kubectl create serviceaccount admin-user -n kubernetes-dashboard
kubectl create clusterrolebinding admin-user \
  --clusterrole=cluster-admin \
  --serviceaccount=kubernetes-dashboard:admin-user
# To fetch token for login (run on controller VM):
# kubectl -n kubernetes-dashboard create token admin-user
```

Create an Ingress to access the dashboard; to allow the dashboard backend to use HTTPS while the external access is HTTP, add the nginx.ingress.kubernetes.io/backend-protocol: "HTTPS" annotation to the Ingress metadata. If full SSL is required, provide HTTPS via an ingress controller that has TLS configured on port 443.

## Step 23: Install Istio
Preparing the cluster for Istio ensures advanced service-mesh features can be used later. The recommended installation steps are similar to the in-class exercise:

1. Download Istio (example: istio-1.25.2) and unpack it:

```bash
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.25.2 sh -
cd istio-1.25.2
```

2. Add istioctl to PATH for the vagrant user (so it can be used directly):

```bash
# move binary into a directory on PATH or add its folder to PATH
sudo mv bin/istioctl /usr/local/bin/ && sudo chown vagrant:vagrant /usr/local/bin/istioctl
```

3. Install Istio using istioctl (optionally provide a config file):

```bash
istioctl install -y
# or with a custom config:
# istioctl install -y -f config.yml
```

This allows customizing components and, similar to ingress installation, you can set which services get loadBalancer IPs or other settings via the Istio install config. 

