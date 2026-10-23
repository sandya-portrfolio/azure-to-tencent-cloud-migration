# Container Image Migration: Azure ACR to Tencent TCR

## 1. Objective

Migrate required container images from Azure Container Registry (ACR) to Tencent Container Registry (TCR) to support cloud infrastructure migration.

This document describes the prerequisites, image transfer procedure, verification commands, migration tracking, and post-migration deployment validation.

## 2. Source and Target

| Item | Source | Target |
|---|---|---|
| Registry | Azure Container Registry (ACR) | Tencent Container Registry (TCR) |
| Registry endpoint | `<SOURCE_ACR_REGISTRY>` | `<TARGET_TCR_REGISTRY>` |
| Namespace | Not applicable | `<TARGET_NAMESPACE>` |
| Image format | `<SOURCE_REGISTRY>/<IMAGE>:<TAG>` | `<TARGET_REGISTRY>/<NAMESPACE>/<IMAGE>:<TAG>` |
| Transfer method | Docker pull | Docker tag and push |
| Verification | Source image and tag | Destination manifest and digest |
| Deployment platform | Existing container platform | Tencent Kubernetes Engine (TKE) |

Replace the placeholders with environment-specific values. Do not publish private registry endpoints, account identifiers, or credentials in a public portfolio.

## 3. Prerequisites

### 3.1 Access and permissions

- Access to the source Azure subscription and ACR registry.
- Permission to authenticate and pull source images.
- Access to the target Tencent Cloud account and TCR instance.
- Permission to access target repositories and push images.
- Approved authentication credentials or temporary tokens.
- Confirmation of required image repositories, tags, and destination mappings.

### 3.2 Required tools

- Azure CLI
- Docker CLI and a running Docker daemon
- Access to Tencent TCR through the console or approved CLI/API workflow
- Network connectivity to both registries

Verify the installed tools:

```bash
az version
docker --version
docker info
```

### 3.3 Network and storage

- The migration host can reach both registry endpoints over HTTPS.
- Firewall, proxy, and network rules allow registry access.
- Sufficient local disk space is available for downloaded images.
- Sufficient network bandwidth is available for image transfer.

### 3.4 Migration planning

- Inventory all required source repositories and tags.
- Confirm the destination namespace and repository names.
- Confirm whether target repositories already exist.
- Identify repositories with unresolved target mappings.
- Record source image digests where available.
- Plan deployment validation and rollback.
- Ensure credentials are not stored in source code or committed to Git.

## 4. Registry Authentication and Discovery

### 4.1 Authenticate to Azure

```bash
az login
az account show

az acr login --name "<SOURCE_ACR_NAME>"
```

Use an approved Azure identity with permission to pull images.

### 4.2 List source repositories

```bash
az acr repository list \
  --name "<SOURCE_ACR_NAME>" \
  --output table
```

### 4.3 List available image tags

```bash
az acr repository show-tags \
  --name "<SOURCE_ACR_NAME>" \
  --repository "<SOURCE_IMAGE>" \
  --output table
```

Confirm the required source tag with the application owner or deployment configuration.

### 4.4 Authenticate to Tencent TCR

```bash
docker login "<TARGET_TCR_REGISTRY>"
```

Enter the approved username and token/password when prompted. Use the authentication method supported by your TCR instance.

Do not hardcode credentials in shell scripts, Terraform files, or CI/CD configuration.

## 5. Image Migration Procedure

The migration follows this sequence:

**Authenticate → Pull → Tag → Push → Verify → Update Deployment → Validate**

### Step 1 — Define image variables

```bash
export SOURCE_ACR_REGISTRY="<SOURCE_ACR_REGISTRY>"
export SOURCE_IMAGE="<SOURCE_IMAGE>"
export SOURCE_TAG="<SOURCE_TAG>"

export TARGET_TCR_REGISTRY="<TARGET_TCR_REGISTRY>"
export TARGET_NAMESPACE="<TARGET_NAMESPACE>"
export TARGET_IMAGE="<TARGET_IMAGE>"
export TARGET_TAG="<TARGET_TAG>"
```

Use the exact source and target image paths confirmed during discovery.

### Step 2 — Pull the source image

```bash
docker pull \
  "$SOURCE_ACR_REGISTRY/$SOURCE_IMAGE:$SOURCE_TAG"
```

Confirm the image is available locally:

```bash
docker image ls
```

### Step 3 — Create the destination tag

Create a new local reference that points to the Tencent TCR destination.

```bash
docker tag \
  "$SOURCE_ACR_REGISTRY/$SOURCE_IMAGE:$SOURCE_TAG" \
  "$TARGET_TCR_REGISTRY/$TARGET_NAMESPACE/$TARGET_IMAGE:$TARGET_TAG"
```

Tagging does not upload the image or modify the source image in ACR.

### Step 4 — Push the image to Tencent TCR

```bash
docker push \
  "$TARGET_TCR_REGISTRY/$TARGET_NAMESPACE/$TARGET_IMAGE:$TARGET_TAG"
```

Wait for Docker to report that the push has completed successfully.

### Step 5 — Verify the destination manifest

```bash
docker manifest inspect \
  "$TARGET_TCR_REGISTRY/$TARGET_NAMESPACE/$TARGET_IMAGE:$TARGET_TAG"
```

A successful response confirms that the destination registry can return a manifest for the specified image reference.

For multi-platform images, verify that the required platform manifests are present.

### Step 6 — Compare image digests

Where supported, compare the source and destination registry digests using the registry's manifest inspection APIs or approved CLI tools.

Digest comparison provides stronger evidence of image integrity than checking manifest existence alone. Be aware that manifest formats, multi-platform indexes, and registry transformations can affect which digest should be compared.

### Step 7 — Repeat for each approved image

Repeat the pull, tag, push, and verification steps for every image repository and tag approved for migration.

Record the source reference, destination reference, tag, verification result, and any outstanding issues.

## 6. Reusable Migration Script

The following Bash script performs the pull, tag, push, and manifest verification steps for one image. Authenticate to both registries first.

```bash
#!/usr/bin/env bash
set -euo pipefail

: "${SOURCE_ACR_REGISTRY:?Set source registry}"
: "${SOURCE_IMAGE:?Set source image}"
: "${SOURCE_TAG:?Set source tag}"
: "${TARGET_TCR_REGISTRY:?Set target registry}"
: "${TARGET_NAMESPACE:?Set target namespace}"
: "${TARGET_IMAGE:?Set target image}"
: "${TARGET_TAG:?Set target tag}"

SOURCE_REF="${SOURCE_ACR_REGISTRY}/${SOURCE_IMAGE}:${SOURCE_TAG}"
TARGET_REF="${TARGET_TCR_REGISTRY}/${TARGET_NAMESPACE}/${TARGET_IMAGE}:${TARGET_TAG}"

echo "Pulling source image: ${SOURCE_REF}"
docker pull "${SOURCE_REF}"

echo "Tagging destination image: ${TARGET_REF}"
docker tag "${SOURCE_REF}" "${TARGET_REF}"

echo "Pushing image to Tencent TCR"
docker push "${TARGET_REF}"

echo "Verifying destination manifest"
docker manifest inspect "${TARGET_REF}" >/dev/null

echo "PASS: Destination manifest exists for ${TARGET_REF}"
```

Save as `migrate-image.sh`, review it, and run it only after confirming the image mappings and registry permissions.

```bash
chmod +x migrate-image.sh
./migrate-image.sh
```

The script deliberately does not delete source images or local images.

## 7. Migration Tracking

Maintain a table for all images in scope.

| Source image | Source tag | Target image | Target tag | Status |
|---|---|---|---|---|
| `IMAGE1` | `<TAG>` | `IMAGE1` | `<TAG>` | Pending verification |
| `IMAGE2` | `<TAG>` | `IMAGE2` | `<TAG>` | Pending verification |
| `IMAGE3` | `<TAG>` | `IMAGE3` | `<TAG>` | Pending mapping confirmation |

Use consistent status values:

- **Pending:** Migration not started.
- **Mapping required:** Target repository or required tag is unknown.
- **Migrated:** Push completed successfully.
- **Manifest verified:** Destination manifest confirmed.
- **Deployment validated:** Application workload validated on Tencent Kubernetes.

Do not mark an image as deployment-validated solely because its push or manifest verification succeeded.

## 8. Post-Migration Deployment Validation

After the images have been transferred:

1. Confirm all required target images and tags exist in Tencent TCR.
2. Update Terraform, Helm, Kubernetes manifests, or GitOps configuration to reference Tencent TCR.
3. Verify that Tencent Kubernetes workloads have the required image-pull permissions.
4. Confirm network connectivity from the cluster to the target registry.
5. Deploy through the approved CI/CD and GitOps workflow.
6. Check pod and deployment status.
7. Review image-pull errors and application logs.
8. Verify readiness and liveness probes.
9. Run application smoke tests and validate dependent services.
10. Record deployment validation evidence.

Example Kubernetes checks:

```bash
kubectl get deployments -n <NAMESPACE>
kubectl get pods -n <NAMESPACE>
kubectl describe pod <POD_NAME> -n <NAMESPACE>
kubectl get events -n <NAMESPACE> --sort-by=.lastTimestamp
```

Use the correct cluster context and namespace before executing these commands.

## 9. Security Controls

- Use approved identities and least-privilege registry permissions.
- Prefer temporary credentials where supported.
- Do not embed tokens or passwords in scripts or repository files.
- Avoid exposing credentials in terminal logs and CI/CD output.
- Use immutable or explicitly versioned image tags for production deployments.
- Record approved image digests where practical.
- Retain source images until target deployment validation and retirement approval are complete.

## 10. Completion Criteria

The migration is complete when:

- All in-scope source repositories and tags have confirmed target mappings.
- Required images have been pushed to Tencent TCR.
- Destination manifests and, where required, image digests have been verified.
- Deployment configuration references the correct target images.
- Tencent Kubernetes can pull the images.
- Application health and dependency checks pass.
- Outstanding gaps are resolved or formally accepted.
- Legacy image and infrastructure cleanup is approved.

## 11. Scope and Limitations

Manifest verification confirms that the destination registry can return the specified image manifest. It does not independently establish byte-for-byte equivalence, application compatibility, or successful startup.

Application deployment validation is a separate step. Source images and legacy infrastructure must not be deleted solely because the image push succeeded.

