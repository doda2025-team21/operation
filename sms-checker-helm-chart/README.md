# sms-checker Helm chart

## What it does
Deploys the app and model services with a single Helm release. Assumes the cluster already has:
- An ingress controller (default class name `nginx`, configurable).
- External IP for the ingress service (e.g., MetalLB in Vagrant, `minikube tunnel` in Minikube).

## Install
```bash
helm upgrade --install sms-checker ./sms-checker-helm-chart \
    -f sms-checker-helm-chart/values.yaml \
    --set app.ingress.hosts.stable=sms.local \
    --set app.ingress.hosts.preview=sms-preview.local \
    --set app.ingress.className=nginx \
    --kubeconfig kubeconfig
```

## Point hosts to your ingress IP
- Vagrant + MetalLB: `echo "192.168.56.90 sms.local sms-preview.local" | sudo tee -a /etc/hosts`
- Minikube: `minikube tunnel` then take the ingress controller EXTERNAL-IP and add the same `/etc/hosts` entries.

If you do not need a preview host, set `app.ingress.hosts.preview=""` (the extra rule will be omitted).
