# Kubernetes Day 4 — Services & Networking Interview Answers

## Q1. Why do we need a Service instead of directly using Pod IPs?

### Production Answer

Kubernetes Pods are ephemeral, so their IP addresses should not be treated as stable application endpoints.

If a Pod is replaced because of scaling, node failure, or another event, the replacement Pod can receive a different IP address. If the frontend directly depends on the old Pod IP, communication can break.

A Service provides a stable network endpoint and uses selectors to identify the appropriate backend Pods. Kubernetes maintains the backend endpoint information through EndpointSlices.

So the architecture becomes:

```text
Frontend
   ↓
Service
   ↓
Selector
   ↓
EndpointSlices
   ↓
Backend Pods
```

The frontend doesn't need to know the individual Pod IPs.

### Important Caveat

A container restart inside the same Pod normally does not change the Pod IP. Pod replacement is what can result in a new Pod IP.

---

# Q2. Explain `port`, `targetPort`, and `containerPort`.

Given:

```yaml
ports:
  - port: 80
    targetPort: 8080
```

### Production Answer

`port` is the port exposed by the Service. Clients access the Service on port 80.

`targetPort` is the port on the backend Pod where the Service sends traffic. In this example, traffic received on Service port 80 is forwarded to port 8080 on the backend.

`containerPort` is specified in the container definition and describes the port the container is intended to expose/listen on. It does not itself make the application listen on that port.

For example:

```text
Application → actually listens on 8080
containerPort → 8080
Service targetPort → 8080
Service port → 80
```

The application itself must actually bind to the target port.

---

# Q3. Explain ClusterIP, NodePort, LoadBalancer, and ExternalName.

### ClusterIP

ClusterIP is the default Service type and is mainly used for internal cluster communication.

```text
Frontend Pod
     ↓
ClusterIP Service
     ↓
Backend Pods
```

### NodePort

NodePort exposes a Service through a port on each node.

```text
External Client
     ↓
NodeIP:NodePort
     ↓
Service
     ↓
Pods
```

The default NodePort range is commonly 30000–32767 unless configured differently.

### LoadBalancer

LoadBalancer exposes a Service through an external load-balancing mechanism, commonly provided by a cloud platform.

```text
Internet
    ↓
Cloud Load Balancer
    ↓
Service
    ↓
Pods
```

### ExternalName

ExternalName provides a DNS-based alias from a Kubernetes Service name to an external hostname.

For example:

```text
Application
    ↓
external-db
    ↓
database.example.com
```

It does not select Kubernetes Pods through a normal selector.

---

# Q4. A Service has no endpoints, but Pods are Running. How would you troubleshoot it?

### Production Answer

I would troubleshoot the Service layer by layer.

First:

```bash
kubectl get svc backend-service
kubectl describe svc backend-service
```

I would check the Service selector.

Then:

```bash
kubectl get pods --show-labels
```

I would compare the Service selector with the Pod labels.

For example:

```text
Service:
app=backend

Pod:
app=backend
```

They must match.

Next I would inspect EndpointSlices:

```bash
kubectl get endpointslice
```

I would also check whether the Pods are Ready:

```bash
kubectl get pods
kubectl describe pod <pod-name>
```

Finally, I would check events:

```bash
kubectl get events
```

Common causes include selector mismatch, incorrect labels, Pods not being Ready, or Service configuration problems.

---

# Q5. Explain the traffic flow when a frontend Pod accesses `backend-service:80`.

### Production Answer

First, the frontend Pod uses Kubernetes DNS to resolve the Service name.

For a normal ClusterIP Service, DNS resolves the Service name to the Service's virtual IP.

The request then goes to the Service, and Kubernetes service networking uses the available backend endpoint information to direct the traffic toward one of the backend Pods.

Conceptually:

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
Backend endpoint
      ↓
Backend Pod
```

`kube-proxy` commonly participates in implementing Service networking on nodes, depending on the cluster networking implementation.

---

# Q6. What is a Headless Service?

### Production Answer

A Headless Service is a Service configured with:

```yaml
clusterIP: None
```

It does not provide the normal virtual ClusterIP.

Instead, Kubernetes DNS can expose the individual endpoints associated with the Service.

This is useful when an application needs to discover individual instances rather than communicate through a single virtual IP.

A common example is a StatefulSet-based application.

Conceptually:

```text
Normal Service:

Client
  ↓
ClusterIP
  ↓
Pod


Headless Service:

Client
  ↓
DNS
  ↓
Individual Pod endpoints
```

---

# Q7. Explain Kubernetes Service DNS.

Suppose:

```text
Service: backend-service
Namespace: backend
```

### Production Answer

The general Kubernetes Service DNS format is:

```text
<service>.<namespace>.svc.<cluster-domain>
```

With the common cluster domain:

```text
backend-service.backend.svc.cluster.local
```

If a client is in another namespace, using the namespace-qualified name avoids ambiguity and correctly targets the Service in the `backend` namespace.

---

# Q8. How would you troubleshoot this production architecture?

```text
Internet
   ↓
Load Balancer
   ↓
Ingress
   ↓
Service
   ↓
Pods
```

### Production Answer

I would troubleshoot from the outside toward the application.

First, I would check DNS and confirm that the hostname resolves correctly.

Then I would check the external Load Balancer.

Next, I would inspect the Ingress:

```bash
kubectl get ingress
kubectl describe ingress <name>
```

Then I would check the Service:

```bash
kubectl get svc
kubectl describe svc <name>
```

Next, I would inspect EndpointSlices:

```bash
kubectl get endpointslice
```

Then I would check the backend Pods:

```bash
kubectl get pods
kubectl describe pod <pod-name>
```

and inspect logs:

```bash
kubectl logs <pod-name>
```

For internal connectivity, I can test the Service from another Pod:

```bash
kubectl exec -it <pod-name> -- curl http://backend-service:80
```

This allows me to identify the layer where the failure occurs instead of randomly troubleshooting components.

---

# Q9. Why does a Service with `selector: app=api` not select Pods with `app=backend`?

### Production Answer

Kubernetes Services use explicit label selectors.

If the Pods have:

```yaml
labels:
  app: backend
```

but the Service has:

```yaml
selector:
  app: api
```

the selector does not match the Pod labels.

Therefore, the Service will not identify those Pods as its backend endpoints.

Kubernetes does not infer that `api` and `backend` refer to the same application.

The selector must explicitly match the intended Pod labels.

---

# Q10. Service vs Ingress

### Production Answer

A Service and an Ingress solve different problems.

A Service provides stable network access to a set of Pods.

Ingress provides HTTP/HTTPS routing rules that direct external requests to Services.

For example:

```text
Internet
    ↓
Load Balancer
    ↓
Ingress Controller
    ↓
Ingress
   /       \
  /         \
Frontend    Backend
Service     Service
  ↓            ↓
Pods          Pods
```

With host-based routing:

```text
frontend.example.com → frontend-service
api.example.com      → backend-service
```

This allows multiple HTTP/HTTPS applications to be routed through an Ingress architecture rather than exposing every application individually through a separate external endpoint.

---

# Q11. What is wrong if `targetPort` is 8080 but the application listens on 9090?

### Production Answer

The Service is forwarding traffic to the wrong backend port.

For example:

```yaml
port: 80
targetPort: 8080
```

but the application listens on:

```text
9090
```

The Service should be configured as:

```yaml
port: 80
targetPort: 9090
```

Changing only:

```yaml
containerPort: 9090
```

does not fix the problem if the Service still has:

```yaml
targetPort: 8080
```

`containerPort` does not control which port the application actually listens on.

---

# Q12. Explain Kubernetes Services and networking as a production design.

### Production Answer

In Kubernetes, Pods are ephemeral, so their IP addresses should not be treated as stable application endpoints. When Pods are replaced, their IPs can change.

To solve this, Kubernetes provides Services. A Service provides a stable network endpoint and uses label selectors to identify the appropriate backend Pods.

For example, a frontend Pod can communicate with:

```text
backend-service:80
```

instead of directly using backend Pod IPs.

Kubernetes DNS provides Service discovery, and EndpointSlices maintain the current backend endpoint information.

For a normal Service, the traffic flow can be represented as:

```text
Frontend Pod
     ↓
DNS
     ↓
Service
     ↓
EndpointSlices
     ↓
Backend Pod
```

There are different Service types. ClusterIP is generally used for internal communication, NodePort exposes a Service through a node port, LoadBalancer provides external load-balancing integration, and ExternalName provides a DNS alias to an external hostname.

For HTTP/HTTPS applications, we can use an Ingress architecture:

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
Pods
```

This allows host-based or path-based routing to different Services.

In production, the important principle is that Services provide stable connectivity while Pods remain replaceable and scalable.

---

# Day 4 — Quick Interview Revision

## Remember these five statements

### 1.

> **Pods are ephemeral; Services provide stable network access.**

### 2.

> **Selectors connect Services to Pods through matching labels.**

### 3.

> **DNS discovers the Service; it does not directly select the backend Pod.**

### 4.

> **`port` is the Service port, `targetPort` is the backend application port, and `containerPort` is descriptive configuration in the container specification.**

### 5.

> **Service provides stable access to Pods; Ingress provides HTTP/HTTPS routing to Services.**

---

# Day 4 Production Mental Model

```text
                         Internet
                            ↓
                     Load Balancer
                            ↓
                    Ingress Controller
                            ↓
                         Ingress
                       /        \
                      /          \
                     ↓            ↓
              Frontend Service  Backend Service
                     ↓            ↓
               Frontend Pods  EndpointSlices
                                  ↓
                             Backend Pods
```

The key Kubernetes networking chain to remember is:

```text
Service
   ↓
Selector
   ↓
EndpointSlices
   ↓
Pods
```

And the external traffic chain is:

```text
Internet
   ↓
Load Balancer
   ↓
Ingress
   ↓
Service
   ↓
Pods
```
