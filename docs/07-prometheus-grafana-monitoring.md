# Prometheus and Grafana on Kubernetes — Complete Monitoring Lab

![Kubernetes](https://img.shields.io/badge/Kubernetes-Minikube-326CE5?logo=kubernetes&logoColor=white)
![Prometheus](https://img.shields.io/badge/Monitoring-Prometheus-E6522C?logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Visualization-Grafana-F46800?logo=grafana&logoColor=white)
![Helm](https://img.shields.io/badge/Package%20Manager-Helm-0F1689?logo=helm&logoColor=white)
![AWS](https://img.shields.io/badge/Cloud-AWS%20EC2-FF9900?logo=amazonaws&logoColor=white)

A hands-on Kubernetes monitoring lab demonstrating how to install Prometheus and Grafana on an Ubuntu EC2 instance using Minikube and Helm, access their web interfaces through port forwarding, and explore Kubernetes monitoring dashboards.

## Table of Contents

- [1. Lab Overview](#1-lab-overview)
- [2. Architecture](#2-architecture)
- [3. Environment and Tools](#3-environment-and-tools)
- [4. Docker Installation](#4-docker-installation)
- [5. Install kubectl and Minikube](#5-install-kubectl-and-minikube)
- [6. Install Helm](#6-install-helm)
- [7. Install Prometheus](#7-install-prometheus)
- [8. Prometheus Service and NodePort](#8-prometheus-service-and-nodeport)
- [9. Access Prometheus Using Port Forwarding](#9-access-prometheus-using-port-forwarding)
- [10. Install Grafana](#10-install-grafana)
- [11. Access Grafana Using Port Forwarding](#11-access-grafana-using-port-forwarding)
- [12. Windows-to-EC2 SSH Port Forwarding](#12-windows-to-ec2-ssh-port-forwarding)
- [13. Configure Prometheus as a Grafana Data Source](#13-configure-prometheus-as-a-grafana-data-source)
- [14. Explore Kubernetes Dashboards](#14-explore-kubernetes-dashboards)
- [15. Understanding Port Forwarding](#15-understanding-port-forwarding)
- [16. Troubleshooting](#16-troubleshooting)
- [17. Security Considerations](#17-security-considerations)
- [18. Lab Checklist](#18-lab-checklist)
- [19. References](#19-references)

## 1. Lab Overview

The objective of this lab is to understand how monitoring tools are deployed and accessed in a Kubernetes environment.

We launched an Ubuntu EC2 instance, installed Docker, configured Minikube, installed Helm, and deployed Prometheus and Grafana.

We then used Kubernetes port forwarding and SSH tunneling to access the monitoring interfaces from a Windows laptop.

### What we learned

- Deploying monitoring tools using Helm charts.
- Understanding Kubernetes Pods and Services.
- Understanding `ClusterIP` and `NodePort`.
- Accessing Kubernetes applications through `kubectl port-forward`.
- Accessing remote applications through SSH local port forwarding.
- Configuring Prometheus as a Grafana data source.
- Exploring Kubernetes CPU, memory, network, namespace, and Pod metrics.
- Troubleshooting connectivity issues.

## 2. Architecture

```text
                     WINDOWS LAPTOP
                  +-------------------+
                  | Chrome Browser    |
                  | localhost:9090    |
                  | localhost:3000    |
                  +---------+---------+
                            |
                            | SSH local port forwarding
                            v
                    AWS EC2 INSTANCE
                  +-------------------+
                  | Ubuntu Linux      |
                  | kubectl           |
                  | port-forward      |
                  +---------+---------+
                            |
                            v
                    MINIKUBE CLUSTER
                  +-------------------+
                  | Prometheus        |
                  | Grafana           |
                  | Node Exporter     |
                  | kube-state-metrics|
                  +-------------------+
```

### Request flow

The browser runs on Windows, but the monitoring applications run inside Kubernetes on the EC2 instance.

There are two separate forwarding layers:

1. **Kubernetes to EC2:** `kubectl port-forward` connects a local EC2 port to a Kubernetes Pod or Service.
2. **Windows to EC2:** SSH local port forwarding connects a Windows localhost port to a port on the EC2 instance.

This allows us to access the applications without directly exposing the monitoring web interfaces to the public internet.

## 3. Environment and Tools

| Component | Configuration |
|---|---|
| Cloud provider | AWS |
| Virtual machine | EC2 |
| Operating system | Ubuntu Linux |
| RAM | Approximately 4 GB |
| Storage | 30 GB EBS volume |
| Container runtime | Docker |
| Kubernetes distribution | Minikube |
| Kubernetes CLI | kubectl |
| Package manager | Helm |
| Metrics collection | Prometheus |
| Visualization | Grafana |
| Client machine | Windows |
| Browser | Google Chrome |

### Installed versions recorded during the lab

| Tool | Version |
|---|---|
| Docker Engine | 29.9.2 |
| kubectl | v1.37.1 |
| Minikube | v1.39.0 |
| Kubernetes | v1.37.0 |
| Helm | Check with `helm version` |

Versions may differ if the installation commands are executed later.

## 4. Docker Installation

Docker provides the container runtime used by Minikube in this lab.

### Update the system and install Docker

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl ca-certificates docker.io
```

Enable Docker and verify the service:

```bash
sudo systemctl enable --now docker
sudo systemctl status docker --no-pager
sudo docker version
```

Test the installation:

```bash
docker run --rm hello-world
```

### Docker permission issue

During setup, we encountered a permission error when accessing the Docker socket.

We added the Ubuntu user to the Docker group:

```bash
sudo usermod -aG docker "$USER"
```

A new login session is required for the updated group membership to take effect. A new group session can also be started with `newgrp docker`.

**Note:** Membership in the Docker group grants highly privileged access to the host.

## 5. Install kubectl and Minikube

### Install kubectl

```bash
KUBECTL_VERSION=$(curl -L -s https://dl.k8s.io/release/stable.txt)

curl -LO "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl"

sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

rm kubectl
```

Verify:

```bash
kubectl version --client
```

### Install Minikube

```bash
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64

sudo install minikube-linux-amd64 /usr/local/bin/minikube

rm minikube-linux-amd64
```

Verify:

```bash
minikube version
```

### Configure Docker as the Minikube driver

```bash
minikube config set driver docker
```

Start the cluster:

```bash
minikube start --driver=docker --cpus=2 --memory=2560mb --disk-size=20g
```

The initial attempt requested 5500 MB, which exceeded the available system memory. Minikube also warned that the requested memory could cause stability issues. The cluster subsequently started with its existing profile; check `minikube status` before making configuration changes.

### Verify the cluster

```bash
minikube status
kubectl get nodes -o wide
kubectl get pods -A
free -h
```

The Minikube node eventually reached the `Ready` state.

## 6. Install Helm

Helm manages Kubernetes applications through charts.

Install Helm using the installation method from the lab:

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify:

```bash
helm version
```

## 7. Install Prometheus

Prometheus collects and stores time-series metrics and provides PromQL for querying them.

### Add the Prometheus Community repository

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

Update the repositories:

```bash
helm repo update
```

Install Prometheus:

```bash
helm install prometheus prometheus-community/prometheus
```

### Verify the installation

```bash
kubectl get pods
kubectl get svc
```

The chart deployed resources including:

- Prometheus server
- Alertmanager
- Node Exporter
- kube-state-metrics
- Pushgateway

### Understand the Prometheus Service

The Prometheus server Service was created as a `ClusterIP` Service.

Its Service port was `80`, while the Prometheus container listened on port `9090`.

The in-cluster DNS name was:

```text
prometheus-server.default.svc.cluster.local
```

The Service is primarily intended for communication inside the Kubernetes cluster.

## 8. Prometheus Service and NodePort

During the lab, we created an additional NodePort Service:

```bash
kubectl expose service prometheus-server --type=NodePort --target-port=9090 --name=prometheus-server-ext
```

Inspect the Service:

```bash
kubectl get svc
```

Retrieve the Minikube IP:

```bash
minikube ip
```

The cluster returned:

```text
192.168.49.2
```

The initial NodePort assigned in our command history was `32241`. Kubernetes may assign a different port when the Service is recreated.

### Test the endpoint

```bash
curl -I http://192.168.49.2:32241
```

We received an HTTP `405 Method Not Allowed` response to the HEAD request. This showed that the endpoint was reachable but did not accept that HTTP method.

We then tested the health endpoint:

```bash
curl http://192.168.49.2:32241/-/healthy
```

Expected response:

```text
Prometheus Server is Healthy.
```

The successful health response confirmed that the Prometheus endpoint was responding.

**Important:** A Minikube NodePort address is not automatically reachable from a Windows laptop or the public internet. Minikube is running inside Docker on EC2, so the node's private cluster address is not the EC2 public IP.

## 9. Access Prometheus Using Port Forwarding

We used the following command:

```bash
kubectl port-forward svc/prometheus-server 9090:80
```

The intention was to expose the Prometheus Service locally on the EC2 instance using port `9090`.

### Understand the command

```text
kubectl port-forward RESOURCE LOCAL_PORT:REMOTE_PORT
```

- `svc/prometheus-server`: Kubernetes Service being forwarded to.
- `9090`: Local port on the EC2 machine.
- `80`: Service port on the Kubernetes Service.

The forwarding process must remain running while the connection is needed.

### Important observation

In the captured output, Kubernetes reported:

```text
Forwarding from 127.0.0.1:9090 -> 9090
```

This differs from the intended Service mapping of `9090:80`. Verify the actual forwarding target if you repeat the command. For a Service port-forward, the destination is the Service port; for a Pod port-forward, the destination is the Pod's port.

## 10. Install Grafana

Grafana visualizes monitoring metrics through dashboards, graphs, tables, and other panels.

### Add the Grafana repository

```bash
helm repo add grafana https://grafana.github.io/helm-charts
```

Update the chart repositories:

```bash
helm repo update
```

Install Grafana:

```bash
helm install grafana grafana/grafana
```

The chart installation completed successfully. The chart also displayed a deprecation warning, so check the chart's current maintenance status before using it for a new production deployment.

### Retrieve the Grafana admin password

```bash
kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

The command retrieves the decoded admin password from the Kubernetes Secret.

Login details:

| Field | Value |
|---|---|
| Username | `admin` |
| Password | Output of the Secret command |
| Grafana Service | `grafana` |
| Service port | `80` |

Do not publish the generated password in this repository.

### Verify Grafana

```bash
kubectl get pods -n default
kubectl get svc grafana
```

The Grafana Pod reached the `Running` state, and the Service was created as a `ClusterIP` Service on port `80`.

## 11. Access Grafana Using Port Forwarding

We ran:

```bash
kubectl port-forward -n default svc/grafana 3000:80
```

This forwards local port `3000` on EC2 to the Grafana Service port `80`.

The forwarding process must remain active while the application is being accessed.

The browser URL, when the browser runs on the same machine as this command, is:

```text
http://localhost:3000
```

Login with username `admin` and the generated password.

## 12. Windows-to-EC2 SSH Port Forwarding

The browser runs on Windows, while Grafana and Prometheus run inside Kubernetes on EC2.

To connect the Windows browser to a local port on EC2, use SSH local port forwarding.

### Grafana SSH tunnel

Run the SSH command in **Windows PowerShell**, not inside the remote EC2 shell:

```powershell
ssh -i "$HOME\Downloads\K8s_Grafana&Promethus.pem" -N -L 3000:127.0.0.1:3000 ubuntu@EC2_PUBLIC_IP
```

Replace `EC2_PUBLIC_IP` with the current public IP of the instance. Adjust the PEM path if the key is stored elsewhere.

### Prometheus SSH tunnel

Open another Windows PowerShell window:

```powershell
ssh -i "$HOME\Downloads\K8s_Grafana&Promethus.pem" -N -L 9090:127.0.0.1:9090 ubuntu@EC2_PUBLIC_IP
```

The corresponding `kubectl port-forward` process must already be running on EC2.

### Understand the SSH options

| Option | Purpose |
|---|---|
| `-i` | Selects the SSH private key |
| `-N` | Does not execute a remote shell command |
| `-L` | Creates local port forwarding |
| `3000:127.0.0.1:3000` | Maps Windows port 3000 to EC2 localhost port 3000 |
| `ubuntu@EC2_PUBLIC_IP` | Connects as the Ubuntu user |

### Access the applications

After starting the appropriate tunnel, use these addresses on Windows:

- Grafana: `http://localhost:3000`
- Prometheus: `http://localhost:9090`

Keep the relevant SSH tunnel and Kubernetes port-forward sessions running.

### SSH troubleshooting

During the lab, an SSH command was attempted from EC2 and returned:

```text
Warning: Identity file K8s_Grafana&Promethus.pem not accessible: No such file or directory.
Permission denied (publickey).
```

The private key was not available at the path supplied to SSH. Run SSH from the Windows machine where the PEM file is stored, and verify that the key belongs to the EC2 instance.

The commands above show the correct forwarding pattern; the successful Windows SSH command was not included in the supplied terminal history.

## 13. Configure Prometheus as a Grafana Data Source

Grafana needs a data source to retrieve Prometheus metrics.

### Configuration steps

1. Open Grafana in the browser.
2. Navigate to **Connections → Data sources**.
3. Open the Prometheus data source or add one.
4. Configure the Prometheus server URL.
5. Select **Save & test**.

### Recommended Prometheus URL

Because Grafana and Prometheus are running inside the same Kubernetes cluster, use the internal Service DNS address:

```text
http://prometheus-server.default.svc.cluster.local
```

This lets Grafana communicate directly with Prometheus inside Kubernetes.

### Why `http://localhost:9090` may fail

Inside a Grafana Pod, `localhost` refers to the Grafana Pod itself, not the Windows laptop and not automatically the EC2 host.

Therefore, configuring Grafana to use `http://localhost:9090` can produce a connection-refused error.

Use the Prometheus Service DNS name instead, assuming both applications are in the `default` namespace as in this lab.

## 14. Explore Kubernetes Dashboards

After configuring the monitoring tools, we explored a Kubernetes dashboard in Grafana.

The dashboard showed panels for:

- Node CPU utilization
- Node memory utilization
- Node CPU cores
- Namespace resource information
- Pod and workload counts
- Network traffic

The imported dashboard used Chinese-language titles. This was a dashboard-specific language issue rather than a problem with Prometheus or Grafana installation.

To use an English dashboard, import an English-language Kubernetes dashboard or edit the panel titles and descriptions.

Dashboard panels only display meaningful metrics when the required metrics are available in Prometheus and the panel queries match those metrics.

## 15. Understanding Port Forwarding

Port forwarding was one of the most important concepts in this lab.

### Kubernetes port forwarding

```bash
kubectl port-forward -n default svc/grafana 3000:80
```

This maps a local port on the EC2 host to the Grafana Service port inside Kubernetes.

### SSH local port forwarding

```powershell
ssh -i "KEY_PATH" -N -L 3000:127.0.0.1:3000 ubuntu@EC2_PUBLIC_IP
```

This maps the Windows local port to port `3000` on the EC2 host.

### Combined request path

```text
Windows Browser
      |
      | http://localhost:3000
      v
Windows Local Port 3000
      |
      | SSH Tunnel
      v
EC2 Local Port 3000
      |
      | kubectl port-forward
      v
Grafana Kubernetes Service :80
      |
      v
Grafana Pod
```

The same concept applies to Prometheus using port `9090`.

### Port mapping versus port forwarding

Port mapping commonly describes mapping one network port to another, such as Docker's `-p` option.

Kubernetes `kubectl port-forward` creates a temporary forwarding session to a Kubernetes resource.

SSH local port forwarding creates an encrypted tunnel from a local machine to a remote destination.

These mechanisms work together in this lab, but they are not the same operation.

## 16. Troubleshooting

### A. Browser shows `ERR_CONNECTION_REFUSED`

Possible causes:

- The port-forward process is not running.
- The application Pod is not ready.
- The wrong port was selected.
- The SSH tunnel is not running.

Useful checks:

```bash
kubectl get pods -n default
kubectl get svc
```

Check the terminal where the port-forward command was started.

### B. Browser shows `ERR_CONNECTION_TIMED_OUT`

Possible causes:

- The wrong IP address or port is being used.
- A NodePort is not reachable from the client network.
- The EC2 security group or network path blocks the connection.
- The Minikube node IP is not externally routable.

For this lab, localhost access through SSH tunneling avoids the need to expose monitoring web ports publicly.

### C. SSH reports `Identity file not accessible`

Check the private key path on Windows.

Use the full path to the PEM file and run SSH from the Windows terminal where that file exists.

### D. SSH reports `Permission denied (publickey)`

Verify:

- The correct private key is selected.
- The correct EC2 public IP is used.
- The login username is `ubuntu`.
- The key belongs to the EC2 instance.
- SSH access is permitted by the EC2 security group.

### E. Grafana cannot connect to Prometheus

Check the Prometheus Pod and Service:

```bash
kubectl get pods
kubectl get svc prometheus-server
```

Use the internal Service DNS name in Grafana:

```text
http://prometheus-server.default.svc.cluster.local
```

Do not use Windows `localhost` as the Prometheus data source address.

### F. NodePort is not working

Check the assigned port:

```bash
kubectl get svc prometheus-server-ext
```

Verify that the Service has an endpoint and that the Prometheus Pod is ready.

A NodePort assigned to Minikube does not necessarily mean the application is reachable directly from the Windows machine.

## 17. Security Considerations

- Do not commit PEM private keys to GitHub.
- Never publish the Grafana admin password.
- Restrict SSH access to trusted IP addresses.
- Avoid exposing Prometheus and Grafana directly to the public internet for this lab.
- Use SSH tunneling for temporary administrative access.
- Remember that `kubectl port-forward` and SSH tunnels stop when their respective processes terminate.
- The Grafana chart reported that persistence was disabled. Dashboard and application data may be lost if the Grafana Pod is recreated.
- For production, configure persistent storage, authentication, TLS, appropriate permissions, and a supported chart version.

## 18. Lab Checklist

- [x] Created an Ubuntu EC2 environment.
- [x] Installed Docker.
- [x] Installed kubectl.
- [x] Installed Minikube.
- [x] Started the Minikube cluster.
- [x] Installed Helm.
- [x] Installed Prometheus using Helm.
- [x] Created and tested a Prometheus NodePort Service.
- [x] Installed Grafana using Helm.
- [x] Retrieved the Grafana admin password.
- [x] Ran Kubernetes port-forward commands for Prometheus and Grafana.
- [x] Accessed the monitoring interfaces in the browser.
- [x] Explored a Kubernetes monitoring dashboard.
- [ ] Record the exact successful Windows SSH tunnel commands.
- [ ] Verify and save the Grafana Prometheus data source configuration.

## 19. References

- [Prometheus and Grafana Zero to Hero](https://github.com/iamsaikishore/prometheus-Grafana-Zero-to-Hero)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Minikube Documentation](https://minikube.sigs.k8s.io/docs/)
- [Helm Documentation](https://helm.sh/docs/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)

---

**Lab summary:** This lab demonstrates how Prometheus and Grafana can be installed on a Minikube cluster running on AWS EC2, and how to access those applications from Windows through Kubernetes port forwarding and SSH tunneling.
