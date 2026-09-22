# Kubernetes Day 6 — Storage

## 1. Why Kubernetes Needs Persistent Storage

Container filesystems are generally ephemeral.

When a container or Pod is replaced, data stored only inside the container filesystem can be lost.

Stateful applications such as MySQL, PostgreSQL, MongoDB, and Elasticsearch need persistent storage so that data survives Pod or container replacement.

### Basic Storage Model

```text
Pod
 ↓
PVC
 ↓
PV
 ↓
Actual Storage
