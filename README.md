# azure-to-tencent-cloud-migration
Repository name: `azure-to-tencent-cloud-migration`

File name: `README.md`

Copy the following content directly into your GitHub repository.

# Azure to Tencent Cloud Migration Runbook

## 1. Overview

This document provides a step-by-step guide for migrating infrastructure, databases, object storage, and applications from Microsoft Azure to Tencent Cloud.

The migration includes:

- Azure Blob Storage → Tencent Cloud Object Storage (COS)
- Azure PostgreSQL → TencentDB for PostgreSQL
- Azure-hosted applications → Tencent Kubernetes Engine (TKE) or other appropriate compute services
- Networking, DNS, secrets, security, and monitoring configuration
- Migration validation, cutover, and rollback

All application and database names are represented using generic identifiers such as `APP1`, `APP2`, `DB1`, and `DB2`.

## 2. Prerequisites

### Azure Requirements

- Azure subscription access
- Azure CLI or Azure Cloud Shell
- Read access to source infrastructure
- Azure Storage Account and container access
- Database administrator or migration permissions
- Database connection details
- Application configuration and deployment details
- Network connectivity information
- Existing backups and recovery procedures

### Tencent Cloud Requirements

- Tencent Cloud account
- Tencent Cloud COS buckets
- Tencent CAM user and policies
- Tencent Cloud Secret Manager (SSM)
- TencentDB for PostgreSQL instances
- Tencent Kubernetes Engine (TKE), if applicable
- Tencent VPC, subnets, routing, and security groups
- Container Registry (TCR), if using containers
- Terraform infrastructure configuration
- Required migration-service permissions

### Required Tools

| Tool                             | Purpose                           |
| -------------------------------- | --------------------------------- |
| Azure CLI                        | Discover Azure infrastructure     |
| Tencent Cloud Console            | Configure destination resources   |
| Terraform                        | Provision Tencent infrastructure  |
| Git                              | Manage infrastructure code        |
| PostgreSQL CLI                   | Validate database connectivity    |
| pg_dump / pg_restore             | PostgreSQL migration              |
| kubectl                          | Kubernetes application management |
| Tencent Object Storage Migration | Azure Blob to COS migration       |

## 3. Migration Architecture

**Source: Microsoft Azure**

- Application: APP1
- Application: APP2
- Database: DB1
- Database: DB2
- Blob Storage: STORAGE1
- Blob Storage: STORAGE2
- Azure networking and application services

**Destination: Tencent Cloud**

- Application: APP1 → TKE
- Application: APP2 → TKE
- Database: DB1 → TencentDB PostgreSQL
- Database: DB2 → TencentDB PostgreSQL
- STORAGE1 → COS_BUCKET1
- STORAGE2 → COS_BUCKET2
- Azure networking → Tencent VPC and related networking resources

## 4. Azure Infrastructure Discovery

### Step 1: Login to Azure

Open Azure Cloud Shell or a terminal with Azure CLI installed.

```
az login
```

Check the current subscription:

```
az account show --output table
```

List available subscriptions:

```
az account list --output table
```

Select the correct subscription:

```
az account set --subscription "<SUBSCRIPTION_ID>"
```

### Step 2: List Azure Resource Groups

```
az group list --output table
```

### Step 3: List Azure Resources

```
az resource list --output table
```

Record the resource type, region, resource group, dependencies, and migration destination.

### Step 4: Identify Storage Accounts

```
az storage account list \
  --query "[].{Name:name,ResourceGroup:resourceGroup,Location:location}" \
  --output table
```

### Step 5: Identify Blob Containers

```
az storage container list \
  --account-name "<STORAGE_ACCOUNT>" \
  --auth-mode login \
  --output table
```

### Step 6: Identify Blob Objects

```
az storage blob list \
  --account-name "<STORAGE_ACCOUNT>" \
  --container-name "<CONTAINER_NAME>" \
  --auth-mode login \
  --output table
```

### Step 7: Calculate Blob Count and Size

```
az storage blob list \
  --account-name "<STORAGE_ACCOUNT>" \
  --container-name "<CONTAINER_NAME>" \
  --auth-mode login \
  --query "[].{name:name,size:properties.contentLength}" \
  --output json > blobs.json
```

Calculate the size:

```
python3 - <<'PY'
import json

with open("blobs.json") as f:
    blobs = json.load(f)

total = sum(blob.get("size") or 0 for blob in blobs)

print("Total Objects:", len(blobs))
print("Total Size (Bytes):", total)
print("Total Size (GB):", round(total / 1e9, 3))
PY
```

For large containers, confirm the Azure CLI returns the complete paginated inventory. Do not rely on a truncated listing.

## 5. Azure Blob Storage to Tencent COS Migration

### Step 1: Collect Azure Storage Details

Collect the following:

| Field                     | Description                       |
| ------------------------- | --------------------------------- |
| Storage Account           | Azure source storage account      |
| Container Name            | Source Blob container             |
| Endpoint                  | Azure Blob service endpoint       |
| Storage Account Key / SAS | Approved migration authentication |
| Object Count              | Total source objects              |
| Total Size                | Source data size                  |
| Region                    | Azure source region               |

To find these details:

1. Open Azure Portal.
2. Navigate to Storage Accounts.
3. Select the required storage account.
4. Open Containers.
5. Record the container names.
6. Open Endpoints to find the Blob service URL.
7. Obtain the required source credentials through an approved access process.

Do not commit storage keys or SAS tokens to GitHub.

### Step 2: Prepare Tencent COS

1. Login to Tencent Cloud Console.
2. Navigate to Cloud Object Storage (COS).
3. Create or select the destination bucket.
4. Select the required region.
5. Configure access policies and encryption.
6. Verify bucket availability.

Example mapping:

| Azure Source          | Tencent Destination |
| --------------------- | ------------------- |
| STORAGE1 / CONTAINER1 | COS_BUCKET1         |
| STORAGE1 / CONTAINER2 | COS_BUCKET2         |
| STORAGE2 / CONTAINER3 | COS_BUCKET3         |
| STORAGE3 / CONTAINER4 | COS_BUCKET4         |

Each mapping must be confirmed with the application owner before migration.

### Step 3: Create Tencent CAM Migration Permissions

Create a dedicated CAM user, such as:

`azure-cos-migration-user`

Configure the permissions required for:

- Listing destination buckets
- Listing objects
- Uploading objects
- Reading objects for validation
- Multipart uploads, where required
- Access to the approved destination buckets

If using Tencent's managed migration service, also configure its documented MSP permissions and any required service role.

Avoid granting unnecessary permissions to unrelated buckets.

### Step 4: Configure Tencent Secret Manager

Create a secret:

`azure-cos-migration-credentials`

Store the migration identity credentials securely.

Example secret structure:

```
{
  "user_name": "azure-cos-migration-user",
  "secret_id": "<SECRET_ID>",
  "secret_key": "<SECRET_KEY>"
}
```

The actual values must be stored in the secret manager, not in GitHub or Terraform source files.

**Important:** Creating a CAM user or an empty SSM secret does not automatically populate the credentials. The API key must be generated and its SecretId and SecretKey stored using a secure process. Terraform-managed credentials may be recorded in Terraform state.

### Step 5: Create Migration Task

Open Tencent Cloud Console.

Navigate to:

**Cloud Migration → Object Storage Migration → Create Task**

Configure:

**Source Configuration**

- Source provider: Azure Blob Storage
- Storage Account: STORAGE1
- Container: CONTAINER1
- Authentication: Approved Azure storage credentials
- Endpoint suffix: Appropriate Azure environment endpoint
- Object listing mode: Scan source objects

**Destination Configuration**

- Destination provider: Tencent COS
- SecretId: Tencent CAM SecretId
- SecretKey: Tencent CAM SecretKey
- Destination bucket: COS_BUCKET1
- Destination root directory: Required prefix, if any
- Storage class: Required COS storage class
- Overwrite behavior: Based on approved migration plan

Review all values before creating the task.

### Step 6: Start Migration

1. Create the migration task.
2. Start the task.
3. Monitor migration progress.
4. Review object transfer errors.
5. Retry failed objects where supported.
6. Record migration task details.

### Step 7: Validate Migration

Check:

- Source object count
- Destination object count
- Source total size
- Destination total size
- Object names and paths
- Content type and metadata
- Representative object checksums
- Application read/write access

Object count and size comparisons alone are not sufficient to prove data integrity.

### Step 8: Final Synchronization

If source objects can change during migration:

1. Agree on a source write-freeze or synchronization strategy.
2. Run the supported incremental or repeat migration process.
3. Validate changed objects.
4. Confirm all required data is available in COS.
5. Obtain approval before switching the application to Tencent COS.

## 6. PostgreSQL Database Migration

### Step 1: Identify Azure Databases

Open Azure Portal and identify:

- Database server
- Database engine and version
- Database names
- Database size
- Database users and permissions
- Extensions
- Networking and firewall configuration
- Backup settings

Example:

| Source Database | Destination   |
| --------------- | ------------- |
| DB1             | TencentDB DB1 |
| DB2             | TencentDB DB2 |
| DB3             | TencentDB DB3 |

### Step 2: Check Database Version

Connect to the Azure PostgreSQL source using an approved secure connection.

```
psql "host=<AZURE_DB_HOST> port=5432 dbname=<DB1> user=<DB_USER> sslmode=require"
```

Run:

```
SELECT version();
```

List databases:

```
SELECT datname
FROM pg_database
WHERE datistemplate = false;
```

Check extensions:

```
SELECT extname, extversion
FROM pg_extension;
```

Check database sizes:

```
SELECT
  datname,
  pg_size_pretty(pg_database_size(datname))
FROM pg_database
WHERE datistemplate = false;
```

### Step 3: Prepare Tencent PostgreSQL

1. Create the TencentDB PostgreSQL instance.
2. Confirm PostgreSQL version compatibility.
3. Configure VPC and security groups.
4. Configure approved database users.
5. Verify network connectivity.
6. Configure backups and monitoring.

### Step 4: Choose Migration Method

Choose one of the following based on database size, compatibility, connectivity, and downtime requirements:

**Option A: PostgreSQL Native Migration**

Use `pg_dump` and `pg_restore` for an approved offline migration.

**Option B: Tencent Data Transfer Service (DTS)**

Use Tencent DTS where the source and destination engines, versions, connectivity, and required migration mode are supported.

### Step 5: PostgreSQL Export

For a native migration:

```
pg_dump \
  -h <AZURE_DB_HOST> \
  -U <DB_USER> \
  -d <DB1> \
  -Fc \
  -f DB1.dump
```

The database password should be supplied securely, not embedded in the command or committed to Git.

### Step 6: PostgreSQL Restore

```
pg_restore \
  -h <TENCENT_DB_HOST> \
  -U <DB_USER> \
  -d <DB1> \
  --no-owner \
  --no-acl \
  -v DB1.dump
```

Ensure the target database exists and required roles, extensions, and permissions are prepared before restoration.

Review any restore errors and reconcile permissions, sequences, and other database objects as needed.

### Step 7: Database Validation

Validate:

- Database schemas
- Table counts
- Row counts for critical tables
- Indexes and constraints
- Sequences
- Extensions
- Users and privileges
- Application connectivity
- Critical business queries

For a consistent final cutover, prevent new source writes or use a supported replication/synchronization approach.

## 7. Application Migration

### Step 1: Identify Applications

Create an application inventory.

| Application | Source | Destination |
| ----------- | ------ | ----------- |
| APP1        | Azure  | Tencent TKE |
| APP2        | Azure  | Tencent TKE |
| APP3        | Azure  | Tencent TKE |

### Step 2: Identify Dependencies

For every application, record:

- Database connections
- Blob Storage dependencies
- Redis/cache dependencies
- APIs and external services
- Secrets and environment variables
- DNS and SSL certificates
- Network connectivity
- Logging and monitoring
- Persistent storage requirements

### Step 3: Prepare Tencent Infrastructure

Use Terraform to provision:

- TKE cluster or target compute
- Namespaces
- Service accounts and permissions
- Tencent Container Registry (TCR)
- TencentDB PostgreSQL
- Redis, if required
- COS buckets
- Load balancers and ingress
- DNS and certificates
- Secrets and monitoring

### Step 4: Build and Deploy Applications

1. Build application container images.
2. Scan images for vulnerabilities.
3. Push images to Tencent TCR.
4. Configure Kubernetes deployment manifests or Helm charts.
5. Configure secrets and environment variables.
6. Deploy applications to TKE.
7. Verify pod health and application logs.

Example:

```
kubectl get pods -n <NAMESPACE>
```

```
kubectl get services -n <NAMESPACE>
```

```
kubectl get ingress -n <NAMESPACE>
```

### Step 5: Update Application Configuration

Update application dependencies to point to Tencent services:

| Configuration      | Old                       | New                               |
| ------------------ | ------------------------- | --------------------------------- |
| Database endpoint  | Azure PostgreSQL          | Tencent PostgreSQL                |
| Object storage     | Azure Blob                | Tencent COS                       |
| Container registry | Azure registry            | Tencent TCR                       |
| Redis endpoint     | Azure Redis               | Tencent Redis                     |
| Monitoring         | Existing Azure monitoring | Approved Tencent monitoring stack |

Never hardcode credentials in application manifests.

### Step 6: Application Validation

Validate:

- Application startup
- Database connectivity
- COS object read/write operations
- Authentication
- API endpoints
- External integrations
- Logging and monitoring
- Performance
- Error rates

## 8. Migration Cutover

Before cutover:

1. Confirm Blob migration completion.
2. Confirm database migration completion.
3. Validate application deployments.
4. Verify secrets and connectivity.
5. Approve the maintenance window.
6. Prepare the rollback procedure.

During cutover:

1. Pause or drain application writes where required.
2. Perform final database and object synchronization.
3. Update application endpoints.
4. Switch DNS or load balancer routing.
5. Run application smoke tests.
6. Monitor service health.

After cutover:

1. Verify application functionality.
2. Monitor database performance.
3. Monitor COS access.
4. Review logs and alerts.
5. Obtain business approval.

## 9. Rollback Plan

Rollback may be required if:

- Application health checks fail
- Database consistency checks fail
- Critical APIs are unavailable
- Data is missing
- Performance falls outside approved thresholds

Rollback steps:

1. Stop or isolate writes to the Tencent environment.
2. Assess data written after cutover.
3. Reconcile any data differences before returning to Azure.
4. Restore previous application configuration.
5. Switch traffic back to Azure.
6. Validate Azure application health.
7. Confirm database and storage consistency.
8. Record the rollback outcome.

Do not switch writes back to Azure without accounting for data created or modified in Tencent after cutover.

## 10. Final Migration Checklist

### Azure Discovery

- Resource inventory completed
- Storage accounts identified
- Blob containers identified
- Database inventory completed
- Application dependencies documented

### Tencent Infrastructure

- COS buckets available
- CAM permissions configured
- Secrets securely stored
- Database instances available
- TKE/application infrastructure ready
- Networking and DNS configured

### Migration

- Blob migration completed
- Database migration completed
- Applications deployed
- Final synchronization completed
- Data validation completed

### Cutover

- Application smoke tests passed
- DNS/routing updated
- Monitoring verified
- Rollback procedure available
- Business approval received

## 11. Security Considerations

- Use dedicated migration identities.
- Apply least-privilege CAM policies.
- Do not store credentials in GitHub.
- Store secrets in an approved secret manager.
- Protect Terraform state and CI/CD logs.
- Enable encryption in transit and at rest.
- Rotate temporary migration credentials after completion.
- Retain Azure resources until decommissioning is approved.

## 12. Official Documentation

**Azure**

- [Azure CLI documentation](https://learn.microsoft.com/en-us/cli/azure/)
- [Azure Blob Storage documentation](https://learn.microsoft.com/en-us/azure/storage/blobs/)
- [Azure Database for PostgreSQL](https://learn.microsoft.com/en-us/azure/postgresql/)

**Tencent Cloud**

- [Tencent Cloud COS documentation](https://www.tencentcloud.com/document/product/436)
- [Tencent Cloud Migration documentation](https://www.tencentcloud.com/document/product/1036)
- [Tencent Cloud CAM documentation](https://www.tencentcloud.com/document/product/598)
- [TencentDB for PostgreSQL](https://www.tencentcloud.com/document/product/409)
- [Tencent Kubernetes Engine](https://www.tencentcloud.com/document/product/457)

---

**Document Version:** 1.0

**Migration Type:** Microsoft Azure → Tencent Cloud

**Scope:** Infrastructure, Blob Storage, PostgreSQL databases, and applications

**Status:** Draft — review and approve before production execution

This is ready to paste into GitHub as `README.md`.
