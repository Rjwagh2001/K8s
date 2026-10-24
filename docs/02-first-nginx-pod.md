# 02 --- First NGINX Pod

## Goal

Create a Pod running NGINX, inspect it, and verify that the web server
returns HTTP content.

## Create the Pod

The first run used the imperative command:

``` bash
kubectl run nginx-pod --image=nginx:alpine
```

Expected response:

``` text
pod/nginx-pod created
```

Inspect it:

``` bash
kubectl get pods
kubectl get pods -o wide
kubectl describe pod nginx-pod
```

Observed successful state:

-   Pod: `nginx-pod`
-   Namespace: `default`
-   Status: `Running`
-   Ready: `1/1`
-   Restart count: `0`
-   Image: `nginx:alpine`
-   Pod IP observed in this run: `10.244.0.3`

Pod IPs can change when Pods are recreated; do not hard-code this IP in
application configuration.

## Understand the lifecycle

1.  `kubectl` submits the request to the Kubernetes API server.
2.  Kubernetes records the desired Pod specification.
3.  The scheduler assigns the Pod to a node.
4.  The node's kubelet asks its container runtime to pull the image and
    start the container.
5.  Kubernetes reports the Pod and container status.

Useful `describe` events include `Scheduled`, `Pulling`, `Pulled`,
`Created`, and `Started`. These events help explain what Kubernetes did.

## Test NGINX through port forwarding

In one EC2 SSH terminal:

``` bash
kubectl port-forward pod/nginx-pod 8080:80
```

Keep this command running. In a second SSH session to the same EC2
instance:

``` bash
curl http://127.0.0.1:8080
```

Success criteria: the response includes `Welcome to nginx!`.

Port mapping format:

``` text
8080:80
  │   └── port 80 inside the Pod
  └────── local port 8080 on the machine running kubectl
```

Port forwarding is temporary and normally listens on loopback by
default. It is useful for development and troubleshooting, not a
permanent public access method.

Stop it with `Ctrl+C` in the port-forward terminal.

## Inspect logs

``` bash
kubectl logs nginx-pod
```

NGINX access logs appear after requests are sent. The exact output
depends on the requests made.

## Reusable YAML manifest

The file `../manifests/nginx-pod.yaml` defines the Pod declaratively.
Apply it with:

``` bash
kubectl apply -f manifests/nginx-pod.yaml
```

Check the result:

``` bash
kubectl get -f manifests/nginx-pod.yaml
kubectl describe -f manifests/nginx-pod.yaml
```

A standalone Pod is useful for learning, but real applications are
commonly managed by a Deployment so Kubernetes can maintain the desired
number of replicas and support controlled updates.

## Troubleshooting checklist

If the Pod is not `Running`:

``` bash
kubectl get pods -o wide
kubectl describe pod nginx-pod
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl logs nginx-pod
```

Use `kubectl logs` if the container has started and produced logs. For a
container that repeatedly crashes, try:

``` bash
kubectl logs nginx-pod --previous
```

`--previous` only helps when a previous container instance exists.

## Cleanup

Delete the Pod when you want to repeat the exercise:

``` bash
kubectl delete pod nginx-pod
```

Or, if you created it from the manifest:

``` bash
kubectl delete -f manifests/nginx-pod.yaml
```

Verify:

``` bash
kubectl get pods
```

Do not run cleanup until you are ready to remove the workload.
