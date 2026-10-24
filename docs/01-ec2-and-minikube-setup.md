# 01 --- EC2 and Minikube Setup

## Purpose

Prepare a single-node Kubernetes learning cluster on an AWS EC2
instance. This document records the environment used in this lab.
Package versions and installation instructions can change; verify vendor
documentation if reproducing the setup later.

## Environment recorded

-   Cloud: AWS EC2
-   Operating system: Red Hat Enterprise Linux 10.2 (x86_64)
-   Instance: 2 vCPUs, approximately 3.5 GiB RAM
-   Root EBS volume: increased from 10 GiB to 25 GiB and filesystem
    expanded
-   Container engine: Docker Engine
-   Kubernetes learning cluster: Minikube using the Docker driver
-   CLI: `kubectl`
-   Login user: `ec2-user`

Never commit the private SSH key (`K8s.pem`) or any AWS credentials to
Git.

## What each component does

-   **EC2:** the virtual machine where the lab runs.
-   **Docker Engine:** runs the Minikube node container.
-   **Minikube:** creates and manages a Kubernetes cluster for learning.
-   **Kubernetes:** orchestrates Pods and their containers.
-   **kubectl:** sends commands to the Kubernetes API server.

This setup runs Kubernetes inside a Docker container on the EC2 host.
Docker is the Minikube driver in this lab; Kubernetes workloads inside
Minikube use the node's configured container runtime.

## Connection

From a trusted local terminal, connect using your own key and instance
DNS name:

``` bash
ssh -i "K8s.pem" ec2-user@YOUR_EC2_PUBLIC_DNS
```

Replace `YOUR_EC2_PUBLIC_DNS` with the current public DNS name. Do not
put the private key in the repository.

## Verify installation

``` bash
uname -r
cat /etc/redhat-release
df -h /
kubectl version --client
docker --version
minikube version
```

Check Docker:

``` bash
sudo systemctl status docker
sudo docker run --rm hello-world
```

If you have configured the `ec2-user` account for Docker group access,
reconnect or start a new login session and verify with `docker ps`.
Membership in the `docker` group grants highly privileged access to the
host; only do this for a trusted account.

## Start and verify Minikube

The cluster was started with:

``` bash
minikube start --driver=docker
```

Check status:

``` bash
minikube status
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A
```

During startup, the node can briefly show `NotReady`. Wait a short time
and check again. If it remains unhealthy, collect diagnostics:

``` bash
kubectl describe node minikube
kubectl get pods -A
minikube logs --problems
docker ps
free -h
df -h /
```

Do not assume a transient startup message is a persistent failure. Use
current node conditions, Pod status, events, and logs to decide.

## Resource sizing note

The host has about 3.5 GiB RAM, and Minikube was configured to use 3072
MiB. This leaves limited memory for RHEL, Docker, and SSH. If memory
pressure occurs, stop the cluster and restart it with a lower memory
allocation, for example:

``` bash
minikube stop
minikube start --driver=docker --memory=2500 --cpus=2
```

Confirm Minikube accepts the requested configuration for your installed
version. Lowering memory may limit what you can run in the cluster.

## Troubleshooting history

### Docker socket permission denied

A non-root user may not have permission to access
`/var/run/docker.sock`. Check:

``` bash
ls -l /var/run/docker.sock
groups
docker ps
```

Use `sudo docker ps` to confirm whether the engine itself is healthy. Do
not blindly change permissions on the socket. If using the `docker`
group, understand that group membership is effectively root-level
access.

### Docker networking modules unavailable after kernel update

This lab previously encountered missing kernel networking modules with
the old running kernel. After booting into the newer installed kernel,
the required modules could be loaded and Docker started. If a similar
issue occurs, inspect the running kernel and module errors rather than
copying commands blindly:

``` bash
uname -r
sudo modprobe ip_tables
sudo modprobe iptable_nat
sudo modprobe xt_addrtype
sudo systemctl status docker
```

Module names and availability depend on the kernel and distribution.

## Cleanup

To stop the learning cluster without deleting its configuration:

``` bash
minikube stop
```

To delete the Minikube cluster entirely:

``` bash
minikube delete
```

Deletion removes the cluster and its workloads. Do this only when you
are sure you no longer need them.
