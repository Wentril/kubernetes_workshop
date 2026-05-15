# Ingress vs Gateway API

Ingress is still widely used and is sufficient for many basic HTTP routing scenarios.  
Gateway API is the newer Kubernetes networking API designed to address limitations of Ingress, especially around expressiveness, role separation, and portability across implementations.

## Ingress NGINX and Ingate
About 50% cloud-native environments are/were dependent on Ingress NGINX ([source - awx.amazon.com blog 03/2026](https://aws.amazon.com/blogs/networking-and-content-delivery/navigating-the-nginx-ingress-retirement-a-practical-guide-to-migration-on-aws/)).  
Ingress NGINX has historically been the default choice in many clusters because it was simple to adopt and supported a broad set of features.  
Under the hood it were Nginx pods, that heavily (mis)used Lua scripting to create server blocks of Nginx configs.  
  
`ingress-nginx` itself has also had a long history of bugs, complexity growth, and security fixes. 

## Gateway API
Gateway API is a set of Kubernetes networking resources developed under SIG Network. It is not limited to classic ingress use cases. The goal is to provide a more expressive and more portable way to configure traffic entering or moving inside a cluster.

![Gateway Model](../images/gateway-resource-model.png)

## Why Ingress was not enough

The built-in Ingress API was intentionally small. That made initial adoption easy, but it also created several practical limitations:
- it mainly targets HTTP/HTTPS traffic
- features such as header matching, traffic splitting, retries, or advanced TLS handling are usually controller-specific
- many production manifests rely heavily on annotations
- it is difficult to express who is allowed to attach routes to shared infrastructure

In other words, Ingress gave Kubernetes a common minimum, but not a strong common model for real-world traffic management.

## Gateway API resource model

Gateway API splits the problem into multiple resources with clearer ownership.

### `GatewayClass`

`GatewayClass` is similar to `IngressClass`.
- It defines which controller implementation should manage Gateways of that class
- It is usually created by the platform team or installed by the controller itself

### `Gateway`

`Gateway` represents the network entry point managed by the infrastructure team.
- It defines listeners such as protocol, port, hostname, and TLS configuration
- It is comparable to the load balancer or proxy instance that will receive traffic
- One Gateway can expose multiple listeners, for example HTTP on port 80 and HTTPS on port 443

### `HTTPRoute`

`HTTPRoute` defines how HTTP traffic should be matched and forwarded.
- It contains hostnames, path matching, header matching, filters, and backend references
- It is typically the resource application teams work with most often
- Multiple routes can attach to the same Gateway when permitted by policy

### Other route types

Gateway API is not limited to HTTP. Depending on implementation and maturity level, you may also encounter:
- `GRPCRoute`
- `TLSRoute`
- `TCPRoute`
- `UDPRoute`

This is one of the major differences from Ingress, which was primarily designed around HTTP routing.

## Separation of responsibilities

One of the biggest improvements in Gateway API is role separation.

Typical split:
- platform team defines `GatewayClass` and shared `Gateway` resources
- application teams create `HTTPRoute` resources for their own services

### Implementations

Feature support is controller-dependent. Not every controller supports every route type or feature level.

- Official implementation list: [gateway-api.sigs.k8s.io](https://gateway-api.sigs.k8s.io/docs/implementations/list/)
- Before choosing a controller, check support for the exact resources and features you plan to teach or use
