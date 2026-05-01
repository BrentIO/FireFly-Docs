# func-api-registration-keys-post

## Overview

Deploys the Lambda function that handles `POST /registration-keys`. Generates one-time 6-character device registration keys stored in DynamoDB with a 30-minute TTL. The route is authenticated via the Cognito JWT authorizer (any authenticated user).

## CloudFormation Stack

`firefly-func-api-registration-keys-post`

## CloudWatch Logs

| Setting | Value |
|---|---|
| Log group | `/aws/lambda/firefly-func-api-registration-keys-post` |
| Retention | 30 days |

## Dependencies

### Deploy Dependencies

| Workflow | Reason |
|---|---|
| [api-gateway](./api-gateway.md) | `ApiId` and `AuthorizerId` resolved from stack outputs |
| [dynamodb-registration-keys](./dynamodb-registration-keys.md) | Table must exist before the function is deployed and granted write access |
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
| [delete-dynamodb-registration-keys](./dynamodb-registration-keys.md) | IAM permissions referencing the table must be removed first |
| [delete-shared-layer](./shared-layer.md) | Layer reference must be removed before the layer stack is deleted |

## IAM Permissions

The Lambda execution role (`firefly-func-api-registration-keys-post-role`) is granted:

- `dynamodb:PutItem` on `firefly-registration-keys`
- `appconfig:StartConfigurationSession`, `appconfig:GetLatestConfiguration` on `*`

## Deploy Workflow

### Description

Resolves the HTTP API Gateway ID, JWT Authorizer ID, shared layer ARN, and AppConfig extension layer ARN from CloudFormation stack outputs, then performs a SAM deploy.

### Steps

1. Configure AWS credentials.
2. Look up `ApiId` from the `firefly-api-gateway` stack output.
3. Look up `AuthorizerId` from the `firefly-api-gateway` stack output.
4. Look up `SharedLayerArn` from the `firefly-shared-layer` stack output.
5. Look up `AppConfigExtensionLayerArn` from the `firefly-shared-layer` stack output.
6. SAM deploy `firefly-func-api-registration-keys-post` with parameters:
   - `ApiId`
   - `AuthorizerId`
   - `SharedLayerArn`
   - `AppConfigExtensionLayerArn`

## Delete Workflow

### Description

Calls `sam delete` to remove the Lambda function, its IAM role, and the API Gateway route integration. Also deletes the CloudWatch log group.

### Steps

1. Configure AWS credentials.
2. SAM delete `firefly-func-api-registration-keys-post`.
3. Delete CloudWatch log group `/aws/lambda/firefly-func-api-registration-keys-post`.

## Failure Scenarios

| Scenario | Behavior |
|---|---|
| `firefly-api-gateway` stack not found | `describe-stacks` returns an error; workflow fails before SAM deploy. Deploy `api-gateway` first. |
| `firefly-dynamodb-registration-keys` stack not deployed | Function deploys but returns errors at runtime. Deploy `dynamodb-registration-keys` first. |
| `firefly-shared-layer` stack not found | Layer ARN lookup fails; SAM deploy is not attempted. Deploy `shared-layer` first. |
