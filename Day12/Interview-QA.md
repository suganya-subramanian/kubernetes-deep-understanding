# Kubernetes Day 12: Interview Questions and Answers

## Topic: Kubernetes Security

This document contains:

- The five Day 12 interview questions.
- The learner's evaluation scores.
- Corrections to the original answers.
- Improved interview-ready answers.
- Practical examples.
- Production-oriented explanations.

---

# 1. Overall Evaluation

| Question | Topic | Score |
|---|---|---:|
| Q1 | Authentication, Authorization, and Admission Control | 6.5/10 |
| Q2 | Kubernetes RBAC | 6.5/10 |
| Q3 | Pod Security Context | 7.5/10 |
| Q4 | Kubernetes Secrets | 6.5/10 |
| Q5 | Production Security Scenario | 7.5/10 |
| **Total** | | **34.5/50** |

**Overall Score:** 6.9/10

**Current Level:** Early Intermediate, progressing toward Intermediate.

## Strengths

- Understands the principle of least privilege.
- Understands the purpose of RBAC.
- Recognizes the importance of non-root containers.
- Understands that Base64 is not encryption.
- Recognizes the value of external secret managers.
- Can identify multiple security controls for production workloads.

## Areas to Improve

- Explain Kubernetes request processing precisely.
- Distinguish namespace-scoped and cluster-scoped RBAC permissions.
- Distinguish Kubernetes API permissions from container security.
- Explain Secret storage and encryption accurately.
- Connect each production security risk to the appropriate control.

---

# 2. Question 1: Authentication vs. Authorization vs. Admission Control

## Question

What is the difference between authentication, authorization, and admission control in Kubernetes? Explain how a request passes through these stages.

## Learner's Original Answer: Summary

The learner explained that:

- Authentication identifies who the requester is.
- Authorization determines what the requester can do.
- RBAC and least privilege restrict access.
- Admission control is another mechanism involved in controlling Kubernetes activities.

The learner understood the overall purpose of these mechanisms but did not clearly distinguish admission control from authentication and authorization.

## Evaluation

**Score: 6.5/10**

### What Was Correct?

- Authentication identifies the requester.
- Authorization determines the requester's permissions.
- RBAC is used to manage Kubernetes API permissions.
- Least privilege is an important security principle.

### What Needed Improvement?

Admission control is not simply another name for RBAC or authorization.

Admission controllers evaluate API requests after authentication and authorization and before the resulting objects are persisted.

They can validate requests, reject requests that violate policies, or modify requests according to configured rules.

## Improved Interview Answer

Authentication verifies the identity of the requester making a Kubernetes API request.

Authorization checks whether the authenticated identity is permitted to perform the requested operation. RBAC is a common authorization mechanism.

Admission control applies additional policies to API requests before they are persisted. It can accept, modify, or reject requests according to those policies.

For example, a developer might be authorized to create Pods in a namespace, but an admission policy can reject a Pod that attempts to run in privileged mode.

A simplified request flow is:

```text
Client
   |
   v
API Server
   |
   v
Authentication
   |
   v
Authorization
   |
   v
Admission Control
   |
   v
Persistence
```

This is a simplified view of Kubernetes API request processing.

## Practical Example

A developer attempts to create a Pod.

1. Authentication identifies the developer.
2. Authorization checks whether the developer can create Pods.
3. Admission control checks whether the Pod complies with applicable policies.
4. If the request passes the required checks, the resulting object can be persisted.

A request may be rejected at any applicable stage.

## Production Perspective

RBAC controls which identities can perform Kubernetes API operations.

Admission policies help enforce requirements on the requested resources.

Both mechanisms are important, but they address different aspects of security.

## Key Interview Point

**Authentication = Who are you?**

**Authorization = What are you allowed to do?**

**Admission control = Does this API request satisfy the applicable policies?**

---

# 3. Question 2: Kubernetes RBAC

## Question

What is the difference between Role, RoleBinding, ClusterRole, and ClusterRoleBinding?

Give a real-world example of granting a developer read-only access to Pods in a specific namespace.

## Learner's Original Answer: Summary

The learner explained that:

- Role defines permissions.
- RoleBinding associates a Role with an identity.
- ClusterRole defines reusable or cluster-level permissions.
- ClusterRoleBinding grants ClusterRole permissions.
- A developer can receive read-only Pod access in a specific namespace.

The learner also mentioned creating a ServiceAccount and assigning permissions to a workload.

## Evaluation

**Score: 6.5/10**

### What Was Correct?

- Correctly identified the purpose of Role and RoleBinding.
- Recognized that ClusterRole and ClusterRoleBinding can provide broader access.
- Understood that users, groups, and ServiceAccounts can be RBAC subjects.
- Understood that permissions should be restricted to the required namespace.

### What Needed Improvement?

A Role is not limited to Pods or containers.

A Role defines namespace-scoped permissions for applicable resources, including Pods, Deployments, Services, ConfigMaps, and Secrets.

A ClusterRole can define permissions for cluster-scoped resources and namespaced resources.

A RoleBinding can reference either a Role or a ClusterRole. When a ClusterRole is referenced through a RoleBinding, the permissions apply within the RoleBinding's namespace for namespaced resources.

A ClusterRoleBinding grants the referenced ClusterRole's permissions at the cluster level.

## Improved Interview Answer

Role-Based Access Control, or RBAC, manages which identities can perform specific operations on Kubernetes resources.

### Role

Defines permissions within a namespace.

### RoleBinding

Associates a Role or ClusterRole with users, groups, or ServiceAccounts within a namespace.

### ClusterRole

Defines reusable permissions for cluster-scoped resources, namespaced resources, or applicable non-resource URLs.

### ClusterRoleBinding

Associates a ClusterRole with identities at the cluster level.

## Practical Example: Read-Only Pod Access

Create a Role:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: development
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

Create a RoleBinding:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: development
subjects:
  - kind: User
    name: developer1
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Apply the files:

```bash
kubectl apply -f role.yaml
kubectl apply -f rolebinding.yaml
```

Verify read permissions:

```bash
kubectl auth can-i list pods \
  --as=developer1 \
  -n development
```

Expected result:

```text
yes
```

Verify that deletion is not permitted:

```bash
kubectl auth can-i delete pods \
  --as=developer1 \
  -n development
```

Expected result:

```text
no
```

These checks use impersonation and require the caller to have permission to impersonate the specified identity.

## Production Perspective

Use Role and RoleBinding when permissions should be limited to a namespace.

Use ClusterRole and ClusterRoleBinding when broader permissions are genuinely required.

Avoid granting cluster-wide administrative access to developers or applications that only need read-only access to selected resources.

## Key Interview Point

**Role defines permissions. RoleBinding grants those permissions within a namespace.**

**ClusterRole defines reusable or cluster-level permission rules. ClusterRoleBinding grants permissions at the cluster level.**

---

# 4. Question 3: Pod Security Context

## Question

How would you secure a Pod in production using securityContext?

Explain:

- runAsNonRoot
- allowPrivilegeEscalation
- readOnlyRootFilesystem
- capabilities.drop: ["ALL"]

## Learner's Original Answer: Summary

The learner explained that:

- `runAsNonRoot` prevents root execution.
- `allowPrivilegeEscalation: false` restricts privilege escalation.
- `readOnlyRootFilesystem` prevents modifications to the container's root filesystem.
- `capabilities.drop: ["ALL"]` removes Linux capabilities.

The learner correctly identified these settings as container security controls.

## Evaluation

**Score: 7.5/10**

### What Was Correct?

- Correctly identified the purpose of all four settings.
- Understood why non-root execution is important.
- Recognized the security benefit of dropping unnecessary capabilities.
- Understood that a read-only root filesystem limits modifications.

### What Needed Improvement?

These settings do not primarily manage Kubernetes API permissions.

RBAC controls what an identity can do through the Kubernetes API.

SecurityContext controls aspects of container and process execution.

The application image must also support the selected user ID and filesystem restrictions.

## Improved Interview Answer

I would use a securityContext to reduce unnecessary privileges in the container execution environment.

### 1. runAsNonRoot

```yaml
runAsNonRoot: true
```

Requires the container to run as a non-root user.

An appropriate user ID must be configured or provided by the image.

### 2. allowPrivilegeEscalation

```yaml
allowPrivilegeEscalation: false
```

Prevents privilege escalation through mechanisms such as set-user-ID or set-group-ID execution.

### 3. readOnlyRootFilesystem

```yaml
readOnlyRootFilesystem: true
```

Makes the container's root filesystem read-only.

Writable directories must be provided through appropriate volumes when required.

### 4. Drop Linux Capabilities

```yaml
capabilities:
  drop:
    - ALL
```

Drops the container's Linux capabilities.

Only capabilities that are genuinely necessary should be added back.

## Example Configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  automountServiceAccountToken: false
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: your-nonroot-application-image
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
```

The example image must support the configured user and read-only filesystem.

## Production Perspective

SecurityContext reduces the privileges available to container processes.

For a complete security approach, combine it with:

- Least-privilege RBAC.
- Appropriate ServiceAccount configuration.
- NetworkPolicies.
- Secret protection.
- Pod Security Admission.
- Image security practices.

## Key Interview Point

**RBAC secures Kubernetes API access. SecurityContext hardens container execution.**

Both are necessary because they address different security risks.

---

# 5. Question 4: Kubernetes Secrets

## Question

Are Kubernetes Secrets secure by default?

Explain how Secret data is stored, why Base64 encoding is not encryption, and how you would protect sensitive information in production.

## Learner's Original Answer: Summary

The learner explained that:

- Kubernetes Secret data may be Base64-encoded.
- Base64 is not encryption.
- Anyone who obtains the encoded data can decode it.
- External secret managers can help protect sensitive information.

## Evaluation

**Score: 6.5/10**

### What Was Correct?

- Correctly identified the difference between encoding and encryption.
- Recognized the risks of exposing encoded Secret data.
- Suggested external secret managers as a production security measure.

### What Needed Improvement?

The statement that Kubernetes Secrets are not secure in any way is too broad.

Kubernetes Secrets provide a mechanism for managing sensitive data, but creating a Secret does not automatically guarantee encryption at rest.

Depending on cluster configuration, Secret data may be stored in etcd without encryption at rest.

Security also depends on RBAC, access to etcd and backups, and the way applications consume the data.

## Improved Interview Answer

Kubernetes Secrets are designed to store sensitive information such as passwords, API tokens, and certificates.

Secret data is commonly represented using Base64 encoding, but Base64 is not encryption.

Anyone who obtains the encoded value can decode it.

To protect Secrets in production, I would:

1. Enable encryption at rest for Secret data.
2. Apply least-privilege RBAC.
3. Restrict access to etcd and backups.
4. Avoid committing secrets to source control.
5. Avoid exposing Secret values in logs.
6. Rotate credentials when appropriate.
7. Consider an external secret manager such as AWS Secrets Manager.

## Production Perspective

Secrets must be protected throughout their lifecycle.

This includes storage, access, delivery to applications, logging, backup, and rotation.

An external secret manager can improve centralized management, but the Kubernetes integration and workload permissions must also be secured.

## Key Interview Point

**Base64 encoding is not encryption. Kubernetes Secret protection requires appropriate storage encryption, access controls, and secure handling.**

---

# 6. Question 5: Production Security Scenario

## Question

A developer deploys an application Pod that runs as root, has privileged access, and can communicate with every Pod in the cluster.

As a DevOps Engineer, how would you reduce these security risks?

## Learner's Original Answer: Summary

The learner identified that:

- Running as root can create unnecessary risk.
- Privileged access should be restricted.
- RBAC should follow least privilege.
- The application requirements should be assessed.
- SecurityContext, external secret management, and network security can help protect the workload.

## Evaluation

**Score: 7.5/10**

### What Was Correct?

- Identified the need to assess application requirements.
- Recognized that unnecessary privileges should be removed.
- Included RBAC and container security.
- Recognized the importance of network restrictions.
- Demonstrated a layered-security mindset.

### What Needed Improvement?

Each risk needs an appropriate control.

RBAC does not directly prevent a container from running as root.

SecurityContext does not replace RBAC.

NetworkPolicies restrict network communication but do not control Kubernetes API permissions.

These controls must be applied together according to the risks involved.

## Improved Interview Answer

As a DevOps Engineer, I would first assess the application's requirements and identify unnecessary privileges.

Then I would apply the following controls.

### 1. Container Security

Use an appropriate non-root user and disable unnecessary privileges.

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  seccompProfile:
    type: RuntimeDefault
```

At the container level:

```yaml
securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
```

I would also avoid privileged mode unless a justified infrastructure requirement exists.

The application image must support the chosen configuration.

### 2. RBAC

Use least-privilege permissions.

Prefer namespace-scoped Roles when they meet the application's requirements.

Avoid unnecessary ClusterRoleBindings and broad access to Secrets.

### 3. ServiceAccount

Assign a dedicated ServiceAccount to the workload.

Disable automatic token mounting if the application does not need Kubernetes API access.

### 4. NetworkPolicy

Restrict ingress and egress traffic to the communication paths the application requires.

Verify that the cluster's networking implementation supports and enforces NetworkPolicies.

### 5. Pod Security Admission

Use an appropriate Pod Security Standard, such as Restricted, where compatible with the application.

Test enforcement before applying it to production workloads.

### 6. Secret Management

Restrict access to sensitive data and use an external secret manager when appropriate.

### 7. Validation

After applying the changes, test the application to verify that it works with the reduced privileges.

Monitor the workload and investigate any denied operations or unexpected failures.

## Production Perspective

The goal is not simply to disable every permission.

The goal is to identify what the application actually needs and grant only those permissions.

A workload may require a specific Linux capability or writable directory. In that case, provide the narrowest suitable exception instead of granting broad privileges.

## Key Interview Point

**Secure production workloads using multiple layers: container hardening, least-privilege RBAC, ServiceAccount restrictions, NetworkPolicies, admission policies, and secure Secret management.**

---

# 7. Important Security Concepts to Remember

| Concept | Purpose |
|---|---|
| Authentication | Identifies the requester |
| Authorization | Determines whether an action is permitted |
| Admission Control | Applies additional policies to API requests |
| Role | Defines namespace-scoped permissions |
| RoleBinding | Grants permissions within a namespace |
| ClusterRole | Defines reusable or cluster-level permission rules |
| ClusterRoleBinding | Grants permissions at the cluster level |
| ServiceAccount | Provides an identity for a workload |
| SecurityContext | Configures aspects of container execution security |
| Kubernetes Secret | Stores sensitive configuration data |
| Encryption at Rest | Protects stored data against unauthorized disclosure |
| NetworkPolicy | Restricts selected Pod network traffic |
| Pod Security Standards | Define workload security requirements |
| Pod Security Admission | Enforces Pod Security Standards at the namespace level |

---

# 8. Day 12 Revision Checklist

Before an interview, make sure you can explain these concepts without reading the notes.

- [ ] Authentication vs. authorization vs. admission control.
- [ ] Role vs. RoleBinding.
- [ ] ClusterRole vs. ClusterRoleBinding.
- [ ] Namespace-scoped vs. cluster-level permissions.
- [ ] `kubectl auth can-i`.
- [ ] ServiceAccounts and automatic token mounting.
- [ ] `runAsNonRoot`.
- [ ] `allowPrivilegeEscalation`.
- [ ] `readOnlyRootFilesystem`.
- [ ] Linux capabilities.
- [ ] Seccomp `RuntimeDefault`.
- [ ] Kubernetes Secrets and Base64.
- [ ] Encryption at rest.
- [ ] External secret managers.
- [ ] NetworkPolicies.
- [ ] Pod Security Standards.
- [ ] Pod Security Admission.
- [ ] Production defense in depth.

---

# 9. Final Assessment

**Topic:** Kubernetes Security

**Score:** 34.5/50

**Percentage:** 69%

**Current Level:** Early Intermediate, progressing toward Intermediate.

## Strengths

The learner understands core Kubernetes security concepts and can suggest appropriate controls for production workloads.

The answers show a good foundation in least privilege, RBAC, container security, Secret management, and network restrictions.

## Improvement Priorities

1. Explain Kubernetes security mechanisms precisely.
2. Understand the scope of RBAC permissions.
3. Distinguish Kubernetes API access from container execution privileges.
4. Explain Secret storage and encryption accurately.
5. Map each production security risk to the appropriate control.

## Recommended Practical Exercises

Before moving to Day 13, complete these exercises in a test cluster:

### Exercise 1: RBAC

Create a Role and RoleBinding that grant read-only Pod access.

Verify allowed and denied actions using:

```bash
kubectl auth can-i list pods \
  --as=developer1 \
  -n development

kubectl auth can-i delete pods \
  --as=developer1 \
  -n development
```

### Exercise 2: Pod Security

Deploy a non-root Pod with privilege escalation disabled and unnecessary Linux capabilities dropped.

Verify that the workload operates correctly.

### Exercise 3: NetworkPolicy

Create a NetworkPolicy that allows traffic from frontend Pods to backend Pods on a specific port.

Test permitted and denied connections.

Verify that the cluster's network plugin enforces the policy.

---

# 10. Next Learning Topic

**Day 13: Kubernetes Troubleshooting**

Planned areas include:

- Pod status and failure diagnosis.
- Events and application logs.
- ImagePullBackOff and CrashLoopBackOff.
- Readiness, liveness, and startup probes.
- Service and networking problems.
- Resource constraints and OOMKilled.
- Systematic root-cause analysis.
- Production troubleshooting workflows.

**Day 12 Status: Completed**
