# IDP Service Registration and Secrets Management

## 1. Purpose

Define the standard process for registering application services in the Internal Developer Portal (IDP), configuring service access, and managing application secrets during migration from Azure to Tencent Cloud.

## 2. Prerequisites

- Target Tencent TKE cluster and Kubernetes namespace are available.
- Application repository and deployment pipeline are configured.
- Tencent TCR repository is available.
- Required infrastructure, database, and network connectivity details are collected.
- Approved secrets management solution and access permissions are available.

## 3. Service Registration in IDP

Register each application or service in IDP with the following details.

| Field | Description |
|---|---|
| Service name | `APP1` |
| Environment | DEV / QA / PROD |
| Owner/team | Responsible application team |
| Repository | Application Git repository |
| Kubernetes cluster | Target Tencent TKE cluster |
| Namespace | Target Kubernetes namespace |
| Application port | Port on which the application listens |
| Service port | Kubernetes Service port |
| Ingress domain | Application domain, such as `app1.example.com` |
| Ingress path | `/` or application-specific path |
| TLS/SSL | HTTPS and certificate configuration |
| Container image | Tencent TCR repository and tag |
| Deployment method | Argo CD / GitOps |
| Health checks | Readiness and liveness probe paths |
| Dependencies | Database, cache, storage, and other services |

### Validation

- Verify the application and Kubernetes ports.
- Confirm ingress routing, DNS, and TLS configuration.
- Validate the cluster, namespace, image, and deployment configuration.
- Confirm service ownership and dependencies.

## 4. Secrets Management

### 4.1 Identify Required Secrets

Inventory all secrets required by the application.

Examples:
- Database usernames and passwords
- Redis credentials
- API tokens and third-party credentials
- Object storage credentials, where required
- External service credentials
- Certificates and private keys, where applicable

### 4.2 Store Secrets Securely

- Use the approved secrets manager, such as Tencent Cloud Secrets Manager or Tencent SSM, based on the platform design.
- Maintain separate secrets for DEV, QA, and PROD.
- Use meaningful secret names, for example:
  - `APP1-DEV-DB-CREDENTIALS`
  - `APP1-QA-DB-CREDENTIALS`
  - `APP1-PROD-DB-CREDENTIALS`
- Restrict access using least-privilege IAM/CAM policies.
- Never commit plaintext credentials to Git or store them in container images.
- Avoid placing secret values directly in IDP metadata or Terraform files.

### 4.3 Integrate Secrets with the Application

Configure the approved mechanism to make secrets available to the application at runtime, such as:
- A supported secrets-store integration
- A secrets synchronization controller
- A secure CI/CD deployment mechanism, where appropriate

Ensure the application receives only the secrets it needs.

### 4.4 Secret Rotation

- Define a rotation procedure for passwords, tokens, and credentials.
- Update the secrets manager and ensure applications can consume the rotated values.
- Validate application connectivity after rotation.
- Revoke old credentials once the replacement is verified.

## 5. Application Integration

- Configure database and other dependency endpoints.
- Configure the application to use the correct environment-specific secret references.
- Verify network access to databases, caches, and external services.
- Confirm application settings and non-sensitive configuration.
- Ensure credentials are not printed in CI/CD logs.

## 6. Deployment and Validation

1. Build the application image and push it to Tencent TCR.
2. Update the deployment configuration in Git.
3. Deploy through the approved CI/CD and Argo CD workflow.
4. Verify pods, Kubernetes Services, and Ingress.
5. Confirm the application can retrieve its required secrets.
6. Test database and dependency connectivity.
7. Validate the application using its ingress domain.
8. Review logs and health checks for errors.

## 7. Security Checklist

- [ ] No plaintext secrets in Git repositories.
- [ ] Separate secrets for DEV, QA, and PROD.
- [ ] Least-privilege access policies configured.
- [ ] Secret access restricted to authorized workloads and teams.
- [ ] Secret rotation process documented.
- [ ] Credentials excluded from application and CI/CD logs.
- [ ] Database and external service connectivity verified.
- [ ] Ingress domain and TLS/SSL validated.
- [ ] Old Azure credentials revoked after migration validation and approval.

## 8. Completion Criteria

The service registration and secrets setup are complete when the IDP metadata is accurate, networking and deployment details are validated, the application retrieves secrets securely, and the deployed application passes health and connectivity checks.
