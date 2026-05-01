# dynamodb-devices

## Overview

Provisions the DynamoDB table that holds registered device records. Each record is keyed by device UUID and includes product identity, MCU details, network interfaces, partition table, public key, and registration metadata. The table has a GSI on `product_hex` for product-scoped queries.

## CloudFormation Stack

`firefly-dynamodb-devices`

## Dependencies

### Deploy Dependencies

None — this workflow has no prerequisites.

### Delete Dependencies

| Workflow | Reason |
|---|---|
| [delete-func-api-devices-register-post](./func-api-devices-register-post.md) | Lambda IAM permissions referencing this table must be removed first |
| [delete-func-api-devices-registration-get](./func-api-devices-registration-get.md) | Lambda IAM permissions referencing this table must be removed first |
| [delete-func-api-devices-get](./func-api-devices-get.md) | Lambda IAM permissions referencing this table must be removed first |

## Required By

### Required By Deploy

| Workflow | Reason |
|---|---|
| [func-api-devices-register-post](./func-api-devices-register-post.md) | Table must exist before the function is deployed and granted write access |
| [func-api-devices-registration-get](./func-api-devices-registration-get.md) | Table must exist before the function is deployed and granted read access |
| [func-api-devices-get](./func-api-devices-get.md) | Table must exist before the function is deployed and granted scan access |

### Required By Delete

None.

## Deploy Workflow

### Description

Deploys the `firefly-dynamodb-devices` CloudFormation stack using `aws cloudformation deploy`.

### Steps

1. Configure AWS credentials.
2. Deploy `firefly-dynamodb-devices` from `templates/dynamodb-devices.yaml`.

## Delete Workflow

### Description

Calls `sam delete` to remove the `firefly-dynamodb-devices` stack and its associated DynamoDB table. All Lambda functions that reference this table must be deleted first.

The table has `DeletionProtectionEnabled: true` — deletion protection must be manually disabled in the AWS Console before the stack can be deleted.

### Steps

1. Configure AWS credentials.
2. SAM delete `firefly-dynamodb-devices`.

## Failure Scenarios

| Scenario | Behavior |
|---|---|
| Deletion protection enabled | Stack deletion fails with `DELETE_FAILED`. Disable deletion protection on the table in the AWS Console, then re-run. |
| Dependent Lambda stacks not deleted first | Stack deletion fails because IAM resource-based policies still reference the table. Delete Lambda stacks first. |
