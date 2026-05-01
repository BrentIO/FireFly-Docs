# func-api-devices-get

## Description

Returns all registered device records from the `firefly-devices` DynamoDB table. Powers the **Registered Devices** page in the management console.

Access is restricted to super users. Non-super-user callers receive `403 Forbidden`.

## Invocation

Invoked by **API Gateway** on an HTTP `GET /devices` request, authenticated via the Cognito JWT authorizer.

## API Endpoints

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/devices` | Cognito JWT (super users only) | List all registered devices |

## Response Body

```json
{
  "devices": [
    {
      "uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "product_id": "FFC0806-2505",
      "product_hex": "0x08062505",
      "device_class": "controller",
      "registration_date": "2025-04-01T12:00:00Z",
      "registering_application": "Hardware-Registration-and-Configuration",
      "registering_version": "2025.04.01",
      "mcu": { ... },
      "network": [ ... ],
      "partitions": [ ... ]
    }
  ]
}
```

Results are sorted ascending by `registration_date`.

## Response Codes

| Code | Reason |
|---|---|
| `200 OK` | List returned (may be empty) |
| `403 Forbidden` | Caller is not in the `super_users` group |
| `500 Internal Server Error` | Unhandled exception |

See the [API Reference](/cloud/api_reference) for full schema documentation.

## Deployment

See the [deployment workflow documentation](../github_actions/func-api-devices-get.md) for workflow steps, infrastructure dependencies, and failure scenarios.
