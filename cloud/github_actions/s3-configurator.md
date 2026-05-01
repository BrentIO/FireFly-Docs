# s3-configurator

## Overview

Provisions the private S3 bucket used to store the Configurator UI static assets. The bucket has all public access blocked and is accessed exclusively through the `cloudfront-configurator` CloudFront distribution via Origin Access Control. Contents are synced by the `deploy-configurator-ui` workflow on each deploy.

## CloudFormation Stack

`firefly-configurator-s3`

## CloudWatch Logs

None — this stack contains only an S3 bucket.

## Dependencies

### Deploy Dependencies

None — this workflow has no prerequisites.

### Delete Dependencies

| Workflow | Reason |
|---|---|
| [cloudfront-configurator](./cloudfront-configurator.md) | CloudFront distribution must be deleted before the origin bucket can be removed |

## Required By

### Required By Deploy

| Workflow | Reason |
|---|---|
| [cloudfront-configurator](./cloudfront-configurator.md) | Bucket must exist before the distribution can reference it |

### Required By Delete

None.

## Deploy Workflow

### Description

Deploys the `firefly-configurator-s3` CloudFormation stack, creating a private S3 bucket with all public access blocked.

### Steps

1. Configure AWS credentials.
2. Install SAM CLI.
3. SAM deploy `firefly-configurator-s3` with parameter:
   - `BucketName` (from `S3_CONFIGURATOR_BUCKET_NAME` secret)

## Delete Workflow

### Description

Empties the S3 bucket (including aborting any in-progress multipart uploads) before deleting the CloudFormation stack. Exits cleanly if the bucket does not exist.

### Steps

1. Configure AWS credentials.
2. Install SAM CLI.
3. Recursively delete all objects in the bucket. Skip if bucket does not exist.
4. Abort all in-progress multipart uploads. Skip if bucket does not exist.
5. SAM delete `firefly-configurator-s3`.

## Failure Scenarios

| Scenario | Behavior |
|---|---|
| Bucket non-empty at stack deletion time | The workflow empties the bucket before calling `sam delete`; re-run if the empty step failed |
| `DELETE_FAILED` — CloudFront origin still active | `delete-cloudfront-configurator` did not complete first; delete the CloudFront stack, then re-run |
