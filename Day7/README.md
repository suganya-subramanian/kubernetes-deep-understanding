# Kubernetes Day 7 — Ingress

## 1. Why Do We Need Ingress?

A Kubernetes Service provides a stable network endpoint for Pods.

For example:

```text
Internet
   ↓
LoadBalancer Service
   ↓
Application Service
   ↓
Pods
```

But when we have multiple applications, exposing every application separately can become difficult and expensive.

For example:

```text
app.example.com
api.example.com
admin.example.com
```

Or:

```text
example.com/
example.com/api
example.com/admin
```

Ingress provides HTTP/HTTPS routing to different Kubernetes Services based on:

- Hostname
- URL path
- TLS configuration

---

# 2. What Is Ingress?

An **Ingress** is a Kubernetes API resource used to define rules for routing HTTP and HTTPS traffic to Services.

Ingress itself does not normally process the network traffic.

An **Ingress Controller** watches the Ingress resources and implements the routing rules.

### Basic Architecture

```text
Internet
    ↓
Load Balancer
    ↓
Ingress Controller
    ↓
Ingress Rules
    ↓
Services
    ↓
Pods
```

### Important

> Ingress defines **what routing should happen**.

> Ingress Controller implements **how that routing actually happens**.

---

# 3. Ingress vs Ingress Controller

These two concepts are commonly confused.

## Ingress

Ingress is a Kubernetes API object.

It defines:

- Host rules
- Path rules
- Backend Services
- TLS configuration

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

The Ingress resource describes the desired routing configuration.

---

## Ingress Controller

The Ingress Controller is the component that watches Ingress resources and implements the routing behavior.

Examples include:

- NGINX Ingress Controller
- Traefik
- HAProxy
- Cloud-provider-specific ingress/load-balancing implementations

### Mental Model

```text
Ingress
   ↓
Defines WHAT should happen

Ingress Controller
   ↓
Implements HOW it happens
```

An Ingress without an appropriate controller only defines routing configuration; the controller is responsible for actually processing traffic according to those rules.

---

# 4. Kubernetes Ingress Architecture

A typical production architecture looks like:

```text
                 Internet
                    ↓
                   DNS
                    ↓
             Load Balancer
                    ↓
          Ingress Controller
                    ↓
             Ingress Rules
              ↙          ↘
       Service A       Service B
           ↓               ↓
         Pods            Pods
```

For example:

```text
app.example.com
      ↓
Ingress Controller
      ↓
frontend-service
      ↓
Frontend Pods
```

And:

```text
api.example.com
      ↓
Ingress Controller
      ↓
backend-service
      ↓
Backend Pods
```

---

# 5. Ingress Controller Is Required

Creating an Ingress object alone does not automatically make traffic work.

An appropriate Ingress Controller must be installed and configured.

The flow is:

```text
Ingress Resource
      ↓
Ingress Controller
      ↓
Service
      ↓
Pods
```

The Ingress Controller watches the Kubernetes API for Ingress resources and configures its proxy/load-balancing behavior accordingly.

---

# 6. Host-Based Routing

Host-based routing routes traffic based on the hostname.

For example:

```text
app.example.com
      ↓
frontend-service

api.example.com
      ↓
backend-service
```

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: application-ingress
spec:
  rules:

    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80

    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 80
```

### Traffic Flow

```text
app.example.com
       ↓
Ingress Controller
       ↓
frontend-service
       ↓
Frontend Pods
```

```text
api.example.com
       ↓
Ingress Controller
       ↓
backend-service
       ↓
Backend Pods
```

---

# 7. Path-Based Routing

Path-based routing routes traffic based on the URL path.

For example:

```text
example.com/
      ↓
frontend-service

example.com/api
      ↓
backend-service
```

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-ingress
spec:
  rules:
    - host: example.com
      http:
        paths:

          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 80

          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

### Traffic Flow

```text
example.com/api
       ↓
Ingress Controller
       ↓
backend-service
       ↓
Backend Pods
```

```text
example.com/
       ↓
Ingress Controller
       ↓
frontend-service
       ↓
Frontend Pods
```

---

# 8. Host-Based and Path-Based Routing Can Be Combined

Host-based routing and path-based routing are not mutually exclusive.

They can be combined to create more specific routing rules.

Example:

```text
api.example.com/users
        ↓
user-service

api.example.com/orders
        ↓
order-service

app.example.com/
        ↓
frontend-service
```

### Mental Model

```text
Host-based routing
→ Which domain?

Path-based routing
→ Which URL path?
```

Both can be used together.

---

# 9. PathType

Ingress paths use a `pathType`.

Common values include:

- `Prefix`
- `Exact`
- `ImplementationSpecific`

## Prefix

Matches the path and its subpaths.

Example:

```yaml
path: /api
pathType: Prefix
```

It can match paths such as:

```text
/api
/api/
/api/users
/api/products
```

---

## Exact

Matches the exact path.

Example:

```yaml
path: /login
pathType: Exact
```

This is intended for the exact `/login` path.

---

## ImplementationSpecific

The matching behavior depends on the Ingress Controller implementation.

For portable Kubernetes configurations, prefer `Prefix` or `Exact` when they meet your requirements.

---

# 10. Ingress and Service

Ingress normally routes traffic to a **Service**, not directly to Pods.

The flow is:

```text
Client
  ↓
Ingress
  ↓
Service
  ↓
EndpointSlices
  ↓
Pods
```

Responsibilities are different:

```text
Ingress
→ HTTP/HTTPS routing

Service
→ Stable network endpoint

Service Selector
→ Identifies matching Pods

EndpointSlices
→ Represent backend endpoints

Pods
→ Run the application
```

---

# 11. Ingress vs Service

| Feature | Service | Ingress |
|---|---|---|
| Purpose | Stable network access to Pods | HTTP/HTTPS routing |
| Main Layer | Mainly L4 | Mainly L7 |
| Pod discovery | Uses selectors and EndpointSlices | Routes to Services |
| Host routing | No | Yes |
| Path routing | No | Yes |
| TLS termination | Not its primary purpose | Common use |
| External exposure | NodePort/LoadBalancer can expose | Usually through Ingress Controller |

### Simple Mental Model

```text
Service
→ Provides stable network access to Pods.

Ingress
→ Decides which Service should receive an HTTP/HTTPS request.
```

---

# 12. Ingress vs LoadBalancer

A LoadBalancer and an Ingress have different responsibilities.

A common architecture is:

```text
Internet
   ↓
Cloud Load Balancer
   ↓
Ingress Controller
   ↓
Service
   ↓
Pods
```

The Load Balancer provides external connectivity.

The Ingress Controller provides HTTP/HTTPS routing.

For example:

```text
app.example.com
       ↓
Load Balancer
       ↓
Ingress Controller
       ↓
frontend-service
```

And:

```text
api.example.com
       ↓
Load Balancer
       ↓
Ingress Controller
       ↓
backend-service
```

---

# 13. TLS / HTTPS

Ingress can be used to configure TLS for HTTPS traffic.

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
spec:
  tls:
    - hosts:
        - example.com
      secretName: example-tls

  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

The TLS certificate and private key can be stored in a Kubernetes TLS Secret.

---

# 14. TLS Secret

A TLS Secret commonly contains:

- TLS certificate
- Private key

Example:

```bash
kubectl create secret tls example-tls \
  --cert=tls.crt \
  --key=tls.key
```

Check:

```bash
kubectl get secrets
```

The Ingress references the Secret:

```yaml
tls:
  - hosts:
      - example.com
    secretName: example-tls
```

---

# 15. TLS Termination

TLS termination means the TLS connection from the client is terminated at the Ingress or another infrastructure component.

Example:

```text
Client
   │
   │ HTTPS
   ▼
Ingress Controller
   │
   │ TLS Termination
   ▼
HTTP
   │
   ▼
Service
   │
   ▼
Pods
```

The Ingress Controller:

1. Receives the HTTPS connection.
2. Uses the TLS certificate/private key.
3. Performs the TLS handshake.
4. Decrypts the request.
5. Can inspect HTTP information.
6. Routes the request to the appropriate Service.

### Important

TLS termination does not necessarily mean the entire architecture must use HTTP internally.

Other designs can keep backend traffic encrypted.

---

# 16. TLS Passthrough

TLS passthrough means the Ingress or Load Balancer does **not terminate TLS**.

The encrypted TLS connection is passed through to the backend.

```text
Client
   │
   │ HTTPS
   ▼
Ingress
   │
   │ HTTPS
   ▼
Backend
```

The backend application is responsible for TLS.

### Simple Definition

> TLS Passthrough = Do not decrypt TLS at the Ingress; pass the encrypted connection to the backend.

This behavior depends on the capabilities of the chosen Ingress Controller.

---

# 17. TLS Re-Encryption

Re-encryption means TLS is terminated at the Ingress and then a **new TLS connection** is established from the Ingress to the backend.

```text
Client
   │
   │ HTTPS
   ▼
Ingress
   │
   │ TLS Termination
   ↓
HTTP
   │
   │ New TLS Connection
   ↓
HTTPS
   │
   ▼
Backend
```

The result is:

```text
Client
   │ HTTPS
   ▼
Ingress
   │ HTTPS
   ▼
Backend
```

The backend connection remains encrypted.

### Important Difference

```text
TLS Termination
→ HTTPS → HTTP

TLS Passthrough
→ HTTPS → HTTPS
  without TLS termination at Ingress

TLS Re-encryption
→ HTTPS → HTTPS
  with a new TLS connection to backend
```

---

# 18. TLS Termination vs Passthrough vs Re-Encryption

| Architecture | TLS at Ingress | Backend Connection |
|---|---|---|
| TLS Termination | Terminated | HTTP |
| TLS Passthrough | Not terminated | Original HTTPS connection |
| TLS Re-encryption | Terminated | New HTTPS connection |

### Mental Model

```text
Termination:

Client
  ↓ HTTPS
Ingress
  ↓ HTTP
Backend
```

```text
Passthrough:

Client
  ↓ HTTPS
Ingress
  ↓ HTTPS
Backend
```

```text
Re-encryption:

Client
  ↓ HTTPS
Ingress
  ↓ HTTPS
Backend
```

The difference between passthrough and re-encryption is **where TLS is terminated and whether a new TLS connection is established**.

---

# 19. cert-manager

`cert-manager` can automate certificate lifecycle management in Kubernetes.

It can help with:

- Certificate issuance
- Certificate renewal
- Certificate storage
- Integration with Certificate Authorities

For example, cert-manager can work with Let's Encrypt.

Conceptually:

```text
Ingress
   ↓
cert-manager
   ↓
Let's Encrypt
   ↓
TLS Certificate
   ↓
Kubernetes Secret
   ↓
Ingress Controller
```

### Important

cert-manager manages certificate lifecycle.

The Ingress Controller handles the actual network traffic and TLS behavior according to its configuration.

---

# 20. HTTPS Request Flow

A typical HTTPS request can flow like this:

```text
User
  ↓
DNS
  ↓
Cloud Load Balancer
  ↓
Ingress Controller
  ↓
TLS Termination
  ↓
Ingress Rules
  ↓
Service
  ↓
EndpointSlices
  ↓
Pod
  ↓
Application
```

### Detailed Flow

```text
1. User enters:
   https://example.com

2. DNS resolves:
   example.com → external endpoint

3. Client connects to the external endpoint.

4. Load Balancer forwards traffic
   toward the Ingress Controller.

5. Ingress Controller handles TLS
   if TLS termination is configured.

6. Ingress Controller inspects
   HTTP information such as:
   - Host
   - Path
   - Headers
   - HTTP method

7. Ingress rules determine
   the target Service.

8. Service provides stable access
   to backend Pods.

9. Kubernetes maintains EndpointSlices
   for matching backend endpoints.

10. Traffic reaches a suitable
    backend Pod.

11. Application processes the request.
```

---

# 21. DNS and Ingress

Suppose we have:

```text
app.example.com
```

DNS must resolve the hostname to the appropriate external endpoint.

Conceptually:

```text
app.example.com
      ↓
Load Balancer IP / DNS name
      ↓
Ingress Controller
```

### Important

Ingress does not automatically create a public DNS record in every environment.

DNS configuration depends on:

- DNS provider
- Cloud environment
- Ingress Controller
- DNS automation tools

---

# 22. IngressClass

`IngressClass` identifies which Ingress Controller should handle an Ingress resource.

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
spec:
  controller: k8s.io/ingress-nginx
```

An Ingress can reference the class:

```yaml
spec:
  ingressClassName: nginx
```

### Mental Model

```text
Ingress
   ↓
IngressClass
   ↓
Ingress Controller
```

This becomes particularly important when multiple Ingress Controllers exist in the same cluster.

---

# 23. Ingress Annotations

Ingress Controllers often support controller-specific annotations.

They may be used for features such as:

- HTTP to HTTPS redirects
- Path rewriting
- Request size limits
- Timeouts
- Rate limiting
- Proxy configuration

Example:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
```

### Important

Annotations are often **controller-specific**.

An annotation supported by NGINX may not work with Traefik or another controller.

---

# 24. Ingress on K3s

K3s commonly includes **Traefik** as its default Ingress Controller unless it has been disabled or replaced.

Conceptually:

```text
Internet
   ↓
Traefik
   ↓
Ingress
   ↓
Service
   ↓
Pods
```

Check the cluster:

```bash
kubectl get pods -A
```

Check IngressClass:

```bash
kubectl get ingressclass
```

Check Ingress resources:

```bash
kubectl get ingress -A
```

---

# 25. Ingress on AWS EKS

On EKS, the architecture depends on the chosen ingress solution.

For example, the AWS Load Balancer Controller can integrate Kubernetes resources with AWS load-balancing services.

Another architecture can use:

```text
Internet
   ↓
AWS Load Balancer
   ↓
NGINX / Traefik Ingress Controller
   ↓
Services
   ↓
Pods
```

The important point is:

> The chosen Ingress Controller determines how the Kubernetes Ingress rules are implemented.

---

# 26. Ingress Troubleshooting

When an Ingress is not working, troubleshoot it layer by layer.

```text
Client
 ↓
DNS
 ↓
Load Balancer
 ↓
Ingress Controller
 ↓
IngressClass
 ↓
Ingress Rules
 ↓
Service
 ↓
EndpointSlices
 ↓
Pod Readiness
 ↓
Application
```

Do not immediately assume that the Ingress resource itself is the problem.

---

# 27. Step 1 — Check DNS

Verify that the hostname resolves correctly.

```bash
nslookup example.com
```

or:

```bash
dig example.com
```

Check whether the hostname resolves to the correct external endpoint.

---

# 28. Step 2 — Check the Load Balancer

Verify:

- External endpoint
- Load Balancer health
- Listener configuration
- Target health
- Security rules
- Network connectivity

The exact commands depend on the cloud provider and Load Balancer implementation.

---

# 29. Step 3 — Check Ingress Resource

```bash
kubectl get ingress
kubectl describe ingress <ingress-name>
```

Check:

- Host
- Path
- Backend Service
- Backend Port
- TLS
- Events
- IngressClass

---

# 30. Step 4 — Check IngressClass

```bash
kubectl get ingressclass
```

Then:

```bash
kubectl describe ingressclass <class-name>
```

Verify that the Ingress is associated with the expected controller.

---

# 31. Step 5 — Check Ingress Controller

Check the controller Pods:

```bash
kubectl get pods -A
```

For example, with NGINX:

```bash
kubectl get pods -n ingress-nginx
```

Check logs:

```bash
kubectl logs -n ingress-nginx <controller-pod>
```

The namespace depends on the controller installation.

Look for:

- Configuration errors
- TLS errors
- Upstream connection errors
- Routing errors
- Controller startup problems

---

# 32. Step 6 — Check the Service

Verify that the Ingress references the correct Service.

```bash
kubectl get svc
kubectl describe svc <service-name>
```

Check:

- Service name
- Service port
- Target port
- Selector

---

# 33. Step 7 — Check EndpointSlices

Verify whether the Service has backend endpoints.

```bash
kubectl get endpointslices
```

Also:

```bash
kubectl describe svc <service-name>
```

If there are no usable endpoints, investigate:

- Pod labels
- Service selector
- Pod readiness
- Pod status

---

# 34. Step 8 — Check Pods

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

Verify:

- Pods are Running
- Pods are Ready
- Application is listening on the expected port
- Application itself is healthy

---

# 35. Common Ingress Problems

## Problem 1 — DNS Is Incorrect

```text
DNS
 ↓
Wrong Endpoint
```

The request may never reach the intended infrastructure.

---

## Problem 2 — Wrong IngressClass

```text
Ingress
 ↓
Wrong Controller
```

The expected controller may not process the resource.

---

## Problem 3 — Wrong Service Name

Ingress references one Service, but the actual Service has a different name.

---

## Problem 4 — Wrong Service Port

Ingress forwards to the wrong Service port.

---

## Problem 5 — Service Has No Endpoints

The Service selector may not match the Pod labels.

---

## Problem 6 — Pods Are Not Ready

Pods can be Running but not Ready.

In that case, traffic should not normally be sent to those Pods.

---

## Problem 7 — TLS Certificate Problem

Possible causes:

- Secret does not exist
- Incorrect Secret name
- Invalid certificate
- Incorrect hostname
- Certificate expired
- Certificate configuration problem

---

# 36. Ingress vs LoadBalancer vs NodePort

| Feature | NodePort | LoadBalancer | Ingress |
|---|---|---|---|
| External access | Yes | Yes | Through controller |
| HTTP routing | No | Generally no | Yes |
| Host routing | No | No | Yes |
| Path routing | No | No | Yes |
| TLS termination | Not primary purpose | Depends on implementation | Common |
| Multiple applications | Possible but inconvenient | Usually separate exposure | Designed for HTTP routing |

### Simple Mental Model

```text
NodePort
→ Exposes a Service through a node port.

LoadBalancer
→ Provides external load-balancer connectivity.

Ingress
→ Routes HTTP/HTTPS requests to Services.
```

---

# 37. Production Best Practices

## 1. Use HTTPS

Use TLS for external traffic.

## 2. Use Trusted Certificates

Use an appropriate Certificate Authority and automate certificate renewal where possible.

## 3. Use DNS Correctly

Ensure DNS points to the correct external endpoint.

## 4. Keep Routing Rules Clear

Use meaningful host and path rules.

## 5. Monitor the Ingress Controller

Monitor:

- Request rate
- Error rate
- Latency
- 4xx responses
- 5xx responses
- Controller health

## 6. Secure the Ingress

Consider:

- TLS
- Authentication
- Authorization
- Rate limiting
- WAF
- Network controls

## 7. Use Readiness Probes

Do not send application traffic to Pods that are not ready.

## 8. Plan for High Availability

Run sufficient Ingress Controller replicas and use an appropriate external load-balancing architecture.

## 9. Understand Controller-Specific Features

Do not assume that annotations or configuration supported by one controller work with another.

---

# 38. Production Ingress Architecture

A typical production architecture:

```text
                         Internet
                            ↓
                           DNS
                            ↓
                    Cloud Load Balancer
                            ↓
                   Ingress Controller
                            ↓
                    Ingress Resources
                     ↙      ↓      ↘
                Frontend  Backend  Admin
                 Service  Service  Service
                    ↓       ↓        ↓
                  Pods     Pods      Pods
```

### TLS Termination Architecture

```text
Client
  │
  │ HTTPS
  ▼
Load Balancer
  │
  │ HTTPS
  ▼
Ingress Controller
  │
  │ TLS Termination
  │
  │ HTTP
  ▼
Service
  │
  ▼
Pods
```

### Re-Encryption Architecture

```text
Client
  │
  │ HTTPS
  ▼
Ingress
  │
  │ New HTTPS connection
  ▼
Backend
```

---

# 39. Important Commands

## Ingress

```bash
kubectl get ingress
kubectl get ingress -A
kubectl describe ingress <ingress-name>
```

## IngressClass

```bash
kubectl get ingressclass
kubectl describe ingressclass <class-name>
```

## Services

```bash
kubectl get svc
kubectl describe svc <service-name>
```

## EndpointSlices

```bash
kubectl get endpointslices
```

## Pods

```bash
kubectl get pods -A
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

## Controller

```bash
kubectl get pods -A
kubectl logs <controller-pod> -n <namespace>
```

## DNS Testing

```bash
nslookup example.com
dig example.com
```

---

# 40. Complete Ingress Troubleshooting Model

When a user says:

> "My application is not accessible through the Ingress."

Follow this order:

```text
1. DNS
   ↓
2. Load Balancer
   ↓
3. Ingress Controller
   ↓
4. IngressClass
   ↓
5. Ingress Rules
   ↓
6. Service
   ↓
7. EndpointSlices
   ↓
8. Pod Readiness
   ↓
9. Application
```

This provides a structured way to isolate the failure instead of making random configuration changes.

---

# 41. Key Mental Models

## Ingress

```text
Ingress
=
HTTP/HTTPS routing rules
```

## Ingress Controller

```text
Ingress Controller
=
Component that implements Ingress routing
```

## Service

```text
Service
=
Stable network endpoint for Pods
```

## Host-Based Routing

```text
app.example.com
      ↓
frontend-service

api.example.com
      ↓
backend-service
```

## Path-Based Routing

```text
example.com/
      ↓
frontend-service

example.com/api
      ↓
backend-service
```

## Combined Routing

```text
api.example.com/users
        ↓
user-service

api.example.com/orders
        ↓
order-service
```

## HTTPS Request Flow

```text
Client
 ↓
DNS
 ↓
Load Balancer
 ↓
Ingress Controller
 ↓
TLS Termination
 ↓
Ingress Rules
 ↓
Service
 ↓
EndpointSlices
 ↓
Pods
```

---

# 42. Final Day 7 Summary

The most important concepts to remember:

1. Ingress provides HTTP/HTTPS routing rules.
2. An Ingress Controller implements those rules.
3. Creating an Ingress object alone does not expose an application.
4. Ingress normally routes traffic to Services.
5. Services provide stable access to Pods.
6. Host-based routing routes traffic based on the hostname.
7. Path-based routing routes traffic based on the URL path.
8. Host-based and path-based routing can be combined.
9. `Prefix` and `Exact` are important path types.
10. TLS can be configured for HTTPS traffic.
11. TLS certificates and private keys can be stored in Kubernetes TLS Secrets.
12. TLS termination, TLS passthrough, and TLS re-encryption are different architectures.
13. cert-manager can automate certificate issuance and renewal.
14. `IngressClass` identifies the intended Ingress Controller.
15. EndpointSlices represent the backend endpoints used by Services.
16. A Running Pod is not necessarily a Ready Pod.
17. DNS, Load Balancer, Ingress Controller, Service, EndpointSlices, and Pods should be troubleshooting layers.
18. Production Ingress should consider TLS, monitoring, security, rate limiting, availability, and certificate renewal.

---

# Final Mental Model

```text
                         Internet
                            ↓
                           DNS
                            ↓
                    Load Balancer
                            ↓
                   Ingress Controller
                            ↓
                    Ingress Rules
                     ↙      ↓      ↘
                Service A Service B Service C
                    ↓        ↓        ↓
               EndpointSlices
                    ↓
                   Pods
```

### The Most Important Difference

```text
Service
→ Provides stable network access to Pods.

Ingress
→ Decides which Service should receive an HTTP/HTTPS request.

Ingress Controller
→ Actually implements the Ingress routing.
```
