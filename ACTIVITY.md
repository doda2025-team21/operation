# W2: Nov 17-23
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

- Ceylin Ece: I worked on F5 and F6. For F5, I implemented multi-stage builds on the Dockerfile of the 'model-service' repository. The final image size decreased from 933 MB to 659 MB. For F6, I made ports and URLs configurable via ENV variables with default values. My PR of this week is: https://github.com/doda2025-team21/operation/pull/5 

# W3: Nov 24-30
- Guotao Gou, [finished step 1 to 4](https://github.com/doda2025-team21/operation/pull/8).
- Dibyendu:
    - Worked on Steps 9-15
    - [PR for this week](https://github.com/doda2025-team21/operation/pull/13): All the steps were included into this 1 PR (I acknowledge that this isn't the best software development practice, but it had to be done because the person who was supposed to work on steps 9-12 didn't complete the work on time). This causes a cascade of dependencies on these tasks which needed to be implemented before the other ones could be attempted and tested for correctness.
    - Install container runtime (containerd), Kubernetes components (kubelet, kubeadm, kubectl), Helm, and configure containerd with systemd cgroups. (Steps 9-12 & 16)
    - On the controller (ctrl), run kubeadm init with the correct pod CIDR, then set up the kubeconfig so kubectl works. (Steps 13-14)
    - Apply the Flannel manifest so nodes can communicate (kubectl apply -f kube-flannel.yml), enabling networking across the cluster. (Step 15)


- Madhav Chawla: For Steps 21–23, I set up the cluster’s networking and management layers by installing the Nginx Ingress Controller via Helm and assigning it a static MetalLB IP for external routing (Step 21), then deployed the Kubernetes Dashboard using Helm, created an admin ServiceAccount with full cluster access, and exposed the dashboard through an Ingress rule (Step 22). Finally, I installed Istio by copying the Istio distribution into the controller VM, installing istioctl, allowing Istio with a custom operator configuration that assigns the Istio ingress gateway its own static MetalLB IP (Step 23).

- Tae Yong Kwon: I prepared the automation to finish the Kubernetes cluster by adding Helm support, automating worker node joining, and installing MetalLB. For Helm, I added tasks to install the helmdiff plugin. For worker nodes, I used Ansible to generate a single join command on the controller and ensured each worker only joins if it hasn't already, keeping the process idempotent. Finally, I created a separate finalization playbook that installs MetalLB, sets up an IP pool (192.168.56.90–99), configures an L2Advertisementand verifies the deployment once the cluster is fully running (task 17-20). This corresponds to the W2 assignment.
- PR for this week : https://github.com/doda2025-team21/operation/pull/10

- Ceylin Ece: I implemented the steps 5-8. For step 5, I used the 'ansible.builtin.shell' module to disable SWAP on the running system. I also removed the SWAP entry from /etc/fstab using the 'ansible.builtin.lineinfile' module. For step 6, I used the 'ansible.builtin.copy' module to add the two modules that are 'overlay' and 'br_netfilter'. Using the 'community.general.modprobe' module, I loaded 'br_netfilter' and 'overlay' in the running system. For step 7, using 'ansible.posix.sysctl', I enabled the properties 'net.ipv4.ip_forward', 'net.bridge.bridge-nf-call-iptables', and 'net.bridge.bridge-nf-call-ip6tables' by setting the value to '1'. For step 8, I implemented the Advanced solution. Please check templates/hosts.j2 in addition to general.yml for this solution. I used the 'ansible.builtin.copy' module and Jinja2 loop for this step. PR of this week: https://github.com/doda2025-team21/operation/pull/9  

# W4: Dec 1-7

- Taeyong Kwon:
    - This week, I integrated full Prometheus-based monitoring across the operation, app, and model-service repositories.
    - In the operation repo, I added kube-prometheus-stack as a Helm dependency and created ServiceMonitors for both the app and model-service.
    - I added metrics ports to the relevant Deployments and Services.
    - I configured Prometheus and Grafana ingresses,and fixed the Prometheus ingress service name.
    - I also fixed the ServiceMonitor path for the app, rebuilt and pushed updated Docker images, and forced Kubernetes to pull the latest images(since it was using the local cache and not updating the metrics)
    - In the app repo, I added Micrometer + Prometheus dependencies in pom.xml, implemented custom Counter, Gauge, and Timer metrics in FrontendController, and enabled Spring Actuator on port 9090 to expose /actuator/prometheus.
    - In the model-service repo, I added prometheus_client to requirements.txt, implemented Counter, Gauge, and Histogram metrics in serve_model.py, and exposed /metrics on port 9091.
    - With these changes, I fully instrumented all services and enabled end-to-end scraping and dashboarding via Prometheus and Grafana.
    - This completes the “Monitoring” part of the W3 assignment.

    -My PRs for this week:
    https://github.com/doda2025-team21/operation/pull/20

    https://github.com/doda2025-team21/model-service/pull/8

    https://github.com/doda2025-team21/app/pull/8

- Ceylin Ece:  
    - I implemented the Grafana part of the assignment. Currently, our "enable monitoring" step is not fully completed yet, so we are using placeholder metrics. If my PR is not merged yet, you can switch to the "grafana" branch and run it. I implemented it in a way that it is not required to manually import the dashboard, I provided a ConfigMap, which is "grafana-a3.yaml" under "templates." 
    - You can see the JSON files for our dashboards under the "dashboards" directory. 
    - I implemented various types of visualizations such as: Gauge, Bar Chart, Heatmap, Time series
    - I also created a placeholder yaml file, for now. It will be removed once the previous steps are completed fully. 
    - My PR of this week is: https://github.com/doda2025-team21/operation/pull/19 
    - Please follow the instructions in our README file to run my implementation. 


- Guotao Gou, I [fixed ingress LoadBalancer problem, finished A2 remaining steps(step 21 to step 23) and add and tested README docs for both helm chart local and VM cluster installation. Add Docs for A2 and A3. Tested helm local installation and VM helml installation. Tested host sms.local, sms-preview.local for local installation and VM installation.](https://github.com/doda2025-team21/operation/pull/18)

- Dibyendu Gupta:
    - Worked on docker migration and creation of helm-charts
    - My contributions for this week is alongside Martin's (Guatao Gou) in the PR that we both contributed for [PR for docker migration & Helm-chart](https://github.com/doda2025-team21/operation/pull/18).
    - I am also currently working on integrating configMaps and secrets as placeholders in helm-chart for an advanced solution. A PR for this branch hasn't been made yet, it should be available sometime this weekend and the activity file will be update!


# W5: Dec 8-14

- Ceylin Ece: 
    - I implemented the "Additional Use Case" task this week.
    - I decided to implement rate limiting. 
    - The PR of this week is: https://github.com/doda2025-team21/operation/pull/30 
    - You can find the instructions on how to run/test rate limiting in our README.md file. 
    - For testing purposes I kept the numbers for "max_tokens" and "tokens_per_fill" relatively small. Otherwise, you would need to wait for a long time period to see that it is working. 
    - You can look at "rate-limiting.yaml" and "values.yaml" for my implementation.
    - I also worked on Grafana again, since we had issues with Prometheus last week, I couldn't properly put our custom metrics, now it works fully, and custom dashboards are automatically added. (It is a3's PR, but I am putting the Grafana PR here again, now it is fully functional https://github.com/doda2025-team21/operation/pull/19 )
 
- TaeYong Kwon:
    -  I did the "traffic management" part of the A4 Assignment
    -  I added new helm templates : istio-gateway.yaml for sms-istio local, istio-virtualservice.yaml for canary based routing and istio- destinationrule.yaml for sticky sessions
    -  i also split the app deployment into app-stable and app-canary deployments and did the version label for istio routing
    -  I also fixed problems with monitoring, as some metrics were not working (custom metrics).
    -  Deployed few releases for app / model-service in order to fix problems with prometheus
    -  My pr for this week : https://github.com/doda2025-team21/operation/pull/27
  
- Guotao Gou, 
    1. I finished the Continiouse Experiment part.
        - [Finished the document](https://github.com/doda2025-team21/operation/pull/29) and some [minor update](https://github.com/doda2025-team21/operation/pull/32).
    2. [Designed the new UI for our app frontend](https://github.com/doda2025-team21/app/pull/9) and [debugging them](https://github.com/doda2025-team21/app/pull/10/files) and [rendering debugging](https://github.com/doda2025-team21/app/pull/10) and [container image and workflow debugging](https://github.com/doda2025-team21/app/pull/11)

- Dibyendu Gupta:
    - This week I was responsible for implementing [secrets and configMap](https://github.com/doda2025-team21/operation/pull/28) from the previous assignment A3. 
    - I'm also currently working on "alerting" from the previous assignment (A3) because the person responsible for this has left the course.
    - Had some issues with running and testing the cluster this week and was sick, so didn't work to a 100% capacity this week. 
    - Will complete "alerting" from A3 , and work on some tasks from A4 (additional use case) next week.

# W6: Dec 15-21

- Ceylin Ece:
    - My PR of the week: https://github.com/doda2025-team21/operation/pull/34 (This was an improvement on A2)
    - Our Vagrantfile was not generating 'inventory.cfg.' It was a separate, static file. Therefore, I fixed it. 
    - I made Vagrantfile generate 'inventory.cfg' automatically and made Ansible use it. 
    - Our 'inventory.cfg' contains only active nodes.
 
- Taeyong Kwon
    - My PR of the week: https://github.com/doda2025-team21/operation/pull/33
    - This PR implements canary deployment support for the model service to enable consistent version routing between app and model service deployments (stable-to-stable, canary-to-canary), which is part of the a4 requirement.
    - I added separate stable and canary deployments for the model service with appropriate version labels
    - I introduced Istio VirtualService for model service routing based on source labels
    - I then clarified destinationrule for model service traffic management, because only for app was existing so far.

- Guotao Gou
    - My PR of the week: https://github.com/doda2025-team21/app/pull/13
    - Out previous app docker image will run mvn run, and this process will download all dependency files, which makes it very slow.
    - In my PR, I optimized out app image so it will download the jar file, and just simply run java -jar, so it'll be much faster.
    - I tested and debuged many times for this app image using github action, resolved the github action authentication and ghcr.io maven privilege problem.

- Dibyendu Gupta
    - I was ill during this week and wasn't able to make active contribution through a PR for this week. I had contributions through commits during the week and during christmas break to make up for it. 
    - You can have a look at this PR: [Non-working implementation of alerting](https://github.com/doda2025-team21/operation/pull/39) to view the commit history for active contribution during this week. 


# W7: Jan 5-11
- Ceylin Ece:
    - My PR of the week: https://github.com/doda2025-team21/operation/pull/37 (This is an improvement on A4)
    - Previously, we only had global-rate-limiting. Now, I have also implemented user-based rate-limiting. 
    - For the cases that the request is made with a known user ID, then the user-specific rate-limiting will be applied to that user (Look at our README.md file to test it out.)
    - In the case that there is an unknown user or there are no headers, we fall back to global-rate-limiting.
 
- Taeyong Kwon:
    - My PR of the week: https://github.com/doda2025-team21/operation/pull/33
    - Made readme to verify the istio, and to observe the traffic distribution between stable and canary version

- Dibyendu Gupta:
    - I had no contributions this week.

- Guotao Gou:
    - My PR of the week: https://github.com/doda2025-team21/operation/pull/36
    - Tested our Ansible/Vagrant system and Kubernetes system works correctly with our optimized app image version.

# W8: Jan 12-18
- Guotao Gou:
    - My PR of the week: https://github.com/doda2025-team21/operation/pull/45
    - According to A1 requirement, changed our docker compose model/app version the same with Kubernetes.
    - Tested and make sure docker compose works with new model/app version number.

- Ceylin Ece:
    - My PR of the week: https://github.com/doda2025-team21/operation/pull/44 
    - This is an improvement on A3 Kubernetes Usage
    - Now, all VMs mount the same shared VirtualBox folder as /mnt/shared into the VM.
    - And, the deployed application mounts this path as a hostPath Volume into at least one Deployment (In our case that is the stable version of app). 
    - Created a verification file "shared/a3-kubernetes-proof.txt" to showcase that the same data is visible on multiple VMs, and the deployed app pod can access the shared directory. 
    - Please follow the instructions in README.md to test it out.
 
- Taeyong Kwon:
    - My PR of the week : https://github.com/doda2025-team21/operation/pull/47
    - Built a grafana dashboard to illustrate the difference between deployed version. It tracks the sms request rate between stable and canary app pods, compares the latency, and shows the visual traffic distribution
    - Stable is in blue and canary is in orange
    - Added the grafana dashboard to grafana configmap so that it will get deployed with the helm chart

- Dibyendu Gupta:
    - My PR for this week was on the alerting feature from A3: [Altering through emails](https://github.com/doda2025-team21/operation/pull/40).
    - This feature implements email alerts when the app received more than 2 requests/min for a sustained period of 1 minute. 
    - The implementation required scraping of frontend application metrics, defining a rule in Prometheus, and use AlertManager to enable and configure email-based notifications. The dashboard in Grafana can be used to visualize alerts too.

# W9: Jan 19-25

- Guotao Gou:
    - My PRs of the week: https://github.com/doda2025-team21/operation/pull/49, https://github.com/doda2025-team21/operation/pull/50
    - Finished first part of extension-proposal of A4. Added smoke-test extension proposal and fills in the implementation plan and pipeline graph.
    - Finished deployment.md a1 part.

- Dibyendu Gupta:
    - My PRs for this week:
        - [A4: Additional use case - shadow launch](https://github.com/doda2025-team21/operation/pull/48) 
        - [A4: Extension Proposal about quality checks in ML models](https://github.com/doda2025-team21/operation/pull/54)
        - [A2 (Fix): Ansible provisioning through vagrant](https://github.com/doda2025-team21/operation/pull/55)
    - This PR is for aids real-life distributed systems to evaluate a new version of a service or model under real production traffic without affecting end users.
    - The implementation involved creating a second model-service deployment (only difference is in `version: shadow`), which is then mirrored through an `istio VirtualService` in the traffic management layer, and we can view the traffic flow from the logs of each service. 
    - The extension proposal discusses the shortcomings of our current project with regards to the quality of the model that we deploy, and a proposal that uses release-engineering concepts that were learnt during this course.
    - Ansible provisioning through vagrant was lacking the controller provisioning for steps 20-23. This PR intended on provisioning MetalLB, Istio, Nginx Ingress Controller, and Kubernetes Dashboard for the controller node in the `Vagrantfile`.

- Taeyong Kwon:

  - My Prs of the week: https://github.com/doda2025-team21/operation/pull/52 , doda2025-team21/model-service#10
  - Merged the A4 grafana dashboard with A3 dashboard, while meeting all the requirements
  - Fixed the consistent routing, did the istio canary documentation
  - Model service was failing with resource stopwords not found error, because NLTK data wasnt included in the docker image. Fixed and built a new image

- Ceylin Ece:
    - My PR of the week: https://github.com/doda2025-team21/operation/pull/53 
    - Now our app-canary version is also using ConfigMap and Secrets values (it was only stable before, extended it to canary)
    - Previously, the stable deployment referenced the model service using a fixed service name. (canary was fine, only changed stable)
    - Now, the model service can be relocated just by changing the Kubernetes config.

# W10: Jan 26-27

- Dibyendu Gupta:
    - Working on deployment doc and presentation slides