# Kubernetes Day 3 — ReplicaSet & Deployment

## 1. Day 3 Overview

Day 2 focused on Pods.

Day 3 builds the next layer:

```text
Deployment
     ↓
ReplicaSet
     ↓
Pod
     ↓
Container
```

The main topics are:

* ReplicaSet
* Desired vs Actual State
* ReplicaSet self-healing
* Labels and Selectors
* Deployment
* Deployment vs ReplicaSet
* Rolling Updates
* Recreate Strategy
* maxUnavailable
* maxSurge
* Revision History
* Rollback
* Scaling
* Production Deployment
* Troubleshooting

---

# 2. Why Do We Need ReplicaSets?

A standalone Pod is ephemeral.

If we create:

```text
Pod
```

and the Pod is deleted:

```text
Pod
 ↓
Deleted
```

there is no ReplicaSet or Deployment to maintain it.

For production applications, we may need:

```text
3 replicas
```

We don't want to manually monitor and recreate Pods.

We want Kubernetes to continuously maintain:

```text
Desired = 3
Actual  = 3
```

This is the responsibility of a ReplicaSet.

---

# 3. What Is a ReplicaSet?

A ReplicaSet ensures that a specified number of matching Pods are running.

Example:

```yaml
spec:
  replicas: 3
```

Conceptually:

```text
ReplicaSet
     │
     ├── Pod A
     ├── Pod B
     └── Pod C
```

The key responsibility is:

> **ReplicaSet maintains the desired number of matching Pods.**

---

# 4. ReplicaSet and Desired State

Suppose:

```text
Desired = 3
Actual  = 3
```

Everything is healthy.

Now Pod B is deleted:

```text
Desired = 3
Actual  = 2
```

The ReplicaSet controller detects the difference.

It creates a replacement Pod.

```text
Pod A
Pod B → deleted
Pod C

        ↓

Pod A
Pod C
Pod D → replacement
```

Eventually:

```text
Desired = 3
Actual  = 3
```

This is reconciliation.

---

# 5. ReplicaSet Does Not Repair the Deleted Pod

If Pod B is deleted:

```text
Pod B
   ↓
Deleted
```

ReplicaSet does not bring Pod B back.

It creates a **new Pod**.

Therefore:

```text
Pod B ≠ Pod D
```

The new Pod can have:

* New UID
* New IP
* New container instances

This is an important production concept.

---

# 6. ReplicaSet Uses Labels and Selectors

ReplicaSets identify the Pods they manage using selectors.

Example:

```yaml
selector:
  matchLabels:
    app: nginx
```

Pod template:

```yaml
template:
  metadata:
    labels:
      app: nginx
```

Conceptually:

```text
ReplicaSet
selector:
app=nginx
       ↓
┌─────────────────────┐
│ Pod A app=nginx     │ ✓
│ Pod B app=nginx     │ ✓
│ Pod C app=frontend  │ ✗
└─────────────────────┘
```

This connects directly with Day 2's Labels and Selectors.

---

# 7. ReplicaSet YAML

Example:

```yaml
apiVersion: apps/v1
kind: ReplicaSet

metadata:
  name: nginx-rs

spec:
  replicas: 3

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

Important sections:

```text
replicas
selector
template
```

---

# 8. Who Does What?

This is one of the most important Day 3 concepts.

```text
ReplicaSet
    ↓
How many Pods are required?

Scheduler
    ↓
Which node should an unscheduled Pod run on?

kubelet
    ↓
Manage the Pod's containers on that node

Container Runtime
    ↓
Run the containers
```

Remember:

> **ReplicaSet decides how many. Scheduler decides where. kubelet manages the containers on the selected node.**

---

# 9. Deployment

A Deployment is a higher-level Kubernetes workload controller used to manage application Pods through ReplicaSets.

Relationship:

```text
Deployment
     ↓
ReplicaSet
     ↓
Pods
     ↓
Containers
```

Deployments provide features such as:

* Rolling updates
* Rollbacks
* Revision history
* Scaling
* ReplicaSet management
* Declarative updates

---

# 10. Why Use Deployment Instead of Direct ReplicaSet?

A ReplicaSet can maintain replicas:

```text
ReplicaSet
    ↓
3 Pods
```

But production applications frequently need:

* Version updates
* Rolling deployments
* Rollbacks
* Revision history
* Controlled application releases

Deployment provides these higher-level capabilities.

Therefore, the normal production pattern for stateless applications is:

```text
Deployment
     ↓
ReplicaSet
     ↓
Pods
```

---

# 11. Important Correction

Do NOT say:

> Deployment is required for self-healing.

A ReplicaSet itself provides replica reconciliation.

Correct:

```text
ReplicaSet
    ↓
Maintains desired Pod count
```

Deployment adds:

```text
Deployment
    ↓
Manages ReplicaSets
    ↓
Rollouts
Rollback
Revision history
Scaling
```

---

# 12. Deployment YAML

Example:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx-deployment

spec:
  replicas: 3

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

Check:

```bash
kubectl get deployments
kubectl get rs
kubectl get pods
```

---

# 13. Deployment Hierarchy

The complete conceptual hierarchy is:

```text
Deployment
     │
     ├── ReplicaSet
     │      ├── Pod
     │      ├── Pod
     │      └── Pod
     │
     └── Rollout / Revision History
```

Deployment does not directly run containers.

The complete path eventually becomes:

```text
Deployment
     ↓
ReplicaSet
     ↓
Pod
     ↓
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

This is a conceptual flow; Kubernetes operates through continuously reconciled control loops rather than one synchronous chain.

---

# 14. Deployment vs ReplicaSet vs Pod

| Component  | Main Responsibility                                  |
| ---------- | ---------------------------------------------------- |
| Pod        | Runs application containers                          |
| ReplicaSet | Maintains desired number of matching Pods            |
| Deployment | Manages ReplicaSets, rollouts, versions and rollback |

Mental model:

```text
Pod
↓
"Run my application"

ReplicaSet
↓
"Keep N Pods running"

Deployment
↓
"Manage application releases and ReplicaSets"
```

---

# 15. Rolling Update

Suppose production is running:

```text
replicas: 4
image: app:v1
```

Current state:

```text
Pod 1 → v1
Pod 2 → v1
Pod 3 → v1
Pod 4 → v1
```

Now we update to:

```text
image: app:v2
```

Deployment creates/manages a new ReplicaSet.

Conceptually:

```text
Deployment
   │
   ├── Old ReplicaSet → v1
   │
   └── New ReplicaSet → v2
```

The new ReplicaSet gradually scales up while the old ReplicaSet scales down.

---

# 16. Rolling Update Example

Initial:

```text
v1
v1
v1
v1
```

During rollout:

```text
v2
v1
v1
v1
```

Then:

```text
v2
v2
v1
v1
```

Then:

```text
v2
v2
v2
v1
```

Finally:

```text
v2
v2
v2
v2
```

The exact sequence depends on:

* `maxUnavailable`
* `maxSurge`
* Pod readiness
* Scheduling capacity
* Other Deployment conditions

---

# 17. Deployment Does Not Replace All Pods at Once

A RollingUpdate is designed to gradually transition from the old version to the new version.

Conceptually:

```text
Old Capacity
     ↓
Old + New
     ↓
More New
     ↓
New Capacity
```

This reduces the risk of taking the entire application offline during an update.

---

# 18. maxUnavailable

`maxUnavailable` controls how many desired Pods can be unavailable during a rolling update.

Example:

```yaml
rollingUpdate:
  maxUnavailable: 1
```

This does NOT mean:

> Create one new Pod.

It means:

> Allow up to the configured number of desired Pods to be unavailable during the rollout, subject to Deployment behavior.

It can also be a percentage:

```yaml
maxUnavailable: 25%
```

Important:

> **maxUnavailable cannot be negative.**

---

# 19. maxSurge

`maxSurge` controls how many extra Pods can temporarily exist above the desired replica count during a rolling update.

Example:

```yaml
replicas: 4

strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
```

During rollout, Kubernetes may temporarily have:

```text
5 Pods
```

instead of only 4.

Mental model:

```text
maxSurge
    ↓
How many extra Pods can temporarily exist?
```

---

# 20. maxUnavailable vs maxSurge

| Setting        | Meaning                                      |
| -------------- | -------------------------------------------- |
| maxUnavailable | How much desired capacity may be unavailable |
| maxSurge       | How many extra Pods may temporarily exist    |

Example:

```yaml
replicas: 4

rollingUpdate:
  maxUnavailable: 1
  maxSurge: 1
```

Conceptually:

```text
Desired = 4

During rollout:
Old Pods + New Pods
     ↓
Maximum extra capacity controlled by maxSurge
     ↓
Maximum unavailable capacity controlled by maxUnavailable
```

These values should be chosen based on:

* Availability requirements
* Application startup time
* Cluster capacity
* Traffic patterns
* Deployment speed

---

# 21. Default Deployment Strategy

The default strategy is:

```text
RollingUpdate
```

This gradually replaces old Pods.

The other common strategy is:

```text
Recreate
```

---

# 22. Recreate Strategy

With:

```yaml
strategy:
  type: Recreate
```

the old Pods are terminated before the new Pods are created.

Conceptually:

```text
Old Pods
   ↓
Terminate
   ↓
No old Pods
   ↓
Create new Pods
   ↓
New version
```

This can cause downtime.

Use it when old and new versions cannot safely run at the same time or when simultaneous versions are undesirable.

For most stateless production applications, RollingUpdate is usually preferred.

---

# 23. Rolling Update vs Recreate

| RollingUpdate                     | Recreate                                    |
| --------------------------------- | ------------------------------------------- |
| Gradual replacement               | Old Pods removed first                      |
| Better availability               | Can cause downtime                          |
| Old and new may coexist           | Old/new don't normally coexist              |
| Default Deployment strategy       | Explicitly configured                       |
| Common for stateless applications | Used for special compatibility requirements |

---

# 24. Deployment Revision History

When Deployment configuration changes, Kubernetes maintains rollout history.

Example:

```text
Revision 1 → app:v1
Revision 2 → app:v2
Revision 3 → app:v3
```

Check:

```bash
kubectl rollout history deployment/nginx-deployment
```

Inspect a revision:

```bash
kubectl rollout history deployment/nginx-deployment --revision=2
```

Revision history is important for rollback.

---

# 25. Old ReplicaSet After Rollout

An important correction:

Do not say:

> The old ReplicaSet is deleted immediately.

During a successful rollout, the old ReplicaSet is generally scaled down to zero and retained according to Deployment revision history.

Conceptually:

```text
Old ReplicaSet
     ↓
Scaled to 0
     ↓
Retained for revision history
```

Example:

```text
Deployment
│
├── Old ReplicaSet
│      └── 0 Pods
│
└── New ReplicaSet
       └── 4 Pods
```

---

# 26. Rollback

Suppose:

```text
v1 → healthy
v2 → healthy
v3 → broken
```

Production users experience errors after v3.

Rollback:

```bash
kubectl rollout undo deployment/nginx-deployment
```

Kubernetes moves the Deployment back toward a previous revision.

Check:

```bash
kubectl rollout status deployment/nginx-deployment
```

---

# 27. Rollback Flow

```text
Revision 1
     ↓
Revision 2
     ↓
Revision 3
     ↓
Production problem
     ↓
rollout undo
     ↓
Previous revision
```

Rollback should be treated as a service-restoration mechanism.

It does not eliminate the need for root-cause analysis.

---

# 28. Scaling

Suppose:

```text
replicas: 3
```

Scale to 5:

```bash
kubectl scale deployment nginx-deployment --replicas=5
```

Now the desired state becomes:

```text
Desired = 5
Actual = 3
```

ReplicaSet reconciles the difference.

```text
Need 2 more Pods
     ↓
New Pod
New Pod
```

Eventually:

```text
Desired = 5
Actual = 5
```

---

# 29. Scaling from 3 to 8

Suppose:

```text
Current = 3
Desired = 8
```

Then:

```text
8 - 3 = 5
```

Five additional Pods are required.

The conceptual flow:

```text
Deployment
     ↓
Desired replicas = 8
     ↓
ReplicaSet
     ↓
Creates/requests 5 additional Pod objects
     ↓
Scheduler
     ↓
Selects suitable nodes
     ↓
kubelet
     ↓
Container Runtime
     ↓
Containers start
```

---

# 30. Important Scaling Correction

Do not say:

> Scheduler decides that five Pods are required.

Incorrect.

The ReplicaSet determines that more Pods are required.

The Scheduler decides where the **unscheduled Pods** should run.

Remember:

```text
ReplicaSet → How many?
Scheduler   → Where?
kubelet     → Run them on the node
```

---

# 31. What If There Isn't Enough Cluster Capacity?

Suppose:

```text
Desired = 8
Running = 5
Pending = 3
```

If the cluster doesn't have enough CPU or memory, some Pods may remain:

```text
Pending
```

The Scheduler may generate events indicating reasons such as:

```text
Insufficient cpu
Insufficient memory
```

Other scheduling constraints can also cause Pending Pods.

Examples:

* Taints
* Tolerations
* Node affinity
* Node selectors
* Pod affinity/anti-affinity
* Topology constraints
* Resource requests

---

# 32. Important Troubleshooting Distinctions

Do not mix these statuses.

## Pending

Usually means the Pod has not successfully reached the running state; scheduling/resource/volume or other admission issues may be involved.

Example:

```text
Insufficient CPU
```

---

## CrashLoopBackOff

The container repeatedly starts and crashes.

Conceptually:

```text
Container starts
     ↓
Crashes
     ↓
Restart
     ↓
Crashes
     ↓
Restart
     ↓
CrashLoopBackOff
```

---

## ImagePullBackOff

Kubernetes cannot successfully pull the container image.

Possible causes:

* Wrong image name
* Wrong tag
* Registry authentication problem
* Registry unavailable

Mental model:

```text
Insufficient resources → Pending

Container repeatedly crashes → CrashLoopBackOff

Image cannot be pulled → ImagePullBackOff
```

---

# 33. Production Rollout Troubleshooting

Suppose a rollout is stuck.

Start with:

```bash
kubectl rollout status deployment/<name>
```

Then:

```bash
kubectl get deployment
kubectl get rs
kubectl get pods
```

Inspect:

```bash
kubectl describe deployment <name>
kubectl describe pod <pod-name>
```

Check events:

```bash
kubectl get events
```

Check logs:

```bash
kubectl logs <pod-name>
```

Previous container logs:

```bash
kubectl logs <pod-name> --previous
```

---

# 34. Production Rollback Scenario

Suppose:

```text
v1 → healthy
v2 → deployed
```

Kubernetes reports:

```text
rollout successful
```

But users report:

```text
HTTP 500
```

Important:

```text
Deployment successful
        ≠
Application functionally healthy
```

Investigate:

```text
Rollout status
      ↓
Pods
      ↓
Pod events
      ↓
Application logs
      ↓
Readiness/health
      ↓
Metrics
      ↓
Compare v1 vs v2
```

If v2 is confirmed as the cause:

```bash
kubectl rollout undo deployment/<name>
```

Then verify:

```bash
kubectl rollout status deployment/<name>
kubectl get pods
```

After service restoration:

> Investigate the root cause with application/development teams using logs, metrics, traces, configuration, and deployment differences.

---

# 35. Deployment Success vs Application Health

This is an important production concept.

Kubernetes might see:

```text
Pod = Running
Pod = Ready
Rollout = Successful
```

But the application can still have business-level failures.

Example:

```text
HTTP health endpoint → 200
Payment processing   → broken
```

Therefore production deployments should use:

* Readiness probes
* Liveness probes
* Startup probes
* Application metrics
* Logs
* Monitoring
* Alerts

These topics will be covered in later days.

---

# 36. Hands-On Lab — Deployment

Create:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx-deployment

spec:
  replicas: 3

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx:1.27
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

Check:

```bash
kubectl get deployment
kubectl get rs
kubectl get pods
```

Understand the hierarchy.

---

# 37. Hands-On Lab — Self-Healing

Get Pods:

```bash
kubectl get pods
```

Delete one:

```bash
kubectl delete pod <pod-name>
```

Immediately:

```bash
kubectl get pods
```

Observe the replacement.

Ask:

```text
Who detected the missing Pod?
Who created the replacement?
Who selected the node?
Who started the container?
```

Expected mental model:

```text
ReplicaSet
   ↓
Replacement Pod
   ↓
Scheduler
   ↓
kubelet
   ↓
Container Runtime
```

---

# 38. Hands-On Lab — Scaling

Scale up:

```bash
kubectl scale deployment nginx-deployment --replicas=5
```

Watch:

```bash
kubectl get pods -w
```

Then:

```bash
kubectl scale deployment nginx-deployment --replicas=2
```

Observe how the desired state changes and the ReplicaSet reconciles the difference.

---

# 39. Hands-On Lab — Rolling Update

Initial image:

```text
nginx:1.27
```

Update:

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.28
```

Watch:

```bash
kubectl rollout status deployment/nginx-deployment
```

Also:

```bash
kubectl get rs
kubectl get pods -w
```

Observe the old and new ReplicaSets during the rollout.

---

# 40. Hands-On Lab — Revision History

Run:

```bash
kubectl rollout history deployment/nginx-deployment
```

Inspect:

```bash
kubectl rollout history deployment/nginx-deployment --revision=1
```

Update again:

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.29
```

Then:

```bash
kubectl rollout history deployment/nginx-deployment
```

Observe the revisions.

---

# 41. Hands-On Lab — Rollback

Rollback:

```bash
kubectl rollout undo deployment/nginx-deployment
```

Watch:

```bash
kubectl rollout status deployment/nginx-deployment
```

Verify:

```bash
kubectl describe deployment nginx-deployment
```

Understand which revision became active.

---

# 42. Hands-On Lab — RollingUpdate Configuration

Example:

```yaml
strategy:
  type: RollingUpdate

  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```

Try changing:

```text
maxUnavailable
maxSurge
```

Observe the difference in rollout behavior.

---

# 43. Important Production Best Practices

## 1. Use Deployments for stateless applications

```text
Deployment
    ↓
ReplicaSet
    ↓
Pods
```

---

## 2. Don't manually manage individual Pods

For production application workloads, use appropriate controllers.

---

## 3. Use versioned container images

Avoid relying on:

```text
latest
```

for production deployments.

Prefer explicit versions or immutable image digests.

---

## 4. Configure readiness probes

A newly created Pod should not receive traffic until the application is ready.

---

## 5. Configure appropriate rollout settings

Consider:

```text
maxUnavailable
maxSurge
```

based on availability and capacity requirements.

---

## 6. Maintain rollback capability

Keep appropriate revision history.

---

## 7. Monitor application health

Don't rely only on:

```text
kubectl get pods
```

Use:

* Logs
* Metrics
* Probes
* Monitoring
* Alerts
* Tracing where appropriate

---

# 44. Day 3 Most Important Interview Concepts

### ReplicaSet

> Maintains the desired number of matching Pods.

### Deployment

> Manages application releases and ReplicaSets, including rolling updates and rollback.

### Scheduler

> Selects a suitable node for an unscheduled Pod.

### kubelet

> Manages Pods and their containers on a worker node.

### RollingUpdate

> Gradually replaces old application Pods with new ones.

### maxUnavailable

> Controls how much desired capacity may be unavailable during a rollout.

### maxSurge

> Controls how many extra Pods may temporarily exist during a rollout.

### Rollback

> Moves the Deployment back toward a previous revision.

---

# 45. Day 3 Component Responsibility Map

```text
                    Deployment
                         │
                         │
                  Manages releases
                         │
                         ↓
                    ReplicaSet
                         │
                         │
                 Maintains replicas
                         │
                         ↓
                       Pods
                         │
                         │
                    Scheduler
                         │
                         │
                  Selects node
                         │
                         ↓
                    Worker Node
                         │
                      kubelet
                         │
                         ↓
                Container Runtime
                         │
                         ↓
                     Container
```

---

# 46. Critical Day 3 Corrections

### Correction 1

❌ Deployment is required for self-healing.

✅ ReplicaSet itself maintains its desired Pod count.

---

### Correction 2

❌ ReplicaSet decides where Pods run.

✅ Scheduler selects the node.

---

### Correction 3

❌ Deployment directly creates/manages Pods.

✅ Deployment manages ReplicaSets; ReplicaSets maintain Pods.

---

### Correction 4

❌ Old ReplicaSet is deleted immediately after rollout.

✅ Old ReplicaSet is generally scaled to zero and retained according to revision history.

---

### Correction 5

❌ `maxUnavailable: -2`

✅ `maxUnavailable` must be a valid non-negative value or percentage.

---

### Correction 6

❌ Insufficient cluster resources → CrashLoopBackOff.

✅ Insufficient scheduling capacity commonly results in `Pending`.

---

### Correction 7

❌ Image pull problem → Pending only.

✅ Image pull failures commonly produce `ImagePullBackOff` or related image-pull errors.

---

### Correction 8

❌ Deployment rollout successful means application is healthy.

✅ Kubernetes rollout success does not guarantee business/application-level health.

---

# 47. Day 3 Command Cheat Sheet

```bash
# Deploy
kubectl apply -f deployment.yaml

# Deployments
kubectl get deployments
kubectl describe deployment <name>

# ReplicaSets
kubectl get rs
kubectl describe rs <name>

# Pods
kubectl get pods
kubectl get pods -o wide
kubectl describe pod <name>

# Scaling
kubectl scale deployment <name> --replicas=5

# Update image
kubectl set image deployment/<name> <container>=<image>:<tag>

# Rollout status
kubectl rollout status deployment/<name>

# Rollout history
kubectl rollout history deployment/<name>

# Inspect revision
kubectl rollout history deployment/<name> --revision=<number>

# Rollback
kubectl rollout undo deployment/<name>

# Pause
kubectl rollout pause deployment/<name>

# Resume
kubectl rollout resume deployment/<name>

# Events
kubectl get events

# Logs
kubectl logs <pod-name>

# Previous container logs
kubectl logs <pod-name> --previous
```

---

# 48. Day 3 Final Mental Model

Think about Kubernetes as a set of responsibilities:

```text
"What should exist?"
        ↓
Deployment
        ↓
"What number of Pods should exist?"
        ↓
ReplicaSet
        ↓
"Where should each Pod run?"
        ↓
Scheduler
        ↓
"How do I run this Pod on my node?"
        ↓
kubelet
        ↓
"How do I execute the container?"
        ↓
Container Runtime
```

For a production update:

```text
v1
 ↓
Deployment updated
 ↓
New ReplicaSet
 ↓
New Pods
 ↓
RollingUpdate
 ↓
New Pods become Ready
 ↓
Old ReplicaSet scales down
 ↓
Old revision retained
```

For a production failure:

```text
Pod disappears
 ↓
ReplicaSet detects replica mismatch
 ↓
Replacement Pod
 ↓
Scheduler selects node
 ↓
kubelet starts containers
```

---

# Day 3 Final Takeaway

The most important thing to understand is not the individual commands.

It is the **responsibility chain**:

```text
Deployment
   ↓
manages versions & ReplicaSets

ReplicaSet
   ↓
maintains replica count

Scheduler
   ↓
selects nodes

kubelet
   ↓
manages containers on the node
```

Once this becomes clear, **scaling, self-healing, rolling updates, rollback, and Deployment troubleshooting all become connected concepts instead of separate topics.**

# Day 3 Revision Checklist

* [ ] ReplicaSet
* [ ] Desired vs Actual State
* [ ] ReplicaSet self-healing
* [ ] Labels and Selectors
* [ ] Deployment
* [ ] Deployment vs ReplicaSet
* [ ] Deployment hierarchy
* [ ] RollingUpdate
* [ ] Recreate
* [ ] maxUnavailable
* [ ] maxSurge
* [ ] Revision history
* [ ] Rollback
* [ ] Scaling
* [ ] Scheduler responsibility
* [ ] kubelet responsibility
* [ ] Pending vs CrashLoopBackOff
* [ ] ImagePullBackOff
* [ ] Production rollout troubleshooting
* [ ] Production rollback
* [ ] Deployment success vs application health
* [ ] Production best practices
* [ ] Kubernetes component responsibility chain
