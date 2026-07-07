# Miscellaneous resources  
## Monitoring
### Logs
Usually there is an agent (daemonset) running in the cluster collecting all logs and shipping them to general log storage.  
[VictoriaLogs Agent](https://docs.victoriametrics.com/victorialogs/vlagent/)  
[Vector](https://vector.dev/)  
[Grafana Alloy](https://grafana.com/docs/alloy/latest/)  
[Fluent Bit](https://fluentbit.io/)  
[OpenTelemetry Collector](https://opentelemetry.io/docs/collector/)  
[Filebeat](https://www.elastic.co/beats/filebeat)  
[Fluentd](https://www.fluentd.org/)  

Modern logging expects structured logs (json) to be produced by the application.  

### Metrics
[Metrics - Getting started](https://prometheus-operator.dev/docs/developer/getting-started/)  
Standard: `monitoring.coreos.com/v1` defines 2 types of probes: `PodMonitor` and `ServiceMonitor`.  
[VictoriaMetrics Agent](https://docs.victoriametrics.com/victoriametrics/vmagent/)  
[Grafana Alloy](https://grafana.com/docs/alloy/latest/)  
[Prometheus Agent](https://prometheus-operator.dev/docs/platform/prometheus-agent/)  

Do not log monitoring data, use `/metrics` to expose Prometheus metrics.  

### Traces
Collection of traces usually follows a "push" data model and "usually" via an SDK directly to  the collector storage.  
[OpenTelemetry](https://github.com/open-telemetry/opentelemetry-collector/blob/main/exporter/otlphttpexporter/README.md)  

## Probes
### Rediness probes
All applications need to have [Readiness probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-readiness-probes) defined. They signal when the pod is ready to accept traffic.  
The readiness probe doesn't include dependencies to services such as:  
- databases  
- database migrations  
- APIs  
- third party services  

### Liveness probes
The [Liveness probe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-liveness-command) is designed to restart your container when it's stuck.  
The Liveness probe should be used as a recovery mechanism only in case the process is not responsive.  
How to do it:  
- Expose an endpoint from your app  
- The endpoint always replies with a success response  
- Consume the endpoint from the Liveness probe  

## Containers should be allowed to crash
[Let it Crash!](https://blog.colinbreck.com/kubernetes-liveness-and-readiness-probes-revisited-how-to-avoid-shooting-yourself-in-the-other-foot/#letitcrash)  
Examples of unrecoverable errors are:  
- an uncaught exception  
- a typo in the code (for dynamic languages)  
- unable to load a header or dependency  

Please note that you should not signal a failing Liveness probe.  
Instead, you should immediately exit the process and let the kubelet restart the container.  

## Independency
When the app starts, it shouldn't crash because a dependency such as a database isn't ready.  
Instead, the app should keep retrying to connect to the database until it succeeds.  
Kubernetes expects that application components can be started in any order.  

## Graceful shutdown
The app doesn't shut down on SIGTERM, but it gracefully terminates connections.  
The app still processes incoming requests in the grace period. Close all idle keep-alive sockets.  

## High-Availability
Run more than one replica for your Deployment.  
Avoid Pods being placed into a single node.  
Set Pod disruption budgets.  

## Topology spread constraints / Anti-affinity
Topology spread constraints and Anti-affinity rules are there to control pod distribution over the specific kubernetes nodes.  
There is little to no difference between Topology spread constraints and Anti-affinity rules for the basic usage, though Topology spread constraints seem to be the more modern approach and offer more functionalities, so use them wherever possible.  
  
To achieve uniform spread over availability zones for pods without allowing more pods of the same "type" to be present on same underlying kubernetes node see example bellow:  
```yaml title=".spec"
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: <application-name>
        app.kubernetes.io/component: <application-component>
    matchLabelKeys:
      - pod-template-hash
      - controller-revision-hash
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: <application-name>
        app.kubernetes.io/component: <application-component>
    matchLabelKeys:
      - pod-template-hash
      - controller-revision-hash
```  

## Tagging (Labeling)
You could tag your Pods with (app.kubernetes.io/) (alphanumeric characters, dashes (-) and SemVer2 for version):  
- name, the name of the application such "user-api"  
- instance, a unique name identifying the instance of an application (you could use the container image tag)  
- version, the current version of the appl (an incremental counter)  
- component, the component within the architecture such as "API" or "database"  
- part-of, the name of a higher-level application this one is part of such as "payment gateway"  
- managed-by, the tool being used to manage the operation of an application such as "kubectl" or "Helm"  
