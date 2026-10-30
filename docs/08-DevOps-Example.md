# Kubernetes & End-to-End DevOps Examples

A practical learning repository for Kubernetes, DevOps, CI/CD, and SRE
examples. Each example should document the complete workflow:
prerequisites, architecture, installation, deployment, verification,
troubleshooting, and cleanup.

-   **Learning repository:** https://github.com/Rjwagh2001/K8s
-   **CI/CD application repository:**
    https://github.com/Rjwagh2001/Jenkins-Zero-To-Hero

> These are learning labs, not production deployment instructions. Never
> commit passwords, access tokens, private keys, kubeconfigs, or secret
> values.

## Repository goals

This repository collects repeatable, end-to-end examples rather than
isolated command notes. Each example should explain:

1.  The problem and architecture.
2.  Prerequisites and infrastructure.
3.  Installation and configuration.
4.  How the application moves from source code to a running workload.
5.  Verification commands and expected results.
6.  Common errors and how to troubleshoot them.
7.  Cleanup and cost considerations.

## Suggested repository structure

``` text
K8s/
├── README.md
├── examples/
│   ├── 01-minikube-basics/
│   │   ├── README.md
│   │   └── manifests/
│   ├── 02-pods-deployments-services/
│   │   ├── README.md
│   │   └── manifests/
│   ├── 03-ingress/
│   │   ├── README.md
│   │   └── manifests/
│   ├── 04-prometheus-grafana/
│   │   └── README.md
│   └── 05-jenkins-docker-argocd-spring-boot/
│       ├── README.md
│       └── manifests/
└── docs/
    ├── troubleshooting.md
    └── commands-cheatsheet.md
```

Keep each example self-contained. Add it to the root README index only
after its files and instructions exist and have been tested.

------------------------------------------------------------------------

# Example 05: Jenkins + Docker + SonarQube + Argo CD + Kubernetes

## Objective

Build a Spring Boot application, analyze its code, package it into a
Docker image, push the image to Docker Hub, update a Kubernetes manifest
in GitHub, and let Argo CD reconcile the Git change into Kubernetes.

## Architecture

``` text
Developer pushes code
        |
        v
      GitHub
        |
        v
      Jenkins
        |
        +--> Maven clean package
        +--> SonarQube analysis
        +--> Docker build
        +--> Push versioned image to Docker Hub
        +--> Update deployment manifest in GitHub
                                  |
                                  v
                               Argo CD
                                  |
                                  v
                         Kubernetes / Minikube
                                  |
                                  v
                       Spring Boot application pods
```

Repositories used:

-   Source code and pipeline:
    https://github.com/Rjwagh2001/Jenkins-Zero-To-Hero
-   Learning examples: https://github.com/Rjwagh2001/K8s

The CI/CD repository paths used in this lab:

``` text
java-maven-sonar-argocd-helm-k8s/
├── spring-boot-app/
│   ├── pom.xml
│   ├── Dockerfile
│   └── JenkinsFile
└── spring-boot-app-manifests/
    ├── deployment.yml
    └── service.yml
```

Check the repository for exact current filenames before running
commands.

## Environment used

-   Ubuntu EC2 instance, approximately 4 GiB RAM and 30 GiB disk
-   Docker Engine
-   kubectl
-   Minikube with Docker driver
-   Single-node Kubernetes cluster
-   Argo CD in the `argocd` namespace
-   Jenkins pipeline using Maven, SonarQube, and Docker Hub
-   Spring Boot 3.4.x application compiled for Java 21
-   Docker Hub image: `shreesakhii/ultimate-cicd`
-   SonarQube endpoint used in the lab: `http://3.111.217.115:9000`

The IP address is lab-specific and may change. Replace it with your
current address. Do not expose administrative services publicly without
access controls.

## Prerequisites

-   Ubuntu EC2 instance with sufficient memory and disk
-   SSH access and private key
-   Docker Hub account and image repository
-   GitHub repository and permission to push manifest changes
-   Jenkins with Git, Pipeline, Docker Pipeline, and credentials support
-   SonarQube reachable from Jenkins
-   Basic Linux, Git, Docker, YAML, and Kubernetes knowledge

Follow the official installation instructions for your Ubuntu release
and desired versions.

## Step 1: Connect to EC2 and prepare Ubuntu

From your local machine:

``` bash
ssh -i "/path/to/your-key.pem" ubuntu@YOUR_EC2_PUBLIC_IP
```

Update the system and install utilities:

``` bash
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install -y curl wget git ca-certificates gnupg
```

Check available resources:

``` bash
free -h
df -h
nproc
```

A small instance may struggle when Jenkins, SonarQube, Minikube, and
application pods run together. Monitor resource use or choose a larger
instance if necessary.

## Step 2: Install and verify Docker

Install Docker Engine using the official Docker instructions for Ubuntu.
Then:

``` bash
sudo systemctl enable --now docker
sudo systemctl status docker
docker --version
docker run --rm hello-world
```

If you need to run Docker without `sudo`:

``` bash
sudo usermod -aG docker "$USER"
```

Log out and reconnect for the group membership to take effect.
Membership in the `docker` group grants powerful privileges, so only
grant it to trusted users.

If Docker is already installed, verify it rather than reinstalling it.

## Step 3: Install kubectl

Install `kubectl` using the official Kubernetes instructions for your OS
and architecture. Verify:

``` bash
kubectl version --client
```

Keep the client version compatible with the cluster.

## Step 4: Install Minikube and start Kubernetes

Install Minikube using its official instructions, then start the Docker
driver:

``` bash
minikube start --driver=docker
```

Verify:

``` bash
minikube status
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A
```

The node should reach `Ready`. If Minikube reports insufficient memory,
check `free -h` and adjust the lab architecture or instance size instead
of allocating most of the host's RAM blindly.

## Step 5: Install Argo CD

Create the namespace and install Argo CD:

``` bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for components:

``` bash
kubectl get pods -n argocd
kubectl rollout status deployment/argocd-server -n argocd
```

For reproducible environments, pin a reviewed Argo CD release rather
than relying on a moving `stable` manifest.

## Step 6: Access Argo CD securely

On EC2, start the port forward:

``` bash
kubectl port-forward -n argocd svc/argocd-server 8081:443
```

Keep it running. On Windows, open another PowerShell terminal:

``` powershell
ssh -i "C:\path\to\your-key.pem" -N -L 8081:127.0.0.1:8081 ubuntu@YOUR_EC2_PUBLIC_IP
```

Open:

``` text
https://localhost:8081
```

A self-signed certificate warning may occur in this lab. It is not a
production TLS setup.

Retrieve the initial admin password if needed:

``` bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
echo
```

The initial username is `admin`. Change the initial password and follow
Argo CD security guidance. Never commit credentials to Git.

## Step 7: Configure Jenkins credentials

In **Manage Jenkins → Credentials**, configure the IDs used by the
pipeline:

  -----------------------------------------------------------------------
  Credential ID           Type                    Purpose
  ----------------------- ----------------------- -----------------------
  `sonarqube`             Secret text             SonarQube token

  `docker-cred`           Username/password       Docker Hub login

  `github`                Secret text in the lab  GitHub token for
                          pipeline                manifest push

  `GithubUsername`        Git credentials, if     Repository checkout
                          used by SCM             
  -----------------------------------------------------------------------

Credential IDs are case-sensitive. Use the credential type expected by
the Jenkinsfile. Do not hardcode tokens in source code.

The Jenkins job used in this lab was `EndToEndDeploy`. Its pipeline
script path was:

``` text
java-maven-sonar-argocd-helm-k8s/spring-boot-app/JenkinsFile
```

Check the job's SCM configuration for the correct repository, branch,
and script path.

## Step 8: Build the Spring Boot application

The Maven project directory is:

``` text
java-maven-sonar-argocd-helm-k8s/spring-boot-app
```

The project uses Java 21. Check `pom.xml` for:

``` xml
<java.version>21</java.version>
```

Build the JAR:

``` bash
cd java-maven-sonar-argocd-helm-k8s/spring-boot-app
mvn clean package
```

Expected artifact:

``` text
target/spring-boot-web.jar
```

Jenkins runs this build in the pipeline. Look for `BUILD SUCCESS`.

## Step 9: Configure the Dockerfile

The runtime image must support the Java version used to compile the
application. The working lab Dockerfile used Java 21:

``` dockerfile
FROM eclipse-temurin:21-jre

WORKDIR /opt/app

ARG artifact=target/spring-boot-web.jar

COPY ${artifact} app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

`EXPOSE` documents the container port; Kubernetes Service configuration
is separate.

Optional manual test from the application directory:

``` bash
docker build -t shreesakhii/ultimate-cicd:local .
docker run --rm -p 8080:8080 shreesakhii/ultimate-cicd:local
```

The Jenkins pipeline uses build-number tags for deployment rather than
the `local` tag.

## Step 10: Configure Kubernetes manifests

The manifest directory is:

``` text
java-maven-sonar-argocd-helm-k8s/spring-boot-app-manifests/
```

### Deployment

The Deployment should run two replicas and use the image tag published
by Jenkins. Its key settings resemble:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: spring-boot-app
  template:
    metadata:
      labels:
        app: spring-boot-app
    spec:
      containers:
        - name: spring-boot-app
          image: shreesakhii/ultimate-cicd:18
          ports:
            - containerPort: 8080
```

Tag `18` was observed during this lab. The CI pipeline should update it
for each subsequent build.

### Service

A ClusterIP Service can expose the application inside the cluster and
work with `kubectl port-forward`:

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: spring-boot-app-service
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: spring-boot-app
  ports:
    - port: 80
      targetPort: 8080
```

Use the actual Service manifest in your repository if it differs. Ensure
its selector matches the pod labels and the target port matches Spring
Boot's port `8080`.

## Step 11: Configure the Jenkins pipeline

The pipeline stages are:

1.  Checkout source code.
2.  Run `mvn clean package`.
3.  Run SonarQube analysis.
4.  Build and push `shreesakhii/ultimate-cicd:${BUILD_NUMBER}`.
5.  Update the image tag in `deployment.yml`.
6.  Commit and push the manifest change to GitHub `main`.

The manifest-update command must match the actual image repository:

``` bash
MANIFEST="java-maven-sonar-argocd-helm-k8s/spring-boot-app-manifests/deployment.yml"

sed -i -E \
  's|(image:[[:space:]]*shreesakhii/ultimate-cicd:).*|\1'"${BUILD_NUMBER}"'|' \
  "$MANIFEST"

grep -n 'image:' "$MANIFEST"
```

Stage and commit only if there is a change:

``` bash
git add "$MANIFEST"

if ! git diff --cached --quiet; then
    git commit -m "Update deployment image to version ${BUILD_NUMBER}"
    # Push using the repository's configured credential-safe authentication.
else
    echo "No manifest changes to commit."
fi
```

Keep shell commands on separate lines inside the Jenkins multiline `sh`
block. A previous failure showed `sed` treating `git add` as a filename
because both commands were accidentally on one line.

Prefer Jenkins Git credentials or another credential-safe push method
instead of embedding a token in a remote URL. Ensure the credential has
permission to push to the intended branch.

## Step 12: Configure the Argo CD application

The lab application name is:

``` text
spring-boot-cicd
```

Expected source configuration:

-   Repository: `https://github.com/Rjwagh2001/Jenkins-Zero-To-Hero.git`
-   Target revision: `main`
-   Path: `java-maven-sonar-argocd-helm-k8s/spring-boot-app-manifests`
-   Destination: in-cluster Kubernetes API
-   Namespace: `default`

**The source path must be the directory containing the YAML manifests,
not the `deployment.yml` file itself.**

Automated sync may be enabled for this lab. Review pruning before
enabling it because it can delete resources removed from Git. If
automated sync is disabled, manually sync the application after Jenkins
updates the manifest.

Check:

``` bash
kubectl get applications -n argocd
```

## Step 13: Verify the deployment

Run:

``` bash
kubectl get applications -n argocd
kubectl get deployment spring-boot-app -n default
kubectl get replicasets -n default
kubectl get pods -n default -o wide
kubectl get svc spring-boot-app-service -n default
kubectl get endpoints spring-boot-app-service -n default
```

Check the image configured in the Deployment:

``` bash
kubectl get deployment spring-boot-app -n default \
  -o jsonpath='{.spec.template.spec.containers[*].image}'
echo
```

Wait for rollout completion:

``` bash
kubectl rollout status deployment/spring-boot-app -n default
```

Expected: two ready replicas and healthy application pods.

Inspect application logs:

``` bash
kubectl logs -n default deployment/spring-boot-app --all-pods=true
```

The successful lab logs showed messages similar to:

``` text
Tomcat started on port 8080 (http)
Started StartApplication
```

The application eventually ran with image
`shreesakhii/ultimate-cicd:18`, Java 21, and two ready pods. Earlier
pods crashed because an older image used Java 11, which could not load
the newer Java bytecode.

## Step 14: Access the app using port 9091

On EC2, forward local port `9091` to Service port `80`:

``` bash
kubectl port-forward -n default svc/spring-boot-app-service 9091:80
```

Keep the command running. In another EC2 terminal:

``` bash
curl -i http://127.0.0.1:9091/
```

To access it from Windows, open another PowerShell terminal:

``` powershell
ssh -i "C:\path\to\your-key.pem" -N -L 9091:127.0.0.1:9091 ubuntu@YOUR_EC2_PUBLIC_IP
```

Then open:

``` text
http://localhost:9091
```

Port flow:

``` text
Windows browser :9091
       |
       v
SSH tunnel to EC2 :9091
       |
       v
kubectl port-forward
       |
       v
Service port 80
       |
       v
Spring Boot container port 8080
```

Both the EC2 port-forward process and the Windows SSH tunnel must remain
running. This is temporary access, not a permanent public endpoint.

------------------------------------------------------------------------

## Troubleshooting

### Java `UnsupportedClassVersionError`

Example:

``` text
class file version 61.0
runtime only recognizes class file versions up to 55.0
```

Class version 55 is Java 11 and version 61 is Java 17. This error
indicates a runtime older than the application's compiled bytecode. This
project uses Java 21, so use a Java 21 runtime in the Docker image.

Check the image and ReplicaSets:

``` bash
kubectl get deployment spring-boot-app -n default \
  -o jsonpath='{.spec.template.spec.containers[*].image}'
echo

kubectl get replicasets -n default \
  -o custom-columns='NAME:.metadata.name,IMAGE:.spec.template.spec.containers[*].image,READY:.status.readyReplicas'

kubectl logs -n default deployment/spring-boot-app --all-pods=true
```

A successful Docker push alone does not deploy the new image. Confirm
that the manifest was updated, pushed to GitHub, and reconciled by Argo
CD.

### `CrashLoopBackOff`

``` bash
kubectl get pods -n default
kubectl logs -n default deployment/spring-boot-app --all-pods=true
kubectl describe deployment spring-boot-app -n default
kubectl get events -n default --sort-by=.lastTimestamp
```

For logs from a previously crashed container:

``` bash
kubectl logs POD_NAME -n default --previous
```

### Jenkins fails at `git commit`

Inspect changes:

``` bash
git diff -- path/to/deployment.yml
git diff --cached
```

Make sure the image replacement matches the actual repository name and
that the shell commands are separated. Skip the commit when there are no
changes.

### Argo CD is `Synced` but `Progressing`

`Synced` means the live resources match Git's desired configuration; it
does not guarantee application health.

``` bash
kubectl get deployment spring-boot-app -n default
kubectl get pods -n default
kubectl describe deployment spring-boot-app -n default
```

### Service has no endpoints

``` bash
kubectl describe svc spring-boot-app-service -n default
kubectl get pods -n default --show-labels
kubectl get endpoints spring-boot-app-service -n default
```

Check that the Service selector matches the pod labels and its target
port is `8080`.

### Port forwarding fails

``` bash
kubectl get svc spring-boot-app-service -n default
kubectl get endpoints spring-boot-app-service -n default
kubectl get pods -n default
```

Keep the port-forward command running. If accessing through Windows,
keep the SSH tunnel running too.

### Repository path not found on EC2

A GitHub repository is not automatically cloned into `/home/ubuntu`.
Jenkins checks out code in its own workspace. In this lab, the workspace
was:

``` text
/var/lib/jenkins/workspace/EndToEndDeploy
```

Check before accessing files:

``` bash
sudo ls -la /var/lib/jenkins/workspace/EndToEndDeploy
```

Use the appropriate working directory for each command.

------------------------------------------------------------------------

## Security and cost checklist

-   Never commit `.pem` files, tokens, passwords, kubeconfigs, or
    Kubernetes Secret values.
-   Store credentials in Jenkins Credentials.
-   Avoid exposing Jenkins, SonarQube, Argo CD, or Kubernetes API ports
    publicly without authentication and network restrictions.
-   Use SSH tunnels for lab-only dashboard access when appropriate.
-   Pin tool and container versions when reproducibility matters.
-   Rotate any token that may have been exposed.
-   Stop or terminate unused EC2 resources and review EBS and public
    IPv4 charges.
-   Minikube on a small EC2 instance is a learning setup, not a
    production Kubernetes architecture.
-   Check the impact before running destructive cleanup commands against
    a namespace.

## How to add future examples

For every new example:

1.  Create a numbered folder under `examples/`, for example
    `examples/06-configmaps-secrets/`.
2.  Add a `README.md` containing goal, architecture, prerequisites,
    commands, expected output, troubleshooting, and cleanup.
3.  Store YAML files in `manifests/` where appropriate.
4.  Use placeholders for secrets and environment-specific IPs.
5.  Include verification commands.
6.  Add the example to the root README index.
7.  Test the steps from a clean environment where practical.
8.  Record software versions and known limitations.

Suggested index format:

``` markdown
## Examples

| Example | Topics | Documentation |
|---|---|---|
| 01 | Minikube basics | [Open](examples/01-minikube-basics/README.md) |
| 02 | Pods, Deployments, Services | [Open](examples/02-pods-deployments-services/README.md) |
| 03 | Ingress | [Open](examples/03-ingress/README.md) |
| 04 | Prometheus and Grafana | [Open](examples/04-prometheus-grafana/README.md) |
| 05 | Jenkins, Docker, Argo CD, Spring Boot | [Open](examples/05-jenkins-docker-argocd-spring-boot/README.md) |
```

Only list examples that have actually been created and tested.

## Useful commands

``` bash
# Kubernetes overview
kubectl get nodes -o wide
kubectl get pods -A
kubectl get deployments -n default
kubectl get svc -n default

# Application
kubectl rollout status deployment/spring-boot-app -n default
kubectl rollout history deployment/spring-boot-app -n default
kubectl describe deployment spring-boot-app -n default
kubectl logs -n default deployment/spring-boot-app --all-pods=true

# Argo CD
kubectl get pods -n argocd
kubectl get applications -n argocd

# Minikube
minikube status
minikube ip
minikube dashboard

# Docker
docker images
docker ps
docker system df
```

## Official learning resources

-   [Kubernetes](https://kubernetes.io/docs/)
-   [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
-   [Services](https://kubernetes.io/docs/concepts/services-networking/service/)
-   [Minikube](https://minikube.sigs.k8s.io/docs/)
-   [Argo CD](https://argo-cd.readthedocs.io/)
-   [Docker](https://docs.docker.com/)
-   [Spring Boot](https://docs.spring.io/spring-boot/index.html)
-   [Jenkins](https://www.jenkins.io/doc/)
-   [SonarQube](https://docs.sonarsource.com/sonarqube-server/)

## Completion checklist for this lab

-   [x] Maven package builds successfully.
-   [x] SonarQube analysis completes successfully.
-   [x] Docker image is pushed to Docker Hub.
-   [x] Argo CD reconciles the deployment manifest.
-   [x] Spring Boot starts with Java 21 in Kubernetes.
-   [x] Two application pods reached `1/1 Running` during the lab.
-   [ ] Recheck Jenkins manifest update on a fresh build.
-   [ ] Verify Service endpoints and test the app through port `9091`.
-   [ ] Confirm Argo CD reports `Synced` and `Healthy`.

Update this checklist after repeating the lab; a previous successful run
does not guarantee the current environment is healthy.
