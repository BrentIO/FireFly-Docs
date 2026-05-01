# Cloud Development Environment

This guide walks through bootstrapping a single AWS account to use the FireFly Cloud deployment workflows.  You will prepare the IAM policy files, create the required AWS resources, and then configure GitHub with the credentials and settings the workflows need.  Once complete, the workflows will automatically create, update, and delete the CloudFormation stacks that make up the FireFly Cloud — including S3 buckets, API Gateway, DynamoDB, and Lambda functions.

::: info Multi-Account Setup
FireFly Cloud uses separate DEV and PROD accounts.  If you are setting up the full account structure for the first time, start with the [AWS Account Setup](/cloud/aws-account-setup) guide, which covers IAM Identity Center, DNS delegation, and walks you through this guide for each account.
:::

This guide assumes Route 53 is already configured for the account with a hosted zone and custom domain name.

::: info AWS Region Support
Only **us-east-1** region is supported.
:::

## Step 1: Prepare Policy Files

Before creating anything in AWS, update the placeholder values in the policy files in the `policies/` directory:

- `AWS_ACCOUNT_ID` — your AWS account ID.
- `AWS_REGION` — the region you plan to deploy to.
- `S3_FIRMWARE_PRIVATE_BUCKET_NAME` — the S3 bucket name you plan to use to store firmware ZIPs.
- `S3_FIRMWARE_PUBLIC_BUCKET_NAME` — the S3 bucket name you plan to use for public OTA firmware delivery.
- `S3_UI_BUCKET_NAME` — the S3 bucket name you plan to use for the UI static files.
- `S3_SAM_DEPLOYMENT_BUCKET_NAME` — the name of the S3 bucket where CloudFormation deployment templates will be stored.
- `ROUTE_53_HOSTED_ZONE_ID` — the Hosted Zone ID for your Route 53 instance.

The following policy files require updates:
- `policies/firefly-github-actions-cloudformation-access-policy.json`
- `policies/firefly-cloudformation-execution-policy.json`
- `policies/firefly-github-actions-role_trust-relationships.json`

## Step 2: AWS Setup

### SAM Deployment Bucket

1. Create an S3 bucket to store CloudFormation deployment templates.  This bucket must be in the same region you plan to deploy to.
2. Name it to match the `S3_SAM_DEPLOYMENT_BUCKET_NAME` value you used in Step 1.


### IAM Roles

#### CloudFormation Execution Role
1. Create a new role named `firefly-cloudformation-execution-role`.
   - Trusted entity type: **AWS service**
   - Service: **CloudFormation**
2. After the role is created, go to **Trust relationships** → **Edit trust policy** and replace the generated policy with the contents of `policies/firefly-cloudformation-execution-role_trust-relationships.json`.

::: info Note
Do not create or attach permissions for the role at this time.
:::

#### GitHub Actions Role
Follow the [GitHub Actions AWS Setup](/cloud/github_actions/aws-oidc-setup) guide to create the OIDC identity provider and the `firefly-github-actions-role` IAM role that GitHub Actions uses to authenticate to AWS.

### IAM Policies

#### CloudFormation Access Policy
This policy allows the GitHub Actions role to execute CloudFormation scripts and assume the CloudFormation Execution role.
1. Create a new policy using the updated statements in `policies/firefly-github-actions-cloudformation-access-policy.json`.
2. Name the policy `firefly-github-actions-cloudformation-access-policy`.
3. Attach IAM role entity `firefly-github-actions-role` to the policy.

#### CloudFormation Execution Policy
This policy allows CloudFormation to deploy and delete the individual AWS services in each stack.
1. Create a new policy using the updated statements in `policies/firefly-cloudformation-execution-policy.json`.
2. Name the policy `firefly-cloudformation-execution-policy`.
3. Attach IAM role entity `firefly-cloudformation-execution-role` to the policy.

## Step 3: GitHub Setup

### GitHub Environments

The workflows deploy to either a `dev` or `production` environment.  Create both environments in the repository settings under **Settings > Environments**.

::: info Note
Secrets and variables must be set at the **environment** level, not at the repository level.
:::

### GitHub Secrets

The following secrets must be configured in each GitHub environment:

| Name | Example Value | Description |
| ---- | ------------- | ----------- |
| `AWS_ACCOUNT_ID` | 1234567890 | Your AWS account ID. |
| `AWS_REGION` | us-east-1 | The AWS region you plan to deploy to. |
| `AWS_ROLE_ARN` | arn:aws:iam::1234567890:role/firefly-github-actions-role | ARN of the IAM role GitHub Actions assumes via OIDC. |
| `GOOGLE_CLIENT_ID` | | OAuth 2.0 Client ID from Google Cloud Console. See [Google Cloud Setup](#google-cloud-setup). |
| `GOOGLE_CLIENT_SECRET` | | OAuth 2.0 Client Secret from Google Cloud Console. See [Google Cloud Setup](#google-cloud-setup). |
| `ROUTE_53_HOSTED_ZONE_ID` | AB1234567 | The Hosted Zone ID for your Route 53 instance. |
| `S3_FIRMWARE_PRIVATE_BUCKET_NAME` | my-firmware-private | The S3 bucket name for storing firmware ZIPs (private). |
| `S3_FIRMWARE_PUBLIC_BUCKET_NAME` | my-firmware-public | The S3 bucket name for OTA firmware binary delivery (public). |
| `S3_UI_BUCKET_NAME` | my-firefly-ui | The S3 bucket name for the UI static files (private, served via CloudFront). |
| `S3_SAM_DEPLOYMENT_BUCKET_NAME` | my-sam-deployment-bucket | The name of the bucket where deployment templates will be stored. |

### GitHub Variables

The following variables must be configured in each GitHub environment:

| Name | Example Value | Description |
| ---- | ------------- | ----------- |
| `API_DOMAIN_NAME` | api.somewhere.com | The domain name for the API gateway. |
| `APPCONFIG_EXTENSION_LAYER_ARM64_ARN` | arn:aws:lambda:us-east-1:027255383542:layer:AWS-AppConfig-Extension-Arm64:250 | ARN for the AppConfig layer; refer to the [AWS version reference](https://docs.aws.amazon.com/appconfig/latest/userguide/appconfig-integration-lambda-extensions-versions.html) for the correct ARN |
| `AUTH_DOMAIN_NAME` | auth.somewhere.com | The custom domain for the Cognito hosted UI (e.g., `auth.example.com`). A Route 53 alias record is created automatically during deployment. |
| `CERTIFICATE_DOMAIN_NAME` | *.somewhere.com | A wildcard to your domain. |
| `CLOUD_FORMATION_EXECUTION_ROLE_NAME` | firefly-cloudformation-execution-role | Name of the execution role. |
| `CLEANUP_TEST_RECORDS` | true | Deletes all test records in DynamoDB from the integration tests when `true` |
| `FIRMWARE_DOMAIN_NAME` | firmware.somewhere.com | The domain name for the CloudFront firmware distribution. |
| `FIRMWARE_TYPE_MAP` | `{"Controller":"FireFly Controller"}` | JSON mapping from URL application name to the firmware type string expected by the device. |
| `UI_DOMAIN_NAME` | `ui.somewhere.com` | The custom domain name for the firmware management UI, without the `https://` scheme. |

## Google Cloud Setup

FireFly uses Google as the only identity provider for the management console.  This is a one-time setup per Google account.

1. Go to the [Google Cloud Console](https://console.cloud.google.com) and create a new project (e.g., `firefly-auth`).
2. In the left menu, go to **APIs & Services** → **OAuth consent screen**.
   - Choose **External** user type.
   - Fill in the required fields (app name, support email, developer contact).
   - Add the scope `openid`, `email`, and `profile`.
   - Add your Google accounts to the **Test users** list while the app is in testing mode.
3. Go to **APIs & Services** → **Credentials** → **Create Credentials** → **OAuth 2.0 Client ID**.
   - Application type: **Web application**.
   - Under **Authorized redirect URIs**, add the Cognito hosted UI callback URL:
     ```
     https://{AUTH_DOMAIN_NAME}/oauth2/idpresponse
     ```
     Replace `{AUTH_DOMAIN_NAME}` with the value of the `AUTH_DOMAIN_NAME` GitHub variable (e.g., `auth.example.com`).
4. Copy the **Client ID** and **Client Secret** — these become the `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` GitHub secrets.

::: info Redirect URI
The redirect URI must be added exactly as shown, including the `/oauth2/idpresponse` path.  Cognito handles the redirect; the SPA callback URL is separate and configured in the User Pool client.
:::

## Adding the First Super User

After deploying the Cognito stack, add the first super user manually before using the UI.  See [Administration → Adding the First Super User](/cloud/administration#adding-the-first-super-user).

## GitHub Actions Workflows

All deployments and deletions are performed through GitHub Actions workflows, each targeting either the `dev` or `production` environment.  Most individual workflows also include an optional **Run integration tests after deploy** checkbox.

### Deploying

For initial setup, use **Deploy All** (`deploy-all`), which deploys all stacks in the correct dependency order and runs integration tests at the end.

For a complete list of individual deploy and delete workflows with descriptions and dependency order, see the [GitHub Actions Workflow Index](/cloud/github_actions/index).

### Integration Tests

The **Run Integration Tests** (`run-integration-tests`) workflow can be run independently to validate a deployed environment without making any changes.  It looks up the API URL from the `firefly-api-gateway` stack output and the UI URL from the `firefly-cloudfront-ui` stack output, then runs the test suite against the live environment.  UI tests are automatically skipped if the `firefly-cloudfront-ui` stack does not exist.

AppConfig tests are excluded by default because they trigger AWS AppConfig deployments and add significant time to the run.  To include them, check **Include AppConfig tests** when triggering the workflow manually.

## Running Tests Locally

Integration tests can also be run locally against any deployed environment.

### Setup

```bash
pip install -r tests/requirements.txt
```

### Environment Variables

| Variable | Required | Description |
|---|---|---|
| `FIREFLY_API_URL` | No | API base URL (defaults to the production URL if not set) |
| `FIREFLY_FIRMWARE_BUCKET` | For upload tests | Private S3 firmware bucket name |
| `FIREFLY_UI_URL` | For UI and CORS tests | Base URL of the firmware management UI (e.g. `https://ui.example.com`) |
| `FIREFLY_UI_BUCKET` | For UI S3 tests | Name of the private S3 bucket serving the UI static files |
| `FIREFLY_COGNITO_USER_POOL_ID` | For auth tests | Cognito User Pool ID |
| `FIREFLY_COGNITO_CLIENT_ID` | For auth tests | Cognito App Client ID |
| `FIREFLY_TEST_USER_EMAIL` | For auth tests | Email of an existing Cognito test user. |
| `FIREFLY_TEST_USER_PASSWORD` | For auth tests | Password for the Cognito test user. |

Authentication tests are automatically skipped when the Cognito environment variables are not set.  All other tests pass auth headers when the variables are present, allowing the full test suite to run against a deployed environment that requires authentication.

The `/users` endpoint tests require additional Cognito admin permissions: the test runner's IAM identity must have `cognito-idp:AdminAddUserToGroup`, `cognito-idp:AdminRemoveUserFromGroup`, and `cognito-idp:ListUsers` on the user pool.  The test suite temporarily adds the test user to the `super_users` group to exercise those endpoints, then removes them at teardown.

AWS credentials must be available via the standard boto3 credential chain.

### Running Tests

```bash
# All tests
pytest tests/integration/ -v

# Skip tests that require S3 upload
pytest tests/integration/ -v -k "not (firmware_item or fresh_firmware_item)"
```

Tests that upload firmware wait up to 60 seconds for the S3 event to propagate and the record to appear in the API before proceeding.
