# Kubernetes Day 11: Helm — Interview Questions and Answers

## Table of Contents

1. [Question 1: What Is Helm?](#question-1-what-is-helm-in-kubernetes-and-why-do-we-need-it)
2. [Question 2: Helm Chart Structure](#question-2-what-happens-when-you-run-helm-create-webapp)
3. [Question 3: Helm Repositories](#question-3-explain-helm-repositories-and-the-installation-workflow)
4. [Question 4: Helm Upgrade and Rollback](#question-4-how-would-you-troubleshoot-a-failed-helm-upgrade-and-roll-back)
5. [Question 5: Helm in Production](#question-5-how-would-you-deploy-a-helm-chart-safely-in-production)
6. [Additional Interview Questions](#6-additional-interview-questions)
7. [Production Scenario Questions](#7-production-scenario-questions)
8. [Common Interview Mistakes](#8-common-interview-mistakes)
9. [Quick Revision Questions](#9-quick-revision-questions)
10. [Final Revision Checklist](#10-final-revision-checklist)

---

# Question 1: What is Helm in Kubernetes, and why do we need it?

## Short Answer

Helm is a package manager for Kubernetes. It simplifies application deployment and lifecycle management by packaging Kubernetes resource templates, configuration values, and metadata into reusable charts.

A chart can be installed as a release. Helm supports installations, upgrades, release history, and rollbacks.

## Detailed Explanation

A Kubernetes application may require multiple resources:

- Deployment
- Service
- ConfigMap
- Secret
- Ingress
- ServiceAccount
- PersistentVolumeClaim
- HorizontalPodAutoscaler

Maintaining these manifests independently for different environments can become difficult.

Helm allows us to organize the application configuration into a reusable chart.

The chart contains templates that Helm renders using the supplied values.

For example:

```bash
helm install webapp ./webapp \
  --namespace development \
  --create-namespace
```

This installs the chart as a release named `webapp` in the `development` namespace.

## Important Terminology

| Term | Meaning |
|---|---|
| Helm | Kubernetes package manager |
| Chart | Package containing templates, values, and metadata |
| Release | Installed instance of a chart |
| Repository | Location that distributes Helm charts |
| Revision | A point in a release's history |

## Helm vs kubectl

`kubectl apply` applies Kubernetes manifests.

```bash
kubectl apply -f ./manifests/
```

Helm provides chart templating and release management.

```bash
helm upgrade --install webapp ./webapp \
  --namespace development \
  --create-namespace
```

Helm does not eliminate kubectl. In real environments, engineers commonly use Helm to manage releases and kubectl to inspect and troubleshoot the resulting resources.

## Example: Multiple Environments

A team wants to deploy the same application in development and production.

Development:

```bash
helm upgrade --install webapp ./webapp \
  --namespace development \
  --create-namespace \
  -f values-development.yaml
```

Production:

```bash
helm upgrade --install webapp ./webapp \
  --namespace production \
  --create-namespace \
  -f values-production.yaml
```

Both releases use the same chart but have different configuration.

Changing the chart does not automatically update every release. Each release must be upgraded separately or managed by automation.

## Production Considerations

- Store chart source in Git.
- Use environment-specific values files.
- Pin application image versions.
- Validate the chart before deployment.
- Protect sensitive configuration.
- Verify application health after deployment.

## Common Interview Mistake

Incorrect:

"Helm automatically updates all namespaces when I change values in one chart."

Correct:

"Each Helm release has its own configuration. I must upgrade each release separately or use automation to manage the releases."

## Model Interview Answer

Helm is a package manager for Kubernetes that simplifies deploying and managing applications. A Helm chart contains reusable Kubernetes resource templates, configuration values, and metadata. An installed instance of a chart is called a release.

Helm allows us to reuse the same chart across environments by providing different values files. It also supports upgrades, release history, and rollbacks. In production, we commonly store charts in Git and deploy them through CI/CD or GitOps workflows.

---

# Question 2: What happens when you run helm create webapp?

## Short Answer

The command creates a starter Helm chart directory named `webapp`, containing chart metadata, default values, templates, and supporting files.

```bash
helm create webapp
```

## Typical Chart Structure

```text
webapp/
├── Chart.yaml
├── values.yaml
├── charts/
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── serviceaccount.yaml
│   ├── _helpers.tpl
│   └── NOTES.txt
└── .helmignore
```

The exact generated files may vary by Helm version.

## Chart.yaml

Contains chart metadata.

Example:

```yaml
apiVersion: v2
name: webapp
description: A Helm chart for a web application
type: application
version: 0.1.0
appVersion: "1.0.0"
```

Important fields:

- `name`: Chart name.
- `version`: Chart package version.
- `appVersion`: Application version represented by the chart.
- `description`: Description of the chart.

The chart version and application version are different concepts.

## values.yaml

Contains default configuration values.

Example:

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.28"
  pullPolicy: IfNotPresent
```

The values are used only when chart templates reference them.

## templates/

Contains the Kubernetes resource templates.

Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
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

Helm renders the template using the configured values.

## _helpers.tpl

Contains reusable template helpers.

For example, helpers can define:

- Common labels.
- Resource names.
- Chart labels.
- Reusable naming logic.

Files beginning with an underscore are not rendered as standalone Kubernetes resource manifests.

## How Do You Change Replicas Without Editing the Deployment Template?

Suppose the template contains:

```yaml
replicas: {{ .Values.replicaCount }}
```

We can override the value:

```bash
helm upgrade --install webapp ./webapp \
  --namespace development \
  --create-namespace \
  --set replicaCount=5
```

Alternatively, create a custom values file:

```yaml
replicaCount: 5
```

Then run:

```bash
helm upgrade --install webapp ./webapp \
  --namespace development \
  -f values-development.yaml
```

## Model Interview Answer

Running `helm create webapp` creates a starter chart directory.

`Chart.yaml` contains chart metadata. `values.yaml` contains default configuration. The `templates/` directory contains Kubernetes resource templates, and `_helpers.tpl` defines reusable template helpers.

To change the replica count without modifying the Deployment template, I can override `replicaCount` using `--set` or provide a custom values file during `helm upgrade`.

---

# Question 3: Explain Helm repositories and the installation workflow

Consider the commands:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami

helm repo update

helm install my-app bitnami/nginx \
  --namespace development \
  --create-namespace
```

## Command 1: helm repo add

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Registers a Helm repository under the alias `bitnami`.

The repository distributes Helm charts.

Adding a repository does not install an application.

## Command 2: helm repo update

```bash
helm repo update
```

Refreshes local metadata for configured Helm repositories.

It allows Helm to discover available charts and versions.

It does not install an application or download all the application's container images.

## Command 3: helm install

```bash
helm install my-app bitnami/nginx \
  --namespace development \
  --create-namespace
```

This command:

- Installs the `nginx` chart from the `bitnami` repository.
- Creates a release named `my-app`.
- Places the release in the `development` namespace.
- Creates the namespace if it does not already exist.

The chart must be available at the specified source, and the cluster must be able to pull the required container images.

## What Is a Helm Repository?

A Helm repository is a location used to distribute Helm charts.

A chart contains templates, values, metadata, and potentially dependencies.

The chart templates commonly reference container images stored in a container registry.

## Repository vs Chart vs Container Registry

| Component | Purpose |
|---|---|
| Helm repository | Distributes Helm charts |
| Helm chart | Defines application resources through templates and values |
| Container registry | Stores and distributes container images |
| Kubernetes cluster | Runs the application workloads |

## How Do You Search for a Chart?

```bash
helm search repo nginx
```

To inspect available versions:

```bash
helm search repo nginx --versions
```

## How Do You Verify the Release?

```bash
helm list -n development
helm status my-app -n development
kubectl get pods -n development
```

## Model Interview Answer

A Helm repository is a location where Helm charts are distributed.

`helm repo add` registers the repository, and `helm repo update` refreshes its local metadata. Neither command installs an application.

`helm install` installs the specified chart as a release. In this example, the release is named `my-app` and is installed into the `development` namespace. The `--create-namespace` option creates that namespace if necessary.

---

# Question 4: How would you troubleshoot a failed Helm upgrade and roll back?

Suppose we deployed an application using:

```bash
helm install my-app ./webapp
```

Later, we upgraded it:

```bash
helm upgrade my-app ./webapp
```

The new version causes problems.

## Step 1: Inspect Release History

```bash
helm history my-app -n development
```

This shows the release revisions and their statuses.

Example:

```text
REVISION  STATUS      CHART
1         superseded  webapp-0.1.0
2         superseded  webapp-0.1.0
3         deployed    webapp-0.2.0
```

The actual revision numbers and statuses depend on the release.

## Step 2: Inspect Release Status

```bash
helm status my-app -n development
```

This provides information about the release's current status.

## Step 3: Inspect Kubernetes Resources

```bash
kubectl get pods -n development
kubectl get deployments -n development
kubectl get events -n development --sort-by=.metadata.creationTimestamp
```

If a Pod is unhealthy:

```bash
kubectl describe pod <pod-name> -n development
```

Inspect application logs:

```bash
kubectl logs <pod-name> -n development
```

## Step 4: Inspect Helm Configuration

Check configured values:

```bash
helm get values my-app -n development
```

Inspect the rendered release manifests:

```bash
helm get manifest my-app -n development
```

Look for:

- Incorrect image tags.
- Invalid environment variables.
- Incorrect resource settings.
- Missing configuration.
- Incorrect Service selectors or ports.
- Probe failures.
- Application startup errors.

## Step 5: Decide Whether to Roll Back

If the previous revision is known to be healthy and the new release is causing a serious incident, rollback may be the safest recovery action.

Suppose revision 2 is the known-good revision.

Run:

```bash
helm rollback my-app 2 \
  --namespace development
```

Replace `2` with the actual revision number.

## Step 6: Verify the Rollback

```bash
helm history my-app -n development
helm status my-app -n development
kubectl get pods -n development
```

Run appropriate application-level health checks.

## Important: Rollback Creates a New Revision

Suppose the release has revisions 1, 2, and 3.

Rolling back to revision 2 can create revision 4.

Revision 4 represents the rollback operation. Helm does not simply delete revision 3 or erase the historical record.

## What Does Helm Rollback Not Do?

Helm rollback does not automatically reverse all external side effects.

Examples include:

- Database schema changes.
- Data written to an external database.
- Messages already sent to another system.
- Changes made by external automation.
- All persistent data modifications.

## Production Troubleshooting Approach

A good production response should consider:

1. User impact and incident severity.
2. Current release status.
3. Whether the previous revision is known to be healthy.
4. Whether rollback is safe.
5. Database and external dependency compatibility.
6. Post-rollback application health.
7. Root-cause analysis after service recovery.

Do not delay recovery unnecessarily during a serious outage when a known-safe rollback is available.

## Model Interview Answer

I would inspect the release history and status using `helm history` and `helm status`. I would then inspect Pod status, Kubernetes events, application logs, and the rendered manifests to identify the cause.

If the new version is causing a serious incident and the previous revision is known to be healthy, I would roll back using:

```bash
helm rollback my-app 2 -n development
```

After rollback, I would verify the release status, Pod readiness, and application functionality. I would also check whether the upgrade introduced database or external changes that Helm cannot reverse automatically.

---

# Question 5: How would you deploy a Helm chart safely in production?

A production deployment should validate the chart, inspect the rendered manifests, deploy with appropriate safeguards, and verify the application afterward.

## Step 1: Validate the Chart

```bash
helm lint ./webapp
```

This checks the chart for common problems.

A successful lint does not guarantee a successful deployment.

## Step 2: Render the Manifests

```bash
helm template my-app ./webapp \
  --namespace production
```

This renders the chart templates locally.

It does not deploy the resources.

Inspect the generated YAML for:

- Correct image tags.
- Correct replica counts.
- Resource requests and limits.
- Security configuration.
- Service ports.
- Labels and selectors.
- ConfigMaps and Secret references.

## Step 3: Preview the Operation

```bash
helm upgrade --install my-app ./webapp \
  --namespace production \
  --dry-run
```

This previews the release operation without applying the release changes.

Check the available dry-run options for your Helm version.

Be careful not to expose sensitive configuration in logs or shared output.

## Step 4: Deploy with Readiness Waiting and Automatic Recovery

```bash
helm upgrade --install my-app ./webapp \
  --namespace production \
  --create-namespace \
  -f values-production.yaml \
  --wait \
  --atomic \
  --timeout 5m
```

### Explanation of the options

#### --wait

Waits for supported Kubernetes resources to meet their readiness conditions.

#### --timeout 5m

Sets the maximum time to wait for the operation.

#### --atomic

For an upgrade, attempts to roll back the failed upgrade automatically.

For an installation, removes the failed release if installation fails.

In Helm 3, `--atomic` also enables waiting for resources to become ready.

## Step 5: Verify the Deployment

```bash
helm status my-app -n production
helm history my-app -n production
kubectl get pods -n production
kubectl get events -n production --sort-by=.metadata.creationTimestamp
```

Then run smoke tests and application-level health checks.

## Step 6: Roll Back if Necessary

If the deployment remains unhealthy and automatic recovery did not restore the expected state, investigate the release history.

```bash
helm history my-app -n production
```

Select the appropriate known-good revision:

```bash
helm rollback my-app 2 -n production
```

Replace `2` with the actual revision number.

## Important Production Considerations

- A ready Pod does not guarantee correct business functionality.
- Rollback does not reverse every database migration.
- Persistent data needs a separate recovery strategy.
- Application-level smoke tests are important.
- Deployment credentials should follow least privilege.
- Secrets should not be exposed in CI/CD logs.
- Chart and image versions should be controlled.
- Rollback itself can fail and must be verified.

## Model Interview Answer

I would start by validating the chart using `helm lint` and rendering the manifests using `helm template`.

I would review the generated configuration and perform an appropriate dry run. Then I would deploy using `helm upgrade --install` with `--wait`, `--atomic`, and a suitable timeout.

After deployment, I would check the Helm release status, Kubernetes resource readiness, and application health. I would use `helm history` to inspect revisions and `helm rollback` if recovery to a previous revision were necessary.

I would also account for database compatibility, external dependencies, and persistent data because Helm rollback cannot automatically reverse every external change.

---

# 6. Additional Interview Questions

## Q6. What is the difference between a Helm chart and a Helm release?

A chart is the application package.

A release is an installed instance of that chart.

One chart can be installed multiple times using different release names, namespaces, and configuration values.

Example:

```bash
helm install webapp-dev ./webapp -n development
helm install webapp-prod ./webapp -n production
```

These are separate releases using the same chart.

## Q7. What is the difference between version and appVersion in Chart.yaml?

`version` identifies the Helm chart package version.

`appVersion` describes the application version represented by the chart.

The chart version may change even when the application version remains the same.

The application version may also change without necessarily requiring a chart version change under every possible workflow, although chart versioning should follow the project's release policy.

## Q8. What is the difference between helm install and helm upgrade?

`helm install` creates a new release.

`helm upgrade` updates an existing release.

For automated workflows:

```bash
helm upgrade --install webapp ./webapp \
  -n development
```

This installs the release if it does not exist or upgrades it if it already exists.

## Q9. What is the difference between helm template and helm install?

`helm template` renders the chart templates locally and displays the generated Kubernetes manifests.

`helm install` installs the chart as a release and submits the resulting resources to Kubernetes.

## Q10. What does helm history show?

It displays the revisions associated with a release.

It helps identify earlier configurations and select a revision for rollback.

```bash
helm history webapp -n development
```

## Q11. What does helm rollback do?

It attempts to restore the release configuration associated with an earlier revision.

```bash
helm rollback webapp 2 -n development
```

The rollback operation creates another revision.

## Q12. What is the purpose of --wait?

It makes Helm wait for supported Kubernetes resources to satisfy their readiness conditions.

It does not guarantee that every application function works correctly.

## Q13. What is the purpose of --atomic?

It enables automatic recovery when an installation or upgrade fails.

For an upgrade, Helm attempts to roll back the failed upgrade. For a failed installation, Helm removes the failed release.

## Q14. What is the difference between a Helm repository and a container registry?

A Helm repository distributes charts.

A container registry stores and distributes container images.

A chart commonly references images stored in a container registry.

## Q15. Does Helm replace kubectl?

No.

Helm manages application charts and releases. kubectl provides direct access to Kubernetes resources and is essential for troubleshooting.

## Q16. Does Helm rollback reverse a database migration?

Not automatically.

Database migrations and other external side effects require their own recovery and compatibility strategies.

## Q17. Where is Helm release information stored?

In the default Helm 3 configuration, release information is stored in Kubernetes Secrets in the release namespace.

Helm supports other storage drivers as well.

## Q18. What is _helpers.tpl?

It contains reusable template helpers, such as common labels and resource naming functions.

It reduces duplicated template logic.

## Q19. How do you inspect the values used by a release?

```bash
helm get values webapp -n development
```

To include chart defaults in the computed values:

```bash
helm get values webapp -n development --all
```

## Q20. How do you inspect the manifests associated with a release?

```bash
helm get manifest webapp -n development
```

This is useful when investigating what Helm rendered and stored for the release.

---

# 7. Production Scenario Questions

## Scenario 1: The Helm command succeeded, but the application is unavailable.

### Investigation

Check the release:

```bash
helm status webapp -n production
```

Check Pods and Services:

```bash
kubectl get pods -n production
kubectl get services -n production
```

Inspect events:

```bash
kubectl get events -n production --sort-by=.metadata.creationTimestamp
```

Inspect Pod details and logs:

```bash
kubectl describe pod <pod-name> -n production
kubectl logs <pod-name> -n production
```

Investigate readiness probes, Services, endpoints, application configuration, and network routing.

A successful Helm operation does not necessarily mean that the application is reachable or functioning correctly.

## Scenario 2: The new version causes a production outage.

### Response

1. Assess the impact.
2. Inspect release history.
3. Determine whether the previous revision is known to be healthy.
4. Roll back if appropriate.
5. Verify application health.
6. Investigate the root cause.
7. Correct the issue and plan a safe redeployment.

Commands:

```bash
helm history webapp -n production
helm rollback webapp <revision-number> -n production
helm status webapp -n production
```

Do not blindly roll back if the previous version is incompatible with the current database or external state.

## Scenario 3: You need five replicas in production and one in development.

Use the same chart with different values files.

Development:

```yaml
replicaCount: 1
```

Production:

```yaml
replicaCount: 5
```

Deploy each release separately:

```bash
helm upgrade --install webapp ./webapp \
  -n development \
  --create-namespace \
  -f values-development.yaml
```

```bash
helm upgrade --install webapp ./webapp \
  -n production \
  --create-namespace \
  -f values-production.yaml
```

## Scenario 4: A release upgrade fails because a Pod cannot pull its image.

Investigate:

```bash
kubectl describe pod <pod-name> -n production
```

Inspect the release manifest:

```bash
helm get manifest webapp -n production
```

Check the image repository, tag, registry credentials, and network connectivity.

Correct the configuration and upgrade the release again.

If the release causes a serious incident and the previous revision is safe, consider rollback.

## Scenario 5: A team stores passwords in values.yaml.

This is a security concern if the file is committed to Git or exposed through CI/CD logs.

Recommended approach:

- Remove plaintext credentials from version-controlled files.
- Rotate credentials if they may have been exposed.
- Use an appropriate secret-management system.
- Restrict Kubernetes RBAC access.
- Avoid printing sensitive values in pipeline output.
- Review Helm release storage access.

A Kubernetes Secret is not automatically safe merely because it is a Secret resource. Access control and encryption at rest matter.

---

# 8. Common Interview Mistakes

## Mistake 1

Saying that `helm repo add` installs the application.

Correct: It registers a repository.

## Mistake 2

Saying that Helm automatically updates every release when a chart changes.

Correct: Each release must be upgraded or managed by automation.

## Mistake 3

Saying that `helm template` deploys Kubernetes resources.

Correct: It renders manifests locally.

## Mistake 4

Saying that `--wait` guarantees the application works.

Correct: It waits for supported Kubernetes resource readiness conditions.

## Mistake 5

Saying that rollback completely reverses a deployment.

Correct: Rollback attempts to restore a previous release configuration but does not reverse every external side effect.

## Mistake 6

Confusing a chart with a release.

Correct: A chart is a package; a release is an installed instance.

## Mistake 7

Confusing chart version with application version.

Correct: `version` identifies the chart package, while `appVersion` describes the application version represented by the chart.

## Mistake 8

Treating Helm as a replacement for Git.

Correct: Git provides source control; Helm manages chart-based deployments and release revisions.

---

# 9. Quick Revision Questions

Try answering these without looking at the explanations.

1. What is Helm?
2. Why do we need Helm?
3. What is a Helm chart?
4. What is a Helm release?
5. What is a Helm repository?
6. What is the difference between a chart and a release?
7. What does `helm create webapp` do?
8. What is the purpose of Chart.yaml?
9. What is the purpose of values.yaml?
10. What is the purpose of templates/?
11. What is _helpers.tpl?
12. What is the difference between chart version and appVersion?
13. What does `helm repo add` do?
14. What does `helm repo update` do?
15. What does `helm install` do?
16. What does `helm upgrade` do?
17. What is the purpose of `helm history`?
18. How do you roll back to a previous revision?
19. What does `helm lint` do?
20. What does `helm template` do?
21. What does `--wait` do?
22. What does `--atomic` do?
23. Where is Helm 3 release information commonly stored?
24. Does Helm rollback reverse database migrations automatically?
25. How would you troubleshoot a failed Helm upgrade?

---

# 10. Final Revision Checklist

Before considering Helm interview preparation complete, verify that you can:

- [ ] Explain Helm in simple terms.
- [ ] Explain chart, release, repository, and revision.
- [ ] Explain the purpose of Chart.yaml.
- [ ] Explain the purpose of values.yaml.
- [ ] Explain the purpose of templates/ and _helpers.tpl.
- [ ] Create a chart using helm create.
- [ ] Validate a chart using helm lint.
- [ ] Render manifests using helm template.
- [ ] Install a release in a specific namespace.
- [ ] Override values using --set and a values file.
- [ ] Upgrade a release.
- [ ] Inspect release history.
- [ ] Roll back to a previous revision.
- [ ] Explain --wait and --atomic.
- [ ] Troubleshoot image pull failures and unhealthy Pods.
- [ ] Explain Helm's limitations around external side effects.
- [ ] Explain how Helm integrates with CI/CD.
- [ ] Explain how Helm can be used in a GitOps workflow.
- [ ] Complete a hands-on installation, upgrade, and rollback.

---

## Final Interview Takeaway

Helm simplifies Kubernetes application packaging and deployment, but a production DevOps engineer must understand more than the commands.

You should be able to explain how chart values become Kubernetes manifests, how releases and revisions work, how to validate changes, how to troubleshoot failed upgrades, and how to recover safely.

**A strong Helm engineer knows not only how to deploy an application, but also how to verify, upgrade, troubleshoot, and recover it safely in production.**
