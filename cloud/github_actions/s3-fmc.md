# s3-fmc

## Overview

Provisions the S3 bucket used to store the FMC static assets. The bucket is fronted by the `cloudfront-fmc` CloudFront distribution and its contents are synced by the `fmc-app` workflow on each deploy.

## CloudFormation Stack

`firefly-s3-fmc`

## Dependencies

### Deploy

None — this workflow has no prerequisites.

### Delete

- `delete-cloudfront-fmc` — the CloudFront FMC distribution must be deleted first (which itself requires `delete-fmc-app`)

## Required By

### Deploy

- `cloudfront-fmc` — bucket name is passed as a parameter to the CloudFront FMC stack

### Delete

None.

---

## Deploy Workflow

### Description

Builds the SAM template and deploys the `firefly-s3-fmc` CloudFormation stack.

### Steps

1. Checkout repository
2. Configure AWS credentials
3. Install SAM CLI
4. `sam build` — template: `templates/s3-fmc.yaml`
5. `sam deploy` — stack: `firefly-s3-fmc`; params: `BucketName` (from secrets)

### Sequence Diagram

[![Deploy s3-fmc sequence](./images/deploy-s3-fmc.svg)](./images/deploy-s3-fmc.svg)

---

## Delete Workflow

### Description

Empties the S3 bucket before deleting the CloudFormation stack. Unlike `s3-firmware-public`, there is no production guard — the bucket is always emptied before stack deletion. The CloudFront FMC distribution must already be deleted so there are no active origins referencing the bucket.

### Steps

1. Configure AWS credentials
2. Install SAM CLI
3. Delete all objects in the bucket recursively
4. `sam delete --stack-name firefly-s3-fmc --no-prompts --region`

### Sequence Diagram

[![Delete s3-fmc sequence](./images/delete-s3-fmc.svg)](./images/delete-s3-fmc.svg)

---

## Failure Scenarios

| Scenario | Cause | Resolution |
|---|---|---|
| Bucket non-empty at stack deletion time | Bucket empty step skipped or failed; CloudFormation cannot delete a non-empty bucket | The workflow empties the bucket before calling `sam delete`; re-run the workflow if the empty step failed |
| `DELETE_FAILED` — CloudFront origin still active | `delete-cloudfront-fmc` did not complete before this job | Ensure the CloudFront FMC distribution is fully deleted, then re-run |
