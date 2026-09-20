Kubernetes Day 5 --- ConfigMaps & Secrets

1. ConfigMap

A ConfigMap stores non-sensitive configuration data separately
from the application image.

Examples: - Application settings - Environment names - Feature flags -
URLs - Log levels - Port numbers

apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
  LOG_LEVEL: info
  DB_HOST: mysql

Do not store passwords, API keys, tokens, or private credentials in a
ConfigMap.

2. Secret

A Secret is intended for sensitive information such as
passwords, API keys, tokens, database credentials, TLS certificates, and
private keys.

apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  DB_USERNAME: admin
  DB_PASSWORD: mypassword

A Secret is not automatically secure just because it is a Secret.
Production security also requires RBAC, encryption at rest, restricted
access, and good secret-management practices.

3. ConfigMap vs Secret

ConfigMap                     Secret

Non-sensitive configuration   Sensitive configuration
Application settings          Passwords
URLs                          API keys
Feature flags                 Tokens
Log levels                    Credentials
Normal configuration          TLS/private-key material

4. Consuming ConfigMaps and Secrets

Two common methods are:

Environment variables

Volume-mounted files

Specific ConfigMap key

env:
  - name: APP_ENV
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: APP_ENV

All ConfigMap keys

envFrom:
  - configMapRef:
      name: app-config

Specific Secret key

env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-secret
        key: DB_PASSWORD

All Secret keys

envFrom:
  - secretRef:
      name: app-secret

5. ConfigMap or Secret as a Volume

A ConfigMap can be mounted as files:

volumes:
  - name: config-volume
    configMap:
      name: app-config

containers:
  - name: app
    image: nginx
    volumeMounts:
      - name: config-volume
        mountPath: /etc/app-config

A Secret can also be mounted:

volumes:
  - name: secret-volume
    secret:
      secretName: app-secret

containers:
  - name: app
    image: nginx
    volumeMounts:
      - name: secret-volume
        mountPath: /etc/app-secret
        readOnly: true

6. Environment Variables vs Volume Mounts

Environment variables

Useful when the application expects configuration as environment
variables.

Important production behavior:

If a ConfigMap or Secret is consumed as an environment variable,
changing the object does not change the environment of an
already-running container. The Pod normally needs to be recreated or
restarted.

Example:

kubectl rollout restart deployment/backend

Volume mounts

Useful when the application expects configuration or credentials as
files.

Kubernetes can update mounted ConfigMap/Secret contents after the
underlying object changes. However, the application must support
rereading or reloading the file before it can use the new value.

7. Base64 Is Not Encryption

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

Do not treat Base64 as a security mechanism.

8. Secret Security in Production

Important controls include:

RBAC

Restrict who can get, list, create, update, and delete Secrets.

Encryption at Rest

Protect Secret data stored in the cluster's backing storage using
encryption at rest.

Avoid Git

Never commit passwords, tokens, API keys, or private keys to Git.

Base64-encoding a password before committing it does not make it safe.

9. External Secret Management

Production environments often use dedicated secret-management systems
such as:

AWS Secrets Manager

HashiCorp Vault

Azure Key Vault

Google Secret Manager

A common architecture is:

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

The external system can act as the source of truth.

10. AWS Production Example

For an application running on AWS:

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

For AWS API access, avoid long-lived AWS access keys in Kubernetes
Secrets when workload identity is available.

Depending on the environment, options include:

EKS Pod Identity

IAM Roles for Service Accounts (IRSA)

This is preferable to distributing long-lived AWS access keys to
workloads.

11. Secret Rotation

Suppose the database password changes:

old-password
      |
      v
new-password

Secret consumed through an environment variable

Secret
  |
  v
Environment variable
  |
  v
Container process

The existing process normally continues using the old environment value.

Restart/recreate the Pods:

kubectl rollout restart deployment/backend
kubectl rollout status deployment/backend

Secret consumed through a volume

Secret
  |
  v
Mounted file
  |
  v
Application

Kubernetes can update the mounted file after the Secret changes, subject
to propagation timing.

The application must still reread or reload the file to use the new
value.

12. Immutable ConfigMaps and Secrets

When configuration should not change after creation, Kubernetes
supports:

immutable: true

Example:

apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
immutable: true
data:
  APP_ENV: production

This helps prevent accidental modification.

13. Production Configuration Design

ConfigMap
  |
  +-- APP_ENV
  +-- LOG_LEVEL
  +-- DB_HOST
  +-- Feature flags

Secret / External Secret
  |
  +-- DB_PASSWORD
  +-- API_TOKEN
  +-- TLS credentials

The container image contains application code, while configuration and
secrets are supplied separately.

14. Troubleshooting

Check the objects:

kubectl get configmaps
kubectl get secrets

Inspect configuration:

kubectl describe configmap app-config
kubectl describe secret app-secret

Check the Deployment:

kubectl describe deployment backend

Verify: - ConfigMap reference - Secret reference - Key names - Volume -
volumeMount - Environment-variable configuration

Check Pods:

kubectl describe pod <pod-name>

For environment variables, be careful because this can expose sensitive
values:

kubectl exec <pod-name> -- env

For mounted files:

kubectl exec <pod-name> -- ls -l /etc/app-config

15. Important Mental Models

ConfigMap = non-sensitive configuration

Secret = sensitive configuration

Base64 = encoding, not encryption

RBAC + Encryption at Rest + Access Control
        = important Secret security controls

Production secret flow:

AWS Secrets Manager / Vault
             |
             v
External Secrets Operator
             |
             v
     Kubernetes Secret
             |
             v
        Application

Remember:

ConfigMap = non-sensitive configuration.

Secret = sensitive configuration.

Base64 = encoding, not encryption.

Environment-variable Secret changes normally require Pod
recreation/restart.

Mounted Secret files can be updated, but the application must reload
them.

Prefer AWS workload identity over long-lived access keys where
supported.

Never commit credentials to Git.
