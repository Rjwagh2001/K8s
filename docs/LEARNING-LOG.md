# Kubernetes Learning Log

Append an entry after each lab. Record what you intended to do, commands
used, actual output, issues, diagnosis, resolution, and the next step.
Redact private IPs if desired, keys, tokens, credentials, and other
sensitive data before committing.

## 2026-10-24 --- EC2, tools, Minikube, and first NGINX Pod

### Environment

-   AWS EC2, Red Hat Enterprise Linux 10.2 (x86_64)
-   2 vCPUs, approximately 3.5 GiB RAM
-   Root EBS volume expanded from 10 GiB to 25 GiB
-   `kubectl` v1.37.1
-   Docker Engine v29.9.2
-   Minikube v1.39.0
-   Kubernetes v1.37.0 in Minikube

### Setup and verification commands

``` bash
uname -r
cat /etc/redhat-release
nproc
free -h
df -hT /
kubectl version --client
docker --version
sudo systemctl status docker
sudo docker run --rm hello-world
minikube version
minikube start --driver=docker
kubectl get nodes
kubectl get pods -A
```

### Issues and resolutions

1.  **Missing Docker networking modules:** the running kernel was older
    than the installed kernel. After rebooting into the newer kernel,
    the modules could be loaded and Docker started.
2.  **Docker socket permission denied:** Docker group membership and a
    refreshed login/group session allowed the user to access Docker.
    Docker group membership is privileged.
3.  **Node briefly NotReady:** later checks showed the node Ready and
    system Pods running; startup was still initializing.
4.  **Low memory overhead:** Minikube used 3072 MiB on a host with about
    3.5 GiB RAM. Consider lower Minikube memory or a larger EC2 instance
    if memory pressure occurs.

### First Pod

``` bash
kubectl run nginx-pod --image=nginx:alpine
kubectl get pods
kubectl get pods -o wide
kubectl describe pod nginx-pod
kubectl port-forward pod/nginx-pod 8080:80
curl http://127.0.0.1:8080
```

### Result

The Pod reached `1/1 Running`, and `curl` returned the NGINX welcome
HTML.

### Concepts learned

-   EC2 is the VM; Docker runs the Minikube node container in this
    setup.
-   `kubectl` is the client for the Kubernetes API.
-   A Pod is the smallest deployable Kubernetes unit and can contain one
    or more containers.
-   `kubectl describe` exposes conditions and events that help
    troubleshooting.
-   Port-forwarding is temporary access for testing.

### Next steps

-   [ ] Create a Deployment and understand desired replicas.
-   [ ] Scale the Deployment and test self-healing.
-   [ ] Create a Service and understand stable access.
-   [ ] Practice rollout update and rollback.

------------------------------------------------------------------------

## Reusable template

### Date and lesson

-   **Objective:**
-   **Prerequisites:**
-   **Why this matters:**
-   **Architecture / request flow:**
-   **Commands used:**
-   **Expected result:**
-   **Actual result:**
-   **Issue / symptom:**
-   **Investigation:**
-   **Root cause:**
-   **Resolution:**
-   **Verification:**
-   **Cleanup:**
-   **Key concepts:**
-   **Next steps:**
