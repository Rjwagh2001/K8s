# Kubernetes Learning Lab

A practical, step-by-step Kubernetes learning repository. It records the
setup, commands, explanations, troubleshooting notes, and reusable
manifests so that the work can be repeated on a new machine and extended
over time.

> **Current environment:** AWS EC2, Red Hat Enterprise Linux 10.2,
> Docker Engine, Minikube, and `kubectl`.
>
> **Current milestone:** A standalone `nginx-pod` is running in
> Minikube, and an HTTP request through `kubectl port-forward` returned
> the NGINX welcome page.

## Goals

-   Keep a reproducible record of how the lab was installed.
-   Understand each Kubernetes resource rather than only copying
    commands.
-   Save reusable YAML manifests in Git.
-   Record troubleshooting and verification steps.
-   Add new lessons without losing the history of previous work.

## Repository map

``` text
k8s-learning-lab/
├── README.md
├── .gitignore
├── docs/
│   ├── 01-ec2-and-minikube-setup.md
│   ├── 02-first-nginx-pod.md
│   └── LEARNING-LOG.md
└── manifests/
    └── nginx-pod.yaml
```

## Current lab architecture

``` text
AWS EC2 instance (RHEL)
└── Docker Engine
    └── Minikube container (Kubernetes node)
        ├── Kubernetes control-plane components
        └── nginx-pod
            └── NGINX container (port 80)
```

`kubectl` is the command-line client used to send requests to the
Kubernetes API server. `minikube` creates and manages the local learning
cluster. In this lab, the Minikube node itself runs as a Docker
container on EC2.

## Quick status checks

Run these on the EC2 instance as `ec2-user`:

``` bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A
kubectl get pods -o wide
kubectl get deployments,services
```

The node should eventually show `Ready`. The first Pod should show
`1/1 Running`.

## Re-create the first Pod

For the initial hands-on exercise:

``` bash
kubectl run nginx-pod --image=nginx:alpine
kubectl get pods -o wide
kubectl describe pod nginx-pod
```

To test the web server, keep this running in one terminal:

``` bash
kubectl port-forward pod/nginx-pod 8080:80
```

Open a second SSH session to the EC2 instance and run:

``` bash
curl http://127.0.0.1:8080
```

Expected result: HTML containing `Welcome to nginx!`.

Stop port forwarding with `Ctrl+C` in the terminal where it is running.
Port forwarding is temporary; it does not create a permanent Service or
make the app publicly available.

## Important: command-created resource vs YAML

The first Pod was created imperatively with `kubectl run`. A reusable
manifest is also included at
[`manifests/nginx-pod.yaml`](manifests/nginx-pod.yaml). To create the
Pod from the manifest instead, use:

``` bash
kubectl apply -f manifests/nginx-pod.yaml
```

Do not create both methods with different names unless you intentionally
want two Pods. If `nginx-pod` already exists, applying the matching
manifest updates/reconciles that resource.

## How to continue learning

Follow this progression, adding notes and manifests as you go:

1.  **Pod** --- inspect, logs, events, delete, recreate.
2.  **Deployment** --- create a Deployment and understand desired
    replicas and self-healing.
3.  **Scaling and rollout** --- scale replicas, update an image, inspect
    rollout history, undo a rollout.
4.  **Service** --- expose a Deployment inside the cluster and compare
    `ClusterIP`, `NodePort`, and `LoadBalancer`.
5.  **Configuration** --- ConfigMaps, Secrets, environment variables.
6.  **Storage** --- volumes, PersistentVolumes, PersistentVolumeClaims.
7.  **Health and resources** --- probes, requests, limits, namespaces,
    labels, selectors.
8.  **Networking** --- DNS, Service discovery, NetworkPolicy, Ingress.
9.  **Troubleshooting** --- `describe`, logs, events, resource status,
    scheduling failures.
10. **Production concepts** --- RBAC, security context, image security,
    observability, upgrades, backup and recovery.

For each lesson, add: - What problem the resource solves. - A small
architecture diagram or flow. - Commands to create and inspect it. -
Expected output or success criteria. - Troubleshooting steps. - Cleanup
commands. - A reusable YAML file where appropriate.

## Save this project to GitHub

### 1. Create a repository

On GitHub, create a new repository named `k8s-learning-lab`. Choose
Public if you want other developers to learn from it; choose Private if
you prefer to keep it private. Do not add AWS private keys, passwords,
tokens, kubeconfig files, or other secrets.

### 2. Initialize and push from your computer

Download/extract this starter repository, open a terminal in the
extracted `k8s-learning-repo` directory, and run:

``` bash
git init
git add README.md .gitignore docs manifests
git commit -m "docs: add Kubernetes learning lab setup"
git branch -M main
git remote add origin https://github.com/YOUR_GITHUB_USERNAME/k8s-learning-lab.git
git push -u origin main
```

Replace `YOUR_GITHUB_USERNAME` with your GitHub username. If Git says
the remote already exists, inspect it with `git remote -v` instead of
adding it again.

### 3. Record every future lesson

After completing a new exercise:

``` bash
git status
git add README.md docs manifests
git commit -m "docs: document Kubernetes deployment and service"
git push
```

Use a commit message that describes the learning milestone. Check
`git status` before committing and review files before pushing.

## EC2 safety and cost notes

-   Minikube is a learning cluster, not a production Kubernetes
    deployment.
-   The EC2 instance has 2 vCPUs and about 3.5 GiB RAM. Minikube was
    configured with 3 GiB, which leaves limited memory for the host. If
    the instance becomes unstable, reduce Minikube memory or use a
    larger instance.
-   Stop or delete cloud resources when not needed, and check the AWS
    billing dashboard. Do not assume an instance is free just because an
    account has credits or a free-tier offer.
-   Keep SSH restricted to your own trusted IP where practical.
-   Do not open port 8080 publicly for this port-forward exercise.
-   The Red Hat registration reminder is separate from Kubernetes. It
    did not prevent this lab from running.

## Learning log

Use [`docs/LEARNING-LOG.md`](docs/LEARNING-LOG.md) to append each
milestone, the exact commands used, results, and questions to revisit.
