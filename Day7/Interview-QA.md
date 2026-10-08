# Kubernetes Day 7 — Ingress Interview Q&A

## Q1. What is the difference between Ingress and an Ingress Controller?

### Answer

**Ingress** is a Kubernetes API resource used to define HTTP/HTTPS routing rules.

For example, it can define:

- Host-based routing
- Path-based routing
- Backend Services
- TLS configuration

The **Ingress Controller** is the component that watches those Ingress resources and actually implements the routing behavior.

### Simple Mental Model

```text
Ingress
   ↓
Defines WHAT should happen

Ingress Controller
   ↓
Implements HOW it happens
```

An Ingress without an appropriate Ingress Controller only defines the desired routing configuration. The controller is responsible for processing traffic according to those rules.

### Production-Ready Answer

> Ingress is a Kubernetes API resource that defines HTTP/HTTPS routing rules, while the Ingress Controller watches those resources and implements the actual proxy and load-balancing behavior. We need both because Ingress describes the desired routing configuration, and the controller is responsible for enforcing that configuration for real network traffic.

---

# Q2. Explain the complete request flow when a user accesses an HTTPS application running on Kubernetes.

### Answer

A typical request flow is:

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
Pod
  ↓
Application
```

### Step-by-Step

### 1. DNS Resolution

The client first resolves the domain name.

For example:

```text
example.com
     ↓
External IP / Load Balancer Endpoint
```

### 2. Load Balancer

The request reaches the external Load Balancer.

The Load Balancer forwards traffic toward the Ingress Controller.

### 3. Ingress Controller

The Ingress Controller processes the routing rules defined by the Ingress resource.

### 4. TLS Termination

If TLS termination is configured at the Ingress Controller:

```text
HTTPS
  ↓
Ingress Controller
  ↓
TLS Termination
  ↓
HTTP
```

The Ingress Controller can then inspect HTTP information such as:

- Host
- Path
- Headers
- HTTP method

### 5. Ingress Routing

The Ingress rules determine which Service should receive the request.

### 6. Service

The Service provides a stable network endpoint for the backend Pods.

### 7. EndpointSlices

Kubernetes maintains EndpointSlices representing the backend endpoints associated with the Service.

### 8. Pod

Traffic is sent to a suitable backend Pod.

### 9. Application

The application inside the Pod processes the request.

### Important Note

TLS termination at Ingress is one possible architecture. TLS can also be passed through or re-encrypted depending on the security requirements and Ingress Controller capabilities.

---

# Q3. What is the difference between host-based routing and path-based routing?

### Answer

**Host-based routing** routes traffic based on the hostname.

Example:

```text
app.example.com
      ↓
frontend-service

api.example.com
      ↓
backend-service
```

**Path-based routing** routes traffic based on the URL path.

Example:

```text
example.com/
      ↓
frontend-service

example.com/api
      ↓
backend-service
```

### Host-Based Routing

The routing condition is primarily the hostname.

```text
api.example.com
```

### Path-Based Routing

The routing condition is primarily the URL path.

```text
/api
```

### They Can Be Combined

They are not mutually exclusive.

For example:

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

### Production-Ready Answer

> Host-based routing uses the hostname to determine the backend Service, while path-based routing uses the URL path. Kubernetes Ingress can also combine both host and path rules to implement more specific routing.

---

# Q4. An Ingress is created, but the application is not accessible. How would you troubleshoot it in production?

### Answer

I would troubleshoot the request path layer by layer rather than immediately changing the Ingress configuration.

The troubleshooting flow is:

```text
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

## Step 1 — Check DNS

```bash
nslookup example.com
dig example.com
```

Verify that the hostname resolves to the correct external endpoint.

---

## Step 2 — Check the Load Balancer

Verify:

- External endpoint
- Listener configuration
- Target health
- Network connectivity
- Security rules

---

## Step 3 — Check the Ingress

```bash
kubectl get ingress
kubectl describe ingress <ingress-name>
```

Verify:

- Host
- Path
- Backend Service
- Backend Port
- TLS
- IngressClass
- Events

---

## Step 4 — Check IngressClass

```bash
kubectl get ingressclass
kubectl describe ingressclass <class-name>
```

Verify that the expected Ingress Controller is handling the Ingress.

---

## Step 5 — Check Ingress Controller

```bash
kubectl get pods -A
```

Then check the controller logs:

```bash
kubectl logs <controller-pod> -n <namespace>
```

Look for:

- Configuration errors
- TLS errors
- Upstream errors
- Controller failures

---

## Step 6 — Check the Service

```bash
kubectl get svc
kubectl describe svc <service-name>
```

Verify:

- Service name
- Service port
- Target port
- Selector

---

## Step 7 — Check EndpointSlices

```bash
kubectl get endpointslices
```

If the Service has no usable endpoints, investigate:

- Pod labels
- Service selector
- Pod readiness
- Pod status

---

## Step 8 — Check Pods

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

Verify:

- Pod is Running
- Pod is Ready
- Application is listening on the expected port
- Application itself is healthy

### Production-Ready Answer

> I would troubleshoot from the client side toward the application: DNS, Load Balancer, Ingress Controller, IngressClass, Ingress rules, Service, EndpointSlices, Pod readiness, and finally the application. This helps isolate whether the problem is DNS, external networking, ingress routing, service discovery, Pod readiness, or the application itself.

---

# Q5. Explain TLS termination at Ingress. Where is the TLS certificate stored in Kubernetes?

### Answer

TLS termination means the Ingress Controller terminates the TLS connection from the client.

The TLS certificate and private key can be stored in a Kubernetes TLS Secret.

```text
TLS Certificate
       +
Private Key
       ↓
Kubernetes TLS Secret
       ↓
Ingress Controller
       ↓
TLS Termination
```

The flow can be:

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

After TLS termination, the Ingress Controller can inspect HTTP information such as:

- Host
- Path
- Headers
- HTTP method

and route the request according to the Ingress rules.

---

# TLS Passthrough

TLS passthrough means the Ingress does not terminate TLS.

The encrypted connection continues to the backend.

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

The backend application handles the TLS connection.

### Simple Definition

> TLS Passthrough means the Ingress passes the encrypted TLS connection to the backend without terminating TLS.

---

# TLS Re-Encryption

Re-encryption means TLS is terminated at the Ingress and then a new TLS connection is established from the Ingress to the backend.

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

So the backend connection remains encrypted.

### Comparison

```text
TLS Termination:

Client
  ↓ HTTPS
Ingress
  ↓ HTTP
Backend
```

```text
TLS Passthrough:

Client
  ↓ HTTPS
Ingress
  ↓ HTTPS
Backend
```

```text
TLS Re-Encryption:

Client
  ↓ HTTPS
Ingress
  ↓ HTTPS
Backend
```

### Important Difference

| Architecture | TLS Terminated at Ingress? | Backend Traffic |
|---|---|---|
| TLS Termination | Yes | HTTP |
| TLS Passthrough | No | Original HTTPS |
| TLS Re-Encryption | Yes | New HTTPS connection |

---

# cert-manager

`cert-manager` can automate certificate lifecycle management.

It can help with:

- Certificate issuance
- Certificate renewal
- Certificate management
- Integration with Certificate Authorities such as Let's Encrypt

Conceptually:

```text
Ingress
   ↓
cert-manager
   ↓
Certificate Authority
   ↓
TLS Certificate
   ↓
Kubernetes Secret
   ↓
Ingress Controller
```

### Important

cert-manager manages the certificate lifecycle.

The Ingress Controller handles the actual network traffic and TLS behavior.

---

# Quick Day 7 Revision

## Ingress

> Kubernetes API resource that defines HTTP/HTTPS routing rules.

## Ingress Controller

> Component that watches Ingress resources and implements the routing.

## Host-Based Routing

> Routes based on hostname.

```text
api.example.com
      ↓
backend-service
```

## Path-Based Routing

> Routes based on URL path.

```text
example.com/api
      ↓
backend-service
```

## Service

> Provides stable network access to Pods.

## EndpointSlices

> Represent the backend endpoints associated with Services.

## TLS Termination

> TLS is terminated at the Ingress and the request may continue as HTTP internally.

## TLS Passthrough

> TLS is not terminated at the Ingress; the encrypted connection continues to the backend.

## TLS Re-Encryption

> TLS is terminated at the Ingress and a new TLS connection is established to the backend.

## Main Request Flow

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
 ↓
Application
```

## Production Troubleshooting Flow

```text
DNS
 ↓
Load Balancer
 ↓
Ingress Controller
 ↓
IngressClass
 ↓
Ingress
 ↓
Service
 ↓
EndpointSlices
 ↓
Pod Readiness
 ↓
Application
```
