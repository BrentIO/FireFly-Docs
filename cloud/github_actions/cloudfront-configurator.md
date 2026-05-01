# cloudfront-configurator

## Overview

Provisions the CloudFront distribution that fronts the `firefly-configurator-s3` bucket for Configurator UI delivery. Also creates the Route 53 alias record mapping `CONFIGURATOR_DOMAIN_NAME` to the CloudFront distribution. All traffic is redirected to HTTPS; TTL is set to 0 so asset updates from `deploy-configurator-ui` are served immediately without a cache invalidation gap.

## CloudFormation Stack

`firefly-configurator-cloudfront`

## CloudWatch Logs

None — this stack contains only a CloudFront distribution and Route 53 record.

## Dependencies

### Deploy Dependencies

| Workflow | Reason |
|---|---|
| [acm](./acm.md) | `CertificateArn` resolved from stack output |
| [s3-configurator](./s3-configurator.md) | Origin bucket must exist before the distribution can reference it |

### Delete Dependencies

None — this workflow has no prerequisites and runs in the first wave of `delete-all`.

## Required By

### Required By Deploy

| Workflow | Reason |
|---|---|
| [configurator-ui](./configurator-ui.md) | CloudFront distribution ID resolved from stack output for cache invalidation |

### Required By Delete

| Workflow | Reason |
|---|---|
| [delete-s3-configurator](./s3-configurator.md) | CloudFront distribution must be deleted before the origin bucket |
| [delete-acm](./acm.md) | ACM certificate must not be in use by any CloudFront distribution at deletion time |

## Deploy Workflow

### Description

Looks up the `CertificateArn` from the `firefly-acm` stack output, then deploys the `firefly-configurator-cloudfront` CloudFormation stack. CloudFront propagation takes 15–20 minutes. The stack also creates a Route 53 ALIAS record for the Configurator domain.

### Steps

1. Configure AWS credentials.
2. Install SAM CLI.
3. Look up `CertificateArn` from the `firefly-acm` stack output.
4. SAM deploy `firefly-configurator-cloudfront` with parameters:
   - `ConfiguratorBucketName` (from `S3_CONFIGURATOR_BUCKET_NAME` secret)
   - `CertificateArn`
   - `ConfiguratorDomain` (from `CONFIGURATOR_DOMAIN_NAME` variable)
   - `HostedZoneId` (from `ROUTE_53_HOSTED_ZONE_ID` secret)

## Delete Workflow

### Description

Deletes the `firefly-configurator-cloudfront` CloudFormation stack, removing the CloudFront distribution and the Route 53 alias record.

### Steps

1. Configure AWS credentials.
2. Install SAM CLI.
3. SAM delete `firefly-configurator-cloudfront`.

## Failure Scenarios

| Scenario | Behavior |
|---|---|
| `CertificateArn` lookup fails | `firefly-acm` stack not deployed or output missing; deploy `acm` first |
| CNAME already associated with another distribution | Another CloudFront distribution has the same alternate domain; remove it from that distribution, then re-run |
| Stack left in `UPDATE_IN_PROGRESS` after cancellation | Wait for the update to finish (up to 20 minutes) before re-running |
