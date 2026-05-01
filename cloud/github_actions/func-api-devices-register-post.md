# func-api-devices-register-post

## Overview

Deploys the Lambda function that handles `POST /devices/register`. Validates a one-time registration key, writes the device record to `firefly-devices`, and deletes the consumed key. This route is **not** Cognito-authenticated — it is authenticated solely by the `X-Registration-Key` header supplied by the HW-Reg application.

## CloudFormation Stack

`firefly-func-api-devices-register-post`

## CloudWatch Logs

| Setting | Value |
|---|---|
| Log group | `/aws/lambda/firefly-func-api-devices-register-post` |
| Retention | 30 days |

## Dependencies

### Deploy Dependencies

| Workflow | Reason |
|---|---|
| [api-gateway](./api-gateway.md) | `ApiId` resolved from stack outputs |
| [dynamodb-devices](./dynamodb-devices.md) | Table must exist before the function is deployed and granted write access |
| [dynamodb-registration-keys](./dynamodb-registration-keys.md) | Table must exist before the function is deployed and granted read/delete access |
| [shared-layer](./shared-layer.md) | Lambda layer must exist before function deployment |

### Delete Dependencies

None — this workflow has no prerequisites.

## Required By

### Required By Deploy

| Workflow | Reason |
|---|---|
| [run-integration-tests](./integration-tests.md) | Endpoint must be live before integration tests run |

### Required By Delete

| Workflow | Reason |
|---|---|
| [delete-api-gateway](./api-gateway.md) | Route registration must be removed before the API Gateway stack is deleted |
| [delete-dynamodb-devices](./dynamodb-devices.md) | IAM permissions referencing the table must be removed first |
| [delete-dynamodb-registration-keys](./dynamodb-registration-keys.md) | IAM permissions referencing the table must be removed first |
| [delete-shared-layer](./shared-layer.md) | Layer reference must be removed before the layer stack is deleted |

## IAM Permissions

The Lambda execution role (`firefly-func-api-devices-register-post-role`) is granted:

- `dynamodb:GetItem`, `dynamodb:PutItem` on `firefly-devices`
- `dynamodb:GetItem`, `dynamodb:DeleteItem` on `firefly-registration-keys`
- `appconfig:StartConfigurationSession`, `appconfig:GetLatestConfiguration` on `*`

## Deploy Workflow

### Description

Resolves the HTTP API Gateway ID, shared layer ARN, and AppConfig extension layer ARN from CloudFormation stack outputs, then performs a SAM deploy. This function does not use a JWT authorizer — no `AuthorizerId` parameter is needed.

### Steps

1. Configure AWS credentials.
2. Look up `ApiId` from the `firefly-api-gateway` stack output.
3. Look up `SharedLayerArn` from the `firefly-shared-layer` stack output.
4. Look up `AppConfigExtensionLayerArn` from the `firefly-shared-layer` stack output.
5. SAM deploy `firefly-func-api-devices-register-post` with parameters:
   - `ApiId`
   - `SharedLayerArn`
   - `AppConfigExtensionLayerArn`

## Delete Workflow

### Description

Calls `sam delete` to remove the Lambda function, its IAM role, and the API Gateway route integration. Also deletes the CloudWatch log group.

### Steps

1. Configure AWS credentials.
2. SAM delete `firefly-func-api-devices-register-post`.
3. Delete CloudWatch log group `/aws/lambda/firefly-func-api-devices-register-post`.

## Failure Scenarios

| Scenario | Behavior |
|---|---|
| `firefly-api-gateway` stack not found | `describe-stacks` returns an error; workflow fails before SAM deploy. Deploy `api-gateway` first. |
| `firefly-dynamodb-devices` stack not deployed | Function deploys but returns errors at runtime when writing device records. Deploy `dynamodb-devices` first. |
| `firefly-dynamodb-registration-keys` stack not deployed | Function deploys but cannot validate registration keys at runtime. Deploy `dynamodb-registration-keys` first. |
| `firefly-shared-layer` stack not found | Layer ARN lookup fails; SAM deploy is not attempted. Deploy `shared-layer` first. |
| Invalid or expired registration key | Lambda returns `401 Unauthorized`. |
| Device UUID already registered | Lambda returns `204` immediately without modifying the existing record (idempotent). |
