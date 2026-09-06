# Operations

## Integration configuration

Local Compose defaults to PostgreSQL, Gemini, and enabled scheduled collection.
Copy `.env.local.example` to `.env.local` at the repository root and supply your
Gemini key. Explicit environment values override Compose defaults. Each source
must independently opt in to scheduling. Set all five AI providers to `stub` for
cloud-free AI development.

AWS production uses [the SAM template](../infrastructure/template.yaml), which
selects DynamoDB and Bedrock and disables scheduled collection. Lambda also disables
ASGI lifespan, so it does not start the process-local scheduler. Manual collection
remains available. Supply `ClientOrigin`, `ClientGatewayUrl`, `ClientGatewayToken`
(if required), and `BedrockModelId` as deployment parameters; the Lambda role supplies
AWS credentials. Do not reuse the local environment file for AWS deployment.

### Switching local AI providers

Set each workflow independently in the repository-root `.env.local`:

```dotenv
FILTER_PROVIDER=gemini
INTERPRETER_PROVIDER=gemini
HYPOTHESIS_PROVIDER=gemini
RISK_PROVIDER=gemini
PLANNER_PROVIDER=gemini
```

Each accepts `gemini`, `bedrock`, or `stub`; mixed selections are supported.
Unsupported provider values fail when the provider is constructed. There is no
automatic fallback to another provider. Effect mapping and relationship inference
support only `stub`.

| Selection | Model settings | Credentials |
| --- | --- | --- |
| `gemini` | `GEMINI_MODEL` (default `gemini-flash-lite-latest`) | `GEMINI_API_KEY` |
| `bedrock` | `BEDROCK_MODEL_ID` and `BEDROCK_REGION` (falls back to `AWS_REGION`) | AWS credentials available to the server process |
| `stub` | None | None |

Model settings are shared by all workflows using that vendor; there are no separate
per-workflow model-ID variables. Panel mode follows `PLANNER_PROVIDER` and supports
1–5 agents (default 3), with numbered planner prompts for model providers.

Compose uses `.env.local` for interpolation only when invoked with
`--env-file .env.local`. Only settings declared in the Compose service environment
are passed into its container; adding an arbitrary variable to the file does not
forward it automatically. Exported shell variables take precedence over the env file.
Running Python or Uvicorn directly does not automatically load `.env.local`.

After editing provider or model settings, apply them with:

```bash
docker compose --env-file .env.local -f compose.dev.yml up -d server
docker compose --env-file .env.local -f compose.dev.yml logs -f server
```

Compose recreates the server when its environment changes. No image rebuild is
needed for environment-only changes. A plain `docker compose restart` does not
apply new environment values, and editing the file is not a live configuration change.

### Bedrock credentials and tuning

Selecting Bedrock locally also requires AWS credentials inside the server process.
Development Compose does not pass `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, or
`AWS_SESSION_TOKEN`, and does not mount the host's AWS profile directory. Putting
credentials in `.env.local` or configuring a host AWS profile alone therefore does
not authenticate the container. Container credential configuration is an additional
setup step; the current development Compose file does not provide it. AWS Lambda
uses its execution role.

These optional settings configure Bedrock calls once credentials are available:

```dotenv
BEDROCK_REGION=ap-southeast-1
BEDROCK_MODEL_ID=global.amazon.nova-2-lite-v1:0
BEDROCK_MAX_ATTEMPTS=3
BEDROCK_SDK_MAX_ATTEMPTS=2
BEDROCK_TIMEOUT_SECONDS=60
BEDROCK_MAX_TOKENS=4096
```

Changing an AI provider does not change persistence, client gateway configuration,
or `ENABLE_SOURCE_SCHEDULER`; those settings are independent.

`BEDROCK_MAX_ATTEMPTS=3` means one initial structured-output request plus at most two
schema-correction requests. `BEDROCK_SDK_MAX_ATTEMPTS=2` independently bounds each
request's transport attempts, making the worst case six HTTP attempts. Reducing either
limit lowers latency and cost. API
authentication, quota, rate-limit, and other HTTP errors fail immediately rather than
consuming schema-repair attempts.

To run only the Bedrock provider contract tests without contacting AWS:

```bash
docker compose --env-file .env.local -f compose.dev.yml exec server pytest tests/test_bedrock_providers.py
```

`HTTPClientGateway` adds correlation and idempotency headers, bounded timeouts and retries, strict response validation, and stable errors.
There is no local operational-data fallback. Missing `CLIENT_GATEWAY_URL` fails clearly
when the gateway dependency is constructed. Use the deployed demo client documented in the [root README](../README.md#quick-start),
or start your own client and point the URL at its versioned integration API. `FakeClientGateway` under `server/tests/fakes/`
is test-only.

### Provider prompt overrides

Operators can edit the Gemini and Bedrock filter, interpreter, and planner system prompts in the
Prompts UI or through `/api/settings/prompts`. Overrides are stored in the platform
database and take effect when a provider is next constructed; resetting an override
restores the built-in default. Stub providers ignore these prompts. Prompt changes do
not bypass structured-output validation, client validation, or human review.

When the platform server runs inside Docker and the demo client runs on the host, use
`http://host.docker.internal:8100/integration/v1` instead of `http://localhost:8100/integration/v1`.

### Client validation failures

Disruption and intervention validation requests include the catalog version advertised
by the connected client. A client-side 4xx response is surfaced by planning as a
sanitized 502 integration error instead of an internal traceback. Check that the client
and platform images were rebuilt against compatible integration contracts.

The deterministic risk stub generates an ordered example window (`effective_from`
before `effective_until`). For nullable JSON Schema unions such as
`{"type": ["string", "null"]}`, it deterministically chooses the first concrete
non-null type so required fixture fields remain useful. Real providers remain
untrusted: their payloads are checked against the advertised JSON Schema and then
against the client's semantic validation rules before a scenario draft is stored.

## Other environment variables

- `DATABASE_URL` — server database connection; Compose uses hostname `database`, while host tools use `localhost:5432`.
- `ENABLE_SOURCE_SCHEDULER` — global opt-in for local periodic source collection;
  defaults to `false` in the application and AWS, and `true` in development Compose. Only enabled website sources with their independent
  `schedule_enabled` option set are collected. Manual **Collect now** requests do
  not require this switch.
- `NEXT_PUBLIC_API_URL` — browser-accessible API URL compiled into the client.
- `CLIENT_ORIGIN` — exact browser origin allowed by FastAPI.
- `BEDROCK_REGION` — AWS Region used by the Bedrock Runtime client.
- `BEDROCK_MODEL_ID` — foundation-model ID or inference-profile ID used by all Bedrock providers.
- `BEDROCK_MAX_ATTEMPTS` — total structured-output and semantic-correction attempts; minimum `1`.
- `BEDROCK_SDK_MAX_ATTEMPTS` — total SDK transport attempts per structured-output request; minimum `1`.
- `BEDROCK_TIMEOUT_SECONDS` — SDK connect and read timeout.
- `BEDROCK_MAX_TOKENS` — maximum tokens requested from Converse.
- `HYPOTHESIS_PROVIDER` — `bedrock`, `gemini`, or `stub`. Development Compose
  explicitly defaults to Gemini and SAM sets Bedrock. Only when the variable is
  absent from the server process does the factory select Bedrock if a model ID is
  configured, or the deterministic stub otherwise.

## Development operations

### DynamoDB Local foundation tests

The local emulator verifies the table schema and repository adapters. Start it on host
port `8001`:

```bash
docker compose --env-file .env.local -f compose.dev.yml --profile dynamodb up -d dynamodb-local
cd server
DYNAMODB_LOCAL_ENDPOINT=http://127.0.0.1:8001 \
  ../.venv/bin/pytest -q tests/test_dynamodb_foundation.py \
  tests/test_dynamodb_codec.py tests/test_dynamodb_local.py
cd ..
```

The fixture uses Region `ap-southeast-1` by default; override it with
`DYNAMODB_LOCAL_REGION`. Each test table is named `test-<uuid>`, created on demand,
and deleted after the test. A failed test may leave an isolated table; restart the
in-memory container to clear all local data:

```bash
docker compose --env-file .env.local -f compose.dev.yml --profile dynamodb restart dynamodb-local
```

If the integration test is skipped, set `DYNAMODB_LOCAL_ENDPOINT`. Connection failures
usually mean the profile service is not running or port `8001` is occupied. Remote
endpoints are deliberately rejected so tests cannot create or delete tables in an AWS
account.

The test fixtures create emulator tables and supply inert local credentials
explicitly. Running the application itself on DynamoDB Local additionally requires
a created table, `PERSISTENCE_BACKEND=dynamodb`, `DYNAMODB_TABLE_NAME`,
`DYNAMODB_ENDPOINT_URL`, and credentials available inside the server process.
Development Compose supplies the backend/table/endpoint settings but does not pass
AWS access-key variables or mount AWS profiles. Merely switching the backend and
starting the profile is therefore not a complete emulator application setup; the
backend test script is the provided emulator verification path.

Production requires `PERSISTENCE_BACKEND=dynamodb`, `DYNAMODB_TABLE_NAME`, and
`AWS_REGION`; omit `DYNAMODB_ENDPOINT_URL` in AWS. Credentials come from boto3's
workload-role/profile chain. The [SAM template](../infrastructure/template.yaml)
creates the encrypted on-demand table, TTL, point-in-time recovery, and a policy with
point, query, batch, and transaction actions but no table-wide read.

SDK requests use standard retry mode with four attempts. Conditional, throttling,
validation, authentication, timeout, transport, and service failures map to sanitized
application persistence errors. They never cause backend switching.

For rollback, stop writers, restore a point-in-time backup to a new table, update
`DYNAMODB_TABLE_NAME`, and redeploy. PostgreSQL fallback is not automatic and requires
a separately planned data migration. Alarm on throttles, system errors, latency, and
unexpected request growth; apply on-demand throughput limits where Region support
allows it.

```bash
docker compose --env-file .env.local -f compose.dev.yml ps
docker compose --env-file .env.local -f compose.dev.yml logs -f
docker compose --env-file .env.local -f compose.dev.yml up -d --build server
docker compose --env-file .env.local -f compose.dev.yml down
```

To also delete the development database and build volumes:

```bash
docker compose --env-file .env.local -f compose.dev.yml down --volumes
```

## Lambda runtime

AWS Lambda invokes `app.main.handler`, a Mangum adapter around the same FastAPI app.
Its ASGI lifespan is disabled, so Lambda web invocations never start APScheduler or
run Alembic. Apply migrations or initialize the selected persistence backend as a
separate deployment operation. Uvicorn retains lifespan behavior for local use.

The Vercel client emits both `robots` no-index metadata and `/robots.txt` with
`Disallow: /`. These are crawler requests, not authentication or access control.

## Standalone PostgreSQL containers

`compose.prod.yml` is the separate PostgreSQL-based container deployment, not the
AWS Lambda production configuration described above.

Set container deployment values in `.env.local`, including `POSTGRES_PASSWORD`,
`CLIENT_ORIGIN`, and `CLIENT_GATEWAY_URL`, then run:

```bash
docker compose --env-file .env.local -f compose.prod.yml up -d --build
docker compose --env-file .env.local -f compose.prod.yml ps
docker compose --env-file .env.local -f compose.prod.yml logs -f
```

Stop without deleting PostgreSQL data:

```bash
docker compose --env-file .env.local -f compose.prod.yml down
```

This container deployment stores PostgreSQL data in the `postgres_prod_data` volume. Do not add `--volumes` unless deletion is intentional.
