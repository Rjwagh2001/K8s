# Kubernetes Learning Lab

A practical, reproducible Kubernetes learning repository built from
hands-on work on AWS EC2. Each lab records prerequisites, setup steps,
commands, what they do, issues encountered, troubleshooting, validation,
and cleanup.

## Current milestone

-   Created an AWS EC2 instance running Red Hat Enterprise Linux 10.2.
-   Installed and verified `kubectl`, Docker Engine, and Minikube.
-   Started a single-node Minikube cluster using the Docker driver.
-   Created `nginx-pod` using `nginx:alpine`.
-   Verified the Pod was `1/1 Running`.
-   Forwarded EC2 loopback port `8080` to Pod port `80`.
-   Confirmed `curl http://127.0.0.1:8080` returned the NGINX welcome
    HTML.

## Repository structure

``` text
K8s/
├── README.md
├── .gitignore
├── docs/
│   ├── 00-prerequisites-and-ec2-creation.md
│   ├── 01-ec2-kubectl-docker-minikube-setup.md
│   ├── 02-first-nginx-pod.md
│   └── LEARNING-LOG.md
└── manifests/
    └── nginx-pod.yaml
```

## Learning path

  ------------------------------------------------------------------------
  Lab                     Topic                    Status
  ----------------------- ------------------------ -----------------------
  00                      Prerequisites and AWS    Documented
                          EC2 creation             

  01                      `kubectl`, Docker,       Documented
                          Minikube installation    
                          and troubleshooting      

  02                      First NGINX Pod and HTTP Completed
                          test                     

  03                      Deployments, replicas,   Next
                          self-healing             

  04                      Services and service     Planned
                          discovery                

  05                      ConfigMaps and Secrets   Planned

  06                      Storage and              Planned
                          PersistentVolumeClaims   

  07                      Probes, resource         Planned
                          requests/limits,         
                          namespaces               

  08                      Networking, DNS, Ingress Planned

  09                      Troubleshooting and      Planned
                          observability            

  10                      Security and             Planned
                          production-readiness     
                          concepts                 
  ------------------------------------------------------------------------

## How to use this repository

1.  Read the lab prerequisites before creating cloud resources.
2.  Follow commands in order and understand the expected result.
3.  Save your actual outputs and any new issues in
    `docs/LEARNING-LOG.md`; redact secrets and sensitive details.
4.  Keep reusable Kubernetes YAML in `manifests/`.
5.  Commit each meaningful learning milestone to Git.

## Quick verification commands

Run on the EC2 instance:

``` bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A
kubectl get pods -o wide
kubectl get deployments,services
```

## Recreate the NGINX Pod from YAML

From the repository root on a machine where `kubectl` is configured for
the intended cluster:

``` bash
kubectl apply -f manifests/nginx-pod.yaml
kubectl get pod nginx-pod -o wide
```

Test it using port-forwarding in one terminal:

``` bash
kubectl port-forward pod/nginx-pod 8080:80
```

In a second terminal on the same EC2 host:

``` bash
curl http://127.0.0.1:8080
```

Expected: HTML containing `Welcome to nginx!`. Stop port-forwarding with
`Ctrl+C`. It is temporary and does not expose the application publicly.

## Git workflow

After changing documentation or manifests:

``` bash
git status
git diff
git add README.md .gitignore docs/ manifests/
git commit -m "docs: describe the next Kubernetes lab"
git push
```

Review `git status` and `git diff` before committing. Do not commit SSH
private keys, AWS credentials, tokens, kubeconfig files, or other
secrets.

## Important limitations and safety

-   This is a single-node Minikube learning environment, not a
    production cluster.
-   The recorded EC2 host has 2 vCPUs and about 3.5 GiB RAM. Minikube
    was configured with 3072 MiB, leaving little memory for the host. If
    memory pressure appears, reduce Minikube memory or choose a larger
    instance.
-   AWS prices, free-tier eligibility, credits, instance availability,
    and console labels can change. Check the AWS console and billing
    dashboard before launching or leaving resources running.
-   Keep SSH access limited to trusted source IPs where practical.
-   Do not open port 8080 publicly for the port-forward exercise.
-   The Red Hat registration reminder seen at login was not blocking the
    lab.
