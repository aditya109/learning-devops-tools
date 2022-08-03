# Telemetry

In order to run `istio`, you need a `envoy` sidecar running alongside application container, functioning as proxy. 
To do that, you need to inject it, by labelling target namespace with the label `istio-injection=enabled`.

Also, you would need the control plane i.e., `istiod`, `kiali`, `jaeger`, `grafana`,etc.  

<span style="color:#f58a42">**To get started with `istio`, you DO NOT need any specific Istio YAML configuration. (No need for VirtualService, Gateways, etc.)**</span>

## Kiali Deeper Dive

Istio provides ***Dynamic Traffic Routing***. 

1. Go to `Service graph` within `Kiali UI`.

2. Click on any service.

3. Go to `Overview` and click on `Actions`. You will see 3 available options.

   - `Create Weighted Routing`
   - `Create Matching Routing`
   - `Suspend Traffic`

   > On clicking any of those, `Istio` will automatically create manifests (`virtualservices` and `destinationrules`)and apply those to the selected service.



 