# Kubernetes Day 2 — Interview Questions & Production Answers

## Q1. What is a Pod in Kubernetes, and why does Kubernetes use a Pod as the smallest deployable unit instead of directly managing individual containers?

### Answer

A Pod is the **smallest deployable unit in Kubernetes**. It can contain one or more containers that are tightly coupled and need to share resources such as networking and, when configured, storage.

Kubernetes uses Pods as the scheduling and lifecycle unit because related containers can be managed together.

Containers within the same Pod:

* Share the network namespace
* Share the Pod IP
* Can communicate using `localhost`
* Can share volumes when the same volume is mounted into them
* Are scheduled onto the same node

For example:

```text
Pod
├── Application Container
└── Logging Sidecar
```

The important point is that Kubernetes does not require every application to use multiple containers in a Pod. Multiple containers should generally be used only when they are tightly coupled.

For example, putting an application and its logging sidecar in the same Pod can make sense, while putting frontend, backend, database, and Redis into one Pod usually does not.

Also, Pods are ephemeral. If a Pod is replaced, the replacement Pod can have a different UID and IP. Therefore, applications should not rely on a Pod IP as a permanent endpoint.

---

## Q2. You have two containers inside the same Pod: a main application container and a logging sidecar. How do these two containers communicate with each other, and what resources are shared between them?

### Answer

Containers in the same Pod share the **network namespace**, so they can communicate with each other using `localhost`.

For example, if the application listens on port 8080, the sidecar can access it using:

```bash
curl localhost:8080
```

They share the Pod's network identity and Pod IP.

They can also share data through a volume when the same volume is explicitly mounted into both containers.

For example:

```text
Pod
│
├── Application
│      └── /shared
│
├── Logging Sidecar
│      └── /shared
│
└── Shared Volume
```

An important distinction is that containers do **not automatically share their filesystems**. A common volume must be configured if they need to share files.

For communication between different Pods, applications typically use a Kubernetes Service and DNS rather than relying on Pod IPs.

---

## Q3. What is the difference between an Init Container and a Sidecar Container? Give one real production scenario where you would use each.

### Answer

An **Init Container** runs before the main application containers and must complete successfully before the application proceeds.

A **Sidecar Container** runs alongside the main application and provides supporting functionality.

### Init Container

Flow:

```text
Init Container
      ↓
Completes successfully
      ↓
Application Container
```

Example production use case:

An init container can prepare configuration files or perform a startup prerequisite before the application starts.

For example:

```text
Init Container
      ↓
Prepare configuration
      ↓
Shared Volume
      ↓
Application
```

### Sidecar

Flow:

```text
Application
     ↕
Sidecar
```

Example production use case:

A logging or telemetry sidecar can collect application logs or provide supporting functionality while the main application is running.

The key difference is:

> **Init Container = initialization before the application**

> **Sidecar = supporting functionality alongside the application**

Modern Kubernetes also supports native sidecars, which are implemented using restartable init containers with a container-level `restartPolicy: Always`.

---

## Q4. A Pod contains one application container. The application crashes repeatedly. Explain what happens from the moment the container crashes until Kubernetes attempts to bring the application back. Also explain the difference between container restart and Pod replacement.

### Answer

When the application container crashes, the container enters a terminated state.

The kubelet on the worker node observes the container state and applies the Pod's restart policy.

For example, with the default:

```text
restartPolicy: Always
```

the container can be restarted.

The simplified flow is:

```text
Application crashes
        ↓
Container terminates
        ↓
kubelet observes failure
        ↓
Restart policy applies
        ↓
Container restarted
```

If the container repeatedly crashes, Kubernetes may show:

```text
CrashLoopBackOff
```

This indicates repeated failures with increasing delays between restart attempts.

Useful troubleshooting commands include:

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
```

For a particular container:

```bash
kubectl logs <pod-name> -c <container-name>
```

### Container Restart

The Pod remains the same:

```text
Same Pod
   ↓
Container crashes
   ↓
kubelet
   ↓
Container restarted
```

### Pod Replacement

If the Pod itself is deleted or lost and a controller such as a ReplicaSet needs to maintain the replica count:

```text
Old Pod
   ↓
Deleted
   ↓
ReplicaSet creates replacement
   ↓
Scheduler selects node
   ↓
kubelet starts containers
```

The replacement is a **new Pod**.

It can have a new:

* Pod UID
* Pod IP
* Container instances

Therefore:

> Container restart and Pod replacement are two different events.

---

## Q5. You have 20 Pods belonging to different applications and environments. How would you use Labels, Selectors, and Namespaces to organize them? Also explain why a Label and a Selector are not the same thing.

### Answer

I would use **Namespaces** for logical separation and **Labels** for identifying and categorizing individual objects.

For example:

```text
Namespace: production

Pod labels:
app=payment
environment=production
version=v2
```

Another Pod could have:

```text
app=frontend
environment=production
version=v3
```

A **Label** is metadata attached to a Kubernetes object.

A **Selector** is a matching rule used to find objects based on their labels.

For example:

```yaml
selector:
  matchLabels:
    app: payment
```

This selector matches Pods with:

```text
app=payment
```

The relationship is:

```text
Label
  ↓
Metadata attached to object

Selector
  ↓
Rule that matches objects using labels
```

Namespaces provide logical organization within the cluster:

```text
Cluster
├── development
├── staging
└── production
```

For example:

```bash
kubectl get pods -n production
```

However, a Namespace should not be considered a complete security boundary. Stronger isolation may require mechanisms such as RBAC, NetworkPolicies, and Pod Security controls.

---

# Day 2 Rapid-Fire Revision

### What is a Pod?

The smallest deployable unit in Kubernetes.

### Same-Pod communication?

`localhost:<port>`

### Do containers automatically share filesystems?

No. They need a common mounted volume.

### Can a Pod IP change?

Yes, when the Pod is replaced.

### Deployment relationship?

```text
Deployment → ReplicaSet → Pod
```

### Who decides where a Pod runs?

Scheduler.

### Who manages containers on a node?

kubelet.

### Init Container?

Runs and completes before application startup.

### Sidecar?

Runs alongside the main application to provide supporting functionality.

### Official Pod phases?

```text
Pending
Running
Succeeded
Failed
Unknown
```

### Container states?

```text
Waiting
Running
Terminated
```

### Default Pod restart policy?

`Always`

### What does CrashLoopBackOff indicate?

Repeated container startup failures with increasing restart delays.

### Label?

Metadata attached to an object.

### Selector?

Rule used to select objects based on labels.

### Annotation?

Additional metadata, generally not used for selection.

### Namespace?

Logical partition within a Kubernetes cluster.

### Most important Pod rule?

**Do not treat a Pod IP as a stable application endpoint.**

### Most important lifecycle distinction?

**Container restart ≠ Pod replacement.**
