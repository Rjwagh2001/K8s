# End-to-End Lab: Docker Image, Kubernetes Deployment, and Service

Build a container image for a Python/Django web application, deploy it to Kubernetes with three replicas, expose it with a Service, and troubleshoot connectivity.

## Table of Contents

- [1. Lab Overview](#1-lab-overview)
- [2. Application Source and Attribution](#2-application-source-and-attribution)
- [3. Repository and File Structure](#3-repository-and-file-structure)
- [4. Request Flow and Architecture](#4-request-flow-and-architecture)
- [5. Build the Docker Image](#5-build-the-docker-image)
- [6. Make the Image Available to Minikube](#6-make-the-image-available-to-minikube)
- [7. Deploy the Application](#7-deploy-the-application)
- [8. Expose the Deployment with a Service](#8-expose-the-deployment-with-a-service)
- [9. Verify Service Endpoints](#9-verify-service-endpoints)
- [10. Test with Port Forwarding](#10-test-with-port-forwarding)
- [11. Troubleshooting Performed](#11-troubleshooting-performed)
- [12. Useful Commands](#12-useful-commands)
- [13. Practice Tasks](#13-practice-tasks)
- [14. Official References](#14-official-references)
- [15. Lab Summary](#15-lab-summary)

## 1. Lab Overview

### Goal

Run a Python/Django web application in Kubernetes and learn how a Deployment and Service work together.

### Environment and components

| Component | Value used in this lab |
|---|---|
| Application source | Python/Django example |
| Docker image | `shreesakhii/python-web-app:v2` |
| Container/application port | `8000` |
| Kubernetes environment | Minikube running on an EC2 host |
| Deployment name | `pyhton-app` |
| Desired replicas | `3` |
| Service name | `python-app-service` |
| Service type | `NodePort` |
| Service port | `8000` |
| Target port | `8000` |
| NodePort | `30080` |

> **Naming note:** `pyhton-app` is misspelled, but that is the name and Pod label used in the lab manifests. Kubernetes label selectors must match the Pod labels exactly. You can rename it to `python-app` in a later cleanup, provided you update the Deployment selector/template labels and Service selector consistently.

## 2. Application Source and Attribution

The application source was taken from the [`python-web-app` example in Docker-Zero-to-Hero](https://github.com/iam-veeramalla/Docker-Zero-to-Hero/tree/main/examples/python-web-app).

The upstream example contains the application files. The Docker image build, Kubernetes Deployment and Service manifests, commands, validation, and troubleshooting documented here describe the work performed in this lab.

## 3. Repository and File Structure

### Working application directory

The working directory used for the lab was `python-web-app`:

```text
python-web-app/
├── Dockerfile
├── devops/
│   └── ... application source and Django files
├── deployment.yaml
└── python-app-service.yaml
```


## 4. Request Flow and Architecture

```text
Application source code
        |
        v
     Dockerfile
        |
        v
Docker image: shreesakhii/python-web-app:v2
        |
        | minikube image load
        v
Kubernetes Deployment: pyhton-app
        |
        | maintains 3 replicas
        v
  +-----------+  +-----------+  +-----------+
  | Pod 1     |  | Pod 2     |  | Pod 3     |
  | port 8000 |  | port 8000 |  | port 8000 |
  +-----------+  +-----------+  +-----------+
        \             |             /
         \            |            /
          v           v           v
       Kubernetes Service: python-app-service
          Service port: 8000
          targetPort: 8000
          NodePort: 30080
                    |
                    v
             Django response
```

The Deployment manages the desired number of Pods. The Service selects matching Pods and provides a stable network endpoint. EndpointSlices record the Service's backend endpoints.

## 5. Build the Docker Image

### Dockerfile used in the lab

The Dockerfile used for this lab installs Python and virtual-environment tooling, copies the application into `/app`, installs Django in a virtual environment, and starts Django on port `8000`.

```dockerfile
FROM ubuntu:latest

WORKDIR /app

RUN apt-get update && \
    apt-get install -y python3 python3-pip python3-venv && \
    rm -rf /var/lib/apt/lists/*

COPY devops/ .

RUN python3 -m venv /app/venv1 && \
    /app/venv1/bin/pip install django

EXPOSE 8000

CMD ["/app/venv1/bin/python", "manage.py", "runserver", "0.0.0.0:8000"]
```

> Check the actual `Dockerfile` in your repository before using this example. If the application defines dependencies in `requirements.txt`, installing the pinned dependencies from that file is usually preferable to installing Django alone. This development server setup is for learning, not a production deployment recipe.

### Build

Run from the directory containing `Dockerfile`:

```bash
docker build -t shreesakhii/python-web-app:v2 .
```

Check the local image:

```bash
docker images
```

### Why the image needed Django

The initial container failed with `ModuleNotFoundError: No module named 'django'`. The image was updated to install Django in the virtual environment, then rebuilt as `shreesakhii/python-web-app:v2`.

## 6. Make the Image Available to Minikube

Load the image into Minikube:

```bash
minikube image load shreesakhii/python-web-app:v2
```

Check available images:

```bash
minikube image ls
```

The Deployment must refer to the same image tag. If using a locally loaded image, ensure Kubernetes is not configured to require a registry pull for that image. If you publish the image to a registry later, configure registry access and pull behavior accordingly.

## 7. Deploy the Application

File: `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pyhton-app
  labels:
    app: python-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: pyhton-app
  template:
    metadata:
      labels:
        app: pyhton-app
    spec:
      containers:
        - name: python-app
          image: shreesakhii/python-web-app:v2
          ports:
            - containerPort: 8000
```

Apply the Deployment:

```bash
kubectl apply -f deployment.yaml
```

Verify:

```bash
kubectl get deployments
kubectl get pods -o wide
kubectl get pods --show-labels
kubectl get all
```

### Expected result

- Deployment shows three desired and available replicas.
- All three Pods reach `Running` and `Ready`.
- The Pods use image `shreesakhii/python-web-app:v2`.
- The application listens on container port `8000`.

`containerPort: 8000` documents the port used by the container; it does not itself start the application. The Docker image's command starts the Django server.

## 8. Expose the Deployment with a Service

File: `python-app-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: python-app-service
spec:
  type: NodePort
  selector:
    app: pyhton-app
  ports:
    - port: 8000
      targetPort: 8000
      nodePort: 30080
      protocol: TCP
```

Apply and inspect:

```bash
kubectl apply -f python-app-service.yaml
kubectl get services
kubectl describe service python-app-service
```

### How a Service works

Pods are replaceable, and their IP addresses can change when they are recreated. A Service provides a stable endpoint for clients and routes traffic to matching backend Pods. Its selector must match the labels on the Pods.

### Understand the ports

| Setting | Meaning |
|---|---|
| `port: 8000` | Port exposed by the Service inside the cluster |
| `targetPort: 8000` | Destination port on the application Pod |
| `nodePort: 30080` | Port exposed through the Kubernetes node's networking interface, subject to the cluster's network setup |
| `protocol: TCP` | Transport protocol used by this port |

### Common Service types

| Type | Purpose |
|---|---|
| `ClusterIP` | Internal cluster access; default Service type |
| `NodePort` | Exposes the Service through a port on each node |
| `LoadBalancer` | Requests an external load balancer when the environment/provider supports it |
| `ExternalName` | Maps a Service name to an external DNS name; does not proxy traffic itself |

## 9. Verify Service Endpoints

Run:

```bash
kubectl get pods --show-labels
kubectl get endpointslices
kubectl describe service python-app-service
```

After the selector was corrected, the EndpointSlice showed three endpoints. The IPs observed during this run were:

```text
10.244.0.27:8000
10.244.0.28:8000
10.244.0.29:8000
```

Pod IPs are dynamic. Use the current command output rather than assuming these addresses will remain the same.

A populated EndpointSlice shows that the Service has discovered backend Pods. It does not, by itself, prove that the application returns a successful HTTP response; test the request path as well.

## 10. Test with Port Forwarding

### Terminal 1: Start port forwarding

On the EC2 host where `kubectl` is configured, run this and leave it running:

```bash
kubectl port-forward service/python-app-service 8080:8000
```

Expected output resembles:

```text
Forwarding from 127.0.0.1:8080 -> 8000
Forwarding from [::1]:8080 -> 8000
```

### Terminal 2: Send an HTTP request

Open a second terminal on the same EC2 host:

```bash
curl -v http://127.0.0.1:8080
```

The returned content depends on the Django URL configuration. Keep Terminal 1 open while testing, then stop the port-forward process with `Ctrl+C`.

**Important:** The port-forward listener runs on the machine where the `kubectl port-forward` command was started. If it was started on the EC2 host, running `curl http://127.0.0.1:8080` inside `minikube ssh` will test the Minikube node's loopback interface instead, not the EC2 host's listener.

## 11. Troubleshooting Performed

### Problem A: Django module missing

**Observed error:**

```text
ModuleNotFoundError: No module named 'django'
```

**Cause:** Django was not installed in the container's Python environment.

**Fix:** Update the Dockerfile to install Django in the virtual environment, rebuild the image with the `v2` tag, and load the image into Minikube.

```bash
docker build -t shreesakhii/python-web-app:v2 .
minikube image load shreesakhii/python-web-app:v2
```

### Problem B: Service had no endpoints

**Observed:** The Service existed, but `kubectl describe service python-app-service` showed no endpoints and the EndpointSlice had no backend addresses.

**Cause:** The Service selector and Pod labels did not match:

```text
Service selector: app=python-app
Pod label:        app=pyhton-app
```

Kubernetes selectors require exact label matches.

**Fix:** Set the Service selector to match the Pod-template label:

```yaml
selector:
  app: pyhton-app
```

Then apply and verify:

```bash
kubectl apply -f python-app-service.yaml
kubectl get pods --show-labels
kubectl get endpointslices
kubectl describe service python-app-service
```

The EndpointSlice subsequently showed all three Pod IPs on port `8000`.

### Problem C: NodePort request failed from the EC2 host

A request to the Minikube node IP and NodePort failed:

```bash
curl http://192.168.49.2:30080
```

**Important distinction:** This failure alone does not prove the application is broken. Minikube's node IP may not be directly reachable from the EC2 host, depending on the Minikube driver and network configuration. First verify Service endpoints and use port forwarding to validate the application; troubleshoot direct NodePort reachability separately.

### Problem D: Port-forward test ran in the wrong terminal

The port-forward command was started and immediately stopped with `Ctrl+C`. A subsequent curl was run inside `minikube ssh`.

**Fix:** Start `kubectl port-forward service/python-app-service 8080:8000` on the EC2 host and leave it running. Run curl from a second terminal on that same host.

## 12. Useful Commands

```bash
# All standard resources
kubectl get all

# Deployment status
kubectl get deployment pyhton-app
kubectl describe deployment pyhton-app
kubectl rollout status deployment/pyhton-app
kubectl rollout history deployment/pyhton-app

# Pods, labels, and logs
kubectl get pods -o wide
kubectl get pods --show-labels
kubectl logs <POD_NAME>
kubectl describe pod <POD_NAME>

# Service and backend endpoints
kubectl get services
kubectl describe service python-app-service
kubectl get endpointslices
```

Replace `<POD_NAME>` with a real Pod name from `kubectl get pods`.

## 13. Practice Tasks

- [ ] Scale from three replicas to five and check the endpoints:
  ```bash
  kubectl scale deployment pyhton-app --replicas=5
  kubectl get pods
  kubectl get endpointslices
  ```
- [ ] Delete one Pod and observe the replacement:
  ```bash
  kubectl delete pod <POD_NAME>
  kubectl get pods -w
  ```
  Stop watching with `Ctrl+C` after the replacement appears.
- [ ] Verify that EndpointSlices update when Pods are recreated.
- [ ] Practice a rolling update and rollback.
- [ ] Compare `ClusterIP`, `NodePort`, `LoadBalancer`, and `ExternalName`.
- [ ] Learn Ingress and how it differs from a Service.
- [ ] Investigate NodePort access from outside Minikube.

## 14. Official References

- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Debugging Services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)
- [Minikube Getting Started](https://minikube.sigs.k8s.io/docs/start/)
- [kubectl Reference](https://kubernetes.io/docs/reference/kubectl/)
- [Upstream Python web application example](https://github.com/iam-veeramalla/Docker-Zero-to-Hero/tree/main/examples/python-web-app)

## 15. Lab Summary

In this lab, I used an existing Python/Django application, built the Docker image `shreesakhii/python-web-app:v2`, loaded it into Minikube, and deployed it using a Kubernetes Deployment with three replicas. I exposed the Pods through a NodePort Service, diagnosed and fixed a Service selector mismatch, verified the EndpointSlices, and learned how to test the Service using `kubectl port-forward`.

**Key lesson:** Troubleshoot application connectivity layer by layer: check Pod readiness and labels, verify the Service selector and EndpointSlices, then test the HTTP response.
