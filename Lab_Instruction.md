# Chaos Engineering: Resilience Testing with Chaos Mesh
## AGH Large Scale Computing Project
### Authors: Tomasz Krępa, Michał Zienkowicz

## Introduction:
This instruction provides materials necessary for experimenting with Chaos Engineering and Resilience Testing.
The project includes deploying a multi-servie application on Kubernetes (Kind) and instrumenting it with basic health metrics. Then systematic failure injection is demonstrated (pod kills, network partitions, latency injections and CPU/memory stress). This excersise, allows you to see how each failure manifests in the metrics and userview, and how the system recovers.

## Tools used:
- Grafana
- Prometheus
- Google Online Boutique - Microservices demo
- Chaos Mesh
- Kind
- Helm
- Locust

# Set-up Instruction:

## 1. Docker Desktop (& WSL)
Firs, we recommend using Docker Desktop, which you can install via following link:
[https://docs.docker.com/desktop/](https://docs.docker.com/desktop/)

Most of tools in this excersise are designed for Linux OS. If you work on machine with Windows, we recommend installing Linux via WSL. In our case we have used [Linux Mint](https://linuxmint.com/), but most of distributions are expected to be compatible:
[https://learn.microsoft.com/en-us/windows/wsl/install](https://learn.microsoft.com/en-us/windows/wsl/install)

### Providing WSL Resources:
You might need to ensure enough RAM and CPU cores for your Docker Desktop Container.
You can do this by modyfying WSL config file. In the **PowerShell**, prompt:

`notepad "$env:USERPROFILE\.wslconfig"` 

Allow for creating new file if asked.

Inside the .wslconfig file insert example settings:
``` TOML
[wsl2]
memory=8GB
processors=4
swap=2GB
```
Then save the file, and close it. Shut down Linux environment (if you had it already running) via **PowerShell**:

``` PowerShell
wsl --shutdown
```
. Restart the Docker Desktop app if required.

If you are using Linux via WSL, make sure to enable WSL integration in Docker Desktop by going to `Settings` -> `Resources` -> `WSL Integration` and enabling the integration.

![alt text](Screenshots\image-2.png)

You can check your setup by running `docker --version` in the Bash (or WSL terminal):
![alt text](Screenshots\image-3.png)

## 2. kubectl 
You can install [kubectl](https://kubernetes.io/docs/reference/kubectl/) via bash commands:

```Bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```
and check the installation using:

```Bash
kubectl version --client
```

![alt text](Screenshots\image-4.png)

## 3. kind
Install [kind](https://kind.sigs.k8s.io/docs/user/quick-start/) using something like:

```Bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind --version
```

![alt text](Screenshots\image-5.png)

## 4. Helm
We recommend installing [Helm](https://helm.sh/docs/intro/install/) using instructions provided on its website:

```Bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
helm version
```

![alt text](Screenshots\image-6.png)

## 5. Creating your cluster  

After succesfull installations, you should be able to create your cluster, which will be your playground for the rest of this excersise. In our case, we named our group of nodes `chaos-lab`. Use provided `kind-config.yaml` file, to create cluster containing 3 nodes (control-plane and two workers).

```Bash
kind create cluster --name chaos-lab --config kind-config.yaml
```

![alt text](Screenshots\image.png)

You should see the cluster running in your Docker Desktop panel: 

![alt text](Screenshots\image-7.png)

*If during your experiments the system will get too damaged (stop responding), you can always delete it using* `delete cluster --name chaos-lab`, *and create a new one.*

## 6. Google Online Boutique

Download Google Online Boutique via GitHub:

```Bash
git clone https://github.com/GoogleCloudPlatform/microservices-demo.git
cd microservices-demo
```

Run the app on the new cluster:

```Bash
kubectl apply -f ./release/kubernetes-manifests.yaml
```

You can spectate the application building process via

```Bash
kubectl get pods --watch
```

You might need to wait up to 10 minutes for all the microservices to have **`Running`** and **`READY`**status.

![alt text](Screenshots\image-8.png)

Then set up port forwarding for the frontend of the store.

```Bash
kubectl port-forward service/frontend-external 8080:80
```

You can then use [https://localhost:8080](https://localhost:8080) to access the site.

## 7. Prometheus and Grafana
We will use Helm to add [Prometheus](https://prometheus.io/) to our application:

```Bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts`
helm repo update
```

![alt text](Screenshots\image-9.png)

We advise creating separate namespace (we named it `monitoring`) and installling the configured Prometheus and [Grafana](https://grafana.com/) in the `kube-prometheus-stack` provided by the distributors. In case of 

```Bash
kubectl create namespace monitoring
helm install monitoring-stack prometheus-community/kube-prometheus-stack --namespace monitoring
```

After succesfull you can check if the pods are running with 

```Bash
kubectl get pods -n monitoring
```

Now, you can start working with Grafana. To access this tool by the browser, we need to first expose the port.

```Bash
kubectl port-forward -n monitoring svc/monitoring-stack-grafana 3000:80
```

After running the command, you can open the bowser (Chrome, Edge, Firefox) and acces your port.

`http://localhost:3000`

You should be able to login using default Helm login username:

* Username: `admin`

Password is accessible via the command, that **you should run in the SEPARATE WSL TERMINAL**

```Bash
kubectl get secret --namespace monitoring monitoring-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

After logging into Grafana, you can access the predefined graphs for Kubernetes. Go to `Dashboards` -> `Kubernetes / Compute Resources / Pod` to see example graphs. Leave `Namespace` as 'default' - this is, where our Boutique is running.

![alt text](Screenshots\image-11.png)
![alt text](Screenshots\image-10.png)

To better see how our interactions with the chaos mesh will affect the cluster we need to increase the scraping frequency.

```Bash
helm upgrade monitoring-stack prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f prometheus-fast-scrape.yaml \
  --reuse-values
```

## 8. Chaos Mesh

To install and use Chaos Mesh we need to add its repository into Helm.

```Bash
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm repo update
```

Then install it to our cluster

```Bash
helm install chaos-mesh chaos-mesh/chaos-mesh \
  -n chaos-mesh --create-namespace \
  --set chaosDaemon.runtime=containerd \
  --set chaosDaemon.socketPath=/run/containerd/containerd.sock \
  --set dashboard.securityMode=false
```

If you want to access the Chaos Mesh dashboard you need to start port forwarding

```Bash
kubectl port-forward -n chaos-mesh svc/chaos-dashboard 2333:2333
```

You can then use [https://localhost:2333](https://localhost:2333) to access the site.

If you run into problems and the pods of Chaos Mesh are not starting you can try increasing open file limits
```Bash
sudo sysctl fs.inotify.max_user_watches=524288
sudo sysctl fs.inotify.max_user_instances=512
```

## 9. Locust

Additionaly, a load generator will prove useful in measuring the effects of the chaos mesh. That's why we use Locust - a simple GUI load generator. To add it to your system use the prepared locust-loadgenerator.yaml file.

```Bash
kubectl apply -f locust-loadgenerator.yaml
```

Wait for the pod to start running

```Bash
kubectl get pods -l app=loadgenerator
```

Then start port forwarding to access the GUI at [https://localhost:8089](https://localhost:8089).

```Bash
kubectl port-forward svc/loadgenerator 8089:8089
```

# Tasks

Formulate a report with answers to questions stated in tasks below.

## Silence before the storm

Go to the boutique site and check available functionalities. Try doing a full buisness path of transaction: from browsing the catalogue to submitting an order.

## Task 1 - Stress test

Open the Grafana dashboard on the checkout pod. Then start the prepared CPU and memory stress test.

```Bash
kubectl replace --force -f stress-test.yaml
# this command will also restart the test if you've already done it
```

Monitor the CPU Throttling and Memory usage graphs. How does the stress test affect those statistics?

When you notice a change, go back to the boutique and try placing an order. Did the order go through? Were there any problems along the way? Include screenshots of any messages recieved during the transactions.

Go back to the Grafana dashboard. Did the CPU Throttling and Memory usage graphs go back to normal? What does it say about the recoverability of the whole system? Include screenshots of the graphs with the incident visible in your report.

## Task 2 - Pod kill 

In this task you will be required to kill pods responsible for two different services. Before starting navigate to a product site (ex. Tank top). Open and inspect the prepared pod-kill-test.yaml file. What service should be down after starting the chaos incident?

Then proceed to initiating the scheduler:

```Bash
kubectl replace --force -f pod-kill-test.yaml
```

Refresh the product page. Did anything change? Has the site remained the same? Write your observations in the report.

Remove the scheduler:

```Bash
kubectl delete -f pod-kill-test.yaml
```

Open and edit the pod-kill-test.yaml file. Replace `recommendationservice` with `productcatalogservice`.

Run the test again and refresh the page. Is the site working? Can you browse it as usual? Write your observations and compare the results of both experiments. Comment on the implied design of the site and which services are required for the site to function.

Remember to remove the scheduler after the tests:

```Bash
kubectl delete -f pod-kill-test.yaml
```

## Task 3 - Network Chaos

In this task we will introduce network latancy to the system and see how it affects the functioning of it.

Before starting the chaos incident navigate to the Locust GUI and start the load generator at 50 maximum users.

Use the command below to find the name of the product catalog service pod and navigate to it in the Grafana dashboard. Leave the output of the command running in the terminal to oversee the pod behaviour.

```Bash
kubectl get pods --watch
```

Open the boutique site on your own as well. Is everything working properly?

Now start the latency injection using the prepared network-latency-test.yaml file.

```Bash
kubectl apply -f network-latency-test.yaml
```

Refresh the page. Do you notice any latencies?

Go to Locust load generator. How large are the latencies for the users overall?

Inspect the Graphana diagrams. Can you see the influence of the latency and users on the metrics?

Now let the latency run out (it should last about a minute) or force remove it using 

```Bash
kubectl delete networkchaos catalog-latency
```

Return to the Graphana dashboard. Are the metrics going back to normal?

Access the boutique site. Is the site back to being usable?

Go to Locust load generator. How are the statistics looking at this moment?

Fill your report and add screenshots with metric diagrams. Comment on how this experiment showed recovery abilities of the system

## Bonus task

Play around using prepared files and attack different services and pods set up in the system. Increase the user load and check how the system behaves and recovers.






