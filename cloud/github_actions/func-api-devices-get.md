# func-api-devices-get

## Overview

Deploys the Lambda function that handles `GET /devices`. Returns all registered device records from the `firefly-devices` DynamoDB table. Powers the **Registered Devices** page in the management console. The route is authenticated via the Cognito JWT authorizer and restricted to super users.

## CloudFormation Stack

`firefly-func-api-devices-get`

## CloudWatch Logs

| Setting | Value |
|---|---|
| Log group | `/aws/lambda/firefly-func-api-devices-get` |
| Retention | 30 days |

## Dependencies

### Deploy Dependencies

| Workflow | Reason |
|---|---|
| [api-gateway](./api-gateway.md) | `ApiId` and `AuthorizerId` resolved from stack outputs |
| [dynamodb-devices](./dynamodb-devices.md) | Table must exist before the function is deployed and granted scan access |
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

## IAM Permissions

The Lambda execution role (`firefly-func-api-devices-get-role`) is granted:

- `dynamodb:Scan` on `firefly-devices`
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
6. SAM deploy `firefly-func-api-devices-get` with parameters:
   - `ApiId`
   - `AuthorizerId`
   - `SharedLayerArn`
   - `AppConfigExtensionLayerArn`

## Delete Workflow

### Description

Calls `sam delete` to remove the Lambda function, its IAM role, and the API Gateway route integration. Also deletes the CloudWatch log group.

### Steps

1. Configure AWS credentials.
2. SAM delete `firefly-func-api-devices-get`.
3. Delete CloudWatch log group `/aws/lambda/firefly-func-api-devices-get`.

## Failure Scenarios

| Scenario | Behavior |
|---|---|
| `firefly-api-gateway` stack not found | `describe-stacks` returns an error; workflow fails before SAM deploy. Deploy `api-gateway` first. |
| `firefly-dynamodb-devices` stack not deployed | Function deploys but returns errors at runtime. Deploy `dynamodb-devices` first. |
| `firefly-shared-layer` stack not found | Layer ARN lookup fails; SAM deploy is not attempted. Deploy `shared-layer` first. |
| Caller is not a super user | Lambda returns `403 Forbidden`. |
