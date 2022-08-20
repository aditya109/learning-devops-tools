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

**Requirement:** A new feature has been release with image tagged as `:6-experimental`, but the requirement is to deploy this as a 10% canary release.*

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

![](https://github.com/aditya109/learning-devops-tools/raw/main/istio/assets/edge-proxy-gateway.svg)

```yaml
# ingress gateway
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ingress-gateway-configuration
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation - ingress-gateway-pod label (pre-existing)
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
```

Upon adding the above gateway configuration, if we go to http gateway endpoint:

```bash
> m service list
|--------------|----------------------------|-------------------|---------------------------|
|  NAMESPACE   |            NAME            |    TARGET PORT    |            URL            |
|--------------|----------------------------|-------------------|---------------------------|
| default      | fleetman-api-gateway       | No node port      |
| default      | fleetman-position-tracker  | No node port      |
| default      | fleetman-staff-service     | No node port      |
| default      | fleetman-vehicle-telemetry | No node port      |
| default      | fleetman-webapp            | http/80           | http://192.168.49.2:30080 |
| default      | kubernetes                 | No node port      |
| istio-system | grafana                    | service/3000      | http://192.168.49.2:31002 |
| istio-system | istio-egressgateway        | No node port      |
| istio-system | istio-ingressgateway       | status-port/15021 | http://192.168.49.2:32063 |
|              |                            | http2/80          | http://192.168.49.2:31200 |  👈
|              |                            | https/443         | http://192.168.49.2:31959 |
|              |                            | tcp/31400         | http://192.168.49.2:31552 |
|              |                            | tls/15443         | http://192.168.49.2:32042 |
| istio-system | istiod                     | No node port      |
| istio-system | jaeger-collector           | No node port      |
| istio-system | kiali                      | http/20001        | http://192.168.49.2:31000 |
|              |                            | http-metrics/9090 | http://192.168.49.2:31178 |
| istio-system | prometheus                 | No node port      |
| istio-system | tracing                    | http-query/80     | http://192.168.49.2:31001 |
| istio-system | zipkin                     | No node port      |
| kube-system  | kube-dns                   | No node port      |
|--------------|----------------------------|-------------------|---------------------------|
```

i.e., http://192.168.49.2:31200 we get a 404, because it does not have any other service routing configuration.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ingress-gateway-configuration
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*" # domain name of the external website
---
kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-webapp
  namespace: default
spec:
  hosts:
    - fleetman-webapp.default.svc.cluster.local
    - "*" # copy the value from the gateway hosts
  gateways:
    - ingress-gateway-configuration
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
```

Now if we do http://192.168.49.2:31200 we get 90-10 split.

> <span style="color:#F1462B"> *The proxies run after a container makes a request !*</span>

### Prefix based routing

**Requirement:** 

1. `/canary` or `/experimental`  - traffic goes to version 1
2. `/` - traffic goes to version 2 

```yaml
kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-webapp
  namespace: default
spec:
  hosts:
    - "*" # copy the value in the gateway hosts
  gateways:
    - ingress-gateway-configuration
  http:
    - match:
      - uri:
          prefix: '/canary' # if
      - uri:
          prefix: '/experimental' # or
      route:
      - destination:
            host: fleetman-webapp.default.svc.cluster.local
            subset: experimental
    - match:
      - uri:
          prefix: '/'
      route:
      - destination:
            host: fleetman-webapp.default.svc.cluster.local
            subset: original
    - route:
        - destination:
            host: fleetman-webapp.default.svc.cluster.local
            subset: original
          weight: 90
        - destination:
            host: fleetman-webapp.default.svc.cluster.local
            subset: experimental
          weight: 10
```

> <span style="color:#F1462B">*`route` field exists in both places inside `match`* and as a separate entity as well.</span>

### Sub-domain Routing

**Requirement:** 

1. `fleetman.com:31200`  - traffic goes to `original` version
2. `experimental.fleetman.com:31200` - traffic goes to `experimental` version

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ingress-gateway-configuration
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*fleetman.com" # domain name of the external website
---
kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-webapp
  namespace: default
spec:
  hosts:
    - "fleetman.com" # copy the value in the gateway hosts
  gateways:
    - ingress-gateway-configuration
  http:
    - route:
        - destination:
            host: fleetman-webapp.default.svc.cluster.local
            subset: original
---
kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-webapp-experimental
  namespace: default
spec:
  hosts:
    - "experimental.fleetman.com" # copy the value in the gateway hosts
  gateways:
    - ingress-gateway-configuration
  http:
    - route:
        - destination:
            host: fleetman-webapp.default.svc.cluster.local
            subset: experimental
```

```bash
❯ curl -s experimental.fleetman.com:31200 | grep title
  <title>Fleet Management Istio Premium Enterprise Edition</title>
❯ curl -s fleetman.com:31200 | grep title
  <title>Fleet Management</title>

```

## Dark Releases

*We use one domain, but differentiate canary release via headers.*

If the request header contains `my-header=canary` it should redirect to experimental version, else original version.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ingress-gateway-configuration
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"   # Domain name of the external website
---
kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-webapp
  namespace: default
spec:
  hosts:      # which incoming host are we applying the proxy rules to???
    - "*"
  gateways:
    - ingress-gateway-configuration
  http:
    - match:
      - headers:  # IF
          x-my-header:
            exact: canary
      route: # THEN
      - destination:
          host: fleetman-webapp
          subset: experimental
    - route: # CATCH ALL
      - destination:
          host: fleetman-webapp
          subset: original
```

> <span style="color:#F1462B">If you are making an api request from a pod to another, you need to do *header propagation*. </span>
>
> **Meaning** if you are pushing any header `x-my-header=canary` from edge service, that same header will have to be propagated throughout the service.

## Fault Injection

https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing

> *The fallacies are*:
>
> 1. The network is reliable;
> 2. Latency is zero;
> 3. Bandwidth is infinite;
> 4. The network is secure;
> 5. Topology doesn't change;
> 6. There is one administrator;
> 7. Transport cost is zero;
> 8. The network is homogeneous.
>
> *The effects of the fallacies:*
>
> 1. Software applications are written with little error-handling on networking errors. During a network outage, such applications may stall or infinitely wait for an answer packet, permanently consuming memory or other resources. When the failed network becomes available, those applications may also fail to retry any stalled operations or require a (manual) restart.
> 2. Ignorance of network latency, and of the packet loss it can cause, induces application- and transport-layer developers to allow unbounded traffic, greatly increasing dropped packets and wasting bandwidth.
> 3. Ignorance of bandwidth limits on the part of traffic senders can result in bottlenecks.
> 4. Complacency regarding network security results in being blindsided by malicious users and programs that continually adapt to security measures.
> 5. Changes in network topology can have effects on both bandwidth and latency issues, and therefore can have similar problems.
> 6. Multiple administrators, as with subnets for rival companies, may institute conflicting policies of which senders of network traffic must be aware in order to complete their desired paths.
> 7. The "hidden" costs of building and maintaining a network or subnet are non-negligible and must consequently be noted in budgets to avoid vast shortfalls.
> 8. If a system assumes a homogeneous network, then it can lead to the same problems that result from the first three fallacies.

**Requirement:** *Fail traffic from a microservice for 14% of the time*

```yaml
---
kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-vehicle-telemetry
  namespace: default
spec:
  hosts:
    - fleetman-vehicle-telemetry.default.svc.cluster.local
  http:
    - match:
      - headers:  # IF
          x-my-header:
            exact: canary
      fault:
        abort:
          httpStatus: 503
          percentage:
            value: 14                # can be changed to simulate a fault injection
      route:
        - destination:
            host: fleetman-vehicle-telemetry.default.svc.cluster.local
---
kind: DestinationRule
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-vehicle-telemetry
  namespace: default
spec:
  host: fleetman-vehicle-telemetry.default.svc.cluster.local
  subsets: []

```

**Requirement:** *delay traffic from a microservice for 53% of the time*

```yaml
---
kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-vehicle-telemetry
  namespace: default
spec:
  hosts:
    - fleetman-vehicle-telemetry.default.svc.cluster.local
  http:
    - match:
      - headers:  # IF
          x-my-header:
            exact: canary
      fault:
        delay:
          percentage:
            value: 53.0                # can be changed to simulate a fault injection
          fixedDelay: 10s
      route:
        - destination:
            host: fleetman-vehicle-telemetry.default.svc.cluster.local
---
kind: DestinationRule
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-vehicle-telemetry
  namespace: default
spec:
  host: fleetman-vehicle-telemetry.default.svc.cluster.local
  subsets: []
```

## Circuit Breaking

### Cascading Failures

A cascading failure is a failure in a system of interconnected parts in which the failure of one or few parts leads to the failure of other parts, growing progressively as a result of positive feedback. 

<span style="color:#269B10">**Solution: Circuit Breaker**</span>

1. A service client should invoke a remote service via a proxy that functions in a similar fashion to an electrical circuit breaker. 
2. When the number of consecutive failures crosses a threshold, the circuit breaker trips, and for the duration of a timeout period all attempts to invoke the remote service will fail immediately. 
3. After the timeout expires the circuit breaker allows a limited number of test requests to pass through. 
4. If those requests succeed the circuit breaker resumes normal operation. Otherwise, if there is a failure the timeout period begins again.

<span style="color:#E74414">Cons for the *classic* circuit breaking</span>:

1. You need to build the circuit breaker into your microservice (every single one of them).
2. You are relying on the programming language for the implementation of the circuit breaker.

***Use istio, instead !***

![](https://raw.githubusercontent.com/aditya109/learning-devops-tools/main/istio/assets/circuit-breaker-fall.svg)

*But what happens if there is just 1 pod ?* (we use **backpressure**, meaning we fail immediately, rather than cascading it further.)

![](https://github.com/aditya109/learning-devops-tools/raw/main/istio/assets/circuit-breaker-backpressure.svg)



### Configuring Outlier Detection (circuit breaking)

Use this to measure curl request:

```bash
❯ while true; do curl -s -w '=> Latency: %{time_total}s\n' http://192.168.49.2:32566/api/vehicles/driver/City%20Truck; echo ; echo ; done
```

```yaml
# generic gateway configuration and virtualservice, we don't really need this.
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ingress-gateway-configuration
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"   # Domain name of the external website
---
# All traffic routed to the fleetman-webapp service
# No DestinationRule needed as we aren't doing any subsets, load balancing or outlier detection.
kind: VirtualService
apiVersion: networking.istio.io/v1alpha3
metadata:
  name: fleetman-webapp
  namespace: default
spec:
  hosts:      # which incoming host are we applying the proxy rules to???
    - "*"
  gateways:
    - ingress-gateway-configuration
  http:
    - route:
      - destination:
          host: fleetman-webapp
```

```yaml
# destinationrule, REQUIRED !
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: circuit-breaker-for-the-entire-default-namespace
spec:
  host: "fleetman-staff-service.default.svc.cluster.local"          # This is the name of the k8s service that we're configuring
  trafficPolicy:
    outlierDetection: # Circuit Breakers HAVE TO BE SWITCHED ON
      maxEjectionPercent: 100
      consecutive5xxErrors: 3
      interval: 20s
      baseEjectionTime: 30s

```

## Mutual TLS

### Why is encryption needed inside a cluster ?

**Requirement:** Is it required that the inter-pod traffic is unencrypted ?

### How Istio can upgrade traffic to TLS 

Basically the normal calls being made from the app pod would still be in `http` but this call goes to the proxy which in turn make an `https` call. The component which enables us to do is `istio-citadel` (part of `istiod` pod now). Now the proxies make calls via `https`. The certificates are going to be issued by the `istio-citadel`.

How to do it ?

1. Enforce a policy that BLOCKS all non TLS traffic.
2. Automatically upgrade all proxy-proxy communication to use mTLS.

<strong>Enabling mTLS - it's automatic, it is already done.</strong>

### STRICT VS PERMISSIVE mTLS+

**Permissive mTLS**

![](https://github.com/aditya109/learning-devops-tools/raw/main/istio/assets/permissive-mTLS.svg)

It is going to stay as in as an `http` call.

![](https://github.com/aditya109/learning-devops-tools/raw/main/istio/assets/strict-mTLS.svg)

It is put in the namespace.

```yaml
# This will enforce that ONLY traffic that is TLS is allowed between proxies
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "default"
  namespace: "istio-system"
spec:
  mtls:
    mode: STRICT # or PERMISSIVE (which is default)
```

<span style="color:orange">**STRICT mTLS works in both directions.**</span>

## Customizing and installing Istio with Istioctl

https://istio.io/latest/docs/setup/getting-started/#download









































