Kubernetes Day 5 --- Interview

1. What is the difference between a ConfigMap and a Secret?

A ConfigMap stores non-sensitive configuration such as application
settings, URLs, log levels, and feature flags.

A Secret is intended for sensitive information such as passwords,
API keys, tokens, database credentials, and TLS material.

A Secret is not automatically secure. Production security also requires
appropriate RBAC, encryption at rest, access control, and secure
secret-management practices.

2. How can you consume a ConfigMap or Secret inside a Pod?

There are two common methods:

Environment variables

A specific ConfigMap key:

env:
  - name: APP_ENV
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: APP_ENV

A specific Secret key:

env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-secret
        key: DB_PASSWORD

All suitable ConfigMap keys:

envFrom:
  - configMapRef:
      name: app-config

All suitable Secret keys:

envFrom:
  - secretRef:
      name: app-secret

Volume mounts

ConfigMaps and Secrets can also be mounted as files.

This is useful when the application expects configuration or credentials
as files.

Important difference

Environment variables are established when the container starts. A
changed Secret does not automatically change the environment of an
already-running container.

Mounted ConfigMap/Secret files can be updated by Kubernetes after the
underlying object changes, but the application must support rereading or
reloading the file.

3. Does Base64 make Kubernetes Secrets secure?

No.

Base64 is encoding, not encryption.

mypassword
    |
    v
Base64
    |
    v
bXlwYXNzd29yZA==

The encoded value can easily be decoded.

Therefore:

Base64 != Encryption

Production protection should include appropriate:

RBAC

Encryption at rest

Access restrictions

Secure cluster configuration

Secret-management practices

External systems such as AWS Secrets Manager or HashiCorp Vault can also
be used.

Never commit credentials to Git just because they have been Base64
encoded.

4. A database password has been rotated, but the application still uses the old password. How would you troubleshoot it?

I would troubleshoot it in layers.

Step 1: Verify the Secret

kubectl get secret db-secret

Avoid unnecessarily exposing the secret value.

Step 2: Check how the Deployment consumes the Secret

kubectl describe deployment backend

Determine whether it is consumed as:

Environment variable

Mounted file

Step 3: If it is an environment variable

The existing container normally retains the old environment value.

Restart/recreate the Pods:

kubectl rollout restart deployment/backend
kubectl rollout status deployment/backend

Step 4: If it is a mounted file

Check the mounted file and whether Kubernetes has propagated the update.

Also check whether the application supports rereading or reloading the
file.

Step 5: Verify the application

Check:

Application logs

Database connectivity

Pod readiness

Events

Application-specific errors

For production, automated secret rotation is preferable to manually
editing credentials and restarting workloads.

5. A developer wants DB_HOST, DB_USERNAME, DB_PASSWORD and AWS_ACCESS_KEY_ID in a ConfigMap. What would you recommend?

I would not store sensitive credentials in a ConfigMap.

A better design is:

ConfigMap
  |
  +-- DB_HOST
  +-- DB_USERNAME

Secret / External Secret
  |
  +-- DB_PASSWORD

For AWS access, I would avoid long-lived AWS access keys when an AWS
workload-identity mechanism is available.

Depending on the environment:

EKS Pod Identity

IAM Roles for Service Accounts (IRSA)

For database credentials, a production design could be:

AWS Secrets Manager
        |
        v
External Secrets Operator
        |
        v
Kubernetes Secret
        |
        v
Application

This keeps sensitive credentials out of Git and allows a dedicated
secret-management system to act as the source of truth.

Additional Production Questions

Why should passwords not be stored in Git even if they are Base64 encoded?

Because Base64 is reversible encoding, not encryption.

A credential committed to Git can remain in Git history, clones, forks,
backups, CI/CD systems, and developer machines.

If a real credential is committed, it should be considered exposed and
rotated.

What happens when a Secret used as an environment variable is updated?

The existing container's environment does not automatically change.

The Pod normally needs to be recreated or restarted to receive the new
environment value.

kubectl rollout restart deployment/backend

What happens when a Secret mounted as a volume is updated?

Kubernetes can update the mounted Secret contents after the Secret
object changes, subject to propagation timing.

However, the application must reread or reload the file for the new
value to take effect.

What is the difference between data and stringData?

data contains Secret values in the encoded representation expected by
the Kubernetes API.

Example:

data:
  DB_PASSWORD: bXlwYXNzd29yZA==

stringData allows users to provide ordinary strings:

stringData:
  DB_PASSWORD: mypassword

Kubernetes handles the conversion into the Secret's data representation.

stringData makes authoring easier; it does not make the value
encrypted.

Day 5 Interview Mental Model

ConfigMap
    |
    +-- Non-sensitive configuration

Secret
    |
    +-- Sensitive configuration

Base64
    |
    +-- Encoding only

RBAC + Encryption at Rest + Access Control
    |
    +-- Security controls

Production:

External Secret Store
        |
        v
External Secrets Operator
        |
        v
Kubernetes Secret
        |
        v
Application

Key Interview Statements

ConfigMap stores non-sensitive configuration.

Secret is intended for sensitive configuration.

Base64 is encoding, not encryption.

Secret does not automatically mean secure.

Environment-variable Secret changes normally require Pod recreation.

Mounted Secret files can be updated, but the application must reload
them.

Avoid long-lived AWS access keys when workload identity is
available.

Never commit credentials to Git.

Use RBAC and encryption at rest as part of Secret security.
