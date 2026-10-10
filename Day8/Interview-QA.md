# Kubernetes Day 8: Interview Questions and Answers

## Topic: Kubernetes Resource Management

This document contains five interview questions with corrected answers, practical examples, and production troubleshooting guidance.

---

# Q1. What is the difference between CPU and memory requests and limits? How does the scheduler use requests?

## Answer

In Kubernetes, requests and limits control resource allocation and consumption for containers.

**Requests** specify the resources a container requests for scheduling.

**Limits** define the maximum resource usage configured for a container.

Example:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "1"
    memory: "1Gi"
```

In this example:

- CPU request: 500m.
- Memory request: 512Mi.
- CPU limit: 1 CPU.
- Memory limit: 1Gi.

The Kubernetes scheduler uses the requests when determining whether a node can accommodate a Pod.

It also considers scheduling constraints such as node affinity, node selectors, taints, tolerations, and topology constraints.

When a container reaches its CPU limit, CPU throttling can occur.

When a container exceeds its memory limit, it may be terminated with an OOMKilled reason.

Requests do not represent the application's continuously consumed resources. They are primarily used for scheduling and resource accounting.

## Example

Suppose a node has 4 CPUs of allocatable resources, and existing Pods have CPU requests totaling 3 CPUs.

A new Pod requests 2 CPUs.

The total requested CPU would become:

```text
3 + 2 = 5 CPUs
```

The new Pod cannot fit on that node based on CPU requests.

The scheduler must find another suitable node, or the Pod may remain Pending.

## Key Points

- Requests influence scheduling.
- Limits constrain resource usage.
- CPU limit enforcement can cause throttling.
- Memory limit enforcement can result in OOMKilled.
- Scheduling considers allocatable resources and multiple scheduling constraints.

---

# Q2. An application is running slowly, and another container repeatedly shows OOMKilled. What do these symptoms mean, and how would you troubleshoot them?

## Answer

CPU throttling is one possible cause of slow application performance.

It occurs when a container is prevented from consuming more CPU because it has reached its configured CPU limit.

For example:

```yaml
resources:
  limits:
    cpu: "500m"
```

If the application needs more CPU than this limit allows, it can experience throttling and increased latency.

However, slow performance can also result from database latency, network problems, application bottlenecks, or other dependencies.

OOMKilled indicates that a container was terminated due to an out-of-memory condition.

This may happen when the container exceeds its configured memory limit. Node-level out-of-memory conditions can also cause containers to be killed.

## Troubleshooting CPU Throttling

First, inspect the Pod:

```bash
kubectl describe pod <pod-name>
```

Check current resource usage:

```bash
kubectl top pod <pod-name>
```

Then inspect:

- CPU requests and limits.
- CPU throttling metrics.
- Application latency.
- Traffic patterns.
- Historical monitoring data.

Prometheus and Grafana can help identify CPU throttling and performance patterns.

If the CPU limit is unnecessarily restrictive, I would evaluate whether increasing it is appropriate.

## Troubleshooting OOMKilled

Inspect the Pod:

```bash
kubectl describe pod <pod-name>
```

Check previous logs:

```bash
kubectl logs <pod-name> --previous
```

Inspect resource configuration:

```bash
kubectl get pod <pod-name> -o yaml
```

Check current memory usage:

```bash
kubectl top pod <pod-name>
```

Then investigate:

- Memory limits.
- Historical memory usage.
- Possible memory leaks.
- Traffic spikes.
- Application behavior.
- Node-level memory pressure.

Based on the evidence, I would decide whether to adjust resources, fix an application issue, or investigate node capacity.

## Key Points

- CPU throttling can cause slow application performance.
- OOMKilled indicates an out-of-memory termination.
- `kubectl top` provides current usage, not historical data.
- Monitoring metrics and Pod events help identify the cause.
- Increasing limits should be based on evidence.

---

# Q3. A Pod requests 2 CPUs and 4Gi memory but remains Pending. What could be the reasons, and how would you investigate?

## Answer

A Pod may remain Pending when Kubernetes cannot find a suitable node or when another scheduling requirement cannot be satisfied.

Insufficient CPU or memory is one possible cause.

Other causes include:

- Node selectors that match no nodes.
- Unsatisfied node affinity.
- Untolerated taints.
- Topology constraints.
- Storage or volume topology issues.
- Other scheduling constraints.

## Step 1: Check Pod Status

```bash
kubectl get pods -o wide
```

## Step 2: Inspect Pod Events

```bash
kubectl describe pod <pod-name>
```

I would examine the Events section for messages such as:

```text
Insufficient cpu
Insufficient memory
```

These messages indicate that the scheduler cannot find a suitable node with sufficient resources for the Pod.

## Step 3: Check Node Resources

```bash
kubectl get nodes
```

```bash
kubectl describe node <node-name>
```

I would inspect:

- Capacity.
- Allocatable resources.
- Allocated resources.
- Existing resource requests.
- Node conditions.
- Scheduling constraints.

## Step 4: Check Namespace Configuration

```bash
kubectl get resourcequota -n <namespace>
```

```bash
kubectl get limitrange -n <namespace>
```

ResourceQuota can reject workload creation when a quota would be exceeded. LimitRange can apply defaults and enforce resource constraints.

These are important checks, although a quota violation itself normally causes the API request to be rejected rather than leaving a created Pod Pending.

## Step 5: Check Storage if Relevant

```bash
kubectl get pvc -n <namespace>
```

If the Pod uses persistent storage, I would check whether its PVC is Bound and whether storage topology or mounting constraints affect scheduling.

## Resolution

Depending on the root cause, I might:

- Adjust resource requests based on actual requirements.
- Add suitable cluster capacity.
- Configure cluster autoscaling.
- Correct node selectors or affinity rules.
- Add appropriate tolerations.
- Resolve storage issues.

I would not immediately reduce resource requests without confirming the cause.

## Key Points

Start with:

```bash
kubectl describe pod <pod-name>
```

**Pod Events are usually the best starting point for diagnosing scheduling failures.**

---

# Q4. What is the difference between ResourceQuota and LimitRange? Give a production example.

## Answer

ResourceQuota and LimitRange help manage resources at the namespace level, but they serve different purposes.

## ResourceQuota

ResourceQuota controls aggregate resource consumption within a namespace.

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

This defines limits for the aggregate CPU and memory requests and limits accounted for in the namespace.

If a new workload would exceed an applicable quota, Kubernetes generally rejects the request.

## LimitRange

LimitRange defines defaults and constraints for individual containers or Pods.

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

This provides default CPU and memory values for containers that do not specify their own values.

LimitRange can also enforce minimum and maximum resource values and maximum limit-to-request ratios.

## Production Example

Suppose a company has a development namespace shared by multiple developers.

I would use ResourceQuota to control the namespace's total resource consumption.

I would use LimitRange to provide sensible default resource values and prevent individual containers from requesting inappropriate amounts.

Together, these mechanisms help manage resource consumption and standardize workload configuration.

## Comparison

| ResourceQuota | LimitRange |
|---|---|
| Controls aggregate namespace consumption | Controls defaults and constraints for individual workloads |
| Defines namespace-wide resource budgets | Defines individual container/Pod defaults and limits |
| Helps prevent namespace overconsumption | Helps establish consistent resource configuration |

---

# Q5. A Deployment has three replicas. During peak traffic, the application becomes slow and memory usage increases significantly. How would you investigate and resolve this?

## Answer

As a DevOps Engineer, I would investigate the problem before deciding whether to change resource requests, limits, or the number of replicas.

## Step 1: Check Application Performance

I would use Prometheus and Grafana, or another monitoring system, to inspect:

- Application latency.
- Request throughput.
- CPU usage.
- CPU throttling.
- Memory usage.
- Container restarts.
- OOMKilled events.
- Database and dependency performance.

## Step 2: Inspect Pod Configuration

```bash
kubectl describe pod <pod-name>
```

```bash
kubectl get pod <pod-name> -o yaml
```

I would review the resource requests and limits and inspect Pod events.

## Step 3: Investigate CPU Usage

If the application is experiencing CPU throttling, I would review its CPU limit and actual workload requirements.

If the limit is too restrictive, I would consider increasing it, provided the node has sufficient capacity and the change is supported by metrics.

## Step 4: Investigate Memory Usage

If memory usage approaches the configured limit, I would investigate:

- Whether the limit is too low.
- Whether the application has a memory leak.
- Whether traffic spikes increase memory requirements.
- Whether the workload has changed.
- Whether the node is experiencing memory pressure.

I would increase the memory limit only when the evidence supports doing so.

## Step 5: Consider Horizontal Scaling

If the application is overloaded and can distribute traffic across multiple replicas, I would consider increasing the replica count or configuring HPA.

For example:

```text
Before:
3 replicas

After:
6 replicas
```

However, additional replicas require sufficient cluster capacity and may increase pressure on shared dependencies such as a database.

I would investigate those dependencies before scaling.

## Step 6: Verify the Solution

After making a change, I would monitor application latency, resource consumption, errors, and restart counts.

I would also perform load testing where appropriate to validate the solution.

## Key Points

- Diagnose before changing resources.
- Increasing requests affects scheduling and resource accounting.
- Increasing limits allows more resource consumption, subject to available capacity and enforcement.
- Increasing replicas distributes workload when the application can scale horizontally.
- HPA can automate replica scaling based on configured metrics.
- More replicas do not automatically solve database or other dependency bottlenecks.

---

# Day 8: Quick Revision

| Concept | Remember |
|---|---|
| Requests | Used for scheduling and resource accounting |
| Limits | Constrain container resource consumption |
| CPU throttling | CPU consumption is restricted by CPU limit enforcement |
| OOMKilled | Container was terminated due to an out-of-memory condition |
| Guaranteed QoS | All containers meet the matching CPU and memory request/limit criteria |
| Burstable QoS | Resource configuration exists, but Guaranteed criteria are not met |
| BestEffort QoS | No CPU or memory requests/limits are configured |
| ResourceQuota | Namespace-wide aggregate resource budget |
| LimitRange | Individual workload defaults and constraints |
| Capacity | Total resources reported by a node |
| Allocatable | Resources available for Kubernetes workloads |
| HPA | Adjusts the number of replicas |
| VPA | Helps recommend or adjust per-Pod resource settings |

## Production Troubleshooting Sequence

```text
Identify the symptom
        |
        v
Check Pod status and events
        |
        v
Inspect resource configuration
        |
        v
Review current and historical metrics
        |
        v
Identify the likely root cause
        |
        v
Choose an appropriate solution
        |
        v
Monitor and validate the result
```

## Final Interview Tip

When answering a Kubernetes troubleshooting question, explain:

1. What the symptom means.
2. Which commands and metrics you would inspect.
3. How you would identify the root cause.
4. What change you would make.
5. How you would verify that the change worked.

This demonstrates practical DevOps troubleshooting ability rather than memorized definitions.
