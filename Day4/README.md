# Kubernetes Day 4 — Services & Networking

## 1. Day 4 Overview

Kubernetes Pods are ephemeral. Their IP addresses can change when Pods are replaced.

A Kubernetes **Service** provides a stable network endpoint through which clients can communicate with a set of Pods.

Core mental model:

```text
Pods are ephemeral
       ↓
Pod IPs can change
       ↓
Service provides stable access
       ↓
Selector identifies Pods
       ↓
EndpointSlices track endpoints
       ↓
DNS provides Service discovery
```

---

# 2. Why Do We Need a Service?

Suppose we have three backend Pods:

```text
backend-pod-1 → 10.42.1.10
backend-pod-2 → 10.42.2.15
backend-pod-3 → 10.42.3.20
```

A frontend Pod should not directly depend on these IP addresses.

If:

```text
backend-pod-2
```

is deleted and Kubernetes creates a replacement:

```text
backend-pod-4 → 10.42.4.25
```

the old IP:

```text
10.42.2.15
```

is no longer valid for that Pod.

Therefore:

```text
Frontend
   ↓
Pod IP
```

is not a reliable production architecture.

Instead:

```text
Frontend
   ↓
backend-service
   ↓
Backend Pods
```

The Service provides a stable endpoint while the backend Pods can change.

---

# 3. What Is a Kubernetes Service?

A Service is a Kubernetes object that provides a stable network endpoint for accessing a group of Pods.

A Service typically uses a **selector** to identify the Pods that should receive traffic.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
```

The Service selects Pods with:

```yaml
labels:
  app: backend
```

Mental model:

```text
Service
   ↓
Selector
   ↓
Matching Pods
```

---

# 4. Service Selector

A selector defines which Pods belong to the Service backend.

Example Pod:

```yaml
metadata:
  labels:
    app: backend
```

Service:

```yaml
spec:
  selector:
    app: backend
```

The labels match, so the Pod becomes a backend for the Service.

If the Service has:

```yaml
selector:
  app: api
```

while Pods have:

```yaml
labels:
  app: backend
```

there is no match.

Therefore:

```text
Service
   ↓
No matching Pods
   ↓
No usable endpoints
```

Important:

> Kubernetes does not infer that `backend` and `api` represent the same application. The selector relationship must be explicitly configured.

---

# 5. `port` vs `targetPort` vs `containerPort`

This is one of the most important Day 4 interview topics.

Consider:

```yaml
ports:
  - port: 80
    targetPort: 8080
```

## `port`

This is the port exposed by the Service.

```text
Service → port 80
```

Clients communicate with the Service on this port.

---

## `targetPort`

This is the port on the backend Pod/container to which the Service sends traffic.

```text
Service port 80
       ↓
targetPort 8080
       ↓
Application
```

---

## `containerPort`

Example:

```yaml
containers:
  - name: backend
    image: my-backend
    ports:
      - containerPort: 8080
```

`containerPort` describes the port that the container is intended to listen on.

### Important correction

`containerPort` does **not** make the application listen on that port.

The application itself must bind/listen on the port.

For example, if the application is actually listening on:

```text
9090
```

then simply specifying:

```yaml
containerPort: 8080
```

does not move the application to port `8080`.

---

## Complete mental model

```text
Application
    ↓
actually listens on 8080

containerPort: 8080
    ↓
describes/document intended container port

Service targetPort: 8080
    ↓
Service forwards traffic here

Service port: 80
    ↓
Clients access Service on port 80
```

Remember:

```text
port       → Service port
targetPort → backend application port
containerPort → container specification/documentation
```

---

# 6. Service Types

Kubernetes provides several Service types.

## 6.1 ClusterIP

ClusterIP is the default Service type.

It provides internal cluster access.

```text
Frontend Pod
      ↓
ClusterIP Service
      ↓
Backend Pods
```

Typical production use:

```text
Frontend → Backend
Backend → Database
Backend → Redis
```

These applications usually don't need direct public exposure.

---

# 6.2 NodePort

NodePort exposes the Service through a port on each node.

Typical flow:

```text
External Client
       ↓
Node IP : NodePort
       ↓
Service
       ↓
Pods
```

The commonly used default NodePort range is:

```text
30000–32767
```

unless the cluster configuration changes it.

NodePort is useful for testing and some architectures, but production HTTP/HTTPS exposure commonly uses a LoadBalancer and/or Ingress architecture.

---

# 6.3 LoadBalancer

LoadBalancer exposes a Service through an external load-balancing mechanism, commonly provided by a cloud platform.

Conceptually:

```text
Internet
    ↓
Cloud Load Balancer
    ↓
LoadBalancer Service
    ↓
Pods
```

Important:

> LoadBalancer Service and Ingress are different Kubernetes concepts.

A LoadBalancer Service provides external network exposure.

Ingress provides HTTP/HTTPS routing rules.

---

# 6.4 ExternalName

ExternalName allows a Kubernetes Service name to act as a DNS alias for an external hostname.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: database.example.com
```

Conceptually:

```text
Application
    ↓
external-db
    ↓
database.example.com
```

It does not select normal Kubernetes Pods in the same way as a selector-based Service.

---

# 7. Headless Service

A Headless Service is created with:

```yaml
spec:
  clusterIP: None
```

It does not provide a normal virtual ClusterIP.

Instead, Kubernetes DNS can expose the individual endpoints associated with the Service.

Normal Service:

```text
Client
   ↓
ClusterIP
   ↓
Pod
```

Headless Service:

```text
Client
   ↓
DNS
   ↓
Individual Pod endpoints
```

A common use case is StatefulSet-based applications where clients need to discover individual instances.

For example:

```text
database-0
database-1
database-2
```

Headless Services are useful when the application needs endpoint-level discovery rather than a single virtual Service IP.

---

# 8. Service Discovery

Kubernetes provides built-in DNS-based Service discovery.

Suppose:

```text
Service: backend-service
Namespace: backend
```

The full DNS name is generally:

```text
backend-service.backend.svc.cluster.local
```

General format:

```text
<service>.<namespace>.svc.<cluster-domain>
```

Common default cluster domain:

```text
cluster.local
```

Therefore:

```text
<service>.<namespace>.svc.cluster.local
```

---

# 9. Namespace and DNS

Suppose:

```text
frontend namespace
backend namespace
```

and:

```text
Service:
backend-service

Namespace:
backend
```

From another namespace, use the namespace-qualified DNS name:

```text
backend-service.backend.svc.cluster.local
```

Short names are namespace-relative.

Therefore, when communicating across namespaces, use an appropriate namespace-qualified Service name.

---

# 10. Service DNS Does Not Select the Pod

A common misunderstanding is:

```text
DNS → directly chooses backend Pod
```

That is not the correct mental model.

For a normal ClusterIP Service:

```text
Frontend Pod
      ↓
DNS resolution
      ↓
Service name
      ↓
ClusterIP
      ↓
Service traffic handling
      ↓
Backend endpoint
      ↓
Pod
```

DNS primarily resolves the Service name to the Service address.

The Service's backend endpoints are maintained separately.

---

# 11. EndpointSlices

Kubernetes needs to know which Pods are currently available as Service backends.

EndpointSlices provide this endpoint information.

Conceptually:

```text
Service
   ↓
EndpointSlice
   ├── Pod IP 1
   ├── Pod IP 2
   └── Pod IP 3
```

When Pods are added, removed, or replaced, the backend endpoint information can change.

Modern Kubernetes uses **EndpointSlices** rather than relying on the older Endpoints API for scalable endpoint tracking.

---

# 12. kube-proxy

`kube-proxy` runs on Kubernetes nodes and helps implement Service networking.

It watches Service and endpoint information and programs the node's networking rules so Service traffic can be directed toward backend endpoints.

Simplified mental model:

```text
Client Pod
    ↓
Service ClusterIP
    ↓
Node networking rules
    ↓
Backend Pod
```

The exact packet-processing implementation depends on the Kubernetes networking configuration.

For interview purposes:

> kube-proxy helps implement Service networking and route/forward Service traffic toward backend endpoints.

---

# 13. Complete Service Traffic Flow

Suppose:

```text
Frontend Pod
```

requests:

```text
http://backend-service:80
```

The conceptual flow is:

```text
Frontend Pod
      ↓
Kubernetes DNS
      ↓
backend-service
      ↓
Service ClusterIP
      ↓
Service networking
      ↓
EndpointSlice
      ↓
One backend endpoint
      ↓
Backend Pod
```

The important components are:

```text
DNS
Service
EndpointSlice
kube-proxy / service networking
Pod
```

---

# 14. Service vs Ingress

This distinction is extremely important.

## Service

Solves:

> How do I provide stable network access to my Pods?

```text
Service
   ↓
Pods
```

## Ingress

Solves:

> How do I route external HTTP/HTTPS requests to different Services?

Example:

```text
Internet
    ↓
Load Balancer
    ↓
Ingress Controller
    ↓
Ingress Rules
   /          \
  /            \
Frontend      Backend
Service       Service
  ↓              ↓
Pods            Pods
```

Example host-based routing:

```text
frontend.example.com → frontend-service
api.example.com      → backend-service
```

---

# 15. Service Without a Selector

A Service does not always have to select Pods using a selector.

A Service without a selector can be used with manually managed EndpointSlices.

This can be useful when the backend is outside the normal Pod-selection model.

Conceptually:

```text
Service
   ↓
Manually managed EndpointSlice
   ↓
External endpoint
```

This is different from a normal selector-based Service.

---

# 16. Session Affinity

By default, Service traffic can be distributed among available backend endpoints.

Kubernetes also supports:

```yaml
sessionAffinity: ClientIP
```

This can provide client-IP-based session affinity.

Conceptually:

```text
Client A
   ↓
Backend Pod 1

Client B
   ↓
Backend Pod 2
```

Session affinity should be used only when the application's requirements justify it.

For scalable applications, stateless design is generally preferred.

---

# 17. `externalTrafficPolicy`

For Services receiving external traffic, you may encounter:

```yaml
externalTrafficPolicy: Cluster
```

or:

```yaml
externalTrafficPolicy: Local
```

## Cluster

Traffic may be routed to an endpoint on another node.

Conceptually:

```text
External Client
      ↓
Node 1
      ↓
Node 2
      ↓
Pod
```

## Local

Traffic is directed only toward local endpoints when possible.

One reason to use `Local` is preserving the original client source IP in scenarios where the networking implementation supports it.

However, Local can create uneven traffic distribution if some nodes have no local endpoints.

---

# 18. Service Troubleshooting

## Scenario

```text
Service exists
Pods are Running
Service has no endpoints
```

First check:

```bash
kubectl get svc backend-service
```

Then:

```bash
kubectl describe svc backend-service
```

Check the selector.

Next:

```bash
kubectl get pods --show-labels
```

Compare:

```text
Service selector
        ↓
Pod labels
```

Then inspect EndpointSlices:

```bash
kubectl get endpointslice
```

Also inspect Pod readiness:

```bash
kubectl get pods
kubectl describe pod <pod-name>
```

And events:

```bash
kubectl get events
```

Possible causes include:

* Selector mismatch
* Pod labels incorrect
* Pods not Ready
* Incorrect Service configuration
* EndpointSlice issues

---

# 19. Service Troubleshooting — Wrong `targetPort`

Suppose:

```yaml
ports:
  - port: 80
    targetPort: 8080
```

but the application listens on:

```text
9090
```

Then the Service is forwarding traffic to the wrong port.

Fix:

```yaml
ports:
  - port: 80
    targetPort: 9090
```

Changing only:

```yaml
containerPort: 9090
```

does not fix a Service whose `targetPort` is still wrong.

The Service must target the port where the application is actually listening.

---

# 20. Production Troubleshooting Flow

If users report:

> Application is not accessible.

Don't immediately jump to the Pod.

Troubleshoot layer by layer:

```text
1. DNS
      ↓
2. Load Balancer
      ↓
3. Ingress Controller
      ↓
4. Ingress rules
      ↓
5. Service
      ↓
6. EndpointSlices
      ↓
7. Pods
      ↓
8. Application
```

Useful commands:

```bash
kubectl get ingress
kubectl describe ingress <name>

kubectl get svc
kubectl describe svc <name>

kubectl get endpointslice

kubectl get pods
kubectl describe pod <pod>

kubectl logs <pod>
```

For internal connectivity:

```bash
kubectl exec -it <pod> -- curl http://backend-service:80
```

The goal is to identify the exact layer where communication fails.

---

# 21. Common Interview Traps

## Trap 1

"Pod restart always changes the Pod IP."

Incorrect.

A container restart inside the same Pod normally keeps the Pod IP.

Pod replacement can result in a new IP.

---

## Trap 2

"`containerPort` makes the application listen on that port."

Incorrect.

The application itself determines which port it listens on.

---

## Trap 3

"`targetPort` and `containerPort` are always the same."

Incorrect.

They can be the same, but they don't have to be.

---

## Trap 4

"DNS selects the backend Pod."

Not exactly.

DNS resolves the Service name. Service networking then directs traffic toward backend endpoints.

---

## Trap 5

"LoadBalancer and Ingress are the same."

Incorrect.

```text
LoadBalancer → external network exposure

Ingress → HTTP/HTTPS routing
```

---

## Trap 6

"Running Pod automatically means Service will send traffic to it."

Not necessarily.

Pod readiness and endpoint conditions matter.

---

# 22. Important Command Cheat Sheet

### Services

```bash
kubectl get svc
kubectl get svc <name>
kubectl describe svc <name>
```

### Pods and labels

```bash
kubectl get pods
kubectl get pods --show-labels
kubectl get pods -l app=backend
```

### EndpointSlices

```bash
kubectl get endpointslice
kubectl get endpointslice -l kubernetes.io/service-name=backend-service
```

### DNS testing

```bash
kubectl exec -it <pod> -- nslookup backend-service
```

or, depending on the image:

```bash
kubectl exec -it <pod> -- getent hosts backend-service
```

### Connectivity testing

```bash
kubectl exec -it <pod> -- curl http://backend-service:80
```

### Ingress

```bash
kubectl get ingress
kubectl describe ingress <name>
```

### Events

```bash
kubectl get events
```

---

# 23. Hands-on Labs

## Lab 1 — ClusterIP

Create:

```text
Backend Deployment
+
ClusterIP Service
```

Test from another Pod:

```bash
kubectl exec -it <frontend-pod> -- curl http://backend-service
```

---

## Lab 2 — Self-Healing Behind Service

1. Create backend Deployment.
2. Create Service.
3. Identify backend Pod.
4. Delete the Pod.
5. Observe replacement Pod.
6. Test Service again.

Important observation:

```text
Old Pod disappears
       ↓
New Pod appears
       ↓
Service continues to work
```

---

## Lab 3 — Break the Selector

Intentionally change:

```yaml
selector:
  app: backend
```

to:

```yaml
selector:
  app: wrong
```

Then:

```bash
kubectl get endpointslice
```

Observe that the Service no longer has the expected backend endpoints.

Fix the selector.

---

## Lab 4 — Wrong `targetPort`

Configure:

```text
Application → 8080
Service targetPort → 9090
```

Observe the failure.

Correct:

```text
targetPort → 8080
```

---

## Lab 5 — NodePort

Create:

```text
NodePort Service
```

Access:

```text
NodeIP:NodePort
```

Observe how the request reaches the Pods.

---

## Lab 6 — Headless Service

Create:

```yaml
clusterIP: None
```

Inspect DNS resolution and observe the endpoint-level discovery behavior.

---

# 24. Production Architecture

A common Kubernetes web application architecture is:

```text
                    Internet
                       ↓
               Cloud Load Balancer
                       ↓
                Ingress Controller
                       ↓
                  Ingress Rules
                  /           \
                 /             \
                ↓               ↓
       frontend-service    backend-service
              ↓                  ↓
         Frontend Pods      Backend Pods
                                  ↓
                           database-service
                                  ↓
                             Database
```

Services provide stable communication between application components.

Ingress handles HTTP/HTTPS routing.

---

# 25. Core Mental Models

### Pod vs Service

```text
Pod
→ ephemeral workload instance

Service
→ stable network endpoint
```

### Selector

```text
Service
   ↓
Selector
   ↓
Matching Pod labels
```

### EndpointSlice

```text
Service
   ↓
EndpointSlices
   ↓
Current backend endpoints
```

### DNS

```text
Service name
   ↓
Service DNS
   ↓
Service address
```

### Ports

```text
port
 ↓
Service port

targetPort
 ↓
Backend application port

containerPort
 ↓
Container specification/documentation
```

### Service vs Ingress

```text
Service
→ stable access to Pods

Ingress
→ HTTP/HTTPS routing to Services
```

---

# 26. Day 4 Revision Checklist

Before considering Day 4 fully revised, you should be able to explain:

* [ ] Why Pod IPs should not be used directly
* [ ] What a Service is
* [ ] Why Services provide stable connectivity
* [ ] Service selectors
* [ ] Pod labels
* [ ] ClusterIP
* [ ] NodePort
* [ ] LoadBalancer
* [ ] ExternalName
* [ ] Headless Service
* [ ] Service Discovery
* [ ] Kubernetes DNS
* [ ] Service DNS format
* [ ] EndpointSlices
* [ ] kube-proxy
* [ ] Service traffic flow
* [ ] `port`
* [ ] `targetPort`
* [ ] `containerPort`
* [ ] Service vs Ingress
* [ ] Selector mismatch troubleshooting
* [ ] No-endpoint troubleshooting
* [ ] Wrong targetPort troubleshooting
* [ ] Production Service architecture

---

# 27. Final Day 4 Mental Map

Remember this:

```text
                    Kubernetes Networking

                           DNS
                            ↓
                     Service Name
                            ↓
                         Service
                            ↓
                        Selector
                            ↓
                     EndpointSlices
                            ↓
                    Backend Endpoints
                            ↓
                           Pods
```

And for external traffic:

```text
Internet
   ↓
Load Balancer
   ↓
Ingress Controller
   ↓
Ingress
   ↓
Service
   ↓
EndpointSlices
   ↓
Pods
```

The most important sentence from Day 4:

> **Pods are ephemeral. Services provide stable connectivity. Selectors connect Services to Pods. DNS makes Services discoverable. EndpointSlices track backend endpoints.**
