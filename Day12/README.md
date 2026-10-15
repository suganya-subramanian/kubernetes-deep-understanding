# Kubernetes Day 12: Kubernetes Security

## 1. Learning Objectives

By the end of this lesson, you should be able to:

- Understand Kubernetes security fundamentals.
- Differentiate authentication, authorization, and admission control.
- Understand Role-Based Access Control (RBAC).
- Explain Role, RoleBinding, ClusterRole, and ClusterRoleBinding.
- Create and manage Kubernetes ServiceAccounts.
- Secure Pods and containers using securityContext.
- Understand Kubernetes Secrets and their limitations.
- Restrict network communication using NetworkPolicies.
- Understand Pod Security Standards and Pod Security Admission.
- Apply security best practices in production Kubernetes environments.
- Troubleshoot common Kubernetes security problems.

---

# 2. Kubernetes Security Fundamentals

Kubernetes security involves protecting the cluster, its API, workloads, network communication, and sensitive information.

The goal is to prevent unauthorized access, reduce the impact of compromised workloads, and ensure that applications operate within their intended permissions.

## 2.1 Important Security Principles

### Least Privilege

Give users, applications, and workloads only the permissions they actually need.

For example, an application that only needs to read configuration should not receive permission to delete Pods or modify Deployments.

### Defense in Depth

Use multiple security controls rather than relying on a single mechanism.

Examples:

- RBAC
- Pod security settings
- NetworkPolicies
- Secret encryption
- Image security scanning
- Admission policies
- Monitoring and auditing

### Secure Defaults

Prefer restrictive configurations unless the application requires additional permissions.

Examples:

- Run containers as non-root.
- Disable privilege escalation.
- Drop unnecessary Linux capabilities.
- Avoid privileged containers.
- Disable automatic ServiceAccount token mounting when API access is unnecessary.

### Minimize the Attack Surface

Remove unnecessary privileges, packages, services, and network access.

The fewer unnecessary capabilities a workload has, the fewer opportunities an attacker has to exploit it.

---

# 3. Kubernetes API Request Security

Kubernetes uses multiple security mechanisms to process API requests.

Three important concepts are:

1. Authentication
2. Authorization
3. Admission control

## 3.1 Authentication

Authentication answers:

**Who are you?**

It verifies the identity of the requester making a Kubernetes API request.

The requester may be:

- A human user
- A ServiceAccount used by a workload
- An identity authenticated through an external identity provider

For example, a developer authenticates using the credentials configured for their Kubernetes access.

Authentication establishes the requester's identity. It does not automatically grant permission to perform every action.

## 3.2 Authorization

Authorization answers:

**Are you allowed to perform this action?**

After authentication, Kubernetes evaluates whether the identified requester has permission to perform the requested operation.

RBAC is a common authorization mechanism.

For example:

A developer might have permission to list Pods in the `development` namespace but not to delete Deployments in the `production` namespace.

Authorization decisions depend on the applicable permissions and authorization configuration.

## 3.3 Admission Control

Admission control answers:

**Should this API request be accepted, modified, or rejected according to applicable policies?**

Admission controllers process API requests after authentication and authorization and before the resulting objects are persisted.

For example:

A developer may be authorized to create Pods, but an admission policy can reject a Pod that violates the cluster's security requirements.

Admission controllers can:

- Validate resource configurations.
- Reject requests that violate policies.
- Apply approved defaults or modifications.
- Enforce additional security requirements.

Examples include Pod Security Admission and other built-in or custom admission mechanisms.

## 3.4 Request Processing Flow

A simplified Kubernetes API request flow is:

```text
Client
   |
   v
Kubernetes API Server
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

Other API processing steps and implementation details are omitted from this simplified diagram.

### Important Interview Point

- Authentication identifies the requester.
- Authorization determines what the requester may do.
- Admission control determines whether the request satisfies additional policies.

RBAC and admission control are different mechanisms. Admission policies do not replace RBAC.

---

# 4. Kubernetes RBAC

RBAC stands for Role-Based Access Control.

It controls which authenticated identities can perform specific actions on Kubernetes resources.

RBAC is commonly used to manage access for:

- Users
- Groups
- ServiceAccounts

## 4.1 Important RBAC Components

Kubernetes RBAC includes four primary resource types:

1. Role
2. RoleBinding
3. ClusterRole
4. ClusterRoleBinding

## 4.2 Role

A Role defines a set of permissions within a specific namespace.

For example, a Role can permit a developer to read Pods in the `development` namespace.

A Role does not grant access by itself. Its permissions must be associated with an identity through a RoleBinding.

### Example: Pod Reader Role

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

Explanation:

- `apiGroups: [""]`: Refers to the core API group.
- `resources: ["pods"]`: Applies to Pods.
- `verbs: ["get", "list", "watch"]`: Allows retrieving individual Pods, listing Pods, and watching changes.

The Role does not grant permission to create or delete Pods.

## 4.3 RoleBinding

A RoleBinding associates a Role or ClusterRole with one or more subjects within a namespace.

Subjects may include:

- Users
- Groups
- ServiceAccounts

### Example: Bind the Role to a Developer

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

This binding grants `developer1` the permissions defined in the `pod-reader` Role within the `development` namespace.

The `developer1` identity must correspond to an identity recognized by the cluster's authentication configuration.

## 4.4 ClusterRole

A ClusterRole defines a set of permissions that can be used at the cluster level.

It can define permissions for:

- Cluster-scoped resources, such as Nodes.
- Namespaced resources, such as Pods.
- Non-resource URL paths, when configured appropriately.

A ClusterRole does not grant permissions by itself. It must be bound to an identity.

## 4.5 ClusterRoleBinding

A ClusterRoleBinding associates a ClusterRole with users, groups, or ServiceAccounts at the cluster level.

Its permissions apply across the cluster, subject to the rules defined in the ClusterRole.

Use ClusterRoleBinding carefully because it can grant broad access.

## 4.6 Role vs. ClusterRole

| Feature | Role | ClusterRole |
|---|---|---|
| Scope of definition | Namespace | Cluster |
| Can define permissions for Pods | Yes | Yes |
| Can define permissions for Nodes | No | Yes |
| Can be bound using RoleBinding | Yes | Yes |
| Can be bound using ClusterRoleBinding | No | Yes |
| Typical use | Namespace-specific permissions | Reusable or cluster-wide permissions |

A ClusterRole used through a RoleBinding grants its permissions only within the RoleBinding's namespace when those permissions concern namespaced resources.

A ClusterRoleBinding grants the referenced ClusterRole's permissions at the cluster level.

## 4.7 RoleBinding vs. ClusterRoleBinding

| Feature | RoleBinding | ClusterRoleBinding |
|---|---|---|
| Scope of grant | One namespace | Cluster-wide |
| Can reference a Role | Yes | No |
| Can reference a ClusterRole | Yes | Yes |
| Common use | Developer access to one namespace | Cluster-wide administrative or read-only access |

### Important

A RoleBinding can reference a ClusterRole.

For example, a reusable ClusterRole defining Pod-read permissions can be bound separately in multiple namespaces using different RoleBindings.

This avoids duplicating the same permission rules.

---

# 5. RBAC Hands-On Lab

## Objective

Give a developer read-only access to Pods in the `development` namespace.

## Step 1: Create the Namespace

```bash
kubectl create namespace development
```

## Step 2: Create the Role

Save the following as `role.yaml`:

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

Apply it:

```bash
kubectl apply -f role.yaml
```

## Step 3: Create the RoleBinding

Save the following as `rolebinding.yaml`:

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

Apply it:

```bash
kubectl apply -f rolebinding.yaml
```

## Step 4: Verify the Permissions

Check whether the user can list Pods:

```bash
kubectl auth can-i list pods \
  --as=developer1 \
  -n development
```

Expected result:

```text
yes
```

Check whether the user can delete Pods:

```bash
kubectl auth can-i delete pods \
  --as=developer1 \
  -n development
```

Expected result:

```text
no
```

The impersonation checks require the caller to have permission to impersonate `developer1`.

These commands test the effective authorization decision for the specified identity and operation.

They do not require the developer to be logged in as `developer1`.

## Step 5: Inspect the Resources

```bash
kubectl get role -n development

kubectl get rolebinding -n development

kubectl describe role pod-reader -n development

kubectl describe rolebinding pod-reader-binding \
  -n development
```

## Production Lesson

Never grant more permissions than an identity needs.

For example, a developer who only needs to inspect Pods should not automatically receive permission to:

- Delete Pods
- Modify Deployments
- Read Secrets
- Create privileged workloads
- Modify RBAC permissions

---

# 6. Kubernetes ServiceAccounts

A ServiceAccount provides an identity for a workload that interacts with the Kubernetes API.

ServiceAccounts are commonly used by Pods and applications that need to communicate with Kubernetes.

They are different from human user identities.

## 6.1 Create a ServiceAccount

```bash
kubectl create serviceaccount app-sa \
  -n development
```

Verify it:

```bash
kubectl get serviceaccounts -n development
```

## 6.2 Assign a ServiceAccount to a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  namespace: development
spec:
  serviceAccountName: app-sa
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
```

Apply the manifest:

```bash
kubectl apply -f app-pod.yaml
```

The Pod uses `app-sa` as its ServiceAccount identity.

However, assigning a ServiceAccount does not automatically grant it broad API permissions. Those permissions depend on applicable authorization rules.

## 6.3 Disable Automatic Token Mounting

If the application does not need to access the Kubernetes API, disable automatic ServiceAccount token mounting.

At the Pod level:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  namespace: development
spec:
  automountServiceAccountToken: false
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
```

This prevents the normal automatic mounting of ServiceAccount API credentials into the Pod.

It does not remove credentials that the application independently receives or mounts through other mechanisms.

### Production Best Practice

Use dedicated ServiceAccounts for workloads with different access requirements.

Avoid granting broad permissions to the default ServiceAccount.

---

# 7. Kubernetes SecurityContext

A securityContext configures security-related settings for a Pod or container.

It can control aspects such as:

- User and group IDs
- Linux capabilities
- Privilege escalation
- Filesystem write access
- Seccomp profiles

SecurityContext is not an RBAC mechanism.

RBAC controls Kubernetes API permissions. SecurityContext controls how the container or its processes run.

## 7.1 runAsNonRoot

```yaml
securityContext:
  runAsNonRoot: true
```

This requires Kubernetes to ensure that the container does not run as UID 0.

The container image and user configuration must support non-root execution.

You can explicitly specify a user ID:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
```

UID 10001 is an example. Use a user ID appropriate for the application image.

## 7.2 allowPrivilegeEscalation

```yaml
securityContext:
  allowPrivilegeEscalation: false
```

This prevents the process from gaining additional privileges through mechanisms such as set-user-ID or set-group-ID execution.

It is an important container-hardening setting.

It does not replace other security controls.

## 7.3 readOnlyRootFilesystem

```yaml
securityContext:
  readOnlyRootFilesystem: true
```

This mounts the container's root filesystem as read-only.

The application cannot write to that filesystem.

If the application needs writable temporary files, logs, caches, or runtime data, provide appropriate writable volumes.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readonly-app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
  containers:
    - name: app
      image: your-nonroot-application-image
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: tmp
      emptyDir: {}
```

Replace the example image with an application image that supports the specified user and filesystem configuration.

The `/tmp` volume is writable even though the container's root filesystem is read-only.

## 7.4 Linux Capabilities

Linux capabilities divide certain root-level privileges into individual permissions.

Containers may receive a set of capabilities depending on the runtime and configuration.

Drop unnecessary capabilities:

```yaml
securityContext:
  capabilities:
    drop:
      - ALL
```

This removes the container's default capability set for the configured process.

Add back individual capabilities only when they are genuinely required.

## 7.5 Seccomp

Seccomp can restrict the system calls a process is permitted to make.

A common configuration is:

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault
```

This uses the container runtime's default seccomp profile.

The exact restrictions depend on the runtime and its configuration.

## 7.6 Example: Hardened Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
  namespace: development
spec:
  automountServiceAccountToken: false
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    runAsGroup: 10001
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

This is a security-hardening example, not a guarantee that every application will run without modification.

Validate the image, filesystem requirements, user IDs, and runtime behavior before deployment.

### Production Lesson

Apply the principle of least privilege to both Kubernetes API access and the container execution environment.

A workload can have no Kubernetes API permissions and still pose a security risk if its container runs with unnecessary operating-system privileges.

---

# 8. Kubernetes Secrets

Kubernetes Secrets are designed to hold sensitive information, including:

- Passwords
- API tokens
- Database credentials
- TLS certificates
- Application credentials

## 8.1 Create a Secret

For a local lab:

```bash
kubectl create secret generic db-secret \
  --from-literal=username=appuser \
  --from-literal=password='example-password' \
  -n development
```

Do not use real production credentials in this example.

Literal values can appear in shell history or process-related diagnostic information. For production workflows, prefer approved secret-management and deployment mechanisms that avoid unnecessary exposure.

## 8.2 Inspect the Secret

```bash
kubectl get secret db-secret -n development
```

View the Secret's metadata and data keys:

```bash
kubectl describe secret db-secret -n development
```

Avoid unnecessarily printing sensitive Secret values into terminal output, logs, or shared sessions.

## 8.3 Base64 Is Not Encryption

Kubernetes Secret manifests often represent data using Base64 encoding.

For example:

```text
YXBwdXNlcg==
```

This is Base64-encoded data, not encryption.

Anyone who obtains the encoded value can decode it.

Therefore, Base64 must never be treated as a security mechanism.

## 8.4 Are Kubernetes Secrets Encrypted at Rest?

Not necessarily.

By default, Kubernetes Secret data may be stored in etcd without encryption at rest, depending on the cluster configuration.

To improve protection:

1. Configure encryption at rest for Secret data.
2. Restrict Secret access using RBAC.
3. Protect access to etcd and cluster backups.
4. Avoid exposing Secret values through logs or source control.
5. Rotate credentials when appropriate.
6. Consider using an external secret manager.

## 8.5 Use an External Secret Manager

For example, AWS Secrets Manager can store sensitive information outside Kubernetes.

A suitable integration can synchronize or retrieve approved secrets for workloads.

The integration may use a supported Secrets Store CSI Driver, External Secrets Operator, or another approved mechanism.

External secret management does not eliminate the need for RBAC, encryption, and secure workload configuration.

## 8.6 Production Secret Security Checklist

- Enable encryption at rest.
- Apply least-privilege RBAC.
- Restrict access to etcd and backups.
- Avoid committing secrets to Git.
- Do not place secrets directly into container images.
- Avoid logging secret values.
- Rotate credentials.
- Monitor access to sensitive resources.
- Use an external secret manager when appropriate.

---

# 9. Kubernetes NetworkPolicies

NetworkPolicies control network traffic to and from selected Pods.

They can restrict:

- Ingress traffic: traffic entering selected Pods.
- Egress traffic: traffic leaving selected Pods.

NetworkPolicies are enforced only when the cluster's network implementation supports and enforces them.

Creating a NetworkPolicy in a cluster without effective policy enforcement does not guarantee that the intended traffic restriction will occur.

## 9.1 Example: Allow Frontend Traffic to Backend

Assume:

- Frontend Pods have the label `app: frontend`.
- Backend Pods have the label `app: backend`.
- The backend application listens on TCP port 8080.
- Both components run in the `development` namespace.

The following policy restricts ingress to backend Pods to traffic from frontend Pods on TCP port 8080.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-ingress-policy
  namespace: development
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

Apply it:

```bash
kubectl apply -f network-policy.yaml
```

Inspect it:

```bash
kubectl get networkpolicy -n development

kubectl describe networkpolicy backend-ingress-policy \
  -n development
```

### Important Details

This policy selects backend Pods and restricts their ingress traffic.

It does not automatically restrict their egress traffic.

The policy also does not create a frontend-to-backend connection or guarantee that the application is healthy.

Ensure the Pod labels and application port match the actual deployment.

### Production Lesson

Start with the application's required communication paths.

Apply NetworkPolicies incrementally, test legitimate traffic, and ensure that required DNS and other dependencies remain reachable.

Avoid applying restrictive policies without understanding the application's network requirements.

---

# 10. Pod Security Standards

Kubernetes defines three Pod Security Standards:

1. Privileged
2. Baseline
3. Restricted

## 10.1 Privileged

The Privileged standard allows broad permissions and provides minimal restrictions.

It may be necessary for specific infrastructure workloads, but it is generally unsuitable for ordinary application workloads.

## 10.2 Baseline

The Baseline standard prevents several known privilege-escalation configurations while allowing common workload patterns.

It provides a moderate level of restriction.

## 10.3 Restricted

The Restricted standard applies stronger security requirements, including restrictions around user privileges, privilege escalation, capabilities, and related security settings.

Some applications need configuration changes to comply.

## 10.4 Comparison

| Standard | General Purpose |
|---|---|
| Privileged | Minimal restrictions |
| Baseline | Prevent commonly known privilege escalations |
| Restricted | Stronger workload security restrictions |

## 10.5 Pod Security Admission

Pod Security Admission enforces Pod Security Standards at the namespace level.

Example:

```bash
kubectl label namespace development \
  pod-security.kubernetes.io/enforce=restricted
```

This enables enforcement of the Restricted standard for applicable Pod creation and update requests in the namespace.

Before enforcing this in production, evaluate existing workloads and identify any that do not comply.

To inspect the namespace labels:

```bash
kubectl get namespace development --show-labels
```

### Warning

Enforcing Restricted may reject workloads that use disallowed configurations.

Test first in a non-production environment.

Pod Security Admission is a policy-enforcement mechanism. It does not replace RBAC, NetworkPolicies, or Secret management.

---

# 11. Production Kubernetes Security Best Practices

## 11.1 Identity and Access

- Use least-privilege RBAC.
- Prefer namespace-scoped permissions when sufficient.
- Avoid unnecessary ClusterRoleBindings.
- Review access to Secrets.
- Use dedicated ServiceAccounts for workloads.
- Disable automatic token mounting when API access is unnecessary.
- Audit permissions regularly.

## 11.2 Container Security

- Run containers as non-root.
- Disable privilege escalation where possible.
- Drop unnecessary Linux capabilities.
- Use read-only root filesystems when compatible.
- Configure appropriate seccomp profiles.
- Avoid privileged containers.
- Use trusted, maintained container images.
- Scan images for known vulnerabilities.

## 11.3 Secret Management

- Enable encryption at rest.
- Restrict access to sensitive information.
- Avoid storing secrets in source control.
- Use external secret-management systems where appropriate.
- Rotate credentials and monitor access.

## 11.4 Network Security

- Use NetworkPolicies to restrict unnecessary communication.
- Validate that the cluster's networking implementation enforces policies.
- Restrict ingress and egress according to application requirements.
- Avoid unnecessary exposure through Services and Ingress resources.
- Verify that DNS and required dependencies continue to work.

## 11.5 Policy Enforcement

- Evaluate Pod Security Standards.
- Use Pod Security Admission where appropriate.
- Test admission policies before production enforcement.
- Review workloads that require exceptions.

## 11.6 Monitoring and Auditing

- Enable appropriate Kubernetes audit logging.
- Monitor unusual API activity.
- Review changes to RBAC resources.
- Monitor access to Secrets.
- Investigate unexpected privileged workloads.
- Alert on security-relevant configuration changes.

---

# 12. Troubleshooting Kubernetes Security Issues

## Problem 1: User Cannot List Pods

Check the user's effective permissions:

```bash
kubectl auth can-i list pods \
  --as=developer1 \
  -n development
```

Inspect Roles:

```bash
kubectl get roles -n development
```

Inspect RoleBindings:

```bash
kubectl get rolebindings -n development
```

Possible causes:

- Missing Role.
- Missing RoleBinding.
- Incorrect namespace.
- Incorrect subject name.
- Missing `list` permission.
- Authentication identity mismatch.

## Problem 2: Pod Is Rejected Because It Runs as Root

Inspect the Pod configuration:

```bash
kubectl get pod <pod-name> \
  -n development \
  -o yaml
```

Check:

- `runAsNonRoot`
- `runAsUser`
- Namespace Pod Security Admission settings
- Admission policy errors

Use an image that supports non-root execution and configure an appropriate UID.

## Problem 3: Application Cannot Write to Files

Check whether the container has:

```yaml
readOnlyRootFilesystem: true
```

Inspect application logs and events.

Possible causes:

- The application writes to its root filesystem.
- A required directory is not writable.
- The mounted volume has incompatible permissions.
- The configured user does not have write access.

Provide writable volumes only for the paths the application needs.

## Problem 4: Application Cannot Reach Another Pod

Inspect NetworkPolicies:

```bash
kubectl get networkpolicies -A
```

Check:

- Source and destination Pod labels.
- Namespace selectors.
- Allowed ports.
- Ingress policies.
- Egress policies.
- DNS access.
- Whether the CNI enforces NetworkPolicies.

## Problem 5: Application Cannot Access a Secret

Check that the Secret exists:

```bash
kubectl get secret db-secret -n development
```

Inspect the Pod configuration and events.

Check:

- Secret name.
- Namespace.
- Volume or environment variable configuration.
- RBAC permissions, if the application retrieves Secrets through the Kubernetes API.
- External secret integration and synchronization status.

Do not print actual Secret values unnecessarily during troubleshooting.

---

# 13. Hands-On Lab Checklist

Complete these tasks in a test cluster.

## Lab 1: RBAC

- [ ] Create the `development` namespace.
- [ ] Create a Role that permits reading Pods.
- [ ] Bind the Role to a developer identity.
- [ ] Verify `list pods` permission.
- [ ] Verify that `delete pods` is denied.

## Lab 2: ServiceAccounts

- [ ] Create a dedicated ServiceAccount.
- [ ] Assign it to a Pod.
- [ ] Disable automatic token mounting for a workload that does not need Kubernetes API access.
- [ ] Understand the difference between identity and authorization.

## Lab 3: Pod Security

- [ ] Deploy an application using a non-root user.
- [ ] Disable privilege escalation.
- [ ] Drop unnecessary Linux capabilities.
- [ ] Enable a read-only root filesystem where compatible.
- [ ] Configure `RuntimeDefault` seccomp.
- [ ] Verify that the application functions correctly.

## Lab 4: Secrets

- [ ] Create a test Secret.
- [ ] Inspect its metadata without unnecessarily exposing its values.
- [ ] Explain why Base64 is not encryption.
- [ ] Review the cluster's Secret encryption configuration.
- [ ] Understand how an external secret manager could be integrated.

## Lab 5: NetworkPolicies

- [ ] Deploy frontend and backend Pods.
- [ ] Create a NetworkPolicy for backend ingress.
- [ ] Test permitted frontend-to-backend communication.
- [ ] Test an unauthorized connection.
- [ ] Confirm that the network plugin enforces the policy.

## Lab 6: Pod Security Admission

- [ ] Review the Baseline and Restricted standards.
- [ ] Test Restricted enforcement in a lab namespace.
- [ ] Identify workloads rejected by the policy.
- [ ] Correct the workload configuration.

---

# 14. Day 12 Key Takeaways

1. Authentication identifies the requester.
2. Authorization determines whether the requester has permission to perform an action.
3. Admission control applies additional policies to API requests.
4. RBAC manages Kubernetes API permissions.
5. Role defines namespace-scoped permissions.
6. RoleBinding grants a Role or ClusterRole's permissions within a namespace.
7. ClusterRole defines reusable or cluster-level permission rules.
8. ClusterRoleBinding grants permissions at the cluster level.
9. ServiceAccounts provide identities for workloads.
10. SecurityContext controls aspects of container execution security.
11. Base64 is not encryption.
12. Kubernetes Secret protection requires suitable encryption and access controls.
13. NetworkPolicies restrict traffic when supported and enforced by the network plugin.
14. Pod Security Standards provide different levels of workload restrictions.
15. Production security requires multiple layers of protection.

---

# 15. Interview Preparation

Practise answering these questions:

1. What is the difference between authentication, authorization, and admission control?
2. What is the difference between Role, RoleBinding, ClusterRole, and ClusterRoleBinding?
3. How would you secure a Pod using securityContext?
4. Are Kubernetes Secrets secure by default?
5. How would you secure a privileged Pod that can communicate with every Pod in the cluster?

Try to answer each question without referring to the notes.

Then compare your response with the concepts and examples in this README.

---

# Day 12 Completion

**Topic:** Kubernetes Security

**Status:** Completed

**Next Topic:** Day 13 — Kubernetes Troubleshooting

The next lesson will focus on systematically diagnosing Kubernetes problems using Pod status, events, logs, probes, Services, networking, resource constraints, and other troubleshooting tools.
