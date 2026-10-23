# Container Image Migration: Azure ACR to Tencent TCR

## Overview

Migrated container images from Azure Container Registry (ACR) to Tencent Container Registry (TCR) to support a cloud infrastructure migration to Tencent Cloud.

## Key Activities

- Assessed source and target container registry repositories.
- Transferred container images using Docker pull, tag, and push workflows.
- Standardized target image repository paths and tags.
- Verified destination image manifests in Tencent TCR.
- Identified unmapped repositories and documented outstanding migration decisions.
- Reviewed image references, Kubernetes image-pull permissions, and deployment dependencies.
- Defined post-migration validation steps for Kubernetes workloads.

## Technology Stack

Azure ACR, Tencent TCR, Docker, Kubernetes (TKE), Terraform, CI/CD.

## Migration Outcome

Seven container images were reported as migrated and manifest-verified. Three additional source repositories required scope and target-mapping confirmation. Application deployment validation remained a separate follow-up activity.
