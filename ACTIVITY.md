# Week 1 (Nov 15+)
- Guotao Gou: I have worked on [F1 and F2](https://github.com/doda2025-team21/lib-version/pull/1) of A1 assignment. Finished lib-version and publish its Maven package to github.
I have worked on [F11](https://github.com/doda2025-team21/lib-version/pull/3), and [add gitbot for automatic tagging and version bumping](https://github.com/doda2025-team21/lib-version/pull/4)

- Dibyendu Gupta: 
    - Worked on F7 and F8
    - F7: Created `docker-compose` file to successfully start the entire app, more information about this feature can be found in [PR for docker-compose](https://github.com/doda2025-team21/operation/pull/1) and [PR for README file](https://github.com/doda2025-team21/operation/pull/2).
    - F8: Made workflows for `model-service` and `app` repositories that automatically version and release images to ghcr registry. The triggers for these need to be adjusted (currently on git tags and manual trigger from github). More infromation can be found from these PRs: [PR for app](https://github.com/doda2025-team21/app/pull/4/) and [PR for model-service](https://github.com/doda2025-team21/model-service/pull/7).
    - Aided the team with uploading all the necessary repositories to app & model-service correctly
    - performed initial commits for Dockerfiles in app and model-service repos (for other team members to start working on their respective features)
    - setting up some branching rules: no pushing directly to main, PRs require atleast 1 approver.

- Madhav Chawla: 
For F9 I Created `.github/workflows/train_model.yml` that works on version tags
also I Automatically allow trains model and uploads `.joblib` artifacts to GitHub releases for public download, for F10
I updated `serve_model.py` to automatically download models from GitHub releases on startup if not present locally
and I updated `Dockerfile` (multi-stage build) and `.dockerignore` to exclude model files from image
and lastly it supports both volume mounts for custom models and automatic downloads as fallback.

- Ashraf El Attachi: I mostly spent my time working on F4; multi-architecture images. For this I created a yml file, which creates a github workflow that creates releases viable for multiple architectures.

- Ceylin Ece: I worked on F5 and F6. For F5, I implemented multi-stage builds on the Dockerfile of the 'model-service' repository. The final image size decreased from 933 MB to 659 MB. For F6, I made ports and URLs configurable via ENV variables with default values. 

# Week 2 (Nov 24+)
- Guotao Gou, [finished step 1 to 4](https://github.com/doda2025-team21/operation/pull/8).
- Dibyendu:
    - Worked on Steps 9-15
    - [PR for this week](https://github.com/doda2025-team21/operation/pull/13): All the steps were included into this 1 PR (I acknowledge that this isn't the best software development practice, but it had to be done because the person who was supposed to work on steps 9-12 didn't complete the work on time). This causes a cascade of dependencies on these tasks which needed to be implemented before the other ones could be attempted and tested for correctness.
    - Install container runtime (containerd), Kubernetes components (kubelet, kubeadm, kubectl), Helm, and configure containerd with systemd cgroups. (Steps 9-12 & 16)
    - On the controller (ctrl), run kubeadm init with the correct pod CIDR, then set up the kubeconfig so kubectl works. (Steps 13-14)
    - Apply the Flannel manifest so nodes can communicate (kubectl apply -f kube-flannel.yml), enabling networking across the cluster. (Step 15)


- Madhav Chawla: For Steps 21–23, I set up the cluster’s networking and management layers by installing the Nginx Ingress Controller via Helm and assigning it a static MetalLB IP for external routing (Step 21), then deployed the Kubernetes Dashboard using Helm, created an admin ServiceAccount with full cluster access, and exposed the dashboard through an Ingress rule (Step 22). Finally, I installed Istio by copying the Istio distribution into the controller VM, installing istioctl, allowing Istio with a custom operator configuration that assigns the Istio ingress gateway its own static MetalLB IP (Step 23).

- Tae Yong Kwon: I prepared the automation to finish the Kubernetes cluster by adding Helm support, automating worker node joining, and installing MetalLB. For Helm, I added tasks to install the helmdiff plugin. For worker nodes, I used Ansible to generate a single join command on the controller and ensured each worker only joins if it hasn't already, keeping the process idempotent. Finally, I created a separate finalization playbook that installs MetalLB, sets up an IP pool (192.168.56.90–99), configures an L2Advertisementand verifies the deployment once the cluster is fully running (task 17-20)

- Ceylin Ece: I implemented the steps 5-8. For step 5, I used the 'ansible.builtin.shell' module to disable SWAP on the running system. I also removed the SWAP entry from /etc/fstab using the 'ansible.builtin.lineinfile' module. For step 6, I used the 'ansible.builtin.copy' module to add the two modules that are 'overlay' and 'br_netfilter'. Using the 'community.general.modprobe' module, I loaded 'br_netfilter' and 'overlay' in the running system. For step 7, using 'ansible.posix.sysctl', I enabled the properties 'net.ipv4.ip_forward', 'net.bridge.bridge-nf-call-iptables', and 'net.bridge.bridge-nf-call-ip6tables' by setting the value to '1'. For step 8, I implemented the Advanced solution. Please check templates/hosts.j2 in addition to general.yml for this solution. I used the 'ansible.builtin.copy' module and Jinja2 loop for this step. 

# Week 3 (Dec 1+)
- Guotao Gou, I [fixed ingress LoadBalancer problem, finished A2 remaining steps(step 21 to step 23) and add and tested README docs for both helm chart local and VM cluster installation. Add Docs for A2 and A3. Tested helm local installation and VM helml installation. Tested host sms.local, sms-preview.local for local installation and VM installation.](https://github.com/doda2025-team21/operation/pull/18)

- Dibyendu Gupta:
    - Worked on docker migration and creation of helm-charts
    - My contributions for this week is alongside Martin's (Guatao Gou) in the PR that we both contributed for [PR for docker migration & Helm-chart](https://github.com/doda2025-team21/operation/pull/18).
    - I am also currently working on integrating configMaps and secrets as placeholders in helm-chart for an advanced solution. A PR for this branch hasn't been made yet, it should be available sometime this weekend and the activity file will be update!