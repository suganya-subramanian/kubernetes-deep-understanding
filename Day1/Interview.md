# Kubernetes Day 1 – Interview Questions & Production-Ready Answers

## Q1. What is Kubernetes, and why do we need Kubernetes when Docker can already run containers?

### Answer

Kubernetes is an open-source container orchestration platform used to manage containerized workloads across a cluster of machines.

Docker primarily provides containerization capabilities such as building and running containers. When the number of containers and hosts increases, manually managing scheduling, scaling, networking, service discovery, rolling updates, and failure recovery becomes difficult.

Kubernetes provides these orchestration capabilities.

For example, if a worker node fails, Kubernetes can detect that the desired workload is no longer satisfied and recreate/reschedule Pods on healthy available nodes.

However, Kubernetes does not normally provision a new physical or virtual server by itself. If additional nodes are required, a configured infrastructure or cluster autoscaling mechanism can provision them.

### Key Point

```text
Docker
→ Build/run containers

Kubernetes
→ Orchestrate containerized workloads across a cluster
```

---

# Q2. Explain the difference between a Kubernetes Control Plane and a Worker Node.

### Answer

The Control Plane manages the Kubernetes cluster, while Worker Nodes run application workloads.

The major Control Plane components are:

* API Server
* etcd
* Scheduler
* Controller Manager

The major Worker Node components are:

* kubelet
* kube-proxy
* Container Runtime
* Pods

The API Server is the main entry point for Kubernetes API operations. etcd stores persistent cluster state. The Scheduler selects suitable nodes for unscheduled Pods, and Controllers continuously reconcile desired state with actual state.

On a Worker Node, kubelet manages the Pods assigned to that node, the container runtime runs the containers, and kube-proxy implements Service-related networking behavior.

### Mental Model

```text
Control Plane = DECIDE + MANAGE

Worker Node = RUN
```

---

# Q3. You run `kubectl apply -f deployment.yaml`. Explain step-by-step what happens internally.

### Answer

When I execute:

```bash
kubectl apply -f deployment.yaml
```

the following high-level process occurs:

1. `kubectl` sends the Deployment request to the Kubernetes API Server.
2. The API Server authenticates and authorizes the request and performs validation/admission processing.
3. The Deployment object's state is persisted through the API Server into etcd.
4. The Deployment Controller observes the desired Deployment state.
5. The Deployment Controller creates or updates a ReplicaSet.
6. The ReplicaSet Controller works to create the required number of Pods.
7. The newly created Pods may initially be unscheduled.
8. The Scheduler evaluates the available nodes and scheduling constraints.
9. The Scheduler assigns each Pod to a suitable node.
10. The kubelet on the selected node observes the assigned Pod.
11. The kubelet works with the container runtime to pull the image if necessary and start the containers.
12. The kubelet reports the Pod status through the API Server.

The overall relationship is:

```text
kubectl
   ↓
API Server
   ↓
etcd
   ↓
Deployment Controller
   ↓
ReplicaSet
   ↓
Pods
   ↓
Scheduler
   ↓
Worker Node
   ↓
kubelet
   ↓
Container Runtime
   ↓
Application
```

An important point is that Kubernetes does not execute the YAML file like a shell script. The YAML declares the desired state, and Kubernetes components work together to achieve that state.

---

# Q4. Three worker nodes have different CPU and memory utilization. How does Kubernetes decide where to schedule a new Pod?

### Answer

The Scheduler is responsible for selecting the node for an unscheduled Pod.

It does not simply choose the node with the lowest current CPU utilization.

The Scheduler evaluates whether nodes satisfy the Pod's requirements and scheduling constraints. Important factors can include:

* CPU requests
* Memory requests
* Node selectors
* Node affinity
* Pod affinity/anti-affinity
* Taints and tolerations
* Topology constraints
* Other scheduling rules

For example, if a Pod requests:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
```

the Scheduler evaluates whether a node has sufficient allocatable capacity for those requests.

Therefore, if Node-2 has the most suitable available capacity and satisfies all scheduling constraints, the Scheduler may select Node-2.

### Key Point

> Kubernetes scheduling is primarily based on resource requests and scheduling constraints, not simply current CPU percentage.

---

# Q5. A production application should have 5 replicas, but a worker node containing two Pods crashes. What happens?

### Answer

Assume:

```text
Desired replicas = 5
```

Two Pods are lost because a worker node fails.

The cluster may temporarily have:

```text
Desired = 5
Actual = 3
```

Kubernetes controllers continuously reconcile desired state with actual state.

The appropriate controller determines that additional Pods are required and works to create replacement Pods.

Those replacement Pods initially need to be scheduled.

The Scheduler then selects suitable healthy nodes.

The kubelet on those nodes works with the container runtime to start the Pods.

Eventually:

```text
Desired = 5
Actual = 5
```

The important distinction is:

```text
Controller
→ Determines replacement Pods are needed

Scheduler
→ Determines where the replacement Pods should run

kubelet
→ Ensures the assigned Pods run on the node
```

This is part of Kubernetes' reconciliation/self-healing behavior.

---

# Q6. The API Server is down, but existing Pods are still running. Will the application continue working?

### Answer

Not necessarily all application functionality is immediately affected.

Existing Pods may continue running even when the API Server is temporarily unavailable because the application containers do not need to communicate with the API Server for every application request.

However, Kubernetes control-plane operations are significantly affected.

For example, operations such as:

```bash
kubectl get pods
kubectl create ...
kubectl delete ...
kubectl scale ...
```

may fail because they require communication with the API Server.

Other Kubernetes management operations such as scheduling and controller reconciliation can also be impaired.

Therefore:

> API Server failure does not necessarily mean that all existing application Pods immediately stop. It primarily causes Kubernetes control-plane and management operations to become unavailable or impaired.

For production, we avoid a single API Server as a single point of failure by using an appropriate highly available control-plane architecture.

---

# Q7. A Pod is stuck in Pending state. What would you investigate?

### Answer

I would first determine why the Pod has not been scheduled or started.

I would begin with:

```bash
kubectl get pod <pod-name>
```

Then:

```bash
kubectl describe pod <pod-name>
```

I would pay particular attention to the Events section.

For example:

```text
FailedScheduling
```

could indicate a scheduling problem.

I would also inspect:

```bash
kubectl get nodes
```

and:

```bash
kubectl describe node <node-name>
```

Possible causes include:

* Insufficient CPU
* Insufficient memory
* Node selector mismatch
* Node affinity mismatch
* Pod affinity/anti-affinity constraints
* Taints without matching tolerations
* Storage/PVC issues
* Topology constraints
* Other scheduling restrictions

The Scheduler is the primary component I would investigate for a Pod that cannot be scheduled.

`kubectl logs` may not be useful initially because the application container may not have started yet.

---

# Q8. etcd becomes unavailable. What happens? Do existing Pods immediately stop?

### Answer

Existing Pods do not necessarily stop immediately when etcd becomes unavailable.

The application containers that are already running can continue running because container execution on worker nodes does not require etcd for every moment of application execution.

However, etcd is critical because it stores Kubernetes' persistent cluster state.

If etcd is unavailable, the API Server cannot reliably perform the required state operations against etcd.

As a result, Kubernetes control-plane functionality can become severely impaired, including:

* Creating resources
* Updating resources
* Persisting state
* Reading persisted state
* Scheduling
* Controller reconciliation
* General cluster management

The exact behavior depends on the scope and duration of the etcd failure and whether the cluster has a healthy etcd quorum.

For production, etcd requires appropriate HA, protection, backups, and restore procedures.

---

# Q9. Why is running only one Control Plane node risky? How would you design it for HA?

### Answer

A single Control Plane node creates a significant single point of failure.

If that node becomes unavailable, Kubernetes control-plane functionality can be disrupted.

Existing workloads may continue running, but operations such as API access, scheduling, and controller reconciliation can be impaired.

For a production environment, I would use an appropriate highly available control-plane architecture.

A common design is:

```text
                    Load Balancer
                         |
             +-----------+-----------+
             |           |           |
           CP-1        CP-2        CP-3
             |           |           |
             +-----------+-----------+
                         |
                     etcd cluster
                         |
              +----------+----------+
              |          |          |
           Worker-1   Worker-2   Worker-3
```

An HA design should provide redundancy for the control-plane components and appropriate etcd availability.

Three control-plane/etcd members are common in many production designs because etcd uses quorum-based consensus and an odd number of members helps with quorum behavior.

The exact architecture depends on whether Kubernetes is managed or self-managed and on the platform being used.

---

# Q10. How would you protect etcd in production, and why are backups important?

### Answer

etcd is critical because it stores Kubernetes cluster state.

A production etcd protection strategy should include:

* High availability where appropriate
* Regular etcd snapshots
* Secure backup storage
* Encryption where appropriate
* Strong access control
* Backup retention policies
* Monitoring backup success/failure
* Regular restore testing

The conceptual flow is:

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

A PersistentVolume should not be considered an etcd backup mechanism by itself.

The important point is that etcd snapshots should be created using the appropriate etcd backup mechanism and stored independently and securely.

Backups are important because if etcd state is lost and there is no valid recovery point, reconstructing the Kubernetes cluster state can be extremely difficult.

A production backup is only truly useful if restoration has been tested successfully.

---

# Day 1 – Interview Quick Revision

## Component → Responsibility

```text
API Server
→ Kubernetes API entry point

etcd
→ Persistent cluster state

Scheduler
→ Selects node for unscheduled Pods

Controller Manager
→ Reconciles desired state and actual state

kubelet
→ Manages Pods on a worker node

kube-proxy
→ Implements Service networking behavior

Container Runtime
→ Runs containers

Pod
→ Smallest deployable workload unit
```

---

# Most Important Architecture Flow

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

# Important Interview Distinctions

### Scheduler vs Controller

```text
Controller:
"Do I have the desired number/state of workloads?"

Scheduler:
"Which node should this unscheduled Pod run on?"
```

### etcd vs PersistentVolume

```text
etcd:
Kubernetes cluster state

PV:
Persistent storage for workloads
```

### API Server vs Worker Node

```text
API Server:
Control-plane entry point

Worker Node:
Runs application workloads
```

### Multiple Control Planes vs Multiple Clusters

```text
Multiple Control Planes:
One HA Kubernetes cluster

Multiple Clusters:
Separate Kubernetes environments
```

### Kubernetes vs Infrastructure Autoscaling

```text
Kubernetes:
Manages workloads

Infrastructure/Cluster Autoscaling:
Can provision additional nodes when configured
```

---

# Day 1 Production Keywords

Before an interview, remember these terms:

* Container orchestration
* Declarative configuration
* Desired state
* Actual state
* Reconciliation
* Self-healing
* Control Plane
* Worker Node
* API Server
* etcd
* Scheduler
* Controller Manager
* kubelet
* kube-proxy
* Container Runtime
* CRI
* Pod
* ReplicaSet
* High Availability
* etcd quorum
* etcd snapshots
* Node failure
* API Server failure
* Pending Pod
* FailedScheduling
* Resource requests
* Taints
* Tolerations
* Node affinity
* RBAC
* Production resilience
