# Azure Blob Storage to Tencent COS Migration

## 1. Objective

This document describes the process for migrating objects from Azure Blob Storage to Tencent Cloud Object Storage (COS) using Tencent Cloud's Object Storage Migration service.

It covers:

- Source storage discovery and migration sizing.
- Tencent CAM user and policy provisioning using Terraform.
- Automated generation and secure storage of CAM credentials in Tencent SSM.
- Migration task configuration and execution.
- Destination verification and application integration.
- Security controls and cleanup.

All resource names and identifiers in this document are placeholders intended for reusable infrastructure documentation.

## 2. Migration Architecture

```text
Azure Blob Storage
        |
        | Source access credentials
        v
Tencent Cloud Object Storage Migration
        |
        | Read source objects
        | Transfer objects
        v
Tencent Cloud COS
        |
        v
Destination verification
        |
        v
Application integration and validation
```

**Migration method:** Tencent Cloud managed Object Storage Migration service.

This approach allows the migration service to transfer objects directly between supported storage systems without requiring a manually managed local download-and-upload workflow.

## 3. Prerequisites

### 3.1 Azure prerequisites

- Access to the Azure subscription and source storage account.
- Permission to inspect the source container and its contents.
- Approved source authentication credentials supported by the migration service.
- Confirmation of the source container name and required objects.
- Network and source access permissions required by the migration service.

### 3.2 Tencent Cloud prerequisites

- Access to the Tencent Cloud account.
- Target COS buckets created and accessible.
- Appropriate CAM permissions for the migration identity.
- Tencent SSM available for credential storage.
- Access to the Tencent Cloud Migration console.
- Approved permissions to create migration tasks and inspect task results.

### 3.3 Local tools

For discovery and infrastructure deployment, the following tools may be needed:

```bash
az version
terraform version
docker --version
```

The `az` and Terraform CLIs are useful for inventory and infrastructure provisioning. The actual object transfer in this workflow is performed by Tencent's migration service.

## 4. Phase 1 — Discover and Size the Source Data

Before creating a migration task, identify the source storage account, container, object count, and total data size.

### 4.1 Identify source storage details

Record:

- Source subscription and storage account.
- Source container.
- Source region.
- Authentication method.
- Object count.
- Total data size.
- Required destination COS bucket.
- Any object metadata or access-control requirements.

### 4.2 Inspect containers using Azure CLI

```bash
az login

az account show

az storage container list \
  --account-name "<SOURCE_STORAGE_ACCOUNT>" \
  --auth-mode login \
  --output table
```

List blobs in a container:

```bash
az storage blob list \
  --account-name "<SOURCE_STORAGE_ACCOUNT>" \
  --container-name "<SOURCE_CONTAINER>" \
  --auth-mode login \
  --query "[].{Name:name,Size:properties.contentLength}" \
  --output table
```

For large containers, use a script or inventory report to aggregate the number of objects and total size instead of printing every blob.

If the migration service uses a different authentication method, ensure the required source permissions and credentials are configured accordingly.

### 4.3 Record the inventory

Maintain a migration inventory:

| Field | Value |
|---|---|
| Source provider | Azure Blob Storage |
| Source storage account | `<SOURCE_STORAGE_ACCOUNT>` |
| Source container | `<SOURCE_CONTAINER>` |
| Destination provider | Tencent COS |
| Destination bucket | `<TARGET_COS_BUCKET>` |
| Destination region | `<TARGET_REGION>` |
| Total objects | `<OBJECT_COUNT>` |
| Total size | `<DATA_SIZE>` |
| Migration status | Pending / In progress / Verified |

The object count and size should be measured from the actual source inventory.

## 5. Phase 2 — Provision Tencent CAM Access Using Terraform

Terraform provisions a dedicated CAM user, a custom COS access policy, a policy attachment, an API access key, and an SSM secret containing the generated credentials.

The following example is sanitized. Replace the placeholders and confirm the required Tencent provider version and resource schemas before use.

### 5.1 CAM user

Create a dedicated API-only migration identity.

```hcl
resource "tencentcloud_cam_user" "migration" {
  count               = var.environment == "prod" ? 1 : 0
  name                = "migration-service-user"
  remark              = "Identity for approved object storage migration"
  console_login       = false
  use_api             = true
  need_reset_password = false

  tags = var.tags
}
```

The example limits creation to the production Terraform environment. Adjust this condition to match the approved ownership and deployment model.

### 5.2 CAM policy

Create a custom policy that grants access only to approved destination buckets.

```hcl
resource "tencentcloud_cam_policy" "migration_cos" {
  count       = var.environment == "prod" ? 1 : 0
  name        = "migration-cos-access"
  description = "Permissions for approved object storage migration"

  document = jsonencode({
    version = "2.0"

    statement = [
      {
        effect = "allow"

        action = [
          "name/cos:GetObject",
          "name/cos:PutObject",
          "name/cos:PostObject",
          "name/cos:HeadObject",
          "name/cos:GetBucket",
          "name/cos:ListMultipartUploads",
          "name/cos:AbortMultipartUpload",
          "name/cos:GetObjectACL",
          "name/cos:PutObjectACL"
        ]

        resource = [
          "qcs::cos:<REGION>:uid/<ACCOUNT_ID>:<BUCKET_NAME>",
          "qcs::cos:<REGION>:uid/<ACCOUNT_ID>:<BUCKET_NAME>/*"
        ]
      },
      {
        effect = "allow"

        action = [
          "name/cos:GetService"
        ]

        resource = ["*"]
      }
    ]
  })

  tags = var.tags
}
```

**Important:** This is an illustrative policy. Validate the exact COS actions and resource formats against the migration service's requirements. Add permissions only when required by the chosen migration operation. For example, destination overwrite or deletion permissions should not be granted by default.

For multiple approved buckets, include each explicitly in the resource list. Replace the region, account ID, and bucket name placeholders with the approved values.

### 5.3 Attach the policy to the CAM user

```hcl
resource "tencentcloud_cam_user_policy_attachment" "migration" {
  count     = var.environment == "prod" ? 1 : 0
  user_name = tencentcloud_cam_user.migration[0].name
  policy_id = tencentcloud_cam_policy.migration_cos[0].id
}
```

This associates the custom policy with the migration identity.

### 5.4 Generate a CAM API access key

```hcl
resource "tencentcloud_cam_access_key" "migration" {
  count      = var.environment == "prod" ? 1 : 0
  target_uin = tencentcloud_cam_user.migration[0].uin
  status     = "Active"

  lifecycle {
    create_before_destroy = true
  }
}
```

Terraform generates an API access key for the CAM user.

Treat the access key and secret key as sensitive credentials. Protect Terraform state and CI/CD logs because generated secret values may be present in state even when they are not displayed in normal output.

### 5.5 Create an SSM secret

```hcl
resource "tencentcloud_ssm_secret" "migration_credentials" {
  count       = var.environment == "prod" ? 1 : 0
  secret_name = "migration-service-credentials"
  description = "Credentials for approved object storage migration"

  tags = var.tags
}
```

This creates the secret container. It does not by itself populate the secret value.

### 5.6 Store generated credentials in an SSM secret version

```hcl
resource "tencentcloud_ssm_secret_version" "migration_credentials" {
  count       = var.environment == "prod" ? 1 : 0
  secret_name = tencentcloud_ssm_secret.migration_credentials[0].secret_name
  version_id  = "v1"

  secret_string = jsonencode({
    SecretId  = tencentcloud_cam_access_key.migration[0].access_key
    SecretKey = tencentcloud_cam_access_key.migration[0].secret_access_key
  })

  depends_on = [
    tencentcloud_cam_user_policy_attachment.migration
  ]
}
```

This stores the generated credentials as a JSON secret value.

Confirm the exact `tencentcloud_cam_access_key` output attributes and `tencentcloud_ssm_secret_version` schema against the pinned provider version. The attachment dependency ensures that the policy is attached before the credentials are stored.

### 5.7 Validate the Terraform changes

```bash
terraform fmt -check
terraform init
terraform validate
terraform plan
```

Review the plan before applying it. Ensure the plan does not unexpectedly replace existing users, keys, policies, or secrets.

Apply only through the approved CI/CD workflow:

```bash
terraform apply
```

Do not enable secret-value output merely to check whether the secret was populated. Verify its existence and version through approved metadata or secret-management controls.

**Security controls:**

- Protect Terraform state with encryption and restricted access.
- Never commit generated credentials to Git.
- Do not print secret values in CI/CD logs.
- Restrict SSM read permissions to authorized identities.
- Rotate credentials and retire temporary access when migration is complete.

## 6. Phase 3 — Create an Object Storage Migration Task

Open the Tencent Cloud console and navigate to **Cloud Migration → Object Storage Migration**.

### 6.1 Configure the task

Enter a descriptive, non-sensitive task name, for example:

`blob-to-cos-migration-dev`

Enter the estimated data size and number of objects based on the inventory. These values are used for migration resource estimation; the actual objects detected by the migration service determine what is transferred.

### 6.2 Configure the source

Select **Azure** as the migration source provider.

Provide the source information requested by the console, such as:

- Azure source authentication credentials.
- Storage account or source endpoint.
- Source container.
- Object list or source bucket selection, depending on the available options.

The exact fields depend on the selected source provider and the migration console version.

If the console offers source scanning, use it to retrieve the available source containers after credentials and access permissions are configured.

### 6.3 Configure object selection

Choose one of the supported methods:

- **Scan the source container:** Transfer eligible objects discovered in the selected container.
- **Use an object list:** Transfer only the specified objects, if supported for the chosen source provider.

For a full-container migration, source scanning is generally appropriate. For a selective migration, verify that the object-list format and filtering behavior are supported.

### 6.4 Configure header and metadata handling

Review the migration service's options for retaining, discarding, or replacing source headers and metadata.

Choose a policy based on application requirements. Verify content types, cache headers, metadata, and any access-control information needed by the destination application.

### 6.5 Configure the destination

Select Tencent COS as the destination and provide the approved destination bucket and required configuration.

Verify:

- Destination bucket and region.
- CAM access permissions.
- Object naming and path mapping.
- Overwrite behavior for existing objects.
- Metadata handling.
- Any supported filtering or transfer options.

Do not assume the source and destination configuration fields are identical; follow the console's prompts for each provider.

### 6.6 Review and submit

Review the task settings before submission:

- Correct source storage account and container.
- Correct target COS bucket.
- Correct credentials and permissions.
- Correct data size and object count estimates.
- Correct object selection and metadata behavior.

Submit the task and record its task ID and status.

## 7. Phase 4 — Execute and Monitor Migration

Use the Tencent migration console to start the task if it was not started automatically.

Monitor:

- Task status.
- Objects discovered and transferred.
- Successful and failed objects.
- Transfer progress and speed.
- Error messages and retry status.

Resolve source permission errors, invalid credentials, destination write failures, and unsupported object metadata issues before declaring completion.

For incremental or repeated migration, verify the service's documented support for re-running tasks and handling objects that changed after the initial scan.

## 8. Phase 5 — Verify Destination COS

After the task reports completion, validate the destination independently.

### 8.1 Compare object counts and total size

Compare the source inventory with the destination bucket inventory. Account for any explicitly excluded objects, filters, or unsupported object versions.

### 8.2 Inspect destination objects

Using an approved COS access method, list destination objects and inspect their sizes and metadata.

The Tencent COS CLI can be used if it is installed and configured with the correct credentials:

```bash
coscli ls cos://<TARGET_COS_BUCKET>/
```

Confirm the exact command syntax for the installed COS CLI version. Alternatively, use the Tencent Cloud console or supported COS APIs.

### 8.3 Validate object integrity

Where supported, compare object checksums or content hashes between source and destination. Do not assume an ETag is always a content MD5, particularly for multipart uploads.

### 8.4 Validate application access

- Confirm the application identity has the required COS permissions.
- Verify object paths and endpoint configuration.
- Test representative reads and writes.
- Check logs for access-denied or missing-object errors.
- Confirm any required metadata and content types are preserved.

A successful migration task does not by itself prove application compatibility.

## 9. Phase 6 — Cutover and Cleanup

Proceed only after destination validation and the required approval.

1. Update application configuration to use Tencent COS.
2. Deploy the updated configuration through the approved CI/CD workflow.
3. Validate application behavior and storage access.
4. Keep the source storage available during the agreed stabilization period.
5. Confirm rollback requirements have been satisfied.
6. Revoke temporary source credentials and unnecessary migration permissions.
7. Retire obsolete source resources only after formal approval.

Do not delete source objects as part of the initial migration unless deletion is explicitly required, authorized, and covered by a verified recovery plan.

## 10. Migration Checklist

### Discovery
- [ ] Source account and container identified.
- [ ] Object count and total data size recorded.
- [ ] Target COS bucket confirmed.
- [ ] Source authentication method validated.

### Infrastructure and credentials
- [ ] CAM user provisioned.
- [ ] Custom policy reviewed and applied.
- [ ] Access key generated.
- [ ] SSM secret and version created.
- [ ] Terraform state and secret access secured.

### Migration
- [ ] Migration task created.
- [ ] Source credentials and container configured.
- [ ] Destination bucket and permissions validated.
- [ ] Task submitted and monitored.
- [ ] Failed objects investigated.

### Verification and cutover
- [ ] Source and destination inventories compared.
- [ ] Object integrity checked where supported.
- [ ] Application storage access tested.
- [ ] Application configuration updated.
- [ ] Cutover approved.
- [ ] Temporary permissions and credentials retired.

## 11. Scope and Limitations

This document describes the managed object storage migration approach and the Terraform pattern for provisioning the required CAM identity and storing its credentials in SSM.

The exact migration task fields, supported source authentication methods, and required COS permissions must be verified against the active Tencent Cloud console and service documentation.

Successful task completion should be supported by destination inventory and integrity checks before the migration is considered complete.
