# AgentGateway
> One high-performance gateway for service, LLM, and MCP traffic.  

> An open source HTTP and gRPC gateway that handles traditional application traffic and AI-native protocols in one data plane. Route, secure, observe, and govern services, LLM provider traffic, MCP tools, and agent-to-agent communication without stitching together separate gateways.  

[agentgateway.dev](https://agentgateway.dev)  
[github: agentgateway/agentgateway](https://github.com/agentgateway/agentgateway)  
[gateway-api.sigs.k8s.io - Implementations](https://gateway-api.sigs.k8s.io/docs/implementations/list/)  

## Helm installation
**CRDS**
```bash
helm upgrade -i --create-namespace \
  --namespace agentgateway-system \
  --version v1.3.1 agentgateway-crds oci://cr.agentgateway.dev/charts/agentgateway-crds
```
**AgentGateway**
```bash
helm upgrade -i -n agentgateway-system agentgateway oci://cr.agentgateway.dev/charts/agentgateway \
--version v1.3.1
```
## AgentGateway Parameters
```yaml
kind: AgentgatewayParameters
apiVersion: agentgateway.dev/v1alpha1
metadata:
  name: agentgateway-proxy-params
  namespace: agentgateway-system
spec:
  logging:
    format: json
  service:
    metadata:
      annotations:
        networking.gke.io/load-balancer-type: "Internal"
        networking.gke.io/load-balancer-ip-addresses: <name_of_the_static_ip>
        networking.gke.io/internal-load-balancer-allow-global-access: "true"
    spec:
      loadbalancerIP: <value_of_the_static_ip>
      externalTrafficPolicy: Local
  deployment:
    spec:
      replicas: 3
      template:
        metadata:
          labels:
            component: agentgateway-proxy
        spec:
          topologySpreadConstraints:
            - maxSkew: 1
              topologyKey: topology.kubernetes.io/zone
              whenUnsatisfiable: ScheduleAnyway
              labelSelector:
                matchLabels:
                  app.kubernetes.io/instance: agentgateway
                  app.kubernetes.io/name: agentgateway
                  component: agentgateway-proxy
              matchLabelKeys:
                - pod-template-hash
                - controller-revision-hash
            - maxSkew: 1
              topologyKey: kubernetes.io/hostname
              whenUnsatisfiable: DoNotSchedule
              labelSelector:
                matchLabels:
                  app.kubernetes.io/instance: agentgateway
                  app.kubernetes.io/name: agentgateway
                  component: agentgateway-proxy
              matchLabelKeys:
                - pod-template-hash
                - controller-revision-hash

```
## AgentGateway Proxy
```yaml
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: agentgateway-proxy
  namespace: agentgateway-system
spec:
  gatewayClassName: agentgateway
  infrastructure:
    parametersRef:
      name: agentgateway-proxy-params
      group: agentgateway.dev
      kind: AgentgatewayParameters
  allowedListeners:
    namespaces:
      from: Selector
      selector:
        matchLabels:
          gateway.networking.k8s.io/allowed-listener-creation: "true"
  listeners:
    - name: http
      protocol: HTTP
      port: 80
```
## ListenerSet
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <app>
  labels:
    gateway.networking.k8s.io/allowed-listener-creation: "true"
```
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: ListenerSet
metadata:
  name: <app-listenerset>
  namespace: <app>
spec:
  parentRef:
    name: parent-gateway
    kind: Gateway
    group: gateway.networking.k8s.io
  - name: http
    hostname: <app.example.com>
    port: 80
    protocol: HTTP
```
## HTTPRoute
```yaml
piVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: <app-route>
  namespace: <app>
spec:
  parentRefs:
  - kind: ListenerSet
    name: <app-listenerset>
    sectionName: http
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: <app-app>
      port: 80
```
