# cloudfront-fmc

## Overview

Provisions the CloudFront distribution that fronts the `firefly-s3-fmc` bucket for FMC delivery. Also creates the Route 53 alias record mapping the FMC domain name to the CloudFront distribution.

## CloudFormation Stack

`firefly-cloudfront-fmc`

## Dependencies

### Deploy

- `acm` — provides the `CertificateArn` for the CloudFront alternate domain
- `s3-fmc` — the origin bucket must exist before the distribution can reference it

### Delete

- `delete-fmc-app` — FMC assets and any active CloudFront references should be cleared first

## Required By

### Deploy

- `fmc-app` — the CloudFront distribution ID and domain are needed to deploy the FMC app and invalidate the cache

### Delete

- `delete-s3-fmc` — CloudFront FMC distribution must be deleted before the origin bucket
- `delete-acm` (transitively)

---

## Deploy Workflow

### Description

Looks up the `CertificateArn` from the `firefly-acm` stack output, then deploys the `firefly-cloudfront-fmc` CloudFormation stack. CloudFront distribution propagation takes 15–20 minutes. The stack also creates a Route 53 ALIAS record for the FMC domain.

### Steps

1. Checkout repository
2. Configure AWS credentials
3. Install SAM CLI
4. Lookup `CertificateArn` from `firefly-acm` stack output
5. `sam build` — template: `templates/cloudfront-fmc.yaml`
6. `sam deploy` — stack: `firefly-cloudfront-fmc`; params: `FmcBucketName`, `CertificateArn`, `FmcDomain`, `HostedZoneId`

### Sequence Diagram

[![Deploy cloudfront-fmc sequence](./images/deploy-cloudfront-fmc.svg)](./images/deploy-cloudfront-fmc.svg)

---

## Delete Workflow

### Description

Deletes the `firefly-cloudfront-fmc` CloudFormation stack, removing the CloudFront distribution and the Route 53 alias record. Must run after `delete-fmc-app` to ensure the distribution is not serving live traffic.

### Steps

1. Configure AWS credentials
2. Install SAM CLI
3. `sam delete --stack-name firefly-cloudfront-fmc --no-prompts --region`

### Sequence Diagram

[![Delete cloudfront-fmc sequence](./images/delete-cloudfront-fmc.svg)](./images/delete-cloudfront-fmc.svg)

---

## Failure Scenarios

| Scenario | Cause | Resolution |
|---|---|---|
| CNAME already associated with another distribution | Another CloudFront distribution in the account has the same alternate domain name | Remove the alternate domain name from the conflicting distribution, then re-run |
| Stack left in `UPDATE_IN_PROGRESS` after cancellation | Workflow was cancelled during the 15–20 minute CloudFront propagation window | Wait for the update to finish before re-running |
| `CertificateArn` lookup fails | `firefly-acm` stack not deployed or output not present | Deploy `acm` first |
