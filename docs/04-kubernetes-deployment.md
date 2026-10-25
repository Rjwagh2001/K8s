# Kubernetes Deployment --- Hands-on Lab

> Personal Kubernetes learning notes based on the hands-on lab completed
> on an Amazon EC2 instance using Minikube, Docker, `kubectl`, and
> NGINX.

## Official documentation

Primary reference: [Kubernetes
Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

The official documentation explains Deployment behavior, updates,
rollbacks, and how Deployments manage ReplicaSets and Pods. Use the
examples below as a practical companion, then explore the official
tutorials and example repository linked at the end.

- **Kubernetes examples repository:** https://github.com/kubernetes/examples
  - Browse practical example applications and configuration files.
  - Use these examples to explore how Kubernetes resources are organized
    and applied; check each example's README and requirements before
    running it.

## Lab environment

-   Host: Amazon EC2 running Red Hat Enterprise Linux
-   Container engine: Docker Engine
-   Local Kubernetes cluster: Minikube using the Docker driver
-   Kubernetes client: `kubectl`
-   Workload: NGINX
-   Namespace: `default`

Check the environment before starting:

``` bash
docker version
minikube status
kubectl get nodes
kubectl config current-context
```

Start the existing Minikube cluster if it is stopped:

``` bash
minikube start --driver=docker
```

Minikube is managed through the `minikube` CLI; it normally does not
install a `minikube.service` systemd unit. `kubectl` is a client, not a
systemd service.

## What is a Deployment?

A Kubernetes Deployment manages a stateless application rollout. It
declares the desired state for Pods and manages ReplicaSets to make the
actual state converge toward that desired state.

A Deployment supports: - Maintaining a desired number of Pod replicas -
Replacing deleted or failed Pods through its ReplicaSet - Scaling the
application up or down - Rolling out Pod-template changes, such as a new
container image - Rolling back to a previous Deployment revision

### Relationship between resources

``` text
Deployment
   |
   v
ReplicaSet
   |
   +---- Pod
   +---- Pod
   +---- Pod
```

-   **Deployment:** owns rollout history and manages ReplicaSets.
-   **ReplicaSet:** works to maintain the desired number of matching
    Pods.
-   **Pod:** runs the container(s).
-   **Service:** a separate resource that can provide a stable network
    endpoint for reaching matching Pods. A Deployment does not
    automatically expose an application externally.

## Deployment manifest used in this lab

The manifest used during the practical was named `ngnix-deployment.yml`
(the filename contains the typo `ngnix`; `nginx-deployment.yml` would be
a clearer name).

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

### Manifest explanation

  -----------------------------------------------------------------------
  Field                               Purpose
  ----------------------------------- -----------------------------------
  `apiVersion: apps/v1`               API group/version for a Deployment.

  `kind: Deployment`                  Declares the Kubernetes resource
                                      type.

  `metadata.name`                     Names the Deployment
                                      `nginx-deployment`.

  `metadata.labels`                   Adds identifying metadata to the
                                      Deployment object.

  `spec.replicas: 3`                  Requests three Pod replicas.

  `spec.selector.matchLabels`         Selects Pods with the label
                                      `app: nginx`.

  `spec.template`                     Blueprint used to create each Pod.

  `template.metadata.labels`          Labels created Pods so they match
                                      the selector.

  `containers.name`                   Container name within each Pod.

  `containers.image`                  Container image used by the Pod.

  `containerPort: 80`                 Documents the port the container is
                                      expected to listen on; it does not
                                      publish the application outside the
                                      cluster.
  -----------------------------------------------------------------------

**Important:** The selector labels must match the Pod-template labels.
In this manifest both use `app: nginx`.

## Practical 1 --- Create the Deployment

From the directory containing the manifest:

``` bash
kubectl apply -f ngnix-deployment.yml
```

Expected confirmation:

``` text
deployment.apps/nginx-deployment created
```

Inspect the resources:

``` bash
kubectl get deployments
kubectl get replicasets
kubectl get pods -o wide
kubectl describe deployment nginx-deployment
```

Expected initial state after Pods become ready: one Deployment, one
ReplicaSet, and three ready Pods.

## Practical 2 --- Test self-healing

List the Pods and select one Pod name:

``` bash
kubectl get pods
```

Delete one Pod managed by the Deployment (replace the example with the
actual Pod name):

``` bash
kubectl delete pod nginx-deployment-<replicaset-hash>-<suffix>
```

Then check again:

``` bash
kubectl get pods -o wide
kubectl get replicasets
```

**Observed in this lab:** after deleting one Deployment-managed Pod,
Kubernetes created a replacement Pod. The Deployment's ReplicaSet worked
to maintain the desired replica count.

Compare with a standalone Pod: if a standalone Pod is deleted and no
controller manages it, a Deployment/ReplicaSet will not recreate it.

## Practical 3 --- Scale from three to five replicas

``` bash
kubectl scale deployment nginx-deployment --replicas=5
```

Verify:

``` bash
kubectl get deployments
kubectl get replicasets
kubectl get pods -o wide
kubectl describe deployment nginx-deployment
```

**Observed in this lab:** Deployment status showed `5/5`
ready/available, and the ReplicaSet showed five desired, current, and
ready replicas.

Note: `kubectl scale` changes the live replica count. The YAML file
still says `replicas: 3`, so the file and live object can differ. If you
later apply that manifest, Kubernetes may return the replica count to
three.

## Practical 4 --- Update the container image

The Deployment initially used `nginx:1.14.2`. Update it to `nginx:1.28`:

``` bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.28
```

Watch the rollout and review history:

``` bash
kubectl rollout status deployment/nginx-deployment
kubectl rollout history deployment/nginx-deployment
kubectl get pods
kubectl describe deployment nginx-deployment
```

A Deployment's rolling-update strategy replaces old Pods with Pods from
the updated template according to its rollout settings. Use a
deliberate, tested image version in real environments; avoid mutable
tags such as `latest` for controlled production releases.

## Practical 5 --- Roll back the image update

``` bash
kubectl rollout undo deployment/nginx-deployment
```

Verify the result rather than relying only on the success message:

``` bash
kubectl rollout status deployment/nginx-deployment
kubectl get deployment nginx-deployment \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
kubectl rollout history deployment/nginx-deployment
kubectl get pods
```

**Observed in this lab:** the image query returned `nginx:1.14.2` after
rollback.

A warning appeared because the Deployment had previously been managed
with `kubectl apply`. The rollback command succeeded, but it does not
rewrite the `kubectl.kubernetes.io/last-applied-configuration`
annotation. Keep your manifest in sync with the intended state and
review it before applying it again.

## Useful inspection and troubleshooting commands

``` bash
# List common resources in the current namespace
kubectl get all

# Get Pods with node and IP information
kubectl get pods -o wide

# Inspect one Pod
kubectl describe pod <pod-name>

# Inspect Deployment events and configuration
kubectl describe deployment nginx-deployment

# Inspect the Deployment's image and replica count
kubectl get deployment nginx-deployment \
  -o jsonpath='Image: {.spec.template.spec.containers[0].image}{"\n"}Replicas: {.spec.replicas}{"\n"}'

# Review rollout history
kubectl rollout history deployment/nginx-deployment

# Wait for the current rollout to finish
kubectl rollout status deployment/nginx-deployment
```

Common mistakes from the lab: - Use `kubectl describe`, not
`kubectl decribe`. - Use `kubectl delete pod <pod-name>`; there is no
general `kubectl stop pod` command. - A `NotFound` error can mean the
Pod was already deleted or the name was mistyped. -
`systemctl status kubectl` and `systemctl status minikube` normally fail
because they are CLI tools, not systemd services. - `kubectl` needs a
reachable Kubernetes API server; Docker running alone does not mean the
Kubernetes cluster is running.

## Declarative vs. imperative management

  ------------------------------------------------------------------------------------------------------------------
  Method                  Example                                                            Use
  ----------------------- ------------------------------------------------------------------ -----------------------
  Declarative             `kubectl apply -f ngnix-deployment.yml`                            Apply configuration
                                                                                             from a YAML manifest.

  Imperative scaling      `kubectl scale deployment nginx-deployment --replicas=5`           Change the live replica
                                                                                             count directly.

  Imperative image update `kubectl set image deployment/nginx-deployment nginx=nginx:1.28`   Change the live
                                                                                             Pod-template image.

  Rollback                `kubectl rollout undo deployment/nginx-deployment`                 Restore a previous
                                                                                             Pod-template revision.
  ------------------------------------------------------------------------------------------------------------------

For repeatable workflows, keep YAML as the source of truth, update it
intentionally, and use version control to review changes.

## Suggested exercises

-   [ ] Create the Deployment from YAML.
-   [ ] Explain `Deployment → ReplicaSet → Pod` in your own words.
-   [ ] Delete one Pod and observe its replacement.
-   [ ] Scale from three to five replicas.
-   [ ] Update the image and verify rollout completion.
-   [ ] Roll back and verify the image version.
-   [ ] Update the YAML so the desired image and replica count match
    your intended state.
-   [ ] Create a Service to access the NGINX Pods.
-   [ ] Add resource requests/limits and readiness/liveness probes in a
    later lab.
-   [ ] Practice declarative changes using `kubectl diff` and
    `kubectl apply`.

## Official documentation and examples to explore

Start with the concept page used for this lab:

1.  [Deployments --- Kubernetes
    Concepts](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
    --- Deployment behavior, ReplicaSets, rolling updates, and rollback.
2.  [Run a Stateless Application Using a
    Deployment](https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/)
    --- create and inspect a Deployment.
3.  [Horizontal Manual Scaling for a
    Deployment](https://kubernetes.io/docs/tasks/run-application/scale-deployment/)
    --- change replica counts.
4.  [Updating a
    Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment)
    --- understand rolling updates.
5.  [Rolling Back a
    Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-back-a-deployment)
    --- inspect and undo revisions.
6.  [Using a Service to Access an
    Application](https://kubernetes.io/docs/tasks/access-application-cluster/service-access-application-cluster/)
    --- expose the workload through a Service.
7.  [Labels and
    Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
    --- understand how controllers identify Pods.

## Lab completion summary

In this lab, I: - Installed and verified Docker, `kubectl`, and Minikube
on EC2. - Started a stopped Minikube cluster and verified that the node
became `Ready`. - Created an NGINX Deployment using a YAML manifest. -
Deleted a managed Pod and observed the ReplicaSet replace it. - Scaled
the Deployment from three to five replicas. - Updated the NGINX image
and monitored the rollout. - Rolled back and verified that the image
returned to `nginx:1.14.2`.

**Next topic:** Kubernetes Services, selectors, ports, and how to reach
the NGINX application.


## Further reading

- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Run a Stateless Application Using a Deployment](https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/)
- [Updating a Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment)
- [Kubernetes examples repository](https://github.com/kubernetes/examples)
- [Expose Your App](https://kubernetes.io/docs/tutorials/kubernetes-basics/expose/expose-intro/)
