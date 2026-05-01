# configurator-ui

## Overview

Builds the FireFly Controller web UI from the FireFly-Controller repository with `VITE_CLOUD_MODE=true` and deploys the compiled assets to the `firefly-configurator-s3` S3 bucket, then invalidates the CloudFront cache. There is no CloudFormation stack — this workflow operates directly against S3 and CloudFront.

The Configurator UI is the same codebase as the on-device Controller web interface, built for cloud hosting. It is triggered automatically via `repository_dispatch` from FireFly-Controller when a new version is released, or manually via `workflow_dispatch`.

## Dependencies

### Deploy Dependencies

| Workflow | Reason |
|---|---|
| [s3-configurator](./s3-configurator.md) | Bucket name resolved from stack output |
| [cloudfront-configurator](./cloudfront-configurator.md) | CloudFront distribution ID resolved from stack output for cache invalidation |

### Delete Dependencies

None — there is no delete workflow. Assets are removed as part of `delete-s3-configurator`.

## Required By

### Required By Deploy

None.

### Required By Delete

None.

## Deploy Workflow

### Description

Checks out the FireFly-Controller repository at the specified ref (or `main` by default), sets build variables from git metadata, installs Node.js 20, runs `npm ci`, and performs a Vite production build with `VITE_CLOUD_MODE=true`. The compiled assets are synced to S3 with `--delete` to remove stale files, and a CloudFront wildcard invalidation ensures fresh content is served.

### Trigger

- `workflow_dispatch` — manually triggered; accepts `target_env` and optional `ref` (branch or tag to deploy from FireFly-Controller)
- `repository_dispatch` (type: `deploy-configurator-ui`) — triggered by FireFly-Controller on release; payload includes `target_env` and `ref`

### Steps

1. Check out the FireFly-Controller repository at the specified `ref`.
2. Set `COMMIT_HASH` and `BRANCH` from git metadata.
3. Configure AWS credentials.
4. Set up Node.js 20.
5. Run `npm ci` in `Controller/ui/`.
6. Run `npm run build` with environment variables:
   - `VITE_UI_VERSION` (branch/tag + commit hash)
   - `VITE_CLOUD_MODE=true`
7. Look up `BucketName` from the `firefly-configurator-s3` stack output.
8. `aws s3 sync Controller/ui/dist/ s3://{BucketName} --delete`
9. Look up `DistributionId` from the `firefly-configurator-cloudfront` stack output.
10. `aws cloudfront create-invalidation --distribution-id {DistributionId} --paths "/*"`

## Failure Scenarios

| Scenario | Behavior |
|---|---|
| npm build fails | No AWS changes are made; check Vite config or missing environment variables before re-running |
| `firefly-configurator-s3` stack lookup fails | Workflow fails before S3 sync; deploy `s3-configurator` first |
| `firefly-configurator-cloudfront` stack lookup fails | Workflow fails before cache invalidation; deploy `cloudfront-configurator` first |
| S3 sync partial failure | Re-run the workflow to retry — `--delete` ensures stale files are still removed |
| CloudFront invalidation fails | Old assets may be served from the edge cache for up to the TTL; re-run the invalidation step manually if needed |
