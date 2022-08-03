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

# Introducing envoy

Difference between `reverse proxy` and `proxy`:

`proxy` - The client **does not know** that the results have come from a proxy.

`reverse proxy` - The client **knows** they have called the proxy and thinks that the results have come from the proxy.
