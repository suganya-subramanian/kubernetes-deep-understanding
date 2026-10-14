# Kubernetes Day 11: Helm — Complete Revision Guide

## Table of Contents

1. [Learning Objectives](#1-learning-objectives)
2. [What Is Helm?](#2-what-is-helm)
3. [Why Do We Need Helm?](#3-why-do-we-need-helm)
4. [Helm vs kubectl](#4-helm-vs-kubectl)
5. [Important Helm Terminology](#5-important-helm-terminology)
6. [Helm Architecture](#6-helm-architecture)
7. [Helm Chart Structure](#7-helm-chart-structure)
8. [Understanding Chart.yaml](#8-understanding-chartyaml)
9. [Understanding values.yaml](#9-understanding-valuesyaml)
10. [Understanding templates](#10-understanding-templates)
11. [Helm Template Syntax](#11-helm-template-syntax)
12. [Creating a Helm Chart](#12-creating-a-helm-chart)
13. [Helm Repositories](#13-helm-repositories)
14. [Installing Applications with Helm](#14-installing-applications-with-helm)
15. [Overriding Helm Values](#15-overriding-helm-values)
16. [Upgrading a Helm Release](#16-upgrading-a-helm-release)
17. [Helm Release History](#17-helm-release-history)
18. [Rolling Back a Helm Release](#18-rolling-back-a-helm-release)
19. [Validating and Rendering Charts](#19-validating-and-rendering-charts)
20. [Waiting for Resources and Atomic Operations](#20-waiting-for-resources-and-atomic-operations)
21. [Uninstalling a Helm Release](#21-uninstalling-a-helm-release)
22. [Helm Dependencies](#22-helm-dependencies)
23. [Helm Troubleshooting](#23-helm-troubleshooting)
24. [Production Best Practices](#24-production-best-practices)
25. [Hands-On Lab](#25-hands-on-lab)
26. [Useful Helm Commands](#26-useful-helm-commands)
27. [Common Mistakes](#27-common-mistakes)
28. [Quick Revision](#28-quick-revision)
29. [Day 11 Completion Checklist](#29-day-11-completion-checklist)

---

## 1. Learning Objectives

By the end of Day 11, you should be able to:

- Explain what Helm is and why Kubernetes engineers use it.
- Understand charts, releases, repositories, and revisions.
- Explain Helm architecture.
- Create a Helm chart.
- Understand Chart.yaml, values.yaml, templates/, and _helpers.tpl.
- Install an application using Helm.
- Configure applications using custom values.
- Upgrade an existing release.
- Inspect release history.
- Roll back to a previous revision.
- Validate and render charts before deployment.
- Use --wait and --atomic appropriately.
- Troubleshoot common Helm deployment failures.
- Explain how Helm fits into a production CI/CD or GitOps workflow.

---

## 2. What Is Helm?

Helm is a package manager for Kubernetes.

It helps us package, configure, deploy, upgrade, and manage Kubernetes applications.

Kubernetes applications commonly require multiple resources, such as:

- Deployment
- Service
- ConfigMap
- Secret
- Ingress
- ServiceAccount
- PersistentVolumeClaim
- HorizontalPodAutoscaler

Maintaining separate YAML manifests for every application and environment can become difficult.

Helm addresses this problem by grouping Kubernetes resource templates, configuration values, and chart metadata into a reusable package called a Helm chart.

### Simple example

Suppose an application requires:

- Deployment with three replicas
- Service exposing port 80
- ConfigMap containing application settings
- Ingress exposing the application externally

Without Helm, we might maintain multiple YAML files.

With Helm, we can organize the resources into one chart and deploy them as a release.

```bash
helm install my-app ./my-app
```

Helm renders the chart templates using the configured values and submits the resulting Kubernetes resources to the cluster.

### Important clarification

Helm does not replace Kubernetes.

Helm helps define and manage application deployments. Kubernetes still creates, schedules, and manages the actual resources.

### Interview answer

Helm is a Kubernetes package manager that simplifies application deployment and lifecycle management. It uses reusable charts containing Kubernetes resource templates and configurable values. Helm supports installation, upgrades, release history, and rollbacks.

---

## 3. Why Do We Need Helm?

Consider an application deployed in three environments:

- Development
- Testing
- Production

The application needs a different replica count in each environment.

| Configuration | Development | Testing | Production |
|---|---:|---:|---:|
| Replicas | 1 | 2 | 5 |
| Log level | DEBUG | INFO | WARN |
| Ingress enabled | false | true | true |

### Without Helm

We might maintain separate Deployment manifests for each environment.

This can cause:

- Duplicate YAML files.
- Configuration inconsistencies.
- Repeated changes.
- Difficulty maintaining multiple applications.
- More complicated upgrade procedures.
- Increased risk of configuration drift.

### With Helm

We can reuse one chart and provide different values files.

```text
webapp/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

Environment-specific configuration:

```text
values-development.yaml
values-testing.yaml
values-production.yaml
```

Deploy to development:

```bash
helm upgrade --install webapp ./webapp \
  -n development \
  --create-namespace \
  -f values-development.yaml
```

Deploy to production:

```bash
helm upgrade --install webapp ./webapp \
  -n production \
  --create-namespace \
  -f values-production.yaml
```

Both releases use the same chart but have different configuration.

### Important point

Changing a chart does not automatically update every release that uses it.

Each release must be upgraded separately, unless a CI/CD pipeline or another automation system manages those upgrades.

### Benefits of Helm

1. Reusable application packages.
2. Centralized configuration.
3. Environment-specific deployments.
4. Simplified upgrades.
5. Release revision history.
6. Rollback support.
7. Dependency management.
8. Integration with CI/CD pipelines.
9. Easier distribution of Kubernetes applications.

---

## 4. Helm vs kubectl

| Feature | kubectl | Helm |
|---|---|---|
| Primary purpose | Interact with Kubernetes resources | Package and manage Kubernetes applications |
| Input | Kubernetes manifests and resource references | Charts, values, and chart references |
| Templating | Not provided by kubectl apply itself | Supported through Helm templates |
| Configuration reuse | Possible with other tools and approaches | Built into chart values and templates |
| Deployment | Applies or changes Kubernetes resources | Installs or upgrades chart releases |
| Release history | Does not provide Helm-style release history | Maintains release revisions |
| Rollback | Requires an appropriate recovery strategy | Supports helm rollback |
| Package distribution | Not a chart package manager | Supports chart repositories and OCI registries |
| Resource inspection | Excellent for inspecting Kubernetes resources | Provides release-level inspection |

### Example using kubectl

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
```

You can also apply a directory:

```bash
kubectl apply -f ./manifests/
```

### Example using Helm

```bash
helm upgrade --install webapp ./webapp \
  --namespace development \
  --create-namespace
```

Helm renders the chart and manages the release operation.

### Does Helm eliminate kubectl?

No.

In real projects, we use both.

Helm manages application releases, while kubectl helps us inspect and troubleshoot the underlying Kubernetes resources.

For example:

```bash
helm status webapp -n development
kubectl get pods -n development
kubectl logs <pod-name> -n development
kubectl describe pod <pod-name> -n development
```

---

## 5. Important Helm Terminology

### 5.1 Helm

The command-line tool used to work with Helm charts and releases.

Example:

```bash
helm version
```

### 5.2 Helm Chart

A package containing the templates, default values, metadata, and optional dependencies required to define a Kubernetes application.

Example:

```text
webapp/
├── Chart.yaml
├── values.yaml
└── templates/
```

### 5.3 Helm Release

An installed instance of a Helm chart.

Example:

```bash
helm install webapp ./webapp
```

Here:

- Release name: webapp
- Chart path: ./webapp

You can install the same chart multiple times using different release names and namespaces.

Example:

```bash
helm install webapp-dev ./webapp -n development
helm install webapp-prod ./webapp -n production
```

These are two separate releases.

### 5.4 Helm Repository

A location where Helm charts are published and distributed.

Example:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

The alias `bitnami` refers to the configured repository.

### 5.5 Helm Revision

A release revision represents a point in a release's history.

Helm records revisions as a release is installed, upgraded, or rolled back.

Example:

```bash
helm history webapp -n development
```

A rollback creates another revision rather than erasing the historical record.

### 5.6 Values

Values are configuration settings supplied to chart templates.

Examples:

```yaml
replicaCount: 3
image:
  repository: nginx
  tag: "1.28"
```

The chart can use these values to generate Kubernetes manifests.

### Quick comparison

| Term | Meaning |
|---|---|
| Helm | Kubernetes package manager |
| Chart | Application package |
| Release | Installed instance of a chart |
| Repository | Location that distributes charts |
| Values | Configuration supplied to templates |
| Revision | Version in a release's history |

---

## 6. Helm Architecture

Helm 3 does not use Tiller.

Tiller was part of Helm 2 and is no longer part of the Helm 3 architecture.

### Simplified architecture

```text
Developer / CI/CD Pipeline
          |
          v
       Helm CLI
          |
          | Reads chart and values
          v
    Template Rendering
          |
          | Generates Kubernetes manifests
          v
    Kubernetes API Server
          |
          v
       Kubernetes
          |
          +--> Deployment
          +--> ReplicaSet
          +--> Pods
          +--> Service
          +--> Other resources
```

### What happens during installation?

Suppose we execute:

```bash
helm install webapp ./webapp \
  --namespace development \
  --create-namespace
```

The workflow is approximately:

1. Helm reads the chart.
2. Helm loads the applicable values.
3. Helm renders the templates.
4. Helm submits the resulting resources through the Kubernetes API.
5. Kubernetes processes the requested resources.
6. Helm records release information in the cluster.
7. The release can be inspected and managed using Helm commands.

### Where does Helm store release information?

In a typical Helm 3 configuration, release information is stored in Kubernetes Secrets in the release namespace.

Example:

```bash
kubectl get secrets -n development
```

You may see Secrets with names resembling:

```text
sh.helm.release.v1.webapp.v1
sh.helm.release.v1.webapp.v2
```

The actual names depend on the release and revision.

Helm can use different storage drivers, so Kubernetes Secrets are the common default rather than the only possible storage mechanism.

### Important security consideration

Helm release records may contain rendered manifests and configuration values, including sensitive information if secrets were placed directly in chart values.

Therefore:

- Restrict access to release Secrets.
- Apply least-privilege RBAC.
- Avoid exposing sensitive values in logs.
- Prefer external secret-management systems when appropriate.
- Never assume that a value is safe merely because it is stored in a Helm chart.

---

## 7. Helm Chart Structure

Create a chart:

```bash
helm create webapp
```

This creates a directory named `webapp` containing a starter chart.

A typical structure looks like:

```text
webapp/
├── .helmignore
├── Chart.yaml
├── values.yaml
├── charts/
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── serviceaccount.yaml
│   ├── hpa.yaml
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   └── tests/
└── ...
```

The exact generated files can vary by Helm version.

### 7.1 Chart.yaml

Contains chart metadata.

Examples include:

- Chart name.
- Chart version.
- Description.
- Application version.
- Chart type.

### 7.2 values.yaml

Contains default configuration values.

Example:

```yaml
replicaCount: 2

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "1.28"

service:
  type: ClusterIP
  port: 80
```

### 7.3 templates/

Contains templates that Helm renders into Kubernetes manifests.

Examples:

```text
templates/
├── deployment.yaml
├── service.yaml
└── ingress.yaml
```

### 7.4 _helpers.tpl

Contains reusable template helpers.

Examples:

- Common labels.
- Resource names.
- Chart labels.
- Reusable naming logic.

Files beginning with an underscore are not rendered as standalone Kubernetes resource manifests.

### 7.5 charts/

Contains packaged chart dependencies when dependencies have been downloaded or packaged.

### 7.6 .helmignore

Specifies files and directories that Helm should ignore when packaging a chart.

### 7.7 NOTES.txt

Can provide useful instructions after chart installation, such as how to access the application.

### Important distinction

`Chart.yaml` describes the chart.

`values.yaml` defines default configuration.

`templates/` contains the resource templates.

`_helpers.tpl` defines reusable template helpers.

---

## 8. Understanding Chart.yaml

Example:

```yaml
apiVersion: v2
name: webapp
description: A Helm chart for a web application
type: application
version: 0.1.0
appVersion: "1.0.0"
```

### Explanation

#### apiVersion

Identifies the chart format version.

For Helm 3 charts, `apiVersion: v2` is commonly used.

#### name

The chart name.

```yaml
name: webapp
```

#### description

A short description of the chart.

#### type

Usually `application` for deployable applications.

Helm also supports library charts, which provide reusable template functionality rather than being installed as standalone applications in the usual way.

#### version

The chart version.

```yaml
version: 0.1.0
```

This identifies the chart package version.

When chart contents change, the chart version should be updated according to the project's versioning policy.

#### appVersion

The application version represented by the chart.

```yaml
appVersion: "1.0.0"
```

This is informational metadata. Helm does not automatically use it as the container image tag unless the chart template explicitly references it.

### Chart version vs application version

| Field | Purpose |
|---|---|
| version | Version of the Helm chart |
| appVersion | Version of the application represented by the chart |

They can change independently.

---

## 9. Understanding values.yaml

The values file separates configuration from Kubernetes resource templates.

Example:

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.28"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
```

These values are useful only when the chart templates reference them.

For example, a Deployment template can use:

```yaml
spec:
  replicas: {{ .Values.replicaCount }}
```

This renders to:

```yaml
spec:
  replicas: 2
```

### Why separate values from templates?

Suppose development requires one replica and production requires five replicas.

Instead of maintaining two different Deployment templates, use different values files.

`values-development.yaml`:

```yaml
replicaCount: 1
```

`values-production.yaml`:

```yaml
replicaCount: 5
```

Deploy development:

```bash
helm upgrade --install webapp ./webapp \
  -n development \
  --create-namespace \
  -f values-development.yaml
```

Deploy production:

```bash
helm upgrade --install webapp ./webapp \
  -n production \
  --create-namespace \
  -f values-production.yaml
```

### Value precedence

When multiple configuration sources provide the same value, the higher-precedence source wins.

A simplified precedence order, from lower to higher, is:

1. Chart default values.
2. Parent chart values for dependencies.
3. User-supplied values files.
4. Values supplied using `--set` and related command-line options.

When multiple values files are provided, later files override earlier files for conflicting values.

Example:

```bash
helm upgrade --install webapp ./webapp \
  -n production \
  -f values.yaml \
  -f values-production.yaml \
  --set replicaCount=6
```

If all three define `replicaCount`, the command-line override takes precedence.

### Production warning

Do not place production passwords, tokens, or private keys in ordinary values files stored in Git.

Use a suitable secret-management approach, such as an external secrets manager or an established encrypted-secret workflow.

---

## 10. Understanding templates

Helm templates use Go template syntax and can reference chart values.

Example template:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: nginx
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 80
```

Suppose the chart is installed using:

```bash
helm install webapp ./webapp \
  --namespace development \
  --set replicaCount=3
```

The generated Deployment contains:

```yaml
spec:
  replicas: 3
```

The release name can also be used to generate resource names.

### Common Helm objects

| Object | Purpose |
|---|---|
| `.Values` | Accesses configured values |
| `.Release.Name` | Release name |
| `.Release.Namespace` | Release namespace |
| `.Chart.Name` | Chart name |
| `.Chart.Version` | Chart version |
| `.Chart.AppVersion` | Application version |

### Example

```yaml
metadata:
  name: {{ .Release.Name }}
  namespace: {{ .Release.Namespace }}
```

These objects allow templates to generate configuration based on the release and chart.

---

## 11. Helm Template Syntax

### 11.1 Values

```yaml
replicas: {{ .Values.replicaCount }}
```

Reads a value from the chart's configuration.

### 11.2 Quote

```yaml
metadata:
  annotations:
    app-version: {{ .Values.appVersion | quote }}
```

The `quote` function converts the value to a quoted string.

This is useful for fields that should be strings, such as annotations.

### 11.3 Default

```yaml
replicas: {{ .Values.replicaCount | default 2 }}
```

Uses a fallback value when the supplied value is considered empty by the template function.

Be careful: `default` also treats values such as `0` and `false` as empty. If zero or false is a meaningful configuration, use validation or another explicit approach instead.

### 11.4 Required

```yaml
image: {{ required "image.repository is required" .Values.image.repository }}
```

Helm reports an error if the required value is empty or missing.

This is useful for mandatory configuration.

### 11.5 Include

```yaml
metadata:
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
```

Calls a named template helper and inserts its output.

### 11.6 With

The `with` statement changes the current context to the specified object when that object is non-empty.

Example:

```yaml
{{- with .Values.nodeSelector }}
nodeSelector:
  {{- toYaml . | nindent 2 }}
{{- end }}
```

This renders the nodeSelector only when the value is present and non-empty.

### 11.7 Range

The `range` statement iterates over lists or maps.

Example:

```yaml
ports:
  {{- range .Values.containerPorts }}
  - containerPort: {{ . }}
  {{- end }}
```

The template expects `containerPorts` to be a list.

### 11.8 toYaml

Converts a value into YAML representation.

Example:

```yaml
resources:
  {{- toYaml .Values.resources | nindent 12 }}
```

This is useful for rendering nested configuration.

### Important template practices

- Keep indentation correct.
- Use `helm lint`.
- Render templates before deployment.
- Validate the resulting Kubernetes manifests.
- Avoid unnecessary template complexity.
- Use helpers for repeated naming and labels.
- Validate required values explicitly.

---

## 12. Creating a Helm Chart

### Step 1: Check Helm

```bash
helm version
```

### Step 2: Create a chart

```bash
helm create webapp
```

### Step 3: Inspect the directory

```bash
ls -R webapp
```

### Step 4: Validate the chart

```bash
helm lint ./webapp
```

### Step 5: Render the manifests

```bash
helm template webapp ./webapp
```

### Step 6: Install the release

```bash
helm install webapp ./webapp \
  --namespace development \
  --create-namespace
```

### Step 7: Check the release

```bash
helm list -n development
helm status webapp -n development
```

### Step 8: Inspect Kubernetes resources

```bash
kubectl get all -n development
```

Note that `kubectl get all` does not literally display every resource type in the namespace. Inspect specific resources separately when necessary.

### Step 9: Inspect the Pods

```bash
kubectl get pods -n development
```

### Step 10: Inspect the rendered configuration

```bash
helm get values webapp -n development
helm get manifest webapp -n development
```

---

## 13. Helm Repositories

A Helm repository distributes charts.

A repository is not the same as an installed application.

### Add a repository

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

This registers the repository under the alias `bitnami`.

### Update repository metadata

```bash
helm repo update
```

This refreshes local metadata for configured repositories.

It does not install every chart or download all application container images.

### List configured repositories

```bash
helm repo list
```

### Search repository charts

```bash
helm search repo nginx
```

### Search for chart versions

```bash
helm search repo nginx --versions
```

### Install a chart

For example, if the chart is available from the configured repository:

```bash
helm install my-app bitnami/nginx \
  --namespace development \
  --create-namespace
```

The command creates a release named `my-app`.

The release belongs to the `development` namespace.

### Repository vs chart vs container registry

| Component | Stores or distributes |
|---|---|
| Helm repository | Helm charts |
| Helm chart | Templates, values, metadata, and optional dependencies |
| Container registry | Container images |
| Kubernetes cluster | Running workloads and Kubernetes resources |

### OCI registries

Helm can also distribute charts using OCI-compatible registries.

For example, chart references may use an OCI URL:

```text
oci://registry.example.com/helm/webapp
```

The exact authentication and installation commands depend on the registry and chart location.

### Production consideration

Before installing a third-party chart, inspect:

- Chart source.
- Chart version.
- Default values.
- Required permissions.
- Generated resources.
- Container image sources.
- Security settings.
- Maintenance status.

Do not blindly install an untrusted chart into production.

---

## 14. Installing Applications with Helm

### General syntax

```bash
helm install RELEASE_NAME CHART
```

Example:

```bash
helm install webapp ./webapp
```

### Install into a namespace

```bash
helm install webapp ./webapp \
  --namespace development
```

The namespace must already exist unless you specify `--create-namespace`.

### Create the namespace if necessary

```bash
helm install webapp ./webapp \
  --namespace development \
  --create-namespace
```

### Install using a values file

```bash
helm install webapp ./webapp \
  --namespace development \
  --create-namespace \
  -f values-development.yaml
```

### Inspect the release

```bash
helm list -n development
helm status webapp -n development
```

### Important behavior

Installing a release does not guarantee that the application will be healthy or reachable.

After installation, verify:

```bash
kubectl get pods -n development
kubectl get services -n development
kubectl get events -n development --sort-by=.metadata.creationTimestamp
```

If the application is not working, inspect Pod events and logs.

---

## 15. Overriding Helm Values

Helm supports several ways to provide configuration.

### Method 1: Default values.yaml

```yaml
replicaCount: 2
```

### Method 2: Custom values file

Create `values-production.yaml`:

```yaml
replicaCount: 5
```

Install or upgrade:

```bash
helm upgrade --install webapp ./webapp \
  --namespace production \
  --create-namespace \
  -f values-production.yaml
```

### Method 3: Command-line override

```bash
helm upgrade --install webapp ./webapp \
  --namespace production \
  --set replicaCount=5
```

### Method 4: String override

When a value must be treated as a string:

```bash
helm upgrade --install webapp ./webapp \
  --namespace production \
  --set-string image.tag=1.2.3
```

### View configured values

```bash
helm get values webapp -n production
```

To inspect all computed values, including chart defaults:

```bash
helm get values webapp -n production --all
```

### Best practice

Use custom values files for complex or environment-specific configuration.

Use `--set` for simple overrides, especially during experimentation or when a pipeline supplies a small number of values.

Avoid exposing secrets through command-line arguments or shell history.

---

## 16. Upgrading a Helm Release

Helm upgrades an existing release when the chart or its configuration changes.

### General syntax

```bash
helm upgrade RELEASE_NAME CHART
```

Example:

```bash
helm upgrade webapp ./webapp
```

### Change replicas

```bash
helm upgrade webapp ./webapp \
  --namespace development \
  --set replicaCount=5
```

This assumes the chart template uses `.Values.replicaCount`.

### Upgrade using a values file

```bash
helm upgrade webapp ./webapp \
  --namespace production \
  -f values-production.yaml
```

### Install or upgrade with one command

```bash
helm upgrade --install webapp ./webapp \
  --namespace development \
  --create-namespace
```

This is useful in automation because the release is installed if it does not exist and upgraded if it already exists.

### Check the upgrade

```bash
helm status webapp -n development
helm history webapp -n development
kubectl get pods -n development
```

### Upgrade considerations

An upgrade can fail because of:

- Invalid chart templates.
- Invalid Kubernetes specifications.
- Image pull failures.
- Insufficient cluster resources.
- Missing configuration.
- RBAC restrictions.
- Immutable field changes.
- Application startup failures.
- Readiness failures.
- Dependency or API incompatibilities.

### Important note

Helm can successfully submit resources even when the application later fails an application-level test.

A production pipeline should include suitable post-deployment validation.

---

## 17. Helm Release History

Helm maintains release revisions.

### Display history

```bash
helm history webapp
```

With a namespace:

```bash
helm history webapp -n development
```

Example:

```text
REVISION  STATUS      CHART
1         superseded  webapp-0.1.0
2         superseded  webapp-0.1.0
3         deployed    webapp-0.2.0
```

The exact output depends on the release.

### What does revision mean?

A revision identifies a point in the release's history.

For example:

- Revision 1: Initial installation.
- Revision 2: First upgrade.
- Revision 3: Second upgrade.

A rollback creates another revision.

### Inspect release status

```bash
helm status webapp -n development
```

### Inspect release values

```bash
helm get values webapp -n development
```

### Inspect all computed values

```bash
helm get values webapp -n development --all
```

### Inspect rendered manifests

```bash
helm get manifest webapp -n development
```

### Inspect release notes

```bash
helm get notes webapp -n development
```

### Why is history useful?

It helps us:

- Identify previous revisions.
- Determine which revision was deployed.
- Investigate a failed upgrade.
- Select a revision for rollback.
- Understand release changes over time.

Release history is not a substitute for Git history, application logs, or monitoring.

---

## 18. Rolling Back a Helm Release

Rollback attempts to restore a release to an earlier revision.

### Step 1: Inspect the history

```bash
helm history webapp -n development
```

### Step 2: Identify the desired revision

Suppose revision 2 is the last known-good release.

### Step 3: Roll back

```bash
helm rollback webapp 2 \
  --namespace development
```

### Step 4: Verify the result

```bash
helm history webapp -n development
helm status webapp -n development
kubectl get pods -n development
```

### What happens during rollback?

Helm attempts to restore the release configuration associated with the selected revision.

It creates a new revision representing the rollback operation.

For example, rolling back to revision 2 from revision 3 can produce revision 4.

Revision 4 represents the rollback operation; it does not replace or erase revision 2.

### Rollback does not reverse everything

Helm rollback is not a universal undo mechanism.

It does not automatically reverse every possible external side effect, such as:

- Database schema migrations.
- Data written to external databases.
- Messages sent to external systems.
- Changes performed by external automation.
- All persistent data modifications.

Database migrations should be designed with compatibility and recovery strategies in mind.

### Production rollback strategy

1. Detect the incident.
2. Assess user impact.
3. Determine whether rollback is safe.
4. Select a known-good revision.
5. Roll back if appropriate.
6. Verify application health.
7. Investigate the root cause.
8. Correct the underlying issue.
9. Document the incident.

Do not delay restoring service when a known-safe rollback is the appropriate response to a serious incident.

---

## 19. Validating and Rendering Charts

Validating charts before deployment reduces avoidable failures.

### 19.1 helm lint

```bash
helm lint ./webapp
```

Checks the chart for common problems.

It is useful for detecting chart structure issues and many common template or configuration mistakes.

However, a successful lint does not guarantee that the chart will deploy successfully.

### 19.2 helm template

```bash
helm template webapp ./webapp
```

Renders the chart templates locally and prints the resulting Kubernetes manifests.

It does not install the application.

Use it to inspect:

- Generated resource names.
- Replica counts.
- Container images.
- Labels and selectors.
- Environment variables.
- Service ports.
- Resource requests and limits.
- Security configuration.

### Render using a namespace

```bash
helm template webapp ./webapp \
  --namespace production
```

### Render using production values

```bash
helm template webapp ./webapp \
  --namespace production \
  -f values-production.yaml
```

### Save the rendered manifests

```bash
helm template webapp ./webapp \
  --namespace production \
  -f values-production.yaml \
  > rendered-manifests.yaml
```

Inspect the output before deployment.

### 19.3 Dry run

```bash
helm upgrade --install webapp ./webapp \
  --namespace production \
  --dry-run
```

A dry run previews the operation without applying the release changes.

Check the documentation for your Helm version and the available dry-run options.

Do not expose rendered Secret data or other sensitive configuration in logs.

### 19.4 Validation workflow

```text
Chart Source
    |
    v
helm lint
    |
    v
helm template
    |
    v
Review Rendered Manifests
    |
    v
Dry Run / Additional Validation
    |
    v
Deploy
    |
    v
Verify Application Health
```

### Important distinction

`helm lint` checks the chart for common problems.

`helm template` renders the manifests.

A dry run previews the proposed release operation.

None of these replaces post-deployment health checks.

---

## 20. Waiting for Resources and Atomic Operations

### 20.1 The --wait option

Example:

```bash
helm upgrade --install webapp ./webapp \
  --namespace production \
  --create-namespace \
  --wait \
  --timeout 5m
```

Helm waits for supported Kubernetes resources to meet their readiness conditions, up to the specified timeout.

The timeout is not a guarantee that every application-level function will work.

### What --wait does not guarantee

It does not prove that:

- Every business workflow works.
- Every external dependency is healthy.
- The application is reachable from every client.
- Database operations are correct.
- The application meets performance requirements.

Use application-level tests and monitoring for those checks.

### 20.2 The --atomic option

Example:

```bash
helm upgrade --install webapp ./webapp \
  --namespace production \
  --create-namespace \
  --wait \
  --atomic \
  --timeout 5m
```

For an upgrade, `--atomic` attempts to roll back the failed upgrade automatically.

For an installation, Helm removes the failed release if installation fails.

In Helm 3, `--atomic` also enables waiting for resources to become ready.

### Important distinction

| Option | Purpose |
|---|---|
| --wait | Wait for supported resources to become ready |
| --timeout | Set the maximum time to wait for the operation |
| --atomic | Attempt automatic recovery when the install or upgrade fails |

### Production caution

Automatic rollback is helpful, but it is not a replacement for a good deployment strategy.

Consider:

- Whether the previous application version remains compatible with the database.
- Whether the upgrade modified persistent data.
- Whether rollback itself can fail.
- Whether external services changed.
- Whether the application needs smoke tests after deployment.

### Example production command

```bash
helm upgrade --install webapp ./webapp \
  --namespace production \
  --create-namespace \
  -f values-production.yaml \
  --wait \
  --atomic \
  --timeout 5m
```

Afterward, verify:

```bash
helm status webapp -n production
kubectl get pods -n production
kubectl get events -n production --sort-by=.metadata.creationTimestamp
```

Run the application's smoke tests as well.

---

## 21. Uninstalling a Helm Release

To uninstall a release:

```bash
helm uninstall webapp -n development
```

This removes the release and the Kubernetes resources managed by that release, subject to Kubernetes resource behavior and the specific resource types involved.

### Check whether the release remains

```bash
helm list -n development
```

### Important considerations

Uninstalling a release does not mean every external resource or side effect will be deleted.

Pay attention to:

- PersistentVolumeClaims.
- PersistentVolumes and reclaim policies.
- External databases.
- Cloud load balancers.
- Resources with retention policies.
- Resources not managed by the release.

Never assume that uninstalling an application is a safe way to clean up all its data.

Before uninstalling production workloads, identify the resources and data that must be preserved.

---

## 22. Helm Dependencies

A chart may depend on other charts.

For example, an application chart may define dependencies on a database or another supporting component.

Dependencies are commonly declared in `Chart.yaml`.

Example:

```yaml
apiVersion: v2
name: webapp
description: Example application chart
type: application
version: 0.1.0

dependencies:
  - name: example-dependency
    version: "1.2.3"
    repository: "https://charts.example.com"
```

The dependency name, version, and repository must correspond to an actual available chart.

### Download dependencies

```bash
helm dependency update ./webapp
```

This resolves the declared dependencies and updates the chart's dependency artifacts and lock information.

### Build dependencies from the lock file

```bash
helm dependency build ./webapp
```

This builds the dependencies using the chart's lock file when available.

### List dependencies

```bash
helm dependency list ./webapp
```

### Production best practices

- Pin dependency versions.
- Review dependency changes.
- Commit the dependency lock file.
- Use trusted chart sources.
- Test dependency upgrades.
- Avoid unnecessary chart dependencies.

For production systems, consider whether databases and other stateful services should be managed as part of the same chart or through a separate lifecycle.

---

## 23. Helm Troubleshooting

Helm troubleshooting requires checking both the Helm release and the Kubernetes resources.

### Scenario 1: Helm installation fails

Check:

```bash
helm status webapp -n development
helm history webapp -n development
```

Inspect Kubernetes resources:

```bash
kubectl get pods -n development
kubectl get events -n development --sort-by=.metadata.creationTimestamp
```

Check whether the failure was caused by invalid templates, permissions, resource constraints, or application startup issues.

### Scenario 2: Pod is in ImagePullBackOff

Check:

```bash
kubectl describe pod <pod-name> -n development
```

Possible causes:

- Incorrect image name.
- Invalid image tag.
- Private registry authentication failure.
- Registry connectivity problems.
- Image does not exist.

Inspect the configured image:

```bash
helm get manifest webapp -n development
```

Correct the chart values and upgrade the release.

### Scenario 3: Release is deployed, but Pods are not ready

Check:

```bash
kubectl get pods -n development
kubectl describe pod <pod-name> -n development
kubectl logs <pod-name> -n development
kubectl get events -n development --sort-by=.metadata.creationTimestamp
```

Possible causes:

- Failed readiness probe.
- Application startup error.
- Missing ConfigMap or Secret.
- Insufficient CPU or memory.
- Incorrect command or arguments.
- Container port or application configuration problems.

### Scenario 4: Wrong replica count

Check the configured values:

```bash
helm get values webapp -n development
```

Inspect the rendered Deployment:

```bash
helm get manifest webapp -n development
```

Check the actual Deployment:

```bash
kubectl get deployment -n development
```

If necessary, correct the configuration and upgrade:

```bash
helm upgrade webapp ./webapp \
  --namespace development \
  --set replicaCount=4
```

### Scenario 5: Upgrade fails

Check the release history:

```bash
helm history webapp -n development
```

Inspect the release status:

```bash
helm status webapp -n development
```

Check events and Pod status.

If the previous revision is known to be healthy and rollback is appropriate:

```bash
helm rollback webapp 2 -n development
```

Replace `2` with the actual target revision.

### Scenario 6: Release already exists

An installation may fail because a release with the same name already exists in the namespace.

Check:

```bash
helm list -n development
```

If you want an install-or-upgrade workflow, consider:

```bash
helm upgrade --install webapp ./webapp \
  --namespace development
```

Do not use this command blindly if the existing release may belong to another deployment process.

### Scenario 7: Resource already exists

Helm may encounter a resource that already exists but is not managed as expected by the release.

Investigate:

- The resource's namespace.
- Its ownership metadata.
- Whether another Helm release manages it.
- Whether another deployment tool created it.
- Whether the resource should be shared or managed separately.

Do not delete production resources merely to make the installation succeed.

### Scenario 8: Template rendering fails

Run:

```bash
helm lint ./webapp
helm template webapp ./webapp
```

Check:

- YAML indentation.
- Missing values.
- Incorrect template expressions.
- Invalid functions.
- Missing named template helpers.
- Incorrect resource structure.

### Troubleshooting principle

Do not investigate only the Helm command output.

Inspect both:

1. The Helm release and rendered configuration.
2. The actual Kubernetes resources and application behavior.

---

## 24. Production Best Practices

### 24.1 Use Git for version control

Store chart source and non-sensitive configuration in Git.

Example:

```text
repository/
├── charts/
│   └── webapp/
├── environments/
│   ├── development/
│   │   └── values.yaml
│   └── production/
│       └── values.yaml
└── README.md
```

The exact structure depends on the team's deployment strategy.

### 24.2 Separate environment configuration

Use environment-specific values files.

```text
values-development.yaml
values-testing.yaml
values-production.yaml
```

Avoid unnecessary duplication of chart templates.

### 24.3 Pin versions

Pin chart dependencies and application image versions.

Avoid using mutable tags such as `latest` in production unless the deployment system provides an explicit and reliable control mechanism.

### 24.4 Validate before deployment

Use:

```bash
helm lint ./webapp
helm template webapp ./webapp
```

Add schema validation, policy checks, and other automated checks where appropriate.

### 24.5 Use resource requests and limits

Configure CPU and memory appropriately for the workload.

Do not copy arbitrary values into production without testing and monitoring.

### 24.6 Configure health probes

Use appropriate readiness, liveness, and startup probes.

Remember that these probes serve different purposes.

### 24.7 Follow least privilege

Restrict access to:

- Kubernetes namespaces.
- Release Secrets.
- ServiceAccounts.
- Cloud credentials.
- Deployment pipelines.

Do not give a Helm deployment pipeline cluster-admin access unless there is a justified and carefully controlled requirement.

### 24.8 Protect secrets

Avoid committing plaintext credentials to Git.

Avoid exposing secret values in:

- Pipeline logs.
- Terminal output.
- Debug artifacts.
- Rendered manifests shared with unauthorized users.

Use a suitable secret-management system.

### 24.9 Plan rollback carefully

Maintain compatible application and database changes.

Rollback cannot automatically reverse every external side effect.

### 24.10 Verify deployments

After deployment, check:

```bash
helm status webapp -n production
kubectl get pods -n production
kubectl get events -n production --sort-by=.metadata.creationTimestamp
```

Then run smoke tests and monitor application metrics and logs.

### 24.11 Integrate Helm with CI/CD

A typical workflow:

```text
Developer Pushes Code
          |
          v
       Git Repository
          |
          v
     CI/CD Pipeline
          |
          v
     Build Application
          |
          v
    Build Container Image
          |
          v
  Push Image to Registry
          |
          v
    Validate Helm Chart
          |
          v
 Render and Review Manifests
          |
          v
    Deploy Using Helm
          |
          v
  Verify Application Health
          |
          v
   Monitor the Application
```

In a GitOps workflow, a controller such as Argo CD can manage synchronization of the desired application configuration. Helm may be used to render the chart as part of that workflow.

Helm and Argo CD serve related but different purposes:

- Helm provides chart packaging and templating.
- Argo CD continuously compares desired configuration with the cluster and reconciles changes according to its configuration.

---

## 25. Hands-On Lab

This lab creates a local Helm chart and practices installation, configuration, upgrades, history, rollback, and cleanup.

### Prerequisites

- Helm installed.
- kubectl installed.
- Access to a Kubernetes cluster.
- Sufficient permissions to create resources in the test namespace.

### Lab 1: Verify Helm

```bash
helm version
kubectl cluster-info
kubectl get nodes
```

### Lab 2: Create a chart

```bash
helm create webapp
```

Inspect the structure:

```bash
ls -R webapp
```

### Lab 3: Validate the chart

```bash
helm lint ./webapp
```

### Lab 4: Render the chart

```bash
helm template webapp ./webapp \
  --namespace development
```

Inspect the generated manifests.

### Lab 5: Create a namespace and install the chart

```bash
helm install webapp ./webapp \
  --namespace development \
  --create-namespace
```

Check the release:

```bash
helm list -n development
helm status webapp -n development
```

### Lab 6: Inspect the Kubernetes resources

```bash
kubectl get pods -n development
kubectl get deployments -n development
kubectl get services -n development
```

### Lab 7: Change the replica count

First, inspect the default value in `webapp/values.yaml`.

The starter chart normally defines `replicaCount`.

Upgrade the release:

```bash
helm upgrade webapp ./webapp \
  --namespace development \
  --set replicaCount=4
```

Check the Deployment:

```bash
kubectl get deployments -n development
kubectl get pods -n development
```

### Lab 8: Inspect the release history

```bash
helm history webapp -n development
```

Record the current revision and the previous revision.

### Lab 9: Perform another upgrade

For example:

```bash
helm upgrade webapp ./webapp \
  --namespace development \
  --set replicaCount=2
```

Check the history again:

```bash
helm history webapp -n development
```

### Lab 10: Roll back

Identify a revision that corresponds to the earlier configuration.

Then run:

```bash
helm rollback webapp <revision-number> \
  --namespace development
```

Replace `<revision-number>` with the actual revision number.

Verify:

```bash
helm history webapp -n development
helm status webapp -n development
kubectl get deployments -n development
```

### Lab 11: Inspect values and manifests

```bash
helm get values webapp -n development
helm get values webapp -n development --all
helm get manifest webapp -n development
```

Compare the values with the rendered resources.

### Lab 12: Clean up

```bash
helm uninstall webapp -n development
```

Check whether the namespace is empty:

```bash
kubectl get all -n development
```

Delete the namespace only if it was created for this lab and no other workloads depend on it:

```bash
kubectl delete namespace development
```

### Lab completion criteria

You should be able to:

- Create a chart.
- Explain the generated directory structure.
- Validate and render a chart.
- Install a release.
- Change the replica count.
- Inspect the release history.
- Roll back to a previous revision.
- Verify the resulting Kubernetes resources.
- Uninstall the release safely.

---

## 26. Useful Helm Commands

### Version and environment

```bash
helm version
helm env
kubectl config current-context
```

### Chart creation and validation

```bash
helm create webapp
helm lint ./webapp
helm template webapp ./webapp
```

### Repository management

```bash
helm repo list
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo nginx
helm search repo nginx --versions
```

### Installation

```bash
helm install webapp ./webapp
helm install webapp ./webapp -n development
helm install webapp ./webapp -n development --create-namespace
```

### Install or upgrade

```bash
helm upgrade --install webapp ./webapp \
  -n development \
  --create-namespace
```

### Release inspection

```bash
helm list
helm list -A
helm status webapp -n development
helm history webapp -n development
```

### Values and manifests

```bash
helm get values webapp -n development
helm get values webapp -n development --all
helm get manifest webapp -n development
helm get notes webapp -n development
```

### Upgrade and rollback

```bash
helm upgrade webapp ./webapp -n development
helm rollback webapp 2 -n development
```

### Uninstall

```bash
helm uninstall webapp -n development
```

### Dependencies

```bash
helm dependency list ./webapp
helm dependency update ./webapp
helm dependency build ./webapp
```

### Troubleshooting with Kubernetes

```bash
kubectl get pods -n development
kubectl describe pod <pod-name> -n development
kubectl logs <pod-name> -n development
kubectl get events -n development --sort-by=.metadata.creationTimestamp
```

---

## 27. Common Mistakes

### Mistake 1: Assuming repository registration installs an application

Incorrect understanding:

`helm repo add` installs the application.

Correct understanding:

It registers a repository. Use `helm install` to install a chart.

### Mistake 2: Assuming one chart means one release

A chart can be installed multiple times, creating separate releases.

### Mistake 3: Assuming all releases update automatically

Each release must be upgraded or managed by an automation system.

### Mistake 4: Confusing chart version with application version

`version` identifies the chart package version.

`appVersion` describes the application version represented by the chart.

### Mistake 5: Assuming helm template deploys resources

`helm template` renders manifests locally. It does not install the release.

### Mistake 6: Assuming --wait guarantees application correctness

`--wait` waits for supported Kubernetes resource readiness conditions. It does not replace application-level tests.

### Mistake 7: Assuming rollback reverses database changes

Helm rollback does not automatically reverse database migrations or every external side effect.

### Mistake 8: Assuming uninstall deletes all application data

Persistent storage and external resources require separate consideration.

### Mistake 9: Storing secrets in Git

Use appropriate secret-management practices instead of committing plaintext credentials.

### Mistake 10: Deploying without inspecting rendered manifests

Always review chart output and validate changes before deploying to production.

---

## 28. Quick Revision

### What is Helm?

A package manager for Kubernetes.

### What is a Helm chart?

A package containing templates, values, metadata, and optional dependencies.

### What is a release?

An installed instance of a Helm chart.

### What is a repository?

A location for distributing Helm charts.

### What does helm create do?

Creates a starter chart directory.

### What does Chart.yaml contain?

Chart metadata.

### What does values.yaml contain?

Default configuration values.

### What is templates/?

The directory containing resource templates.

### What is _helpers.tpl?

A file for reusable template helpers.

### What does helm lint do?

Checks the chart for common issues.

### What does helm template do?

Renders the chart into Kubernetes manifests without installing it.

### What does helm install do?

Installs a chart as a release.

### What does helm upgrade do?

Upgrades an existing release.

### What does helm history do?

Displays the release's revision history.

### What does helm rollback do?

Attempts to restore a previous release revision.

### What does --wait do?

Waits for supported Kubernetes resources to meet their readiness conditions.

### What does --atomic do?

Attempts automatic recovery when an installation or upgrade fails.

### What does helm uninstall do?

Uninstalls a release and removes its managed resources, subject to resource-specific behavior.

---

## 29. Day 11 Completion Checklist

Mark each item only after you can explain it or perform it yourself.

- [ ] Explain Helm and why it is useful.
- [ ] Differentiate Helm from kubectl.
- [ ] Explain chart, release, repository, and revision.
- [ ] Explain Helm 3 architecture and the absence of Tiller.
- [ ] Explain Chart.yaml.
- [ ] Explain values.yaml.
- [ ] Explain templates/ and _helpers.tpl.
- [ ] Create a Helm chart.
- [ ] Add and update a chart repository.
- [ ] Install a chart into a namespace.
- [ ] Override configuration values.
- [ ] Upgrade a release.
- [ ] Inspect release history.
- [ ] Roll back to a previous revision.
- [ ] Validate a chart with helm lint.
- [ ] Render manifests with helm template.
- [ ] Explain --wait, --timeout, and --atomic.
- [ ] Troubleshoot a failed deployment.
- [ ] Explain production security considerations.
- [ ] Complete the hands-on lab.

---

## Final Takeaway

Helm makes Kubernetes application deployment more reusable, configurable, and manageable.

However, understanding Helm is not just about memorizing commands.

A production-ready Kubernetes engineer must understand:

- How chart values become Kubernetes manifests.
- How releases and revisions are managed.
- How to validate changes before deployment.
- How to investigate failed upgrades.
- When rollback is appropriate.
- Why rollback cannot reverse every external change.
- How to verify application health after deployment.
- How to integrate Helm with CI/CD and GitOps workflows.

**Do not just learn how to install a Helm chart. Learn how to operate, troubleshoot, upgrade, and recover an application deployed through Helm.**
