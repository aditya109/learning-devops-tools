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

![image-20220806225530595](/home/adityakumar/work/learning/learning-devops-tools/istio/assets/image-20220806225530595.png)

