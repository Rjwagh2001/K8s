# 02 --- First NGINX Pod

## Goal

Create a Pod running NGINX, inspect its status, and verify that it
serves HTTP.

## 1. Create a Pod

Command used:

``` bash
kubectl run nginx-pod --image=nginx:alpine
```

Expected response:

``` text
pod/nginx-pod created
```

This command creates a standalone Pod, not a Deployment.

## 2. Verify the Pod

Commands used:

``` bash
kubectl get pods
kubectl get pods -o wide
kubectl describe pod nginx-pod
```

Observed result:

-   Name: `nginx-pod`
-   Namespace: `default`
-   Status: `Running`
-   Ready: `1/1`
-   Restart count: `0`
-   Image: `nginx:alpine`
-   Pod IP during this run: `10.244.0.3`

The Pod IP is temporary and can change when a Pod is recreated.

## 3. Understand the events

The `describe` output showed these events:

-   `Scheduled` --- the scheduler assigned the Pod to the Minikube node.
-   `Pulling` / `Pulled` --- Kubernetes fetched the `nginx:alpine`
    image.
-   `Created` --- the container was created.
-   `Started` --- the container process started.

`Ready: True` means Kubernetes considered the container ready at that
time.

## 4. Test HTTP using port-forward

In one SSH terminal on EC2:

``` bash
kubectl port-forward pod/nginx-pod 8080:80
```

Keep the command running. In a second SSH session to the same EC2
instance:

``` bash
curl http://127.0.0.1:8080
```

Observed result: HTML containing `Welcome to nginx!`.

Port-forwarding maps local port `8080` to port `80` on the Pod for as
long as the command runs. It is temporary and does not create a Service
or make NGINX publicly available.

Stop it using `Ctrl+C` in the port-forward terminal.

## 5. Recreate the Pod from YAML

The reusable manifest is `manifests/nginx-pod.yaml`.

From the repository root, on a machine whose `kubectl` points at the
intended cluster:

``` bash
kubectl apply -f manifests/nginx-pod.yaml
kubectl get pod nginx-pod -o wide
```

The YAML and imperative command describe the same named Pod. If it
already exists, `kubectl apply` reconciles it against the manifest.

## 6. Troubleshooting

``` bash
kubectl get pods -o wide
kubectl describe pod nginx-pod
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl logs nginx-pod
```

If the container restarted and a previous instance exists:

``` bash
kubectl logs nginx-pod --previous
```

## 7. Cleanup

When you are ready to remove this Pod:

``` bash
kubectl delete pod nginx-pod
```

Or delete it using the manifest:

``` bash
kubectl delete -f manifests/nginx-pod.yaml
```

Verify:

``` bash
kubectl get pods
```

A Pod managed by a Deployment should generally be managed through the
Deployment, rather than manually deleting individual replicas.
