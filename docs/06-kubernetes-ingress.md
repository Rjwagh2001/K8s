# 06 --- Kubernetes Ingress: Concepts and Hands-on Lab

> Learning path: Pods → Deployments → Services → Ingress\
> Lab environment: Amazon EC2 + Minikube + NGINX Ingress Controller +
> Django/Python web application

## Learning objectives

By the end of this lab, you should be able to:

-   Explain why Ingress is useful even when Kubernetes already provides
    Services, load balancing, auto-healing, and scaling.
-   Compare `ClusterIP`, `NodePort`, and `LoadBalancer` Services with
    Ingress.
-   Explain the difference between an Ingress resource and an Ingress
    Controller.
-   Understand host-based routing, path-based routing, path types, and
    TLS configuration.
-   Deploy an Ingress Controller in Minikube and route traffic to an
    existing application Service.
-   Diagnose DNS/hosts-file, controller, Service, and application-level
    routing problems.

## 1. Quick recap: what we already learned

### Pod

A Pod is the smallest deployable unit in Kubernetes. It runs one or more
containers. Pod IPs can change when Pods are replaced, so clients should
not rely on a Pod IP as a permanent endpoint.

### Deployment

A Deployment manages ReplicaSets and keeps the requested number of
application replicas running. For example, `replicas: 3` asks Kubernetes
to maintain three Pods. If a managed Pod disappears, the Deployment's
control loop works to restore the desired replica count.

### Service

A Service gives a changing set of backend Pods a stable network
endpoint. A Service selector matches Pod labels, and Kubernetes
maintains EndpointSlices describing the eligible backends.

Common Service types:

  -----------------------------------------------------------------------
  Type                    Main purpose            Typical reachability
  ----------------------- ----------------------- -----------------------
  `ClusterIP`             Stable internal         Inside the cluster
                          endpoint                

  `NodePort`              Exposes a Service on a  Through a reachable
                          port of each node       node IP and NodePort

  `LoadBalancer`          Requests an external    External, depending on
                          load balancer from a    provider/network
                          supported               configuration
                          environment/provider    
  -----------------------------------------------------------------------

A Service provides backend endpoint abstraction and traffic
distribution. The exact packet-routing and balancing implementation
depends on the cluster networking implementation. Do not assume every
Kubernetes Service uses a strict round-robin algorithm.

### Auto-healing and scaling are different concerns

-   A Deployment maintains the desired replica count.
-   A HorizontalPodAutoscaler (HPA), when configured with metrics and
    suitable policies, adjusts replica count.
-   A Service provides a stable endpoint to eligible backends.
-   Ingress provides HTTP/HTTPS routing rules across Services.

Ingress does not replace any of these responsibilities.

## 2. Why does Ingress come into the picture?

Imagine a company hosts `frontend`, `payments`, `admin`, and `catalog`
applications.

Each application can have a Deployment and a Service. Internal Services
work well for service-to-service communication. The next question is:
how should external web users reach several applications using normal
URLs and HTTPS?

### Option A --- ClusterIP only

A `ClusterIP` Service is normally reachable only from inside the
cluster. It is suitable for internal communication, but an external
browser cannot ordinarily connect directly to its cluster-internal IP.

### Option B --- NodePort

Each application could be exposed on a node port, for example:

-   `http://<node-ip>:30080` → frontend
-   `http://<node-ip>:30081` → payments
-   `http://<node-ip>:30082` → admin

This can work, but clients need to reach the node address and port.
NodePort itself does not provide HTTP hostname/path routing or central
TLS termination.

### Option C --- One LoadBalancer Service per application

In a supported cloud environment, setting a Service to
`type: LoadBalancer` can request an external load balancer. If an
organization exposes many applications independently this way, it may
end up managing many external load balancers, public entry points, DNS
records, security policies, and certificates. This can increase
operational complexity and cost.

This is a possible architecture, not a requirement: Kubernetes does not
mandate one external load balancer per application, and cloud/provider
implementations differ. An external IP is not always a permanent/static
IP unless the provider and configuration support that.

### Option D --- A shared HTTP/HTTPS entry point with Ingress

Ingress lets an organization define host- and path-based HTTP/HTTPS
rules for multiple Services behind a common entry point.

Examples:

  Request                            Backend
  ---------------------------------- --------------------
  `https://company.example/`         `frontend-service`
  `https://company.example/api`      `api-service`
  `https://admin.company.example/`   `admin-service`

A typical architecture is:

``` text
Browser / API client
        |
        v
DNS name / reachable external entry point
        |
        v
Ingress Controller (for example, NGINX)
        |
        +---- /       ----> frontend-service ----> frontend Pods
        |
        +---- /api    ----> api-service ---------> API Pods
        |
        +---- admin host -> admin-service -------> admin Pods
```

A shared entry point can reduce the need to provision a separate public
load balancer for every web application. The actual entry point might
itself be a cloud load balancer, a reachable node/network endpoint, or
another supported design.

## 3. Does Ingress replace Kubernetes load balancing?

No. There are different layers and responsibilities.

  -----------------------------------------------------------------------
  Component                           Responsibility
  ----------------------------------- -----------------------------------
  Deployment                          Maintains the desired number of
                                      Pods

  HPA                                 Adjusts replica count according to
                                      configured metrics/policies

  Service                             Stable backend endpoint and
                                      abstraction over eligible Pods

  Ingress resource                    Declares HTTP/HTTPS host/path
                                      routing rules

  Ingress Controller                  Implements the rules using a
                                      proxy/load-balancing system

  External load balancer (if used)    Makes the entry point reachable and
                                      forwards traffic according to its
                                      configuration
  -----------------------------------------------------------------------

The Ingress Controller chooses the backend Service according to HTTP
routing rules. It may proxy to Pod endpoints directly, or use another
implementation-specific forwarding path. The Service remains the
declared backend in the Ingress resource and provides Kubernetes service
discovery/endpoint abstraction.

Kubernetes Service traffic distribution is not necessarily the same as
the advanced application-aware routing offered by a reverse proxy or
enterprise load balancer.

## 4. Why is an Ingress Controller required?

Kubernetes stores and validates the Ingress API object, but an Ingress
object alone does not implement network traffic routing.

There are two key pieces:

1.  **Ingress resource** --- the desired configuration, written as YAML.
    It specifies hosts, paths, and backend Services.
2.  **Ingress Controller** --- a running controller that watches
    Kubernetes API resources and reconciles the configuration into a
    working proxy/load-balancer configuration.

The controller typically watches Ingress resources and related objects,
such as Services, EndpointSlices, Secrets, and IngressClasses. It then
configures or updates its data plane.

This is a reconciliation pattern: observe desired state, compare it with
actual state, and make changes to converge on the desired configuration.

Kubernetes has controllers for many resources, but it does not ship one
universal Ingress Controller that automatically supports every vendor's
load balancer and proxy. Different controllers have different
installation procedures and capabilities.

Examples include ingress-nginx, Traefik, and cloud-provider-specific
controllers. Choose a controller based on the environment, supported
features, maintenance status, and operational requirements.

## 5. What can Ingress specify?

The standard Kubernetes Ingress API includes:

-   **Host-based routing:** match a hostname such as `shop.example.com`.
-   **Path-based routing:** match paths such as `/`, `/api`, or
    `/admin`.
-   **Backend Service and port:** choose a Service and its Service port.
-   **Path matching:** `Prefix`, `Exact`, or `ImplementationSpecific`.
-   **TLS configuration:** reference a TLS Secret for HTTPS termination,
    when supported/configured by the controller.
-   **IngressClass:** identify which controller should implement the
    Ingress.

### Path types

-   `Prefix`: path elements are matched by prefix. `/api` matches `/api`
    and `/api/users`, but not `/apix`.
-   `Exact`: matches the specified path exactly.
-   `ImplementationSpecific`: the controller defines the matching
    behavior.

### Features that may depend on the controller

Sticky sessions, ratio/weighted routing, canary behavior, URL rewriting,
redirects, authentication, rate limiting, IP allow/deny rules, WebSocket
handling, advanced TLS behavior, and other enterprise load-balancer
features are not all portable standard Ingress fields. They may require
controller-specific annotations, configuration, or other resources.

Do not assume that writing an Ingress manifest automatically enables
every feature of an enterprise load balancer.

## 6. Ingress vs. an enterprise load balancer / reverse proxy

Before Kubernetes, organizations commonly used physical or virtual
appliances and software reverse proxies/load balancers, such as NGINX,
HAProxy, F5 BIG-IP, or cloud load-balancing services. They could provide
L4/L7 routing, TLS termination, access policies, health checks,
persistence/sticky sessions, weighted routing, and other features
depending on the product and license/configuration.

Kubernetes Ingress gives a Kubernetes-native API for declaring a common
subset of web-routing requirements. An Ingress Controller translates
that desired state into the behavior of its proxy or load balancer.

Ingress is not itself a specific load-balancer product and does not
guarantee every enterprise feature. In some architectures, an external
enterprise/cloud load balancer sits in front of an Ingress Controller;
in others, a cloud controller directly provisions an external load
balancer from Kubernetes resources.

## 7. Lab environment and existing application

This lab uses the existing Python/Django application in the
`python-web-app` directory, running on Minikube hosted on an EC2
instance.

From the lab output, the application Deployment is named `pyhton-app`
(the spelling in the existing resource), with three replicas. The
backend Service is `python-app-service`, exposing Service port `8000`.
Its EndpointSlice showed three backend Pod IPs.

> Use the names that actually exist in your cluster. Do not rename
> `pyhton-app` only because it looks like a typo unless you
> intentionally update the Deployment and related resources.

### Check the current resources

``` bash
kubectl get nodes
kubectl get deployments
kubectl get pods -o wide
kubectl get services
kubectl get endpointslices
```

The existing Service was originally `type: LoadBalancer` and displayed
`<pending>` for `EXTERNAL-IP` in the Minikube/EC2 environment. This does
not automatically mean the application is unhealthy. It means that an
external load-balancer address has not been reported by that
environment.

## 8. Enable the Minikube NGINX Ingress Controller

``` bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
kubectl get ingressclass
```

Wait until the controller Pod is `Running` and ready. The admission Jobs
may show `Completed`; that is normal.

The IngressClass in this lab is `nginx`, with controller
`k8s.io/ingress-nginx`.

## 9. Ingress manifest used in the lab

Save this manifest separately as `manifests/ingress.yaml`. It routes the
host `foo.bar.com` and path `/demo` to the existing `python-app-service`
Service on port `8000`.

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-python-app
spec:
  ingressClassName: nginx
  rules:
    - host: foo.bar.com
      http:
        paths:
          - path: /demo
            pathType: Prefix
            backend:
              service:
                name: python-app-service
                port:
                  number: 8000
```

Apply and inspect it:

``` bash
kubectl apply -f manifests/ingress.yaml
kubectl get ingress
kubectl describe ingress ingress-python-app
```

Expected routing summary:

``` text
Host: foo.bar.com
Path: /demo
Backend: python-app-service:8000
```

The controller's Ingress address in the lab was `192.168.49.2`. This is
a private Minikube address in this environment, not automatically a
public internet address.

## 10. Hostname resolution in the lab

On the EC2 instance, `/etc/hosts` was updated with:

``` text
192.168.49.2 foo.bar.com
```

The entry was added without installing an editor:

``` bash
echo "192.168.49.2 foo.bar.com" | sudo tee -a /etc/hosts
getent hosts foo.bar.com
```

This only maps the hostname on that EC2 machine. It is not public DNS
and does not configure the Windows laptop's hosts file. The IP must be
reachable from the client running the request.

For an alternative that does not change `/etc/hosts`, use curl's
`--resolve` option:

``` bash
curl --resolve foo.bar.com:80:192.168.49.2 -i http://foo.bar.com/demo/
```

Use the current Ingress/Minikube address if it changes.

## 11. Test the traffic path

The lab successfully returned the Django page for:

``` bash
curl -i http://foo.bar.com/demo/
```

The response was `HTTP/1.1 200 OK`, and the body contained the "Free
DevOps Course By Abhishek" HTML page.

The request path was:

``` text
curl
  -> foo.bar.com resolves to 192.168.49.2
  -> NGINX Ingress Controller
  -> host/path rule matches foo.bar.com and /demo
  -> python-app-service:8000
  -> one of the application Pod backends
  -> Django returns the HTML response
```

### Why did `/bar` return 404?

The first Ingress rule used `/bar`. The request reached Django, which
displayed a debug 404 page and listed the application's URL patterns:

-   `demo/`
-   `admin/`

There was no Django URL pattern for `bar`, so Django returned a 404.
This was an **application-level 404**, not proof of an Ingress failure.

Ingress forwards the original path unless rewrite behavior is
configured. It does not automatically translate `/bar` into `/demo/`.

### Why did `/demo` return 301?

The application returned:

``` text
HTTP/1.1 301 Moved Permanently
Location: /demo/
```

Django redirected `/demo` to its canonical slash-terminated URL
`/demo/`. The next request returned `200 OK`.

To follow redirects automatically:

``` bash
curl -i -L http://foo.bar.com/demo
```

## 12. Troubleshooting checklist

### Controller not running

``` bash
kubectl get pods -n ingress-nginx
kubectl describe pods -n ingress-nginx
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller --tail=100
```

### Ingress rule not recognized

``` bash
kubectl get ingressclass
kubectl describe ingress ingress-python-app
kubectl get ingress ingress-python-app -o yaml
```

Check the class name, hostname, path, and backend Service/port.

### Backend Service has no endpoints

``` bash
kubectl get svc python-app-service
kubectl get endpointslices -l kubernetes.io/service-name=python-app-service
kubectl get pods --show-labels
```

Check that the Service selector matches the Pod labels and that the Pods
are ready.

### Interpret HTTP errors carefully

-   **NGINX/default 404:** the request may not match an Ingress
    host/path rule or may reach the controller's default backend.
-   **Application 404:** the request reached the application, but the
    application did not recognize the path. Inspect the response body
    and application logs.
-   **502/503:** may indicate an unavailable/unhealthy backend,
    incorrect Service port, connection refusal, or controller-specific
    upstream problem.
-   **Connection refused/timeout:** investigate whether the client can
    reach the controller IP/port, the listener, security
    groups/firewalls, and the Minikube network path.

Do not diagnose by status code alone; inspect response headers/body,
controller logs, and backend health.

## 13. Security and production considerations

-   Do not expose Django's `DEBUG = True` setting in production. It
    reveals internal URL configuration and diagnostic details.
-   Use HTTPS/TLS with a properly managed certificate for public
    applications.
-   Configure security groups, firewall rules, network policies,
    authentication, and access restrictions according to the threat
    model.
-   Do not treat `/etc/hosts` as a firewall or whitelist. It only
    resolves a hostname locally.
-   Keep the Ingress Controller and its configuration maintained and
    supported.
-   A single controller/entry point can become a critical dependency;
    production design should consider replicas, availability zones,
    capacity, upgrades, monitoring, and failure recovery.
-   Use least privilege and avoid exposing administrative endpoints
    unnecessarily.

## 14. Ingress API and Gateway API

Ingress remains a stable and widely used Kubernetes API, but its feature
development is frozen. Kubernetes recommends considering the Gateway API
for new capabilities and richer traffic-management models. Existing
Ingress resources are not automatically removed, and learning Ingress
remains valuable for understanding existing clusters.

## 15. Interview-ready summary

**Question: If Kubernetes Services already load balance traffic, why do
we need Ingress?**

A strong answer:

> A Kubernetes Service provides a stable endpoint for a set of backend
> Pods and enables traffic distribution to eligible endpoints. It does
> not, by itself, provide a standard HTTP routing configuration for
> multiple applications based on hostnames and URL paths. Exposing many
> applications separately with LoadBalancer Services can create
> additional external load balancers and operational overhead. Ingress
> lets us declare HTTP/HTTPS host- and path-based rules that route
> requests to multiple Services through a shared entry point. The
> Ingress Controller watches the Kubernetes API and implements those
> rules using a proxy or load-balancing system. Features such as TLS
> termination, rewrites, authentication, sticky sessions, and rate
> limiting depend on the controller and its configuration.

### Questions to practise

1.  What is the difference between a Service and an Ingress?
2.  What is the difference between an Ingress resource and an Ingress
    Controller?
3.  Why might `type: LoadBalancer` show `EXTERNAL-IP: <pending>` in
    Minikube?
4.  What is the difference between host-based and path-based routing?
5.  What do `Prefix` and `Exact` mean?
6.  Why can a request pass through Ingress and still receive an
    application 404?
7.  Why doesn't `/etc/hosts` make an application publicly accessible?
8.  Which responsibilities belong to Deployment, HPA, Service, Ingress,
    and the Ingress Controller?
9.  How would you troubleshoot a 404 versus a 503?
10. When might you choose Gateway API instead of Ingress?

## Official references

-   [Kubernetes Ingress
    concepts](https://kubernetes.io/docs/concepts/services-networking/ingress/)
-   [Kubernetes Ingress
    Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
-   [Kubernetes
    Services](https://kubernetes.io/docs/concepts/services-networking/service/)
-   [Minikube Ingress tutorial (Kubernetes v1.33
    documentation)](https://v1-33.docs.kubernetes.io/docs/tasks/access-application-cluster/ingress-minikube/)
-   [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/)
