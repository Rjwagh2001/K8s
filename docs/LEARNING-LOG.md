# Kubernetes Learning Log

Append a new entry after each completed exercise. Keep exact commands
and observed output where useful, but remove credentials, tokens,
private keys, public IPs if you do not want them public, and other
sensitive data.

## 2026-10-24 --- EC2 Minikube setup and first NGINX Pod

### Objective

Run a first Kubernetes workload on an AWS EC2 learning machine and
verify that it serves HTTP.

### Environment

-   RHEL 10.2 on EC2
-   Docker Engine installed and verified using `hello-world`
-   Minikube started with the Docker driver
-   `kubectl` configured for the `minikube` context
-   Root disk expanded to 25 GiB

### Commands used

``` bash
minikube start --driver=docker
kubectl get nodes
kubectl get pods -A
kubectl run nginx-pod --image=nginx:alpine
kubectl get pods
kubectl describe pod nginx-pod
kubectl port-forward pod/nginx-pod 8080:80
curl http://127.0.0.1:8080
```

### Result

-   Minikube node became `Ready`.
-   Kubernetes system Pods were running.
-   `nginx-pod` reached `1/1 Running`.
-   `kubectl describe` showed the container was ready and had zero
    restarts at inspection time.
-   `curl` returned the NGINX welcome HTML.

### Concepts learned

-   A Pod is the smallest deployable Kubernetes unit and wraps one or
    more containers.
-   `kubectl` is used to inspect and manage Kubernetes resources.
-   `kubectl describe` includes conditions and events useful for
    troubleshooting.
-   `kubectl port-forward` gives temporary access for testing.

### Next steps

-   [ ] Learn Pod deletion and recreation.
-   [ ] Create an NGINX Deployment.
-   [ ] Scale replicas and inspect self-healing.
-   [ ] Create a Service and understand stable access.
-   [ ] Practice rollout update and rollback.

------------------------------------------------------------------------

## Template for the next lesson

### Date and lesson

-   **Objective:**
-   **Why this matters:**
-   **Architecture / request flow:**
-   **Commands used:**
-   **Expected result:**
-   **Actual result:**
-   **Troubleshooting and root cause:**
-   **Key concepts:**
-   **Cleanup performed:**
-   **Next steps:**
