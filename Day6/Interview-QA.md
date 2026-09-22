# Kubernetes Day 6 – Storage

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
2. PersistentVolume (PV)

A PersistentVolume (PV) is a Kubernetes resource that represents persistent storage available to the cluster.

A PV can be backed by different storage systems, such as:

AWS EBS
AWS EFS
NFS
Local storage
Other CSI-backed storage systems
Important Point

A PV is the Kubernetes representation of a persistent storage resource.

The actual data ultimately resides in the underlying storage backend.

3. PersistentVolumeClaim (PVC)

A PersistentVolumeClaim (PVC) is a request for storage.

A PVC can specify requirements such as:

Storage capacity
Access mode
StorageClass

Example:

resources:
  requests:
    storage: 10Gi
Simple Understanding
PVC = "I need this type and amount of storage."

PV = "Here is storage that satisfies your request."

The PVC is bound to the PV.

The Pod references the PVC and uses the storage through it.

4. Pod, PVC and PV Relationship

A Pod normally does not directly request a PV.

The Pod references a PVC.

Pod
 ↓
PVC
 ↓
PV
 ↓
Storage Backend

Example:

volumes:
  - name: app-storage
    persistentVolumeClaim:
      claimName: my-pvc

The application accesses the mounted storage through the Pod.

5. PersistentVolume Lifecycle

A PV can move through different states.

Available

The PV exists but is not currently bound to a PVC.

Bound

The PV is successfully bound to a PVC.

Released

The PVC has been deleted, but the PV still requires reclamation according to its reclaim policy.

Reclaimed

The storage is handled according to the configured reclaim policy and storage implementation.

Useful commands:

kubectl get pv
kubectl describe pv <pv-name>
6. StorageClass

A StorageClass defines how storage should be dynamically provisioned.

It can define:

Provisioner
Parameters
Reclaim policy
Volume binding mode
Volume expansion support

Example:

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: ebs.csi.aws.com

The provisioner depends on the storage platform.

For example, the AWS EBS CSI driver can dynamically provision EBS-backed storage.

7. Static vs Dynamic Provisioning
Static Provisioning

In static provisioning, the administrator creates the PV before the application requests storage.

Administrator
     ↓
PV
     ↓
PVC
     ↓
Pod

The PVC binds to an existing suitable PV.

Dynamic Provisioning

In dynamic provisioning, the PVC requests storage through a StorageClass.

PVC
 ↓
StorageClass
 ↓
CSI Provisioner
 ↓
PV
 ↓
Actual Storage
 ↓
Pod

The provisioner automatically creates the required storage resource.

Advantage

Dynamic provisioning avoids manually creating every PV.

8. CSI — Container Storage Interface

CSI (Container Storage Interface) is the standard interface used to integrate Kubernetes with external storage systems.

CSI drivers allow Kubernetes to provision, attach, mount, and manage storage from different storage platforms.

Examples:

AWS EBS CSI Driver
AWS EFS CSI Driver
NFS CSI drivers
Other vendor or community CSI drivers
Production Model
Kubernetes
    ↓
CSI Driver
    ↓
Storage Backend

CSI is an important part of modern Kubernetes storage architecture.

9. Access Modes

Access modes define how a volume can be mounted.

Access Mode	Meaning
RWO	Read-write by a single node
ROX	Read-only by multiple nodes
RWX	Read-write by multiple nodes
RWO — ReadWriteOnce

RWO means the volume can be mounted for read-write access by a single node.

Important Interview Point

RWO does not necessarily mean only one Pod.

For example:

Node 1
 ├── Pod A
 ├── Pod B
 └── Pod C
       ↓
    RWO Volume

Multiple Pods on the same node may be able to use the volume, depending on the storage implementation and mount behavior.

Therefore:

RWO means single-node access, not necessarily single-Pod access.

ROX — ReadOnlyMany

The volume can be mounted as read-only by multiple nodes.

Node 1 ──┐
Node 2 ──┼──> Read-only Volume
Node 3 ──┘
RWX — ReadWriteMany

The volume can be mounted for read-write access by multiple nodes.

Node 1 ──┐
Node 2 ──┼──> Read-write Volume
Node 3 ──┘
Important

Not every storage backend supports every access mode.

The actual capabilities depend on the storage system and CSI driver.

10. Reclaim Policy

The reclaim policy controls what happens to a PV or dynamically provisioned storage when the PVC is deleted.

Common policies are:

Delete
Retain
Delete

With Delete, the dynamically provisioned storage may be deleted when the PVC is deleted, depending on the storage implementation.

This can be useful for disposable or automatically managed storage.

Retain

With Retain, the PV/storage is preserved after the PVC is deleted and requires manual handling.

This can be useful when protecting important data from accidental PVC deletion.

Important

Retain is not a backup.

A production database should still have a separate backup and restore strategy.

Database
   ↓
Persistent Storage
   ↓
Backup System
11. Volume Expansion

A StorageClass can support volume expansion.

Example:

allowVolumeExpansion: true

A PVC can then request additional storage.

Example:

resources:
  requests:
    storage: 20Gi
Important

Volume expansion requires support from the storage backend and CSI driver.

Always verify that the storage implementation supports online or offline expansion as required by the workload.

12. Stateful Applications

Examples of stateful applications:

MySQL
PostgreSQL
MongoDB
Elasticsearch
Kafka

These applications need persistent storage.

Kubernetes StatefulSets can use volumeClaimTemplates to create persistent storage claims for their replicas.

Conceptually:

StatefulSet
    ↓
Pod
    ↓
PVC
    ↓
PV
    ↓
Persistent Storage

Each StatefulSet replica can have its own PVC.

13. Storage and Pod Scheduling

Storage can affect Pod scheduling.

For example, cloud storage may have availability-zone or topology constraints.

Kubernetes may need to consider:

Node
Availability Zone
Volume topology
Storage binding mode
Storage availability

Example:

Pod
 ↓
PVC
 ↓
Volume
 ↓
Availability Zone
 ↓
Compatible Node

A volume being available does not necessarily mean it can be mounted on every node.

14. Local vs Network Storage
Local Storage

The storage is physically associated with a particular node.

Advantages
Low latency
Local access
Challenges
Node dependency
Scheduling constraints
Recovery complexity
Network / Remote Storage

Storage is provided through a network-accessible storage system.

Examples:

AWS EBS
AWS EFS
NFS
Other CSI-backed storage

The appropriate storage type depends on the application's requirements.

15. Storage Performance

When designing production storage, consider:

IOPS

Input/Output Operations Per Second.

It indicates how many input/output operations the storage can handle.

Throughput

The amount of data that can be transferred over time.

Usually measured in MB/s or GB/s.

Latency

The time required to complete a storage operation.

Simple Comparison
IOPS       → How many operations?
Throughput → How much data?
Latency    → How quickly?
16. PVC Troubleshooting

If a PVC is stuck in Pending, start with:

kubectl get pvc
kubectl describe pvc <pvc-name>

Then check:

kubectl get storageclass
kubectl get pv

Possible causes include:

Incorrect StorageClass
No suitable PV
Insufficient storage capacity
Unsupported access mode
CSI provisioner problem
CSI driver problem
Storage backend problem
Topology constraints
17. PVC Bound but Pod Cannot Mount the Volume

A PVC being Bound does not guarantee that the Pod can successfully attach and mount the volume.

Start with:

kubectl describe pod <pod-name>

Look at the Pod events.

Common errors may include:

FailedMount
FailedAttachVolume

Then investigate layer by layer:

Pod
 ↓
PVC
 ↓
PV
 ↓
StorageClass
 ↓
CSI Driver
 ↓
Node / Topology
 ↓
Storage Backend

Useful commands:

kubectl get pods -A

kubectl get pvc
kubectl describe pvc <pvc-name>

kubectl get pv
kubectl describe pv <pv-name>

kubectl get storageclass

kubectl describe pod <pod-name>
18. Production Storage Troubleshooting

A good troubleshooting approach is:

Step 1 — Check the Pod
kubectl get pod
kubectl describe pod <pod-name>

Check events such as:

FailedMount
FailedAttachVolume
Step 2 — Check the PVC
kubectl get pvc
kubectl describe pvc <pvc-name>

Verify:

Status
StorageClass
Capacity
Access mode
Step 3 — Check the PV
kubectl get pv
kubectl describe pv <pv-name>

Verify:

Status
Capacity
Access mode
Reclaim policy
Claim reference
Step 4 — Check StorageClass
kubectl get storageclass
kubectl describe storageclass <storageclass-name>

Verify:

Provisioner
Parameters
Binding mode
Expansion support
Step 5 — Check CSI

Check whether the CSI driver components are running:

kubectl get pods -A

Then investigate relevant CSI controller and node components.

Step 6 — Check Node and Topology

Verify:

Node availability
Availability Zone
Volume topology
Attachment restrictions
Step 7 — Check Storage Backend

Finally, verify the underlying storage system.

19. Storage and Backup

Persistent storage and backup are different concepts.

Persistent Storage

Keeps application data available during normal Pod/container lifecycle changes.

Backup

Provides a separate recoverable copy of data.

Application
     ↓
Persistent Storage
     ↓
Backup System

A production database should have:

Regular backups
Appropriate retention
Tested restore procedures
Recovery objectives
20. Production Storage Design Considerations

Before selecting a storage solution, consider:

Capacity

How much storage is required?

Performance

Consider:

IOPS
Throughput
Latency
Availability

Can the application tolerate storage or node failures?

Access Mode

Does the workload need:

RWO?
ROX?
RWX?
Backup

How will the data be backed up?

Recovery

How quickly can the data be restored?

Expansion

Can storage capacity grow safely?

Security

Consider:

Encryption
RBAC
Access control
Network security
Topology

Consider:

Availability Zones
Node placement
Storage locality
21. Hands-On Labs
Lab 1 — Create a PVC

Create a PVC and verify:

kubectl apply -f pvc.yaml
kubectl get pvc
kubectl describe pvc <pvc-name>
Lab 2 — Create a Pod Using PVC

Create a Pod that mounts the PVC.

Verify:

kubectl get pod
kubectl describe pod <pod-name>
Lab 3 — Write Data

Enter the Pod:

kubectl exec -it <pod-name> -- /bin/sh

Create a file inside the mounted directory.

Lab 4 — Delete and Recreate the Pod

Delete the Pod:

kubectl delete pod <pod-name>

Recreate the Pod and verify whether the data remains.

This demonstrates:

Pod Lifecycle
      ≠
Storage Lifecycle
22. Important Commands
kubectl get pv
kubectl get pvc
kubectl get storageclass

kubectl describe pv <pv-name>
kubectl describe pvc <pvc-name>
kubectl describe storageclass <storageclass-name>

kubectl get pods
kubectl describe pod <pod-name>
23. Key Mental Models
PV
PV = Persistent storage resource
PVC
PVC = Request for storage
StorageClass
StorageClass = Dynamic provisioning definition
CSI
CSI = Standard storage integration interface
Access Modes
RWO = Single node read/write
ROX = Multiple nodes read-only
RWX = Multiple nodes read/write
Reclaim Policy
Delete = Storage may be deleted
Retain = Preserve for manual recovery
Main Storage Flow
Pod
 ↓
PVC
 ↓
PV
 ↓
Actual Storage
Dynamic Provisioning Flow
Pod
 ↓
PVC
 ↓
StorageClass
 ↓
CSI Provisioner
 ↓
PV
 ↓
Storage Backend
Troubleshooting Flow
Pod
 ↓
PVC
 ↓
PV
 ↓
StorageClass
 ↓
CSI
 ↓
Storage Backend
 ↓
Node / Topology / Mount
24. Final Day 6 Summary

The most important concepts to remember:

Container filesystems are generally ephemeral.
PV represents persistent storage in Kubernetes.
PVC is a request for storage.
Pods normally use storage through PVCs.
StorageClass defines how storage is dynamically provisioned.
CSI integrates Kubernetes with external storage systems.
RWO means single-node access, not necessarily single-Pod access.
RWX depends on backend support.
PVC Bound does not guarantee successful mounting.
Reclaim policy controls storage lifecycle after claim deletion.
Retain is not a backup strategy.
Storage can affect Pod scheduling through topology constraints.
Production storage requires capacity, performance, availability, backup, recovery, security, and expansion planning.
Final Mental Model
                  Kubernetes

Pod
 │
 │ references
 ▼
PVC
 │
 │ binds to
 ▼
PV
 │
 │ represents
 ▼
Actual Storage
 │
 ├── EBS
 ├── EFS
 ├── NFS
 ├── Local Storage
 └── Other CSI-backed Storage

Dynamic provisioning:

PVC
 │
 ▼
StorageClass
 │
 ▼
CSI Provisioner
 │
 ▼
PV
 │
 ▼
Actual Storage
