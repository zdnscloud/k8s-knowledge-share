# Service Mesh
> A service mesh is a way to control how different parts of an application share 
> data with one another.
>                      -- redhat

> A service mesh is a dedicated infrastructure layer for making service-to-service 
> communication safe, fast, and reliable. 
>                      -- buoyant

> A service mesh is a configurable, low‑latency infrastructure layer designed to 
> handle a high volume of network‑based interprocess communication among application 
> infrastructure services using application programming interfaces (APIs). A service 
> mesh ensures that communication among containerized and often ephemeral application 
> infrastructure services is fast, reliable, and secure. The mesh provides critical 
> capabilities including service discovery, load balancing, encryption, observability, 
> traceability, authentication and authorization, and support for the circuit breaker 
> pattern.
>                      -- NGINX

![""](servicemesh.png)

# Key features
1. Resiliency for inter-service communication
   - Circuit-breaking
   - Retries and timeout
   - Fault injection
   - Load balancing
1. Service Discovery
1. Routing
1. Observability
1. Security & Access control

# Service Mesh Interface(SMI)
Define a set of CRDs to provide the following functionality
- Traffic policy
- Traffic telemetry
- Traffic management

# Traffic Split
Split traffic targeting the foreground service to backend 
services based on weight, all the services are standard k8s 
services.
1. Foreground service
select all the workload
1. Backend services
select partial workloads which is in a rollout phase.
```yaml
apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: my-trafficsplit
spec:
  # The root service that clients use to connect to the destination application.
  service: my-website
  # Services inside the namespace with their own selectors, endpoints and configuration.
  backends:
  - service: my-website-v1
    weight: 50
  - service: my-website-v2
    weight: 50
```
Note: there is not port in TrafficSplit, the port should be same
in all services.

## Workflow for workload rolling out
For running deployment deploy-v1 with service svc-v1, when rollout deploy-v2
1. Create new service svc, deploy-v2 and service svc-v2
1. Create TrafficSplit set svc-v1 weight to 100 and svc-v2 weight to 0
1. Update TrafficSplit, set svc-v2 weight to 50 
1. Update TrafficSplit, delete svc-v1
```yaml
apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: foobar-rollout
spec:
  service: foobar
  backends:
  - service: foobar-v1
    weight: 100
  - service: foobar-v2
    weight: 0


apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: foobar-rollout
spec:
  service: foobar
  backends:
  - service: foobar-v1
    weight: 100
  - service: foobar-v2
    weight: 50

apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: foobar-rollout
spec:
  service: foobar
  backends:
  - service: foobar-v2
    weight: 50
```

# Traffic Access Control
- Specify which pods could access which pods in which way
- Pod is identified by ServiceAccount
- Access method is defined as http path and http method
```yaml
apiVersion: specs.smi-spec.io/v1alpha1
kind: HTTPRouteGroup
metadata:
  name: api-service-routes
matches:
- name: api
  pathRegex: /api
  methods: ["*"]
- name: metrics
  pathRegex: /metrics
  methods: ["GET"]

---
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha1
metadata:
  name: api-service-metrics
  namespace: default
destination:
  kind: ServiceAccount
  name: api-service
  namespace: default
specs:
- kind: HTTPRouteGroup
  name: api-service-routes
  matches:
  - metrics
sources:
- kind: ServiceAccount
  name: prometheus
  namespace: default
```

# Traffic Metrics
- Provide metrics related to HTTP traffic.
- Normally it's used to fetcher traffic between pods
```yaml
apiVersion: metrics.smi-spec.io/v1alpha1
kind: TrafficMetrics
# See ObjectReference v1 core for full spec
resource:
  name: foo-775b9cbd88-ntxsl
  namespace: foobar
  kind: Pod
edge:
  direction: to
  resource:
    name: baz-577db7d977-lsk2q
    namespace: foobar
    kind: Pod
timestamp: 2019-04-08T22:25:55Z
window: 30s
metrics:
- name: p99_response_latency
  unit: seconds
  value: 10m
- name: p90_response_latency
  unit: seconds
  value: 10m
- name: p50_response_latency
  unit: seconds
  value: 10m
- name: success_count
  value: 100
- name: failure_count
  value: 100
```
Get the metric info through k8s extension Api Server
