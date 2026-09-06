# Deploying AEGIS on AWS

This guide deploys the backend using the current
[SAM template](../infrastructure/template.yaml): FastAPI on AWS Lambda, DynamoDB
for persistence, and Amazon Bedrock for AI providers. Run the commands from the
repository root unless a step explicitly changes directories.

## What gets deployed

| Component | Current configuration |
| --- | --- |
| API | Python 3.13, x86_64 Lambda, `app.main.handler` via Mangum |
| Resources per invocation | 1,024 MB memory; 60-second timeout |
| API endpoint | Lambda Function URL with `AuthType: NONE` |
| Persistence | One DynamoDB table with two indexes, on-demand billing, encryption, TTL, and point-in-time recovery |
| AI | Bedrock for filtering, interpretation, hypotheses, risk, and planning |
| Deterministic providers | Effect mapping and relationships use stubs |
| Credentials | Lambda execution role grants DynamoDB access and `bedrock:InvokeModel` |

The template does not deploy the Next.js frontend or the authoritative simulation
client. Host the frontend separately (the existing application supports Vercel) and
provide a reachable client API implementing the
[client integration contract](client-integration-contract.md).

The Function URL is public and unauthenticated. `ClientOrigin` configures browser
CORS; it does not authenticate callers. `ClientGatewayToken` authenticates outbound
requests to the simulation client, not incoming requests to AEGIS.

## 1. Prepare tools and deployment values

Install AWS CLI v2, AWS SAM CLI, and Docker with a running Docker daemon. Docker
builds dependencies for Lambda's Linux runtime. Configure an AWS profile with
permission to deploy CloudFormation stacks and manage the template's Lambda, IAM,
DynamoDB resources and SAM's S3 deployment artifacts.

```bash
aws --version
sam --version
docker info

export AWS_PROFILE=your-deployment-profile
export AWS_REGION=ap-southeast-1
export AEGIS_STACK_NAME=aegis-production
aws sts get-caller-identity
```

Replace the profile and region for your account. If the profile uses IAM Identity
Center, sign in with `aws sso login --profile "$AWS_PROFILE"` first.

Prepare these CloudFormation parameter values:

| Parameter | What to enter |
| --- | --- |
| `TableName` | A new table name, such as `aegis-production`; template default is `production` |
| `ClientOrigin` | Exact frontend origin, such as `https://your-app.vercel.app`, without a trailing slash or path |
| `ClientGatewayUrl` | Public client API base URL ending in `/integration/v1`, without a trailing slash |
| `ClientGatewayToken` | Client bearer token, or empty if the client does not require one |
| `BedrockModelId` | Model or inference-profile ID usable in the selected region and account |

For an existing deployment, keep its stack and table names. The template creates a
table; entering the name of an unrelated existing table does not attach that table.
Local PostgreSQL data is not copied to DynamoDB by deployment.

The [root README](../README.md#quick-start) provides the existing demo client URL
and demo token. A local `localhost` or `host.docker.internal` address cannot serve
as the client endpoint for this Lambda configuration.

Confirm model access in Amazon Bedrock for the deployment account and region.
Some providers require additional setup; consult AWS's
[model access documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html).
The repository's documented example is `global.amazon.nova-2-lite-v1:0`; verify its
availability for your deployment before using it. The adapter uses forced tool
output for Nova 2 model IDs and JSON-schema structured output for other models,
so a model must support the corresponding Converse features. A model ID alone
does not establish compatibility or access. Inference profiles can also require
permission to invoke models in their destination regions.

AWS deployment does not read `.env.local`. The SAM parameters set runtime values,
and the execution role supplies AWS credentials; no Gemini key is needed.

## 2. Validate and build

```bash
cd infrastructure
sam validate --lint --template-file template.yaml --region "$AWS_REGION"
sam build --use-container --template-file template.yaml
```

Stay in `infrastructure/` for the remaining SAM commands. This keeps generated
artifacts under `infrastructure/.aws-sam/`, which the repository ignores. Edit
`infrastructure/template.yaml` and `server/` source files, then rebuild; generated
`.aws-sam/build/` files are deployment artifacts.

The container build packages `server/requirements.txt` for Lambda, including native
dependencies. See the AWS [sam build reference](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-cli-command-reference-sam-build.html).

## 3. Deploy the stack

```bash
sam deploy --guided \
  --template-file .aws-sam/build/template.yaml \
  --stack-name "$AEGIS_STACK_NAME" \
  --region "$AWS_REGION" \
  --capabilities CAPABILITY_IAM
```

Enter the five parameter values prepared above. Allow SAM to create the execution
role, keep rollback enabled, and review the changeset before applying it. If SAM
asks about deploying the function without authorization, that reflects the public
Function URL configured by this template.

Guided deployment can save settings in `infrastructure/samconfig.toml` for later
deployments. This file is ignored by Git. Treat saved parameter overrides as
sensitive if they contain `ClientGatewayToken`; `NoEcho` on the CloudFormation
parameter does not encrypt a local configuration file. Avoid putting real tokens
in committed examples or shell command history.

SAM uploads the build artifacts and deploys through CloudFormation. See the AWS
[sam deploy reference](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-cli-command-reference-sam-deploy.html)
for guided deployment and configuration options.

Read the resulting endpoint and table name:

```bash
aws cloudformation describe-stacks \
  --stack-name "$AEGIS_STACK_NAME" \
  --region "$AWS_REGION" \
  --query 'Stacks[0].Outputs' --output table

export AEGIS_API_URL="$(aws cloudformation describe-stacks \
  --stack-name "$AEGIS_STACK_NAME" --region "$AWS_REGION" \
  --query 'Stacks[0].Outputs[?OutputKey==`ApiFunctionUrl`].OutputValue | [0]' \
  --output text)"
export AEGIS_API_URL="${AEGIS_API_URL%/}"
```

DynamoDB is initialized by CloudFormation. Do not run Alembic migrations or
`python -m app.seed` for this deployment: those commands target PostgreSQL.
Create sources and evidence through the application after deployment.

## 4. Connect the frontend

In your separately hosted Next.js project's environment settings, configure:

```dotenv
BACKEND_URL=https://your-function-id.lambda-url.your-region.on.aws
NEXT_PUBLIC_API_URL=https://your-function-id.lambda-url.your-region.on.aws
```

Use the `ApiFunctionUrl` output, with its trailing slash removed. `BACKEND_URL`
serves server-side API requests; `NEXT_PUBLIC_API_URL` supplies browser-side
configuration. Rebuild and redeploy the frontend after changing these values.
The backend's `ClientOrigin` must match the frontend origin actually used in the
browser. If that origin changes, rerun guided deployment with the updated value.

## 5. Verify the deployment

```bash
curl --fail-with-body "$AEGIS_API_URL/health"
curl --fail-with-body "$AEGIS_API_URL/health/storage"
curl --fail-with-body "$AEGIS_API_URL/health/client"
```

Expect `status: ok` from each endpoint and `backend: dynamodb` from storage.
Inspect the client response body: it can return HTTP 200 with `status: degraded`
and an `error_code`. A healthy process alone does not prove client connectivity,
Bedrock access, or a complete workflow.

Open the frontend and verify the connection on the home page. Create a source and
evidence item, process it, and inspect the resulting reviewable signal. Then use
the planning workspace to generate a scenario, review it, run its baseline, and
generate and simulate a plan. These steps exercise persistence, Bedrock, and the
separate client simulator. AI requests and simulations may incur service charges.

For automated checks before deployment, follow
[Getting started](getting-started.md). The backend suite uses deterministic
fixtures and DynamoDB Local; it is not a substitute for the deployed smoke checks.

## Updates and troubleshooting

After changing source or infrastructure, rebuild from `infrastructure/` and deploy
using saved guided settings:

```bash
sam build --use-container --template-file template.yaml
sam deploy --template-file .aws-sam/build/template.yaml --confirm-changeset
```

If settings were not saved, repeat the guided command. Preserve the existing stack,
region, and table parameters when updating.

Read Lambda logs and stack events:

```bash
sam logs --name ApiFunction --stack-name "$AEGIS_STACK_NAME" \
  --region "$AWS_REGION" --tail

aws cloudformation describe-stack-events \
  --stack-name "$AEGIS_STACK_NAME" --region "$AWS_REGION" --max-items 20
```

`sam logs --tail` runs continuously; stop it with Ctrl+C before the next command.

| Symptom | Check |
| --- | --- |
| Build fails on native dependencies | Docker is running and `sam build --use-container` is used |
| Table already exists | Use the original stack for updates, or a distinct table name for a new stack |
| Bedrock access or validation error | Model ID, region, provider access, Converse feature support, and execution-role/account policies |
| Request times out | Lambda's total 60-second budget includes client calls and AI retries; the Bedrock SDK timeout is also 60 seconds, so retries can exceed the invocation budget |
| Client health is degraded | Public gateway URL, token, availability, and integration-contract compatibility |
| Browser CORS failure | Exact `ClientOrigin`, including scheme and port, followed by backend redeployment |
| Frontend requests localhost | Hosted frontend environment values and a fresh frontend deployment |
| Storage health fails | Stack table output, Lambda role permissions, and DynamoDB throttling or service errors |

Automatic collection is disabled in this stack. Lambda disables ASGI lifespan and
does not start the process-local scheduler; use **Collect now** for manual collection.
The current template does not provision an external scheduling service.

The table has deletion protection and retain policies. Removing the stack retains
its data table; recreating a stack with that same table name will require an explicit
resource recovery/import plan or a different name. Table replacement and data
migration should be planned separately. For persistence limits and recovery details,
see [Operations](operations.md) and the [DynamoDB data model](dynamodb-data-model.md).
