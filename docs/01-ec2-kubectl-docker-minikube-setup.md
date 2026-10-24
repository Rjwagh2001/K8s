# 01 --- Install kubectl, Docker, and Minikube on EC2

## Purpose

Install the tools needed for the first Kubernetes learning cluster,
verify each component, and document the real issues encountered during
setup.

## Environment

-   AWS EC2
-   Red Hat Enterprise Linux 10.2 (x86_64)
-   2 vCPUs, approximately 3.5 GiB RAM
-   Root EBS volume expanded from 10 GiB to 25 GiB
-   Login user: `ec2-user`

## 1. Pre-installation checks

Run:

``` bash
whoami
cat /etc/redhat-release
uname -r
nproc
free -h
df -hT /
```

Why: establish the OS, running kernel, CPU, memory, and disk before
installing packages. This helps diagnose compatibility and capacity
problems later.

## 2. Install kubectl first

`kubectl` is the Kubernetes command-line client. It talks to a
Kubernetes API server; installing it does not itself create a cluster.

The lab had `kubectl` installed at `/usr/local/bin/kubectl`, version
`v1.37.1`. The exact initial installation command was not preserved in
the session notes, so this document does not claim a particular command
was used. For future reproductions, follow the current official
Kubernetes installation guide for Linux and choose the correct
architecture.

Verify:

``` bash
command -v kubectl
kubectl version --client
```

Expected in this lab: client version `v1.37.1`. Versions can change; use
a client version compatible with the cluster version.

## 3. Install Docker Engine

Docker is used here as Minikube's driver. Minikube creates a container
that acts as the Kubernetes node. Inside that node, the Kubernetes
workload runtime is containerd.

Docker Engine version observed: `29.9.2`. The Minikube node used
containerd `2.3.4`; the EC2 host also had containerd `2.4.2` installed
as part of the Docker setup.

The session used Docker's official RHEL repository. Repository setup and
package names can change; consult the current Docker Engine installation
instructions for RHEL before repeating on a new machine.

Verify Docker:

``` bash
docker --version
sudo systemctl status docker
sudo docker run --rm hello-world
```

Expected: Docker reports its version, the service is active, and
`hello-world` prints `Hello from Docker!`.

### Issue A --- Docker networking kernel modules missing

**Symptom:** Docker initially could not start correctly because kernel
networking modules including `ip_tables`, `iptable_nat`, and
`xt_addrtype` were unavailable for the running kernel.

**Investigation commands:**

``` bash
uname -r
rpm -q kernel
sudo modprobe ip_tables
sudo modprobe iptable_nat
sudo modprobe xt_addrtype
sudo systemctl status docker
```

**Resolution in this lab:** a newer kernel package had been installed,
but the EC2 instance was still running the older kernel
(`6.12.0-211.16.1...`). The system was rebooted into the newer installed
kernel (`6.12.0-211.68.1.el10_2.x86_64`), after which the required
modules could be loaded and Docker started.

**Lesson:** when kernel modules are missing, check the running kernel
against installed kernels. A reboot may be needed after a kernel update.
Reboot only when safe and ensure you can reconnect to the instance.

### Issue B --- Docker socket permission denied

**Symptom:** Minikube could not access Docker through
`/var/run/docker.sock` as the non-root user.

**Investigation commands:**

``` bash
ls -l /var/run/docker.sock
groups
docker ps
sudo docker ps
```

**Resolution in this lab:** the `ec2-user` account was added to the
Docker group and a refreshed group/login session was used.
`newgrp docker` was used during the session, after which Minikube could
access Docker.

Docker group membership effectively grants root-level control over the
host. Only do this for a trusted user. A safer alternative for a one-off
Docker command is `sudo`, but Minikube itself must be configured to use
Docker access consistently.

Verify:

``` bash
docker ps
```

## 4. Install Minikube

Minikube provides a single-node Kubernetes cluster suitable for
learning. Version observed: `v1.39.0`, installed at
`/usr/local/bin/minikube`. The exact original download/install command
was not preserved; follow the current official Minikube Linux
installation guide when reproducing.

Verify:

``` bash
command -v minikube
minikube version
```

## 5. Start the cluster

The command used was:

``` bash
minikube start --driver=docker
```

Minikube downloaded or configured Kubernetes `v1.37.0` and set `kubectl`
to use the `minikube` context.

Verify:

``` bash
minikube status
kubectl config current-context
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A
```

Expected: context is `minikube`; the node eventually becomes `Ready`;
Kubernetes system Pods eventually reach `Running` or their appropriate
completed/healthy state.

### Issue C --- Node briefly showed NotReady

**Symptom:** immediately after startup, `kubectl get nodes` briefly
showed `NotReady`.

**Investigation:**

``` bash
kubectl describe node minikube
kubectl get pods -A
minikube logs --problems
```

**Outcome in this lab:** a later `kubectl describe node minikube` showed
`Ready=True`, no memory/disk/PID pressure, and the Kubernetes system
Pods were running. This was a transient startup state, not a persistent
failure.

**Lesson:** check again after initialization and use current
conditions/events before diagnosing a lasting fault.

### Issue D --- Memory warning

Minikube allocated 3072 MiB to its node while the EC2 host had around
3553 MiB RAM. Startup warned that little host overhead remained.

If memory pressure occurs, stop and restart with a lower allocation:

``` bash
minikube stop
minikube start --driver=docker --memory=2500 --cpus=2
```

Only do this if you are prepared to recreate or reconfigure the cluster,
and verify the configuration accepted by your installed Minikube
version. Another option is to use an EC2 instance with more RAM, subject
to cost.

## 6. Cluster health checks

Run:

``` bash
kubectl get nodes
kubectl get pods -A
kubectl describe node minikube
free -h
df -h /
```

Do not treat an early startup warning as a root cause without checking
current state. Capture the failing command, exact error, events, logs,
and resource usage.

## 7. Stop or delete the cluster

Stop it but retain the cluster configuration:

``` bash
minikube stop
```

Delete the cluster and its workloads:

``` bash
minikube delete
```

Use deletion only when you intend to remove the learning cluster. The
EC2 instance and EBS volume are separate AWS resources and are not
automatically deleted by `minikube delete`.
