# func-api-devices-registration-get

## Overview

Deploys the Lambda function that handles `GET /devices/{uuid}/registration`. Called by the HW-Reg application at boot to verify that a device is registered with FireFly-Cloud. Authenticates the request by verifying an ECDSA P-256 signature over a caller-supplied nonce using the public key stored at registration time. This route has **no** Cognito JWT authorizer — it is authenticated solely by the device's cryptographic signature.

## CloudFormation Stack

`firefly-func-api-devices-registration-get`

## CloudWatch Logs

| Setting | Value |
|---|---|
| Log group | `/aws/lambda/firefly-func-api-devices-registration-get` |
| Retention | 30 days |

## Dependencies

### Deploy Dependencies

| Workflow | Reason |
|---|---|
| [api-gateway](./api-gateway.md) | `ApiId` resolved from stack outputs |
| [dynamodb-devices](./dynamodb-devices.md) | Table must exist before the function is deployed and granted read access |
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
| [delete-shared-layer](./shared-layer.md) | Layer reference must be removed before the layer stack is deleted |

## IAM Permissions

The Lambda execution role (`firefly-func-api-devices-registration-get-role`) is granted:

- `dynamodb:GetItem` on `firefly-devices`
- `appconfig:StartConfigurationSession`, `appconfig:GetLatestConfiguration` on `*`

## Deploy Workflow

### Description

Resolves the HTTP API Gateway ID, shared layer ARN, and AppConfig extension layer ARN from CloudFormation stack outputs, then performs a SAM deploy. This function does not use a JWT authorizer — no `AuthorizerId` parameter is needed.

### Steps

1. Configure AWS credentials.
2. Look up `ApiId` from the `firefly-api-gateway` stack output.
3. Look up `SharedLayerArn` from the `firefly-shared-layer` stack output.
4. Look up `AppConfigExtensionLayerArn` from the `firefly-shared-layer` stack output.
5. SAM deploy `firefly-func-api-devices-registration-get` with parameters:
   - `ApiId`
   - `SharedLayerArn`
   - `AppConfigExtensionLayerArn`

## Delete Workflow

### Description

Calls `sam delete` to remove the Lambda function, its IAM role, and the API Gateway route integration. Also deletes the CloudWatch log group.

### Steps

1. Configure AWS credentials.
2. SAM delete `firefly-func-api-devices-registration-get`.
3. Delete CloudWatch log group `/aws/lambda/firefly-func-api-devices-registration-get`.

## Failure Scenarios

| Scenario | Behavior |
|---|---|
| `firefly-api-gateway` stack not found | `describe-stacks` returns an error; workflow fails before SAM deploy. Deploy `api-gateway` first. |
| `firefly-dynamodb-devices` stack not deployed | Function deploys but returns errors at runtime. Deploy `dynamodb-devices` first. |
| `firefly-shared-layer` stack not found | Layer ARN lookup fails; SAM deploy is not attempted. Deploy `shared-layer` first. |
| Device UUID not found | Lambda returns `401 Unauthorized`. |
| Invalid or mismatched signature | Lambda returns `401 Unauthorized`. |
