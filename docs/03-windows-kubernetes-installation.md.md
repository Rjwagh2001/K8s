# Kubernetes Installation on Windows 11

A practical, repeatable guide to installing `kubectl`, Minikube, WSL 2, Ubuntu, and Docker Desktop on Windows 11, then creating and testing your first Kubernetes Pod.

## What we will build

```text
Windows 11
├── kubectl (Kubernetes command-line client)
├── WSL 2
│   └── Ubuntu 24.04 LTS
├── Docker Desktop (Docker Engine using the WSL 2 backend)
└── Minikube
    └── Local single-node Kubernetes cluster
        └── NGINX Pod
            └── Local access through kubectl port-forward
```

> **Important:** `kubectl` is the client used to communicate with Kubernetes. Minikube creates the local Kubernetes cluster. Docker Desktop provides the container engine used by Minikube's Docker driver. WSL 2 provides a Linux kernel environment for Linux containers on Windows.

## Prerequisites

- Windows 11 with hardware virtualization enabled.
- Administrator access for Windows/WSL installation steps.
- Internet access to download the tools and container images.
- Enough free disk space for Docker Desktop, Ubuntu, and Kubernetes images.

Use **PowerShell** for the commands below unless stated otherwise. Run commands requiring system changes in PowerShell **as Administrator**.

---

## 1. Create a working directory

```powershell
New-Item -Path 'D:\K8s\WindowsInstallationK8s' -ItemType Directory -Force
Set-Location 'D:\K8s\WindowsInstallationK8s'
```

This folder is where the Kubernetes manifest used in this lab will be stored.

## 2. Install kubectl

`kubectl` is the command-line client used to query and manage a Kubernetes cluster.

### Official documentation

- [Install and Set Up kubectl on Windows — Kubernetes documentation](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/)
- [Install kubectl binary on Windows via direct download or curl](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/#install-kubectl-binary-on-windows-via-direct-download-or-curl)

> **Note:** Check the official Kubernetes documentation first for the latest installation instructions and supported versions. The commands below record the steps used in this lab.

### Download the Windows binary

This example installs the `v1.37.0` client, matching the Kubernetes version used in this lab.

```powershell
Set-Location 'D:\K8s\WindowsInstallationK8s'

curl.exe -LO "https://dl.k8s.io/release/v1.37.0/bin/windows/amd64/kubectl.exe"
curl.exe -LO "https://dl.k8s.io/v1.37.0/bin/windows/amd64/kubectl.exe.sha256"
```

The `amd64` download is for 64-bit x86 Windows computers. Choose the appropriate architecture if your device differs.

### Verify the downloaded file

```powershell
(Get-FileHash -Algorithm SHA256 .\kubectl.exe).Hash -eq (Get-Content .\kubectl.exe.sha256)
```

Expected output:

```text
True
```

### Test kubectl

PowerShell does not automatically run executable files from the current directory, so use `.\`:

```powershell
.\kubectl.exe version --client
```

Expected output includes:

```text
Client Version: v1.37.0
```

### Add kubectl to your user PATH

Add the directory containing `kubectl.exe` to your **user** `Path` environment variable:

1. Open Start and search for **Edit environment variables for your account**.
2. Select the user `Path` variable and click **Edit**.
3. Click **New** and add:

   ```text
   D:\K8s\WindowsInstallationK8s
   ```

4. Save the changes and open a new PowerShell window.

Verify:

```powershell
where.exe kubectl
kubectl version --client
```

> If `kubectl` is already available, check `where.exe kubectl` to see which copy is being used. Docker Desktop may also install a copy of `kubectl`.

---

## 3. Install Minikube

Minikube creates a Kubernetes cluster for local development and learning.

### Official documentation

- [Minikube — Get Started](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fwindows%2Fx86-64%2Fstable%2F.exe+download)

> **Note:** Use the official Minikube guide to check the latest download option and installation instructions for your Windows architecture. The commands below document the approach used in this lab.

### Download Minikube

Create its directory:

```powershell
New-Item -Path 'C:\minikube' -ItemType Directory -Force
```

Download the Windows executable:

```powershell
$ProgressPreference = 'SilentlyContinue'

Invoke-WebRequest `
  -Uri 'https://github.com/kubernetes/minikube/releases/latest/download/minikube-windows-amd64.exe' `
  -OutFile 'C:\minikube\minikube.exe'
```

Verify the executable directly:

```powershell
C:\minikube\minikube.exe version
```

The lab used Minikube `v1.39.0`.

### Add Minikube to your user PATH

Add this directory to your **user** `Path` environment variable:

```text
C:\minikube
```

Open a new PowerShell window and verify:

```powershell
where.exe minikube
minikube version
```

> If changing the machine-wide PATH returns “Requested registry access is not allowed,” use the user-level PATH instead. Administrator access is not required to update your user PATH.

---

## 4. Install WSL 2 and Ubuntu

WSL 2 (Windows Subsystem for Linux 2) runs a Linux kernel in a lightweight virtual machine managed by Windows. Docker Desktop can use this backend to run Linux containers.

Open PowerShell **as Administrator**:

```powershell
wsl --install
```

Follow any prompts and restart Windows if requested. The command may start downloading Ubuntu. Let the download and installation complete.

### If the default Ubuntu download gets stuck

Check the available distributions in a separate PowerShell window:

```powershell
wsl --list --online
wsl --list --verbose
```

If no distribution is installed, you can install Ubuntu 24.04 LTS explicitly:

```powershell
wsl --install -d Ubuntu-24.04
```

Avoid launching multiple Ubuntu installations simultaneously. If an installation is clearly stuck, cancel that attempt once with `Ctrl+C` before retrying.

When Ubuntu launches for the first time, create a Linux username and password. The password is not displayed while typing; this is normal.

### Verify WSL and Ubuntu

```powershell
wsl --status
wsl --list --verbose
```

Confirm that Ubuntu shows `VERSION 2`. The state can be `Running` or `Stopped`; either is normal when Ubuntu is not actively being used.

If the distribution is installed as WSL 1, convert it:

```powershell
wsl --set-version Ubuntu-24.04 2
```

If needed, update WSL:

```powershell
wsl --update
```

---

## 5. Install Docker Desktop

Docker provides the container engine used by Minikube in this setup.

1. Download Docker Desktop from the [official Docker Desktop website](https://www.docker.com/products/docker-desktop/).
2. Run the installer.
3. Use the **WSL 2 backend** when offered or supported by the installed version.
4. Complete installation and restart Windows if requested.
5. Launch Docker Desktop and wait for it to finish starting.

In Docker Desktop, open **Settings → General** and verify **Use the WSL 2 based engine** is enabled if that setting is shown.

### Verify Docker

Open a new PowerShell window:

```powershell
docker --version
docker info
```

Run a test container:

```powershell
docker run hello-world
```

Expected output includes:

```text
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

The first run downloads the `hello-world` image if it is not already present.

> If you see `Docker Desktop is unable to start`, open Docker Desktop and wait for the engine to start, then retry `docker info` and `docker run hello-world`. A successful `hello-world` run confirms the engine is responding.

---

## 6. Start a local Kubernetes cluster

Make sure Docker Desktop is running. In PowerShell, run:

```powershell
minikube start --driver=docker
```

On the first run, Minikube downloads its node image and Kubernetes images, creates a Docker container for the node, configures cluster networking, and enables basic add-ons. This can take several minutes.

Expected final message:

```text
Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

### Verify the cluster

```powershell
minikube status
kubectl config current-context
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A
```

The current context should be `minikube`, and the node should eventually show `Ready`.

Example:

```text
NAME       STATUS   ROLES           VERSION
minikube   Ready    control-plane   v1.37.0
```

The age, IP address, and other details will vary.

---

## 7. Create an NGINX Pod

A Pod is the smallest deployable unit in Kubernetes. This exercise creates a standalone Pod, not a Deployment.

In `D:\K8s\WindowsInstallationK8s`, create a file named `pod.yml` with this content:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.14.2
      ports:
        - containerPort: 80
```

You can create or edit the file with Notepad:

```powershell
notepad .\pod.yml
```

Save the YAML, then create the Pod:

```powershell
kubectl create -f .\pod.yml
```

Expected output:

```text
pod/nginx created
```

### Verify the Pod

```powershell
kubectl get pods
kubectl get pods -o wide
kubectl describe pod nginx
```

Wait if the Pod initially shows `ContainerCreating`. The image needs to be downloaded and the container started.

Confirm that:

- `STATUS` is `Running`.
- `READY` is `1/1`.
- `RESTARTS` is `0` for this initial run.
- `kubectl describe pod nginx` reports the container as `Running` and `Ready: True`.

The Pod IP (for example, `10.244.0.3`) is internal to the cluster and may change if the Pod is recreated.

---

## 8. Access NGINX locally with port forwarding

`kubectl port-forward` creates a temporary connection from a local port on your Windows machine to a port in the Pod.

In the first PowerShell window, run:

```powershell
kubectl port-forward nginx 8080:80
```

Keep this window open. Expected output:

```text
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

Open a **second** PowerShell window and run:

```powershell
curl.exe http://127.0.0.1:8080
```

The response should contain the NGINX welcome-page HTML. You can also visit:

<http://127.0.0.1:8080>

A `200 OK` response or the NGINX welcome page confirms that local access works.

> **Networking note:** This is local testing through port forwarding. It does not expose NGINX publicly to the internet. Keep the port-forwarding terminal open while testing; stop it with `Ctrl+C` when finished.

### PowerShell curl note

In Windows PowerShell, `curl` may be an alias for `Invoke-WebRequest`, which can show a script-execution warning and formatted output. Use `curl.exe` to invoke the actual curl program and avoid that alias behavior.

---

## 9. Useful commands

| Task | Command |
|---|---|
| Check Minikube | `minikube status` |
| Start the cluster | `minikube start --driver=docker` |
| Stop the cluster | `minikube stop` |
| Show nodes | `kubectl get nodes` |
| Show Pods in default namespace | `kubectl get pods` |
| Show Pods in every namespace | `kubectl get pods -A` |
| Show Pod IP and node | `kubectl get pods -o wide` |
| Inspect Pod events and details | `kubectl describe pod nginx` |
| View container logs | `kubectl logs nginx` |
| Forward local port 8080 | `kubectl port-forward nginx 8080:80` |
| Delete the Pod | `kubectl delete pod nginx` |

If port `8080` is already in use, choose another local port:

```powershell
kubectl port-forward nginx 8081:80
```

Then test `http://127.0.0.1:8081`.

---

## 10. Cleanup

Stop the port-forwarding process with `Ctrl+C` in its terminal.

Delete the standalone Pod:

```powershell
kubectl delete pod nginx
kubectl get pods
```

Stop Minikube when you are done:

```powershell
minikube stop
```

The next time you want to practice, start Docker Desktop and run:

```powershell
minikube start --driver=docker
kubectl get nodes
```

You do not need to reinstall kubectl, Minikube, WSL, Ubuntu, or Docker for each lab.

## Troubleshooting notes

### `kubectl` is not recognized

- Confirm the directory containing `kubectl.exe` is in your user PATH.
- Open a new PowerShell window.
- Check the executable location:

  ```powershell
  where.exe kubectl
  ```

- If you are in the directory containing the executable, run `.\kubectl.exe version --client`.

### `kubectl cluster-info` cannot connect to `localhost:8080`

This generally means kubectl has no usable cluster configuration or the cluster is not running. Start Minikube and check the active context:

```powershell
minikube status
minikube start --driver=docker
kubectl config current-context
kubectl get nodes
```

### `minikube start` cannot use Docker

- Open Docker Desktop.
- Wait until its engine is running.
- Confirm `docker info` succeeds.
- Retry:

  ```powershell
  minikube start --driver=docker
  ```

### Pod is stuck in `ContainerCreating`

Inspect the Pod and its events:

```powershell
kubectl describe pod nginx
kubectl get pods -o wide
```

Wait for image pulls to finish and check the event messages for errors.

### Port forwarding says the Pod was not found

Use the exact Pod name and spelling:

```powershell
kubectl get pods
kubectl port-forward nginx 8080:80
```

The correct name in this lab is `nginx`, not `ngnix`.

---

## Final result

By completing this guide, you have:

1. Installed `kubectl` on Windows.
2. Installed Minikube.
3. Installed WSL 2 and Ubuntu 24.04 LTS.
4. Installed and verified Docker Desktop.
5. Created a local Kubernetes cluster using the Docker driver.
6. Created and inspected an NGINX Pod.
7. Accessed NGINX locally using `kubectl port-forward`.

This setup is suitable for learning Kubernetes resources, Pods, YAML manifests, basic troubleshooting, and local application access.
