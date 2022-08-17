# Introduction to Istio

Table of contents

## What is Istio ?

Istio is a _service mesh_.

### What is a Service Mesh ?

A _service mesh_ is an extra layer of software you deploy alongside your cluster (any orchestration like Kubernetes).

Service meshes are deployed like sidecars in Kubernetes. The idea is it should readily get integrated without massive changes to existing manifests.

The service meshes like istio have sidecar injector function which make life easier. Right now, the said-container is inside `istiod` or `Istio Daemon`.
In order to attach istio to a system, we do it by attaching a label.

#### Terminology

| Jargon               | Description                                                                                 |
| -------------------- | ------------------------------------------------------------------------------------------- |
| Data Plane           | The proxies/sidecars/envoy containers within the system are collectively called data plane. |
| envoys/sidecar/proxy | They are synonymous and used interchangeably.                                               |
|                      |                                                                                             |

## Introducing envoy

Difference between `reverse proxy` and `proxy`:

`proxy` - The client **does not know** that the results have come from a proxy.

`reverse proxy` - The client **knows** they have called the proxy and thinks that the results have come from the proxy.

## Telemetry

In order to run `istio`, you need a `envoy` sidecar running alongside application container, functioning as proxy.
To do that, you need to inject it, by labelling target namespace with the label `istio-injection=enabled`.

Also, you would need the control plane i.e., `istiod`, `kiali`, `jaeger`, `grafana`,etc.

<span style="color:#f58a42">**To get started with `istio`, you DO NOT need any specific Istio YAML configuration. (No need for VirtualService, Gateways, etc.)**</span>

### Kiali Deeper Dive

Istio provides ***Dynamic Traffic Routing***.

1. Go to `Service graph` within `Kiali UI`.

2. Click on any service.

3. Go to `Overview` and click on `Actions`. You will see 3 available options.

   - `Create Weighted Routing`
   - `Create Matching Routing`
   - `Suspend Traffic`

   > On clicking any of those, `Istio` will automatically create manifests (`virtualservices` and `destinationrules`)and apply those to the selected service.

### Distributed Tracing - Jaeger

#### Setup for application - header propagation

![](https://raw.githubusercontent.com/aditya109/learning-devops-tools/959f52f5240d14ccdcc1fa7fc883c0bc62b74991/istio/assets/header-propagation-jaeger-tracing.svg)

On receiving a request, the proxy if it doesn't see the field `x-request-id` it create a unique guid and attach it to the header.

If there are subsequent requests being made, the application does have to manually do it, meaning manually take out the header.

[1]: https://istio.io/latest/docs/tasks/observability/distributed-tracing/overview/#trace-context-propagation	"Istio trace context propagation"

#### Metrics with Grafana

## Traffic Management

### Introducing Canaries

Deploy a new version of a software components (for us, new image), but only make that new image *live* for a percentage of the time. Most of the time, the old (definitely working) version is the one being used.

### Canaries with Replicas

```yaml
# blue-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: staff-service
spec:
  selector:
    matchLabels:
      app: staff-service
  replicas: 1
  template: # template for the pods
    metadata:
      labels:
        app: staff-service
    spec:
      containers:
      - name: staff-service
        image: richardchesterwood/istio-fleetman-staff-service:6-placeholder
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: fleetman-staff-service
spec:
  selector:
    app: staff-service
  ports:
    - name: http
      port: 8080
  type: ClusterIP

---
# green-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: staff-service
spec:
  selector:
    matchLabels:
      app: staff-service
  replicas: 1
  template: # template for the pods
    metadata:
      labels:
        app: staff-service
    spec:
      containers:
      - name: staff-service
        image: richardchesterwood/istio-fleetman-staff-service:6-placeholder
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: production-microservice
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
```

This is botched canary implementation using Kubernetes.

### Version Grouping

For Kiali we have multiple types of graphs,

1. App graph - An *application* is a group of pods having the same *app* label, meaning you have to use `app` label.
2. Service graph - A service is Kubernetes service here.
3. Versioned app graph - An *versioned app* is a group of pods having the same *version* label, meaning you have to use `version` label.
4. Workload graph - A *workload* is a set of pods under a deployment. 

### Elegant Canaries and Staged Replicas

### What is an Istio VirtualService and DestinationRule?

An *VirtualService* defines a set of traffic routing rules to apply when a host is addressed. ***They are used to dynamically configure envoy sidecar on to accustom to out routing. They are managed by `istio-pilot`.***

Each routing rule defines matching criteria for traffic of a specific protocol. If the traffic is matched, then it is sent to a named destination service (or subset/version of it) defined in the registry.

```yaml
kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name:  fleetman-staff-service-virtualservice  
  namespace: default
spec:
  hosts:
    - fleetman-staff-service.default.svc.cluster.local  
  http:
    - route:
        - destination:
            host: fleetman-staff-service.default.svc.cluster.local # The Target DNS name
            subset: safe-group  # The name defined in the DestinationRule
          weight: 90
        - destination:
            host: fleetman-staff-service.default.svc.cluster.local # The Target DNS name
            subset: risky-group  # The name defined in the DestinationRule
          weight: 10
```

A *DestinationRule* defines policies that apply to traffic intended for a service after routing has occurred. These rules specify configuration for load balancing, connection pool size from the sidecar, and outlier detection settings to detect and evict unhealthy hosts from the load balancing pool. 

```yaml
kind: DestinationRule       # Defining which pods should be part of each subset
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-staff-service-destinationrule 
  namespace: default
spec:
  host: fleetman-staff-service # Service
  subsets:
    - labels:   # SELECTOR.
        version: safe # find pods with label "safe"
      name: safe-group
    - labels:
        version: risky
      name: risky-group
```

![](https://istio.io/latest/img/service-mesh.svg)

## Load Balancing

### Session Affinity (*Stickiness*)

*Is it possible to use the weighted destination rules to make a single user `stick` to a canary ?*

In short, not directly. *DestinationRule* does not natively support it, but has a work-around within itself.

For example, the following rule uses a round robin load balancing policy for all traffic going to the ratings service.

```yaml
kind: DestinationRule       # Defining which pods should be part of each subset
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: grouping-rules-for-our-photograph-canary-release # This can be anything you like.
  namespace: default
spec:
  host: fleetman-staff-service # Service
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
  subsets:
    - labels:   # SELECTOR.
        version: safe # find pods with label "safe"
      name: safe-group
    - labels:
        version: risky
      name: risky-group
```

The following example sets up sticky sessions for the ratings service hashing-based load balancer for the same ratings service using the the User cookie as the hash key. 

```yaml
kind: DestinationRule       # Defining which pods should be part of each subset
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: grouping-rules-for-our-photograph-canary-release # This can be anything you like.
  namespace: default
spec:
  host: fleetman-staff-service # Service
  trafficPolicy:
    loadBalancer:
      consistentHash:
        useSourceIp: true
  subsets:
    - labels:   # SELECTOR.
        version: safe # find pods with label "safe"
      name: safe-group
    - labels:
        version: risky
      name: risky-group
```

>  <span style="color:#CC5500">***Warning !*** This does not really work as load balancers are created after the weighted rules are created. </span> 
> We need to mix it up service gateway to achieve this. https://github.com/istio/istio/issues/9764#issuecomment-631994703

Workaround: *use it without weighted routes*

```yaml
kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: a-set-of-routing-rules-we-can-call-this-anything  # "just" a name for this virtualservice
  namespace: default  
spec:
  hosts:
    - fleetman-staff-service.default.svc.cluster.local  # The Service DNS (ie the regular K8S Service) name that we're applying routing rules to.
  http:
    - route:
        - destination:
            host: fleetman-staff-service.default.svc.cluster.local # The Target DNS name
            subset: all-staff-service-pods  # The name defined in the DestinationRule
          # weight: 100 not needed if there's only one.
---
kind: DestinationRule       # Defining which pods should be part of each subset
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: grouping-rules-for-our-photograph-canary-release # This can be anything you like.
  namespace: default
spec:
  host: fleetman-staff-service # Service
  trafficPolicy:
    loadBalancer:
      consistentHash:
        httpHeaderName: "x-myval"
  subsets:
    - labels:   # SELECTOR.
        app: staff-service # find pods with label "safe"
      name: all-staff-service-pods
```

```bash
# now if you do a curl 
curl --header "x-myval: 192" http://192.168.49.2:30080/api/vehicles/driver/Cit%20Truck
```

## Gateways

### Why do I need an Ingress Gateway ?

*A new feature has been release with image tagged as `:6-experimental`, but the requirement is to deploy this as a 10% canary release.*

```yaml
---
kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-webapp
  namespace: default
spec:
  hosts:
    - fleetman-webapp.default.svc.cluster.local
  http:
    - route:
        - destination:
            host: fleetman-webapp.default.svc.cluster.local
            subset: original
          weight: 90
        - destination:
            host: fleetman-webapp.default.svc.cluster.local
            subset: experimental
          weight: 10
---
kind: DestinationRule
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-webapp
  namespace: default
spec:
  host: fleetman-webapp.default.svc.cluster.local
  subsets:
    - labels:
        version: original
      name: original
    - labels:
        version: experimental
      name: experimental
```

The above however does not result in 90-10 traffic split.
Reason: *The above services run frontend, which directly come from the user, not via the proxy, meaning the reason traffic splitting was possible through Istio was because we were able to re-route the traffic through proxy sidecar, which won't be possible if your pod traffic is coming via the service.*

Solution: <span style="color:#26AF37"> Edge Proxy</span> => Istio Ingress Gateway

### Edge Proxies and Gateways



### Prefix based routing

### Sub-domain Routing































































