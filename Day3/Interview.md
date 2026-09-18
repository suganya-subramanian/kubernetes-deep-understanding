# Kubernetes Day 3 — Interview Questions & Production Answers

## Q1. A Deployment is configured with 3 replicas. All 3 Pods are running successfully. One Pod is suddenly deleted. Explain step-by-step what happens inside Kubernetes.

### Answer

The Deployment specifies the desired number of replicas as 3.

The Deployment manages a ReplicaSet, and the ReplicaSet is responsible for maintaining the desired number of matching Pods.

Initially:

```text
Desired = 3
Actual = 3
```

If one Pod is deleted:

```text
Desired = 3
Actual = 2
```

The ReplicaSet controller observes that the actual replica count is lower than the desired count and creates a replacement Pod object.

The new Pod is initially unscheduled.

The Scheduler evaluates the Pod and selects a suitable worker node based on factors such as:

* Resource requests
* Node availability
* Taints and tolerations
* Node selectors
* Affinity/anti-affinity
* Other scheduling constraints

After the Pod is assigned to a node, the kubelet on that node ensures the Pod's containers are started through the container runtime.

The final state becomes:

```text
Desired = 3
Actual = 3
```

The important component flow is:

```text
Pod deleted
    ↓
ReplicaSet detects replica mismatch
    ↓
Replacement Pod created
    ↓
Scheduler selects node
    ↓
kubelet manages Pod
    ↓
Container Runtime starts containers
```

The replacement is a **new Pod**, not the original deleted Pod. Therefore it can have a new Pod UID and Pod IP.

---

## Q2. What is the difference between a ReplicaSet and a Deployment? Why would you normally use a Deployment in production?

### Answer

A ReplicaSet's primary responsibility is to maintain the desired number of matching Pods.

For example:

```text
replicas: 3
```

means the ReplicaSet works to maintain three matching Pods.

A Deployment is a higher-level workload controller that manages ReplicaSets and provides application release-management capabilities such as:

* Rolling updates
* Rollbacks
* Revision history
* Scaling
* ReplicaSet management

The relationship is:

```text
Deployment
     ↓
ReplicaSet
     ↓
Pods
```

A ReplicaSet itself provides self-healing for its Pods.

Therefore, it is incorrect to say that Deployment is required for self-healing.

In production, Deployments are normally preferred for stateless applications because they provide controlled application updates and rollback capabilities.

For example, if an application is running:

```text
app:v1
```

and needs to be updated to:

```text
app:v2
```

a Deployment can create/manage a new ReplicaSet and perform a controlled rolling update.

---

## Q3. Explain what happens during a Rolling Update when a Deployment changes from app:v1 to app:v2.

### Answer

Suppose the Deployment has:

```text
replicas: 4
image: app:v1
```

Initially:

```text
Old ReplicaSet
├── Pod v1
├── Pod v1
├── Pod v1
└── Pod v1
```

When the image is changed to `app:v2`, the Deployment controller creates/manages a new ReplicaSet representing the new Pod template.

Conceptually:

```text
Deployment
   │
   ├── Old ReplicaSet → v1
   │
   └── New ReplicaSet → v2
```

The new ReplicaSet is gradually scaled up while the old ReplicaSet is gradually scaled down.

The exact rollout behavior is controlled by:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```

`maxUnavailable` controls how much desired capacity may be unavailable during the rollout.

`maxSurge` controls how many additional Pods may temporarily exist above the desired replica count.

For example:

```text
replicas = 4
maxSurge = 1
```

can allow up to five Pods temporarily during the rollout.

The old ReplicaSet is generally scaled down to zero after the rollout and retained according to Deployment revision history.

The objective is to gradually move the application from:

```text
v1
v1
v1
v1
```

to:

```text
v2
v2
v2
v2
```

without unnecessarily taking all application capacity offline at once.

---

## Q4. A production Deployment reports successful rollout, but users receive HTTP 500 errors. How would you investigate and safely roll back?

### Answer

First, I would distinguish between:

```text
Kubernetes rollout success
```

and:

```text
Application functional health
```

A successful rollout does not necessarily mean that the application is functionally healthy.

I would first check the rollout:

```bash
kubectl rollout status deployment/<deployment-name>
```

Then inspect the Deployment:

```bash
kubectl get deployment <deployment-name>
kubectl describe deployment <deployment-name>
```

Next, check the Pods:

```bash
kubectl get pods
kubectl describe pod <pod-name>
```

Then inspect application logs:

```bash
kubectl logs <pod-name>
```

If the application has restarted, I would also check:

```bash
kubectl logs <pod-name> --previous
```

I would check events:

```bash
kubectl get events
```

Then I would compare the new version with the previous version, including:

* Application logs
* Configuration
* Environment variables
* Secrets/ConfigMaps
* Dependencies
* Application metrics
* Health checks
* Recent code/image changes

If the evidence confirms that `v2` caused the production issue, I would roll back:

```bash
kubectl rollout undo deployment/<deployment-name>
```

Then verify:

```bash
kubectl rollout status deployment/<deployment-name>
kubectl get pods
```

After service is restored, I would continue root-cause analysis with the development/application team.

The key production principle is:

> **Rollback restores service; it does not replace root-cause analysis.**

---

## Q5. A Deployment is scaled from 3 replicas to 8. Explain what happens. What happens if the cluster doesn't have enough resources?

### Answer

Initially:

```text
Desired = 3
Actual = 3
```

After scaling:

```text
Desired = 8
Actual = 3
```

The ReplicaSet detects that five additional Pods are required.

Conceptually:

```text
Deployment
    ↓
Desired replicas = 8
    ↓
ReplicaSet
    ↓
5 additional Pod objects
```

The new Pods are initially unscheduled.

The Scheduler evaluates each Pod and selects suitable nodes based on:

* CPU/memory resource requests
* Node availability
* Taints and tolerations
* Node selectors
* Affinity/anti-affinity
* Topology constraints
* Other scheduling rules

After a Pod is assigned to a node:

```text
Scheduler
    ↓
Worker Node
    ↓
kubelet
    ↓
Container Runtime
    ↓
Container
```

If the cluster does not have enough resources, some Pods may remain:

```text
Pending
```

For example:

```text
Desired = 8
Running = 5
Pending = 3
```

The Scheduler may generate events such as:

```text
Insufficient cpu
```

or:

```text
Insufficient memory
```

To troubleshoot, I would use:

```bash
kubectl get pods
kubectl describe pod <pending-pod>
kubectl get events
```

I would look for:

* Insufficient CPU
* Insufficient memory
* Taints
* Affinity constraints
* Node selectors
* Other scheduling constraints

I would **not** classify insufficient cluster resources as `CrashLoopBackOff`.

`CrashLoopBackOff` generally indicates that a container is repeatedly starting and failing.

Similarly, `ImagePullBackOff` indicates an image-pull problem.

Therefore:

```text
Insufficient resources → Pending

Container repeatedly crashes → CrashLoopBackOff

Image cannot be pulled → ImagePullBackOff
```

---

# Day 3 Rapid Revision

## ReplicaSet

**Maintains the desired number of matching Pods.**

## Deployment

**Manages ReplicaSets and application releases, including rolling updates and rollback.**

## Scheduler

**Selects a suitable node for an unscheduled Pod.**

## kubelet

**Manages Pods and their containers on a worker node.**

## RollingUpdate

**Gradually replaces old Pods with new Pods.**

## maxUnavailable

**Controls how much desired capacity may be unavailable during rollout.**

## maxSurge

**Controls how many extra Pods may temporarily exist during rollout.**

## Rollback

```bash
kubectl rollout undo deployment/<name>
```

## Scaling

```bash
kubectl scale deployment/<name> --replicas=5
```

## Rollout status

```bash
kubectl rollout status deployment/<name>
```

## Revision history

```bash
kubectl rollout history deployment/<name>
```

---

# Day 3 Golden Mental Model

```text
Deployment
     │
     │ manages
     ↓
ReplicaSet
     │
     │ maintains
     ↓
Pods
     │
     │ scheduled by
     ↓
Scheduler
     │
     ↓
Worker Node
     │
     │ managed by
     ↓
kubelet
     │
     ↓
Container Runtime
     │
     ↓
Container
```

Remember:

> **Deployment manages the release. ReplicaSet maintains the replicas. Scheduler chooses the node. kubelet runs the workload on the node.**
