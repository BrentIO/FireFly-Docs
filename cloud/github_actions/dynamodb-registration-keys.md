# dynamodb-registration-keys

## Overview

Provisions the DynamoDB table that holds one-time device registration keys. Keys have a 30-minute TTL enforced by DynamoDB's TTL feature. The deploy workflow includes import logic to adopt an existing table into CloudFormation without data loss.

## CloudFormation Stack

`firefly-dynamodb-registration-keys`

## Dependencies

### Deploy Dependencies

None — this workflow has no prerequisites.

### Delete Dependencies

| Workflow | Reason |
|---|---|
| [delete-func-api-registration-keys-post](./func-api-registration-keys-post.md) | Lambda IAM permissions referencing this table must be removed first |
| [delete-func-api-registration-keys-get](./func-api-registration-keys-get.md) | Lambda IAM permissions referencing this table must be removed first |
| [delete-func-api-devices-register-post](./func-api-devices-register-post.md) | Lambda IAM permissions referencing this table must be removed first |

## Required By

### Required By Deploy

| Workflow | Reason |
|---|---|
| [func-api-registration-keys-post](./func-api-registration-keys-post.md) | Table must exist before the function is deployed and granted write access |
| [func-api-registration-keys-get](./func-api-registration-keys-get.md) | Table must exist before the function is deployed and granted scan access |
| [func-api-devices-register-post](./func-api-devices-register-post.md) | Table must exist before the function is deployed and granted read/delete access |

### Required By Delete

None.

## Deploy Workflow

### Description

Deploys the `firefly-dynamodb-registration-keys` CloudFormation stack. The workflow includes import logic to handle the case where the DynamoDB table already exists but the CloudFormation stack does not — this allows adopting a pre-existing table without data loss.

**Normal path** (stack exists): runs `aws cloudformation deploy` with `--no-fail-on-empty-changeset`.

**Import path** (table exists, stack does not):
1. Creates a CloudFormation `IMPORT` changeset to adopt the existing table.
2. Waits for the changeset to reach `CREATE_COMPLETE`.
3. Executes the changeset and waits for `IMPORT_COMPLETE`.
4. Runs a second `aws cloudformation deploy` to add the `Outputs` section (omitted from the import template body).

**Fresh path** (neither stack nor table exists): runs `aws cloudformation deploy` normally.

### Steps

1. Configure AWS credentials.
2. Run the Python deploy script, which:
   - Checks whether the `firefly-dynamodb-registration-keys` CloudFormation stack exists.
   - Checks whether the `firefly-registration-keys` DynamoDB table exists.
   - Follows the appropriate deploy path (normal, import, or fresh) as described above.

## Delete Workflow

### Description

Calls `sam delete` to remove the `firefly-dynamodb-registration-keys` stack and its associated DynamoDB table. All Lambda functions that reference this table must be deleted first.

The table has `DeletionProtectionEnabled: true` — deletion protection must be manually disabled in the AWS Console before the stack can be deleted.

### Steps

1. Configure AWS credentials.
2. SAM delete `firefly-dynamodb-registration-keys`.

## Failure Scenarios

| Scenario | Behavior |
|---|---|
| Table exists, stack does not | Deploy workflow runs the import path to adopt the table into CloudFormation. |
| Deletion protection enabled | Stack deletion fails with `DELETE_FAILED`. Disable deletion protection on the table in the AWS Console, then re-run. |
| Dependent Lambda stacks not deleted first | Stack deletion fails because IAM resource-based policies still reference the table. Delete Lambda stacks first. |
