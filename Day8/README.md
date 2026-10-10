# Kubernetes Day 8: Resource Management

## Overview

Kubernetes Resource Management controls how CPU and memory resources are requested, allocated, and consumed by application workloads.

As a DevOps Engineer, understanding resource management is essential for:

- Scheduling Pods onto appropriate nodes.
- Preventing resource exhaustion.
- Troubleshooting Pending Pods.
- Investigating CPU throttling.
- Troubleshooting OOMKilled containers.
- Managing namespace resource consumption.
- Planning cluster capacity.
- Optimizing infrastructure costs.
- Configuring Horizontal Pod Autoscaler (HPA) and Vertical Pod Autoscaler (VPA).

## Learning Objectives

By the end of this lesson, you should understand:

1. CPU and memory resources.
2. Resource requests and limits.
3. How the Kubernetes scheduler uses resource requests.
4. CPU throttling.
5. OOMKilled containers.
6. Kubernetes QoS classes.
7. ResourceQuota.
8. LimitRange.
9. Node Capacity and Allocatable resources.
10. Resource troubleshooting.
11. Production resource planning.
12. HPA and VPA at a high level.

---

# 1. Why Resource Management Is Needed

Consider a Kubernetes node with the following resources:

```text
Node Capacity:
CPU:    4 cores
Memory: 8 GiB
```

Multiple applications are running on this node.

If one application consumes excessive resources, it can affect other workloads.

Kubernetes provides resource management mechanisms to help control resource consumption and make scheduling decisions.

The two fundamental concepts are:

- Requests
- Limits

## Requests

Requests specify the resources a container requests for scheduling.

The scheduler uses these values when deciding whether a node can accommodate the Pod.

## Limits

Limits specify the maximum resource usage configured for a container.

For CPU, the limit is enforced through CPU throttling.

For memory, exceeding the container's memory limit can result in the container being terminated.

---

# 2. CPU Resources

Kubernetes allows CPU resources to be expressed in cores or millicores.

Examples:

```text
100m  = 0.1 CPU
250m  = 0.25 CPU
500m  = 0.5 CPU
1000m = 1 CPU
2000m = 2 CPU
```

The letter `m` means millicores.

For example:

```yaml
resources:
  requests:
    cpu: "500m"
```

This means the container requests 0.5 CPU.

Another example:

```yaml
resources:
  limits:
    cpu: "1"
```

This configures a CPU limit of one CPU.

## Important Points

- CPU requests influence scheduling decisions.
- CPU limits constrain CPU consumption.
- A CPU request is not a promise that the application will continuously consume that amount.
- Exceeding a CPU limit generally causes throttling rather than immediate container termination.

---

# 3. Memory Resources

Kubernetes memory resources can be specified using units such as:

- `Ki`
- `Mi`
- `Gi`
- `K`
- `M`
- `G`

Binary units such as `Mi` and `Gi` are commonly used for Kubernetes memory configuration.

Examples:

```yaml
resources:
  requests:
    memory: "128Mi"
```

```yaml
resources:
  limits:
    memory: "512Mi"
```

Common values:

```text
128Mi
256Mi
512Mi
1Gi
2Gi
```

## Important Points

- Memory requests influence scheduling decisions.
- Memory limits constrain the container's memory usage.
- Exceeding a memory limit can cause the container to be terminated with an OOMKilled reason.
- Node-level memory pressure can also cause Pods to be evicted or containers to be killed.
- Memory requirements should be based on actual workload behavior and monitoring data.

---

# 4. Resource Requests

A resource request tells Kubernetes how much CPU or memory a container requests for scheduling.

Example:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
```

The container requests:

```text
CPU:    500m
Memory: 512Mi
```

The scheduler uses these requests when determining whether a node can accommodate the Pod.

## How Scheduling Works

```text
Deployment
    |
    v
ReplicaSet
    |
    v
Pod created
    |
    v
Scheduler evaluates the Pod
    |
    +--> CPU requests
    +--> Memory requests
    +--> Node selectors
    +--> Affinity rules
    +--> Taints and tolerations
    +--> Other scheduling constraints
    |
    v
Suitable node selected
    |
    v
Pod assigned to the node
```

The scheduler considers multiple constraints, not just CPU and memory.

## Important Distinction

A request is not the application's actual resource usage.

For example:

```text
CPU request:       500m
Actual CPU usage:  150m
```

The scheduler still considers the configured request when making scheduling decisions.

---

# 5. Resource Limits

A resource limit defines the maximum resource usage configured for a container.

Example:

```yaml
resources:
  limits:
    cpu: "1"
    memory: "1Gi"
```

This configures:

```text
CPU limit:    1 CPU
Memory limit: 1Gi
```

When configuring requests and limits, remember that CPU and memory behave differently.

| Resource | Request | Limit |
|---|---|---|
| CPU | Used for scheduling and resource accounting | Constrains CPU consumption |
| Memory | Used for scheduling and resource accounting | Constrains memory consumption |

## Complete Example

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "1"
    memory: "1Gi"
```

The scheduler considers the request values when selecting a node.

The container can use additional resources up to its configured limits, subject to node capacity and other constraints.

---

# 6. CPU Throttling

CPU throttling occurs when a container is prevented from consuming more CPU because its configured CPU limit has been reached.

Example:

```yaml
resources:
  requests:
    cpu: "250m"
  limits:
    cpu: "500m"
```

The container requests 250m CPU and has a limit of 500m.

If the application needs more CPU than its limit permits, it can experience CPU throttling.

## Possible Symptoms

- Slow application response.
- Increased request latency.
- Timeouts.
- Reduced throughput.
- Poor performance during traffic spikes.

## Troubleshooting CPU Throttling

Check the Pod:

```bash
kubectl describe pod <pod-name>
```

Check current CPU usage:

```bash
kubectl top pod <pod-name>
```

Inspect the configured requests and limits:

```bash
kubectl get pod <pod-name> -o yaml
```

Use Prometheus or another monitoring system to investigate historical CPU usage and throttling metrics.

Possible actions:

- Review CPU limits.
- Review CPU requests.
- Investigate application performance.
- Perform load testing.
- Increase limits if justified by measurements and node capacity.
- Consider HPA if additional replicas can distribute the workload.

A high CPU usage value alone does not prove CPU throttling. Check appropriate throttling metrics when available.

---

# 7. OOMKilled

OOM means Out Of Memory.

`OOMKilled` indicates that a container was terminated because of an out-of-memory condition.

Example:

```yaml
resources:
  requests:
    memory: "128Mi"
  limits:
    memory: "256Mi"
```

If the application exceeds its configured memory limit, it may be terminated.

Kubernetes may report:

```text
Reason: OOMKilled
```

## Troubleshooting OOMKilled

Inspect the Pod:

```bash
kubectl describe pod <pod-name>
```

Check previous container logs:

```bash
kubectl logs <pod-name> --previous
```

Inspect the Pod configuration:

```bash
kubectl get pod <pod-name> -o yaml
```

Check current memory usage when the container is running:

```bash
kubectl top pod <pod-name>
```

For multi-container Pods:

```bash
kubectl top pod <pod-name> --containers
```

## Questions to Investigate

- Is the memory limit too low?
- Is the application experiencing a memory leak?
- Did traffic increase?
- Did the workload change?
- Is there a sudden memory spike?
- Are other containers in the Pod consuming significant memory?
- Is the node experiencing memory pressure?

Use historical monitoring data when available.

`kubectl top` reports current usage and does not provide a complete history of memory consumption.

## Important Distinction

OOMKilled does not always prove that a container exceeded its configured memory limit. Node-level out-of-memory conditions can also cause a container to be killed.

---

# 8. CPU Throttling vs OOMKilled

| CPU throttling | OOMKilled |
|---|---|
| Related to CPU limits | Related to an out-of-memory condition |
| Container is restricted from consuming more CPU | Container may be terminated |
| Application may become slow | Container may restart if its restart policy and workload controller permit it |
| Investigate CPU throttling metrics | Investigate memory limits, usage, logs, and events |

Mental model:

```text
CPU limit reached
       |
       v
CPU throttling
       |
       v
Possible performance degradation


Memory limit exceeded
       |
       v
Possible OOMKilled termination
```

These are common patterns, but the actual cause must be confirmed through evidence.

---

# 9. Resource Configuration for a Container

Example Pod manifest:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
    - name: app
      image: nginx
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
```

Save this file as:

```text
resource-demo.yaml
```

Apply:

```bash
kubectl apply -f resource-demo.yaml
```

Check:

```bash
kubectl get pod resource-demo
```

Inspect:

```bash
kubectl describe pod resource-demo
```

---

# 10. Resource Accounting in Multi-Container Pods

Resource requests and limits are specified for individual containers.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-demo
spec:
  containers:
    - name: app
      image: nginx
      resources:
        requests:
          cpu: "250m"
          memory: "256Mi"
        limits:
          cpu: "500m"
          memory: "512Mi"

    - name: sidecar
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "200m"
          memory: "256Mi"
```

For this example, the combined container requests are:

```text
CPU:
250m + 100m = 350m

Memory:
256Mi + 128Mi = 384Mi
```

For ordinary non-init application containers, the Pod's resource requests and limits are based on the combined requirements of its containers.

Init containers and restartable sidecars have additional resource-accounting rules, so do not assume that every Pod's calculation is always a simple sum of every container listed in the manifest.

---

# 11. Kubernetes QoS Classes

QoS means Quality of Service.

Kubernetes assigns Pods to one of three QoS classes:

1. Guaranteed
2. Burstable
3. BestEffort

QoS classification is based on the resource requests and limits configured for the Pod's containers.

QoS influences how Pods are treated during resource pressure, particularly memory pressure.

## 11.1 Guaranteed

A Pod generally receives Guaranteed QoS when every container has CPU and memory requests equal to their corresponding limits.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-demo
spec:
  containers:
    - name: app
      image: nginx
      resources:
        requests:
          cpu: "500m"
          memory: "256Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
```

The CPU request equals the CPU limit.

The memory request equals the memory limit.

Therefore, this Pod qualifies for Guaranteed QoS.

## 11.2 Burstable

A Pod is generally Burstable when it has resource requests or limits configured but does not meet the Guaranteed criteria.

Example:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "1"
    memory: "512Mi"
```

The requests differ from the limits.

Therefore, this Pod receives Burstable QoS.

## 11.3 BestEffort

A Pod receives BestEffort QoS when none of its containers has CPU or memory requests or limits configured.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-demo
spec:
  containers:
    - name: app
      image: nginx
```

If no applicable defaults inject resource values, this Pod receives BestEffort QoS.

A LimitRange may inject default requests and limits, so always check the actual Pod configuration.

## QoS Comparison

| QoS class | General configuration | Resource pressure |
|---|---|---|
| Guaranteed | Every container has matching CPU and memory requests/limits | Generally receives stronger eviction protection |
| Burstable | Resource configuration exists, but Guaranteed criteria are not met | Eviction behavior depends on usage relative to requests and other factors |
| BestEffort | No CPU or memory requests/limits on any container | Generally most vulnerable to eviction |

Do not assume Kubernetes always evicts Pods in a fixed QoS order. Actual eviction decisions depend on node pressure, usage relative to requests, and other Kubernetes rules.

## Check QoS Class

```bash
kubectl get pod <pod-name> \
  -o jsonpath='{.status.qosClass}'
```

Possible results:

```text
Guaranteed
Burstable
BestEffort
```

---

# 12. ResourceQuota

ResourceQuota limits aggregate resource consumption within a namespace.

Example:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: development-quota
  namespace: development
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
```

This defines namespace-wide limits for the specified resource accounting categories.

For example:

```text
Total CPU requests:    4 CPU
Total memory requests: 8Gi
Total CPU limits:      8 CPU
Total memory limits:   16Gi
```

These values apply to aggregate resource accounting in the namespace.

If creating a new workload would violate the applicable quota, Kubernetes generally rejects the API request.

A quota violation does not normally create a Pod that remains Pending because of the quota itself.

## Create a Namespace

```bash
kubectl create namespace development
```

## Apply ResourceQuota

Save the YAML as:

```text
resource-quota.yaml
```

Apply:

```bash
kubectl apply -f resource-quota.yaml
```

Inspect:

```bash
kubectl get resourcequota -n development
```

Detailed information:

```bash
kubectl describe resourcequota development-quota -n development
```

---

# 13. LimitRange

LimitRange defines default values and resource constraints for individual containers or Pods within a namespace.

It can configure:

- Default resource requests.
- Default resource limits.
- Minimum resource values.
- Maximum resource values.
- Maximum limit-to-request ratios.

Example:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: container-limits
  namespace: development
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
```

If a container does not specify resource values, Kubernetes can apply the configured defaults.

## Apply LimitRange

Save the file as:

```text
limit-range.yaml
```

Apply:

```bash
kubectl apply -f limit-range.yaml
```

Inspect:

```bash
kubectl get limitrange -n development
```

Detailed information:

```bash
kubectl describe limitrange container-limits -n development
```

## ResourceQuota vs LimitRange

| ResourceQuota | LimitRange |
|---|---|
| Controls aggregate resource consumption in a namespace | Defines defaults and constraints for individual containers or Pods |
| Helps control namespace-wide resource budgets | Helps enforce consistent workload configuration |
| Can limit total CPU and memory requests and limits | Can define default, minimum, and maximum values |
| May reject new workloads that exceed quota | May reject workloads that violate applicable constraints |

## Production Example

In a development namespace:

- ResourceQuota limits aggregate CPU and memory consumption.
- LimitRange supplies default resource values and prevents containers from specifying inappropriate resource values.

Using both helps establish predictable resource management.

---

# 14. Node Capacity vs Allocatable

Kubernetes nodes report Capacity and Allocatable resources.

Inspect a node:

```bash
kubectl describe node <node-name>
```

## Capacity

Capacity represents the total resources reported by the node.

Example:

```text
CPU:    4
Memory: 8Gi
```

## Allocatable

Allocatable represents the resources Kubernetes makes available for Pods after accounting for applicable system reservations and other resource reservations.

Example:

```text
Capacity:
CPU:    4
Memory: 8Gi

Allocatable:
CPU:    3.7
Memory: 7Gi
```

The scheduler uses allocatable resources and existing Pod resource requests when making scheduling decisions.

The difference between Capacity and Allocatable helps reserve resources for the operating system and system components.

## Important Distinction

The scheduler does not simply compare requests with current actual CPU and memory usage.

It primarily uses resource requests and allocatable resources for scheduling decisions.

Actual usage remains important for monitoring, performance analysis, and node-pressure troubleshooting.

---

# 15. Why Pods Remain Pending

Suppose a Pod requests:

```yaml
resources:
  requests:
    cpu: "2"
    memory: "4Gi"
```

But the node does not have enough allocatable resources available based on existing requests.

The Pod may remain Pending.

Possible scheduling problems include:

- Insufficient CPU.
- Insufficient memory.
- Node selectors that match no nodes.
- Unsatisfied node affinity.
- Untolerated taints.
- Topology constraints.
- Unavailable storage or volume topology constraints.
- Other scheduling constraints.

## Troubleshooting

Check Pods:

```bash
kubectl get pods -o wide
```

Inspect the Pod:

```bash
kubectl describe pod <pod-name>
```

Look at Events for messages such as:

```text
Insufficient cpu
Insufficient memory
```

Inspect nodes:

```bash
kubectl get nodes
```

```bash
kubectl describe node <node-name>
```

Inspect resource quotas:

```bash
kubectl get resourcequota -n <namespace>
```

Inspect storage if relevant:

```bash
kubectl get pvc -n <namespace>
```

## Important Distinction

A Pending Pod is not necessarily a resource problem.

Use Events to identify the actual cause.

---

# 16. Resource Requests and Actual Usage

Requests should reflect realistic workload requirements.

Suppose an application usually consumes:

```text
CPU:
150m to 350m

Memory:
300Mi to 450Mi
```

An initial configuration might be:

```yaml
resources:
  requests:
    cpu: "200m"
    memory: "384Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

These values are illustrative, not universal recommendations.

The correct values depend on:

- Application behavior.
- Traffic patterns.
- Historical metrics.
- Load testing.
- Startup behavior.
- Memory growth.
- Latency requirements.
- Cluster capacity.

## Over-requesting

If an application uses very little CPU and memory but requests excessive resources, it can lead to:

- Inefficient scheduling.
- Reduced cluster utilization.
- More nodes being required.
- Higher infrastructure costs.
- Pods remaining Pending unnecessarily.

## Under-requesting

If requests are much lower than realistic requirements, Kubernetes may schedule more workloads onto nodes than their actual resource demands can comfortably support.

Possible consequences include:

- Node resource pressure.
- Poor performance.
- Memory pressure.
- Evictions.
- OOM conditions.

Resource requests should be tuned using workload data, not arbitrary guesses.

---

# 17. Production Resource Planning

Consider a Deployment with ten replicas.

Each Pod requests:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
```

Aggregate container requests are approximately:

```text
CPU:
10 × 500m = 5 CPU

Memory:
10 × 512Mi = 5Gi
```

This calculation helps estimate workload resource requirements.

However, cluster planning must also consider:

- Node allocatable resources.
- Other workloads.
- System overhead.
- DaemonSets.
- Scheduling constraints.
- Rolling-update surge Pods.
- Failure scenarios.
- Autoscaling behavior.
- Storage and network requirements.

For example, if a Deployment temporarily creates additional Pods during a rolling update, the cluster may need capacity for more than the desired replica count.

---

# 18. HPA vs VPA

## Horizontal Pod Autoscaler (HPA)

HPA adjusts the number of replicas based on configured metrics.

Example:

```text
3 Pods
   |
   v
6 Pods
```

HPA can use CPU or memory utilization and other supported metrics, depending on configuration.

For CPU utilization-based scaling, resource requests are important because utilization is calculated relative to requests.

## Vertical Pod Autoscaler (VPA)

VPA helps recommend or adjust resource requests and, depending on its configuration and operating mode, limits.

Conceptually:

```text
CPU request:
500m → 1 CPU

Memory request:
512Mi → 1Gi
```

Exact behavior depends on the VPA configuration and update mode.

## Comparison

| HPA | VPA |
|---|---|
| Changes replica count | Recommends or adjusts per-Pod resources |
| Scales horizontally | Scales vertically |
| Adds or removes Pods | Adjusts CPU/memory resource configuration |
| Useful for workloads that can scale across replicas | Useful for workloads whose individual resource needs change |

HPA and VPA must be configured thoughtfully when both operate on the same resource metrics.

---

# 19. Useful Commands

## List Pods

```bash
kubectl get pods
```

## Get Detailed Pod Information

```bash
kubectl describe pod <pod-name>
```

## Check Current Pod Usage

```bash
kubectl top pod <pod-name>
```

## Check Container-Level Usage

```bash
kubectl top pod <pod-name> --containers
```

## Check Node Usage

```bash
kubectl top nodes
```

These metrics commands require Metrics Server or another supported metrics source.

## Inspect Pod YAML

```bash
kubectl get pod <pod-name> -o yaml
```

## Check QoS Class

```bash
kubectl get pod <pod-name> \
  -o jsonpath='{.status.qosClass}'
```

## Inspect Node Resources

```bash
kubectl describe node <node-name>
```

## Inspect ResourceQuota

```bash
kubectl get resourcequota -n <namespace>
```

```bash
kubectl describe resourcequota <quota-name> -n <namespace>
```

## Inspect LimitRange

```bash
kubectl get limitrange -n <namespace>
```

```bash
kubectl describe limitrange <limitrange-name> -n <namespace>
```

## Inspect Previous Container Logs

```bash
kubectl logs <pod-name> --previous
```

For multi-container Pods:

```bash
kubectl logs <pod-name> -c <container-name> --previous
```

---

# 20. Hands-On Lab 1: Requests and Limits

Create a file named:

```text
resource-demo.yaml
```

Add:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
    - name: app
      image: nginx
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
```

Apply:

```bash
kubectl apply -f resource-demo.yaml
```

Verify:

```bash
kubectl get pod resource-demo
```

Inspect:

```bash
kubectl describe pod resource-demo
```

Observe the configured requests and limits.

---

# 21. Hands-On Lab 2: Check QoS Classes

Check the previous Pod:

```bash
kubectl get pod resource-demo \
  -o jsonpath='{.status.qosClass}'
```

Expected:

```text
Burstable
```

The request values differ from the limits.

## Create a Guaranteed Pod

Create:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-demo
spec:
  containers:
    - name: app
      image: nginx
      resources:
        requests:
          cpu: "500m"
          memory: "256Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
```

Apply:

```bash
kubectl apply -f guaranteed-demo.yaml
```

Check:

```bash
kubectl get pod guaranteed-demo \
  -o jsonpath='{.status.qosClass}'
```

Expected:

```text
Guaranteed
```

## Create a BestEffort Pod

Create:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-demo
spec:
  containers:
    - name: app
      image: nginx
```

Apply:

```bash
kubectl apply -f besteffort-demo.yaml
```

Check:

```bash
kubectl get pod besteffort-demo \
  -o jsonpath='{.status.qosClass}'
```

Expected:

```text
BestEffort
```

These examples assume no applicable LimitRange defaults inject resource requests or limits.

---

# 22. Hands-On Lab 3: ResourceQuota

Create a namespace:

```bash
kubectl create namespace resource-demo
```

Create `quota.yaml`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: resource-quota
  namespace: resource-demo
spec:
  hard:
    requests.cpu: "1"
    requests.memory: "1Gi"
    limits.cpu: "2"
    limits.memory: "2Gi"
```

Apply:

```bash
kubectl apply -f quota.yaml
```

Inspect:

```bash
kubectl describe resourcequota resource-quota -n resource-demo
```

Observe the resource quota and current usage.

---

# 23. Hands-On Lab 4: LimitRange

Create `limitrange.yaml`:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: resource-demo
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
```

Apply:

```bash
kubectl apply -f limitrange.yaml
```

Inspect:

```bash
kubectl describe limitrange resource-limits -n resource-demo
```

Create a container without explicitly specifying resources in the same namespace.

Inspect the resulting Pod:

```bash
kubectl get pod <pod-name> -n resource-demo -o yaml
```

Observe whether the configured defaults have been applied.

---

# 24. Hands-On Lab 5: Troubleshoot a Pending Pod

Create a Pod with resource requests that may not fit your cluster:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pending-resource-demo
spec:
  containers:
    - name: app
      image: nginx
      resources:
        requests:
          cpu: "100"
          memory: "100Gi"
```

This example deliberately requests excessive resources for most clusters. It may remain Pending because no node can satisfy the request.

Apply:

```bash
kubectl apply -f pending-resource-demo.yaml
```

Inspect:

```bash
kubectl get pod pending-resource-demo
```

```bash
kubectl describe pod pending-resource-demo
```

Look at Events.

Then inspect:

```bash
kubectl get nodes
```

```bash
kubectl describe node <node-name>
```

Understand why the Pod cannot be scheduled.

Clean up afterward:

```bash
kubectl delete pod pending-resource-demo
```

**Important:** Do not use excessive resource requests in real production workloads. This is a controlled troubleshooting exercise.

---

# 25. Production Troubleshooting Scenarios

## Scenario 1: Pod Is Pending

Symptoms:

```text
Pod status: Pending
```

Investigation:

1. Run `kubectl describe pod`.
2. Inspect Events.
3. Check CPU and memory requests.
4. Check node allocatable resources.
5. Check taints, tolerations, affinity, and selectors.
6. Check ResourceQuota.
7. Check PVCs if storage is involved.

Do not assume every Pending Pod has insufficient CPU or memory.

## Scenario 2: Container Is OOMKilled

Symptoms:

```text
Reason: OOMKilled
```

Investigation:

1. Inspect Pod status and events.
2. Check container memory limits.
3. Check previous logs.
4. Examine historical memory usage.
5. Investigate memory leaks and traffic spikes.
6. Determine whether the issue is container-level or node-level memory pressure.
7. Adjust resources only after identifying the likely cause.

## Scenario 3: Application Is Slow

Possible causes:

- CPU throttling.
- Memory pressure.
- Database latency.
- Network latency.
- Application bottlenecks.
- Dependency failures.

Investigation:

1. Check application latency.
2. Check CPU and memory usage.
3. Inspect resource requests and limits.
4. Check CPU throttling metrics.
5. Review application and dependency metrics.
6. Determine whether scaling or resource adjustment is appropriate.

## Scenario 4: New Pods Cannot Be Scheduled

Investigation:

1. Inspect Pending Pods.
2. Check scheduling events.
3. Review node allocatable resources.
4. Calculate aggregate workload requests.
5. Check node selectors, affinity, taints, and topology constraints.
6. Check whether cluster autoscaling is configured and able to add suitable nodes.

## Scenario 5: Application Is Slow During Peak Traffic

Investigation:

1. Compare application latency before and during peak traffic.
2. Check CPU usage and throttling.
3. Check memory usage and OOMKilled events.
4. Check requests and limits.
5. Determine whether each Pod is overloaded.
6. Check database and other dependencies.
7. Consider HPA if horizontal scaling can help.
8. Verify that additional Pods can be scheduled.
9. Load-test and monitor the change.

---

# 26. Production Best Practices

## Resource Requests

- Set requests based on realistic workload requirements.
- Use historical metrics and load testing.
- Review requests when workload behavior changes.
- Avoid excessive requests that unnecessarily restrict scheduling.

## Resource Limits

- Configure limits according to application behavior and reliability requirements.
- Investigate CPU throttling before increasing CPU limits.
- Monitor memory usage and investigate OOMKilled events.
- Do not assume increasing limits alone will fix application problems.

## Namespace Management

- Use ResourceQuota to control aggregate namespace resource consumption.
- Use LimitRange to provide defaults and enforce appropriate constraints.
- Review quota usage and capacity regularly.

## Monitoring

Monitor:

- CPU usage.
- Memory usage.
- CPU throttling.
- Container restarts.
- OOMKilled events.
- Pod scheduling failures.
- Node memory pressure.
- Application latency.
- Request throughput.

## Capacity Planning

- Account for node allocatable resources.
- Consider scaling and rolling-update surge capacity.
- Plan for node failures.
- Verify that autoscaling can provide suitable capacity.
- Evaluate resource usage and costs over time.

---

# 27. Common Mistakes to Avoid

1. Assuming requests represent actual resource consumption.
2. Assuming CPU limit violations immediately kill containers.
3. Assuming every OOMKilled event is caused by a container memory limit.
4. Assuming every Pending Pod is a resource problem.
5. Confusing ResourceQuota with LimitRange.
6. Confusing node Capacity with Allocatable resources.
7. Increasing replicas without investigating the bottleneck.
8. Increasing resource limits without checking application metrics.
9. Assuming BestEffort Pods are always evicted first.
10. Treating current usage from `kubectl top` as historical monitoring data.
11. Forgetting that a LimitRange can inject default resource values.
12. Assuming that adding more Pods automatically solves every performance issue.

---

# 28. Day 8 Revision Summary

| Concept | Key takeaway |
|---|---|
| CPU request | Used for scheduling and resource accounting |
| Memory request | Used for scheduling and resource accounting |
| CPU limit | Constrains CPU consumption; throttling can occur |
| Memory limit | Constrains memory consumption; OOMKilled is possible |
| Scheduler | Evaluates requests and other scheduling constraints |
| Guaranteed QoS | Every container meets the matching CPU and memory request/limit criteria |
| Burstable QoS | Resource configuration exists, but Guaranteed criteria are not met |
| BestEffort QoS | No CPU or memory requests or limits are configured |
| ResourceQuota | Controls aggregate resource consumption in a namespace |
| LimitRange | Provides defaults and constraints for individual workloads |
| Capacity | Total resources reported by a node |
| Allocatable | Resources available for Kubernetes workloads |
| HPA | Adjusts the number of replicas |
| VPA | Helps recommend or adjust per-Pod resources |

## Final Mental Model

```text
Resource Management
        |
        +-- Requests
        |      |
        |      +-- Scheduling
        |      +-- Resource accounting
        |
        +-- Limits
        |      |
        |      +-- CPU throttling
        |      +-- Memory limit enforcement
        |
        +-- QoS Classes
        |      |
        |      +-- Guaranteed
        |      +-- Burstable
        |      +-- BestEffort
        |
        +-- ResourceQuota
        |      |
        |      +-- Namespace-wide budget
        |
        +-- LimitRange
        |      |
        |      +-- Defaults and constraints
        |
        +-- Monitoring
               |
               +-- CPU usage and throttling
               +-- Memory usage
               +-- OOMKilled events
               +-- Scheduling failures
```

## Day 8 Completion Checklist

- [ ] I can explain requests vs limits.
- [ ] I understand how the scheduler uses resource requests.
- [ ] I can explain CPU throttling.
- [ ] I can troubleshoot OOMKilled containers.
- [ ] I understand Guaranteed, Burstable, and BestEffort QoS.
- [ ] I understand ResourceQuota vs LimitRange.
- [ ] I understand Capacity vs Allocatable.
- [ ] I can investigate a Pending Pod.
- [ ] I can use `kubectl top`, `kubectl describe pod`, and `kubectl describe node`.
- [ ] I understand when to consider resource changes versus horizontal scaling.

---

## Key Takeaway

As a DevOps Engineer, you do not need to memorize every Linux kernel implementation detail to manage Kubernetes resources effectively.

You need to understand how resource configuration affects scheduling, application performance, memory failures, autoscaling, and cluster capacity.

**The production mindset is simple: investigate the evidence, identify the bottleneck, make an appropriate change, and verify the result.**
