# fmc-app

## Overview

Builds the Vite/React FMC and deploys it to S3, then invalidates the CloudFront cache so devices and browsers receive the updated assets immediately. There is no CloudFormation stack — this workflow operates directly against S3 and CloudFront.

## Dependencies

### Deploy Dependencies

| Workflow | Reason |
|---|---|
| [cloudfront-fmc](./cloudfront-fmc.md) | CloudFront distribution ID and S3 bucket name resolved from stack outputs |
| [cognito](./cognito.md) | Cognito Client ID injected as a Vite environment variable at build time |

### Delete Dependencies

None — this workflow has no prerequisites.

## Required By

### Required By Deploy

| Workflow | Reason |
|---|---|
| [run-integration-tests](./integration-tests.md) | FMC must be deployed before integration tests validate the full stack |

### Required By Delete

| Workflow | Reason |
|---|---|
| [delete-cloudfront-fmc](./cloudfront-fmc.md) | S3 bucket must be emptied before the CloudFront/S3 stacks can be deleted |

## Deploy Workflow

### Description

Looks up runtime configuration from three CloudFormation stacks, installs Node.js 20, runs `npm ci`, and performs a Vite production build with environment variables injected. The compiled assets are synced to S3 with `--delete` to remove stale files, and a CloudFront wildcard invalidation ensures fresh content is served.

### Steps

1. Configure AWS credentials.
2. Look up `CognitoClientId` from the `firefly-cognito` stack output.
3. Look up `CloudFrontDistributionId` from the `firefly-cloudfront-fmc` stack output.
4. Look up `BucketName` from the `firefly-s3-fmc` stack output.
5. Set up Node.js 20.
6. Run `npm ci` in the `ui/` directory.
7. Run `npm run build` with environment variables:
   - `VITE_API_URL` (constructed as `https://` + `vars.API_DOMAIN_NAME`)
   - `VITE_COMMIT_SHA` (from `github.sha`)
   - `VITE_COGNITO_DOMAIN` (from `vars.AUTH_DOMAIN_NAME`)
   - `VITE_COGNITO_CLIENT_ID` (from Cognito stack lookup)
   - `VITE_COGNITO_REDIRECT_URI` (from `vars.FMC_DOMAIN_NAME`)
8. `aws s3 sync ui/dist/ s3://{BucketName} --delete`
9. `aws cloudfront create-invalidation --distribution-id {CloudFrontDistributionId} --paths "/*"`

### Sequence Diagram

[![Deploy Sequence](./images/deploy-fmc-app.svg)](./images/deploy-fmc-app.svg)

## Delete Workflow

### Description

Empties the S3 FMC bucket so that the `delete-cloudfront-fmc` and `delete-s3-fmc` workflows can proceed without a non-empty bucket error. Empties the bucket directly using the `S3_FMC_BUCKET_NAME` secret; exits 0 if the bucket does not exist (idempotent).

### Steps

1. Configure AWS credentials.
2. Empty S3 FMC bucket using `S3_FMC_BUCKET_NAME` secret. Exit 0 if bucket does not exist.

### Sequence Diagram

[![Delete Sequence](./images/delete-fmc-app.svg)](./images/delete-fmc-app.svg)

## Failure Scenarios

| Scenario | Behavior |
|---|---|
| npm build fails | No AWS changes are made; check Vite config or missing environment variables before re-running. |
| Cognito stack lookup fails | Build proceeds with an empty/incorrect client ID; authentication will not work at runtime. Deploy `cognito` first. |
| CloudFront distribution ID lookup fails | Workflow fails before S3 sync; deploy `cloudfront-fmc` first. |
| S3 sync partial failure | Some files may not upload; re-run the deploy workflow to retry — `--delete` ensures stale files are still removed. |
| CloudFront invalidation fails | Old assets may be served from the edge cache for up to the TTL; this is not a deployment blocker. Re-run the invalidation step manually if needed. |
