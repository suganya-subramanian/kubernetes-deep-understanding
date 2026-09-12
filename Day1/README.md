# Kubernetes Deep Learning – Day 1

# Kubernetes Architecture

## 1. Learning Objectives

By the end of Day 1, you should be able to:

* Explain why Kubernetes is needed.
* Explain the difference between Docker and Kubernetes.
* Explain Kubernetes cluster architecture.
* Explain Control Plane and Worker Node responsibilities.
* Explain the API Server.
* Explain etcd.
* Explain the Scheduler.
* Explain the Controller Manager.
* Explain kubelet.
* Explain kube-proxy.
* Explain the Container Runtime.
* Understand the Kubernetes Object Model.
* Explain desired state vs actual state.
* Explain how `kubectl apply` works internally.
* Explain how Kubernetes performs self-healing.
* Understand what happens when a worker node fails.
* Understand the impact of API Server failure.
* Understand the impact of etcd failure.
* Understand basic Kubernetes High Availability.
* Troubleshoot a Pod stuck in `Pending`.
* Understand important production best practices.

---

# 2. Why Kubernetes?

## 2.1 The Problem

Docker makes it easy to:

* Build container images.
* Start containers.
* Stop containers.
* Manage containers on a host.

But imagine a production environment with:

* Hundreds of servers.
* Thousands of containers.
* Multiple applications.
* Different replicas.
* Frequent deployments.
* Failed nodes.
* High traffic.
* Requirements for high availability.

Managing all of this manually becomes difficult.

### Example

Suppose an application has:

```text
10 containers
3 servers
```

One server fails.

Without an orchestrator, an administrator may need to:

* Detect the failure.
* Identify affected containers.
* Start replacement containers.
* Place them on another server.
* Reconfigure networking.
* Redirect traffic.

At large scale, this is not practical.

---

# 3. What is Kubernetes?

Kubernetes is an open-source **container orchestration platform**.

It automates:

* Container deployment
* Scheduling
* Scaling
* Self-healing
* Service discovery
* Load balancing
* Rolling updates
* Rollbacks
* Networking
* Storage orchestration
* Workload management

A useful mental model:

> Kubernetes is like an operating system for a cluster of machines.

Instead of managing containers individually on individual servers, Kubernetes manages workloads across a cluster.

---

# 4. Docker vs Kubernetes

Docker and Kubernetes are not direct replacements for each other.

| Docker                              | Kubernetes                          |
| ----------------------------------- | ----------------------------------- |
| Containerization platform           | Container orchestration platform    |
| Builds container images             | Manages containerized workloads     |
| Runs containers                     | Schedules workloads across nodes    |
| Usually focused on individual hosts | Designed for clusters               |
| Manual scaling is possible          | Automated scaling can be configured |
| Limited self-healing                | Built-in workload reconciliation    |
| Container runtime/tooling ecosystem | Cluster management/orchestration    |

### Simple interview explanation

> Docker helps us build and run containers, while Kubernetes manages containerized workloads across multiple machines and provides scheduling, scaling, networking, rolling updates, and self-healing.

---

# 5. Important Correction: Kubernetes Does Not Automatically Create Servers by Itself

A common misconception is:

> "If a Kubernetes worker node fails, Kubernetes automatically creates another server."

This is not generally correct.

Kubernetes can:

```text
Failed Node
     ↓
Replacement Pod
     ↓
Healthy Existing Node
```

If there is insufficient capacity, a configured **cluster/node autoscaling mechanism** may provision additional infrastructure.

Therefore:

```text
Kubernetes
    =
Workload orchestration

Infrastructure autoscaling
    =
Provisioning additional nodes/VMs
```

These are related but different responsibilities.

---

# 6. Kubernetes Cluster

A Kubernetes cluster consists primarily of:

```text
Kubernetes Cluster
│
├── Control Plane
│
└── Worker Nodes
```

The Control Plane manages the cluster.

Worker Nodes run application workloads.

---

# 7. High-Level Kubernetes Architecture

```text
                         USERS / OPERATORS
                                |
                              kubectl
                                |
                                v
                       +------------------+
                       |    API Server    |
                       +------------------+
                         /       |       \
                        /        |        \
                       v         v         v
                    etcd     Scheduler   Controller
                                           Manager
                                              |
                                              |
                         +--------------------+
                         |
             +-----------+-----------+
             |                       |
             v                       v
       +-----------+           +-----------+
       | Worker 1  |           | Worker 2  |
       |-----------|           |-----------|
       | kubelet   |           | kubelet   |
       | kube-proxy|           | kube-proxy|
       | Runtime   |           | Runtime   |
       | Pods      |           | Pods      |
       +-----------+           +-----------+
```

---

# 8. Control Plane

The Control Plane is responsible for managing the Kubernetes cluster.

Its major components are:

```text
Control Plane
│
├── API Server
├── etcd
├── Scheduler
└── Controller Manager
```

Think:

> Control Plane = DECIDES + MANAGES

It does not normally run the application's containers itself.

---

# 9. API Server

## 9.1 What is the API Server?

The Kubernetes API Server is the central entry point to the Kubernetes API.

It acts as the front door of the cluster.

Examples of clients:

```text
kubectl
Controllers
Scheduler
Kubelets
Automation tools
CI/CD systems
```

communicate with the Kubernetes API through the API Server.

---

# 10. API Server Responsibilities

The API Server is responsible for handling Kubernetes API requests.

Important responsibilities include:

* Authentication
* Authorization
* Admission processing
* API validation
* Object creation/update/deletion
* Communication with other Kubernetes components
* Reading/writing cluster state through etcd

---

# 11. Example: kubectl

When we execute:

```bash
kubectl get pods
```

The basic flow is:

```text
kubectl
   |
   v
API Server
   |
   v
etcd / cluster state
   |
   v
API Server
   |
   v
kubectl
```

For creating a workload:

```bash
kubectl apply -f deployment.yaml
```

the flow is more involved and is explained later.

---

# 12. etcd

## 12.1 What is etcd?

`etcd` is the distributed key-value store used by Kubernetes for persistent cluster state.

Think:

> etcd = Kubernetes' persistent cluster-state database.

It stores information about Kubernetes objects and cluster configuration/state.

Examples include:

* Pods
* Deployments
* ReplicaSets
* Services
* Secrets
* ConfigMaps
* Nodes
* PersistentVolume objects
* PersistentVolumeClaim objects
* Cluster configuration/state

---

# 13. Why etcd Is Critical

If etcd is lost without a valid recovery mechanism, recovering the Kubernetes cluster's state can become extremely difficult.

Therefore production environments need:

* etcd high availability where appropriate
* Regular snapshots
* Secure backup storage
* Access control
* Encryption
* Backup retention
* Restore testing

---

# 14. etcd Is Not a PersistentVolume

Important distinction:

```text
etcd
=
Kubernetes cluster state

PersistentVolume
=
Storage resource used by workloads
```

A PersistentVolume does not automatically constitute an etcd backup strategy.

Production etcd backup normally uses an appropriate **etcd snapshot/backup mechanism**, with snapshots stored in a secure backup location.

---

# 15. Scheduler

## 15.1 What is the Scheduler?

The Kubernetes Scheduler determines which worker node should run an unscheduled Pod.

Example:

```text
Pod
 |
 | needs scheduling
 v
Scheduler
 |
 +----> Worker Node A
 +----> Worker Node B
 +----> Worker Node C
```

The Scheduler evaluates whether nodes satisfy the Pod's scheduling requirements.

---

# 16. What Does the Scheduler Consider?

Depending on the workload and configuration, scheduling can consider:

* CPU requests
* Memory requests
* Node availability
* Node selectors
* Node affinity
* Pod affinity
* Pod anti-affinity
* Taints
* Tolerations
* Topology constraints
* Other scheduling constraints

---

# 17. Important Scheduler Concept: Requests

A common beginner mistake is:

> "The Scheduler simply chooses the node with the lowest CPU usage."

Not exactly.

Kubernetes scheduling primarily evaluates **requested resources and scheduling constraints**.

Example:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
```

The Scheduler evaluates whether a node has enough **allocatable capacity** to satisfy these requests along with other constraints.

Actual CPU utilization is a different concept.

This topic becomes much more important on **Day 8 – Resource Management**.

---

# 18. Controller Manager

## 18.1 What is the Controller Manager?

The Controller Manager runs various Kubernetes controllers.

Controllers continuously compare:

```text
Desired State
     vs
Actual State
```

and attempt to reconcile differences.

This is called the **reconciliation loop**.

---

# 19. Desired State vs Actual State

Suppose a Deployment specifies:

```text
replicas: 3
```

Desired state:

```text
3 Pods
```

But only two Pods are currently available.

```text
Desired = 3
Actual  = 2
```

The appropriate controller detects the difference and works toward:

```text
Desired = 3
Actual  = 3
```

This reconciliation model is fundamental to Kubernetes.

---

# 20. Scheduler vs Controller

This distinction is extremely important.

### Scheduler

Answers:

> "Which node should this unscheduled Pod run on?"

### Controller

Answers:

> "Does the actual state match the desired state?"

Example:

```text
Desired: 5 Pods
Actual:  3 Pods
```

Controller:

```text
I need 2 more Pods.
```

Scheduler:

```text
I need to decide where those new Pods should run.
```

---

# 21. Worker Node

Worker Nodes are where application workloads run.

A typical Worker Node contains:

```text
Worker Node
│
├── kubelet
├── kube-proxy
├── Container Runtime
└── Pods
```

Think:

> Worker Node = RUNS

---

# 22. kubelet

## 22.1 What is kubelet?

`kubelet` is the Kubernetes node agent.

It runs on each worker node.

Its responsibilities include:

* Receiving Pod/workload instructions
* Ensuring assigned Pods are running
* Working with the container runtime
* Monitoring containers/Pods
* Reporting status to the API Server
* Reporting node health information

---

# 23. kubelet Flow

Conceptually:

```text
API Server
    |
    v
kubelet
    |
    v
Container Runtime
    |
    v
Container
```

The kubelet does not itself act as the container runtime.

It coordinates with the runtime.

---

# 24. Container Runtime

The container runtime is responsible for actually running containers.

Modern Kubernetes uses the **Container Runtime Interface (CRI)** to communicate with container runtimes.

Common examples include:

* containerd
* CRI-O

Historically, Docker Engine was commonly used with Kubernetes, but modern Kubernetes does not use Docker Engine directly as its CRI runtime.

---

# 25. kube-proxy

`kube-proxy` is a node-level networking component associated with Kubernetes Services.

It helps implement Service networking by programming networking rules according to the cluster's configuration.

Conceptually:

```text
Client
   |
   v
Service
   |
   v
Pod
```

Traffic can be distributed among eligible backend Pods.

Modern Kubernetes networking implementations can differ, and some environments replace or supplement traditional kube-proxy behavior with other networking components.

---

# 26. Pod

A Pod is the smallest deployable unit in Kubernetes.

A Pod can contain:

```text
Pod
│
├── Container 1
├── Container 2
└── Container 3
```

Multiple containers in a Pod share certain Pod-level resources and are intended to work closely together.

Pods will be covered deeply on Day 2.

---

# 27. Kubernetes Object Model

Kubernetes is object-oriented in its API model.

Examples of Kubernetes objects:

* Pod
* Deployment
* ReplicaSet
* Service
* ConfigMap
* Secret
* Node
* PersistentVolume
* PersistentVolumeClaim
* Ingress

These objects represent desired configuration/state in the cluster.

---

# 28. Declarative Model

Kubernetes is primarily declarative.

Instead of telling Kubernetes every individual action to perform, we declare the desired state.

Example:

```yaml
spec:
  replicas: 3
```

This means:

> "I want three replicas."

We don't manually tell Kubernetes:

```text
Create Pod 1
Create Pod 2
Create Pod 3
```

Kubernetes controllers work toward the desired state.

---

# 29. Complete kubectl apply Flow

Consider:

```bash
kubectl apply -f deployment.yaml
```

A simplified architecture flow is:

```text
                     kubectl
                        |
                        v
                  +-----------+
                  | API Server|
                  +-----------+
                        |
                        v
                      etcd
                        |
                        v
               Deployment Controller
                        |
                        v
                   ReplicaSet
                        |
                        v
                 ReplicaSet Controller
                        |
                        v
                      Pods
                        |
                        v
                    Scheduler
                        |
                        v
                  Selected Node
                        |
                        v
                     kubelet
                        |
                        v
                Container Runtime
                        |
                        v
                  Application
```

---

# 30. Step-by-Step kubectl apply

## Step 1 – User executes command

```bash
kubectl apply -f deployment.yaml
```

`kubectl` sends the request to the Kubernetes API Server.

---

## Step 2 – API Server processes the request

The API Server performs appropriate:

* Authentication
* Authorization
* Validation
* Admission processing

---

## Step 3 – Object state is persisted

The Deployment object is stored through the API Server in etcd.

---

## Step 4 – Deployment Controller reacts

The Deployment Controller sees the desired Deployment state.

It creates/manages a ReplicaSet.

---

## Step 5 – ReplicaSet Controller creates Pods

The ReplicaSet specifies how many Pods should exist.

For:

```yaml
replicas: 3
```

the ReplicaSet controller works toward having three Pods.

---

## Step 6 – Pods need nodes

New Pods may initially be unscheduled.

The Scheduler identifies suitable nodes.

---

## Step 7 – Scheduler assigns nodes

The Scheduler selects an appropriate worker node based on resource requests and scheduling constraints.

---

## Step 8 – kubelet receives the assignment

The kubelet on the selected node works to make the Pod run.

---

## Step 9 – Container Runtime starts containers

The kubelet interacts with the container runtime.

The runtime:

* Pulls the image if required
* Creates the container
* Starts the container

---

## Step 10 – Status is reported

The kubelet reports Pod/node status through the API Server.

The cluster now moves toward:

```text
Desired State = Actual State
```

---

# 31. Important Correction: Kubernetes Does Not "Execute YAML Commands One by One"

A YAML file is not a shell script.

For example:

```yaml
spec:
  replicas: 3
```

doesn't mean:

```text
Execute command 1
Execute command 2
Execute command 3
```

It declares:

```text
Desired state = 3 replicas
```

Controllers and other components work together to achieve that state.

---

# 32. Deployment → ReplicaSet → Pod

This hierarchy is extremely important:

```text
Deployment
     |
     v
ReplicaSet
     |
     v
Pod
     |
     v
Container
```

Do not say:

> "Deployment directly creates the Pods."

More accurately:

> A Deployment manages ReplicaSets, and ReplicaSets maintain the desired number of Pods.

This will become a major topic on Day 3.

---

# 33. Self-Healing

One of Kubernetes' important features is reconciliation/self-healing.

Suppose:

```text
Desired replicas = 5
Actual replicas  = 5
```

Everything is healthy.

Now two Pods disappear:

```text
Desired = 5
Actual  = 3
```

The appropriate controller detects:

```text
Actual != Desired
```

and creates replacement Pods.

Then the Scheduler determines suitable nodes for unscheduled replacement Pods.

Finally:

```text
Desired = 5
Actual  = 5
```

---

# 34. Worker Node Failure Scenario

Suppose:

```text
Node-1
 ├── Pod-A
 ├── Pod-B
```

and Node-1 fails.

The cluster may now have:

```text
Desired = 5
Available = 3
```

The control plane detects that the desired workload count is not satisfied.

The appropriate controller works to create replacement Pods.

Then:

```text
Controller
    ↓
Replacement Pods
    ↓
Scheduler
    ↓
Healthy Nodes
    ↓
kubelet
    ↓
Container Runtime
```

---

# 35. Controller vs Scheduler During Node Failure

This is a critical interview distinction.

### Controller

Determines:

> "I need replacement Pods."

### Scheduler

Determines:

> "These replacement Pods should run on these nodes."

### kubelet

Ensures:

> "The assigned Pod actually runs on this node."

---

# 36. API Server Failure

This is a very important production scenario.

Suppose:

```text
API Server
    X
  DOWN
```

It does **not automatically mean all application containers immediately stop**.

Existing workloads may continue running depending on the exact failure state.

However, Kubernetes control-plane operations become unavailable or impaired.

Examples:

```bash
kubectl get pods
kubectl create ...
kubectl delete ...
kubectl scale ...
```

may fail because the API Server is unavailable.

Control-plane activities that depend on API Server communication are also affected.

---

# 37. API Server Failure Mental Model

Think:

```text
API Server DOWN
       |
       +---- Existing workloads may continue running
       |
       +---- kubectl operations affected
       |
       +---- Control-plane management affected
       |
       +---- Scheduling/reconciliation can be impaired
```

Therefore:

> API Server failure is primarily a Kubernetes control-plane availability problem, not necessarily an immediate application runtime failure.

---

# 38. API Server High Availability

Production clusters should avoid a single API Server as a single point of failure.

A common HA architecture:

```text
                    Load Balancer
                         |
             +-----------+-----------+
             |           |           |
             v           v           v
          API/CP-1    API/CP-2    API/CP-3
             |           |           |
             +-----------+-----------+
                         |
                     etcd cluster
```

The exact architecture depends on the Kubernetes distribution and environment.

---

# 39. Multiple Control Planes vs Multiple Clusters

These are different concepts.

### Multiple Control Planes

One HA Kubernetes cluster:

```text
Production Cluster
│
├── Control Plane 1
├── Control Plane 2
└── Control Plane 3
```

### Multiple Clusters

Separate Kubernetes clusters:

```text
Development Cluster
Staging Cluster
Production Cluster
```

Do not use these terms interchangeably.

---

# 40. etcd Failure

Suppose:

```text
etcd
  X
DOWN
```

Existing workloads do not necessarily stop immediately.

Containers that are already running can continue running.

However, Kubernetes' ability to reliably manage and persist cluster state is severely affected.

The API Server cannot normally perform the required persistent state operations against an unavailable etcd.

This can affect:

* Creating objects
* Updating objects
* Reading persisted state
* Scheduling
* Controller reconciliation
* Cluster management

The exact behavior depends on the scope and duration of the etcd failure and cluster topology.

---

# 41. etcd High Availability

Production etcd should be designed for availability.

A common conceptual topology:

```text
             +---------+
             | etcd-1  |
             +---------+
                  |
        +---------+---------+
        |                   |
   +---------+         +---------+
   | etcd-2  |         | etcd-3  |
   +---------+         +---------+
```

etcd uses quorum-based consensus.

This is one reason production systems commonly use an odd number of members.

---

# 42. etcd Backup

Because etcd contains critical Kubernetes cluster state, regular backups are essential.

Production strategy should include:

```text
etcd
 ↓
Snapshot
 ↓
Secure Backup Storage
 ↓
Retention
 ↓
Restore Testing
```

Important practices:

* Take regular snapshots.
* Protect backup storage.
* Encrypt backups where appropriate.
* Restrict access.
* Maintain retention policies.
* Monitor backup success.
* Test restoration regularly.

---

# 43. Why Restore Testing Matters

A backup file existing on disk does not guarantee disaster recovery.

You must verify:

```text
Backup
   ↓
Can it be restored?
   ↓
Does the restored cluster behave correctly?
```

A tested backup is far more valuable than an untested backup.

---

# 44. Pod Pending – Troubleshooting

A Pod in `Pending` means Kubernetes has not yet successfully progressed the Pod to the running state.

One common reason is scheduling failure.

---

# 45. Common Reasons for Pending

Possible causes include:

### Resource shortage

```text
Insufficient CPU
Insufficient Memory
```

### Scheduling constraints

```text
Node selector mismatch
Node affinity mismatch
Pod affinity/anti-affinity constraints
```

### Taints/Tolerations

A node may have a taint that the Pod does not tolerate.

### Storage

A workload may depend on a PVC that is not available/bound.

### Other scheduling constraints

Topology and policy constraints can also prevent scheduling.

---

# 46. Pending Pod Troubleshooting Commands

Start with:

```bash
kubectl get pods
```

Then:

```bash
kubectl describe pod <pod-name>
```

Pay special attention to:

```text
Events
```

For example:

```text
FailedScheduling
```

You can inspect nodes:

```bash
kubectl get nodes
```

and:

```bash
kubectl describe node <node-name>
```

---

# 47. Why kubectl logs May Not Help With Pending Pods

If a Pod is still `Pending`, its application container may not have started.

Therefore:

```bash
kubectl logs <pod-name>
```

may not provide useful application logs.

For scheduling problems, `describe` + Events is usually more useful initially.

---

# 48. Who Is Responsible for Pod Scheduling?

The answer is:

```text
Scheduler
```

Not Controller Manager.

The Controller determines that the desired Pod should exist.

The Scheduler determines where an unscheduled Pod should run.

---

# 49. Architecture Responsibilities – Quick Reference

| Component          | Primary Responsibility             |
| ------------------ | ---------------------------------- |
| API Server         | Kubernetes API entry point         |
| etcd               | Persistent cluster state           |
| Scheduler          | Selects node for unscheduled Pods  |
| Controller Manager | Reconciles desired vs actual state |
| kubelet            | Manages Pods on a node             |
| kube-proxy         | Service networking implementation  |
| Container Runtime  | Runs containers                    |
| Pod                | Smallest deployable workload unit  |

---

# 50. Easy Mental Model

Remember:

### API Server

> "What does the cluster want / what is being requested?"

### etcd

> "What is the persisted cluster state?"

### Scheduler

> "Where should this Pod run?"

### Controller

> "Does actual state match desired state?"

### kubelet

> "Make sure the assigned Pod runs on this node."

### Container Runtime

> "Actually run the container."

### kube-proxy

> "Implement Service-related network routing on the node."

---

# 51. Control Plane vs Worker Node

| Control Plane              | Worker Node       |
| -------------------------- | ----------------- |
| Manages cluster            | Runs workloads    |
| API Server                 | kubelet           |
| etcd                       | kube-proxy        |
| Scheduler                  | Container Runtime |
| Controller Manager         | Pods              |
| Makes/reconciles decisions | Executes workload |

Mental model:

```text
Control Plane = DECIDE + MANAGE

Worker Node = RUN
```

---

# 52. Production Scenario – Application with 5 Replicas

Desired:

```text
5 replicas
```

Actual:

```text
5 replicas
```

Everything is healthy.

Now:

```text
Worker Node fails
```

Two Pods are lost.

Actual:

```text
3 replicas
```

Controller detects:

```text
Desired = 5
Actual = 3
```

It works toward creating:

```text
2 replacement Pods
```

Scheduler assigns them to suitable healthy nodes.

kubelet starts them.

Eventually:

```text
Desired = 5
Actual = 5
```

This is Kubernetes reconciliation/self-healing.

---

# 53. Production Scenario – API Server Down

```text
API Server
    X
```

Potential consequences:

* `kubectl` operations fail.
* New API requests cannot be processed.
* Cluster state operations are affected.
* Scheduling/reconciliation can be impaired.
* Existing workloads may continue running.

Therefore production Kubernetes should use an HA control-plane architecture where appropriate.

---

# 54. Production Scenario – etcd Down

```text
etcd
 X
```

Potential consequences:

* Cluster state cannot be reliably persisted/retrieved.
* API operations depending on etcd are impaired.
* Controllers and Scheduler cannot operate normally.
* Existing workloads may continue running for some time.

The exact behavior depends on the failure and HA topology.

---

# 55. Kubernetes High Availability

A production cluster should avoid unnecessary single points of failure.

Typical design:

```text
                         Users
                           |
                           v
                    Load Balancer
                           |
             +-------------+-------------+
             |             |             |
             v             v             v
           CP-1          CP-2          CP-3
             |             |             |
             +-------------+-------------+
                           |
                      etcd cluster
                           |
            +--------------+--------------+
            |              |              |
            v              v              v
         Worker-1       Worker-2       Worker-3
```

HA design may vary based on:

* Cloud provider
* Kubernetes distribution
* Managed vs self-managed Kubernetes
* etcd topology
* Network architecture

---

# 56. Production Best Practices

## Control Plane

* Use an HA control plane for production where required.
* Avoid unnecessary single points of failure.
* Protect API Server access.
* Use strong authentication and authorization.
* Monitor control-plane health.

## etcd

* Take regular snapshots.
* Secure backup storage.
* Encrypt sensitive backups.
* Restrict access.
* Test restore procedures.
* Monitor backup jobs.

## Worker Nodes

* Monitor node health.
* Maintain sufficient capacity.
* Use appropriate resource requests/limits.
* Plan node maintenance.
* Use multiple worker nodes for availability.

## Networking

* Understand Service networking.
* Use appropriate NetworkPolicies where required.
* Monitor cluster networking.

## Security

* Use RBAC.
* Restrict API Server access.
* Follow least privilege.
* Protect credentials and Secrets.

Security will be covered deeply on Day 12.

---

# 57. Important Interview Corrections

## Incorrect

> Scheduler maintains desired state.

### Correct

> Controller Manager/controllers reconcile desired state with actual state.

---

## Incorrect

> Controller Manager decides which node runs the Pod.

### Correct

> Scheduler selects the node for an unscheduled Pod.

---

## Incorrect

> Deployment directly creates Pods.

### Correct

```text
Deployment
   ↓
ReplicaSet
   ↓
Pod
```

---

## Incorrect

> API Server failure means the entire application immediately goes down.

### Correct

> Existing workloads may continue running, but Kubernetes control-plane operations and management are impaired.

---

## Incorrect

> Kubernetes automatically creates a new physical server when a node fails.

### Correct

> Kubernetes can reschedule workloads onto healthy available nodes. Infrastructure/node autoscaling can provision additional nodes when configured.

---

## Incorrect

> PersistentVolume is the mechanism for backing up etcd.

### Correct

> etcd snapshots/backups should be created using an appropriate etcd backup mechanism and stored securely.

---

## Incorrect

> Pending Pods are primarily investigated with logs.

### Correct

Start with:

```bash
kubectl describe pod <pod-name>
```

and inspect Events, especially scheduling failures.

---

# 58. Hands-on Commands

## Cluster Information

```bash
kubectl cluster-info
```

Purpose:

* Verify cluster connectivity.
* View Kubernetes control-plane information.

---

## List Nodes

```bash
kubectl get nodes
```

More detailed:

```bash
kubectl get nodes -o wide
```

---

## Inspect a Node

```bash
kubectl describe node <node-name>
```

Look for:

* Capacity
* Allocatable resources
* Conditions
* Pods
* Events
* Taints

---

## List System Pods

```bash
kubectl get pods -n kube-system
```

Depending on your Kubernetes distribution, you may see components such as:

* CoreDNS
* Network plugin components
* Metrics components
* Other system workloads

Do not assume every cluster has exactly the same system Pods.

---

## Kubernetes Version

```bash
kubectl version
```

---

# 59. Day 1 Commands – Revision Table

| Command                           | Purpose                           |
| --------------------------------- | --------------------------------- |
| `kubectl cluster-info`            | Cluster/control-plane information |
| `kubectl get nodes`               | List nodes                        |
| `kubectl get nodes -o wide`       | Detailed node information         |
| `kubectl describe node <node>`    | Detailed node information/events  |
| `kubectl get pods -n kube-system` | List system Pods                  |
| `kubectl get pods`                | List Pods                         |
| `kubectl describe pod <pod>`      | Detailed Pod/events               |
| `kubectl version`                 | Client/server version information |

---

# 60. Day 1 Architecture Cheat Sheet

```text
                    KUBERNETES CLUSTER
                           |
             +-------------+-------------+
             |                           |
             v                           v
       CONTROL PLANE                WORKER NODES
             |                           |
     +-------+-------+           +-------+-------+
     |       |       |           |       |       |
     v       v       v           v       v       v
 API      etcd   Scheduler   kubelet kube-proxy Runtime
 Server
     |
 Controller
 Manager
```

---

# 61. Most Important Flow

```text
kubectl
   ↓
API Server
   ↓
etcd
   ↓
Controller
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

---

# 62. Five Concepts You Must Remember

### 1. Kubernetes is declarative

You specify:

```text
What you want
```

Kubernetes works toward that state.

---

### 2. Controllers reconcile state

```text
Desired State
      ↓
Compare
      ↓
Actual State
      ↓
Reconcile
```

---

### 3. Scheduler selects nodes

```text
Unscheduled Pod
       ↓
Scheduler
       ↓
Suitable Worker Node
```

---

### 4. kubelet runs assigned workloads

```text
Assigned Pod
     ↓
kubelet
     ↓
Container Runtime
     ↓
Container
```

---

### 5. etcd stores cluster state

```text
Kubernetes State
       ↓
      etcd
```

Protect it accordingly.

---

# 63. Final Day 1 Revision Checklist

Before moving to Day 2, make sure you can explain these without notes:

* [ ] Why Kubernetes is needed
* [ ] Docker vs Kubernetes
* [ ] Kubernetes Cluster
* [ ] Control Plane
* [ ] Worker Node
* [ ] API Server
* [ ] etcd
* [ ] Scheduler
* [ ] Controller Manager
* [ ] kubelet
* [ ] kube-proxy
* [ ] Container Runtime
* [ ] Pod
* [ ] Kubernetes Object Model
* [ ] Desired State
* [ ] Actual State
* [ ] Reconciliation
* [ ] Deployment → ReplicaSet → Pod
* [ ] `kubectl apply` request flow
* [ ] Worker node failure
* [ ] Self-healing
* [ ] API Server failure
* [ ] etcd failure
* [ ] etcd backup
* [ ] Kubernetes HA
* [ ] Multiple Control Planes vs Multiple Clusters
* [ ] Pod Pending troubleshooting
* [ ] Scheduler vs Controller
* [ ] Production best practices
* [ ] Basic Day 1 kubectl commands

---

# 64. One-Minute Interview Revision

If an interviewer says:

> "Explain Kubernetes architecture."

A strong structure is:

```text
Kubernetes has two major areas:

1. Control Plane
2. Worker Nodes

Control Plane:
- API Server handles API requests.
- etcd stores cluster state.
- Scheduler assigns Pods to nodes.
- Controller Manager reconciles desired and actual state.

Worker Node:
- kubelet manages Pods.
- Container Runtime runs containers.
- kube-proxy implements Service networking.
- Pods run the application workloads.

The API Server is the main communication point.

The overall model is declarative:
we define the desired state, and Kubernetes continuously
reconciles the actual state toward it.
```

Then explain:

```text
kubectl
  ↓
API Server
  ↓
etcd
  ↓
Controllers
  ↓
Scheduler
  ↓
kubelet
  ↓
Container Runtime
  ↓
Pod
```

This is the foundation for the rest of Kubernetes.

---

# Day 1 Final Takeaway

The most important thing is not memorizing component definitions.

Understand the relationship:

```text
                 DESIRED STATE
                      |
                      v
                 API SERVER
                      |
                      v
                     etcd
                      |
             +--------+--------+
             |                 |
             v                 v
        Controllers        Scheduler
             |                 |
             |                 v
             |            Worker Node
             |                 |
             |               kubelet
             |                 |
             +---------> Container Runtime
                               |
                               v
                              Pod
```

Kubernetes continuously works to make:

```text
Actual State = Desired State
```

That reconciliation model is the foundation behind:

* Deployments
* ReplicaSets
* Self-healing
* Scaling
* Scheduling
* Rolling updates
* Stateful workloads
* Production Kubernetes operations
