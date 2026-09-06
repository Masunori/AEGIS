# AEGIS Platform

AEGIS is an evidence and scenario-planning platform for organizations that simulate
their operations. It connects to a client's operational model, turns evidence into
reviewable signals, and coordinates baseline and intervention simulations so people
can compare outcomes and decide how to respond.

The client defines its industry-specific entities, operating rules, possible changes,
and simulation results through the [client integration contract](docs/client-integration-contract.md).
AEGIS manages evidence, review, experiments, and planning decisions around that system.
A simulator needs an adapter implementing this contract; an arbitrary simulation API
cannot be connected without integration work. The current hackathon ranking defaults
still use demo metrics; see [Simulation and planning](docs/simulation-and-planning.md#ranking-defaults).

The local stack provides:

- Next.js client: <http://localhost:3000>
- FastAPI server: <http://localhost:8000>
- Interactive API documentation: <http://localhost:8000/docs>
- PostgreSQL: `localhost:5432`

Deployment configuration selects the runtime settings:

| Setting | Local (`compose.dev.yml`) | AWS (`infrastructure/template.yaml`) |
| --- | --- | --- |
| Persistence | PostgreSQL | DynamoDB |
| AI providers | Gemini | Amazon Bedrock |
| Scheduled evidence collection | Enabled globally; opt in per source | Disabled |
| Client integration | Deployed demo client or local demo client | Deployed client integration API |
| AI credentials | `GEMINI_API_KEY` | Lambda execution role |

These settings follow the deployment configuration, not the browser hostname.
Manual evidence collection is available in both environments. Effect mapping and
relationship providers remain deterministic stubs in both environments.
For cloud-free AI development, set all five provider variables in `.env.local` to
`stub`. Simulations still require the separate demo client.
See [AI and workflow](docs/ai-and-workflow.md) and [operations](docs/operations.md)
for provider configuration and deployment details.

## Backend tests

Install host Python dependencies with `python -m pip install -r server/requirements-dev.txt`
(in a virtual environment), then run the Python suite including DynamoDB Local contracts.
The live Bedrock smoke test remains opt-in:

```bash
scripts/test-backend.sh
```

Install the tracked pre-push hook once to run that suite before every push:

```bash
scripts/install-git-hooks.sh
```

GitHub Actions also runs the same script on every push and pull request.

## Quick start

Prerequisites:

- Docker Engine or Docker Desktop with Docker Compose v2.
- Use the deployed demo client below, or run a demo client locally on port 8100.

From the repository root, create your local configuration:

```bash
cp .env.local.example .env.local
```

Set `GEMINI_API_KEY` in `.env.local` and adjust the demo client URL/token if needed.
The AWS deployment configuration in `infrastructure/samconfig.toml` already points
to a deployed demo client. You can use it from your local platform without starting
a local demo client by setting:

```dotenv
CLIENT_GATEWAY_URL=https://vd5unmdb37jhsj3u4rglhl6wui0ulxus.lambda-url.us-east-1.on.aws/integration/v1
CLIENT_GATEWAY_TOKEN=hackathon-demo-change-this
```

Use the demo token above for this deployed client. Open the home page after startup
to check the connection.
To use your own local demo client instead, keep the example file's
`http://host.docker.internal:8100/integration/v1` URL.

Existing `.env.local` values override the new defaults: replace previous `bedrock`
provider selections with `gemini`, and set `ENABLE_SOURCE_SCHEDULER=true`.
Each scraper must also have **Schedule automatic collection** enabled in Sources.
The development database uses the fixed local credentials `psa` / `psa`.

```bash
docker compose --env-file .env.local -f compose.dev.yml up -d --build --wait
docker compose --env-file .env.local -f compose.dev.yml exec server python -m app.seed
```

Then open <http://localhost:3000>. The active workspaces are `/evidence`, `/sources`, `/review`, `/planning`, and `/prompts`; the home page reports the authoritative client connection and version tuple.

For container-only tests (DynamoDB Local contracts skip), run:

```bash
docker compose --env-file .env.local -f compose.dev.yml exec server pytest
```

## Configuration variables

For local development, keep values in the repository-root `.env.local`, using
[`.env.local.example`](.env.local.example) as the template. Compose reads this file
when you pass `--env-file .env.local`.

| Variable | Required locally? | Value or default |
| --- | --- | --- |
| `CLIENT_GATEWAY_URL` | Yes | Deployed demo client URL above, or `http://host.docker.internal:8100/integration/v1` for a local demo client |
| `CLIENT_GATEWAY_TOKEN` | If the demo client requires authentication | Use `hackathon-demo-change-this` for the deployed demo; `.env.local.example` uses `demo-client-token` for a local client |
| `GEMINI_API_KEY` | When using Gemini (the local default) | Your Gemini API key |
| `GEMINI_MODEL` | No | `gemini-flash-lite-latest` |
| `PERSISTENCE_BACKEND` | No | `postgres` |
| `FILTER_PROVIDER` | No | `gemini` |
| `INTERPRETER_PROVIDER` | No | `gemini` |
| `HYPOTHESIS_PROVIDER` | No | `gemini` |
| `RISK_PROVIDER` | No | `gemini` |
| `PLANNER_PROVIDER` | No | `gemini` |
| `ENABLE_SOURCE_SCHEDULER` | No | `true`; each source must also enable scheduling |
| `GEMINI_MAX_ATTEMPTS` | No | `3` |
| `GEMINI_TIMEOUT_SECONDS` | No | `30` |

Set all five provider variables to `stub` to run without a Gemini key. These are
development Compose defaults; existing `.env.local` values override them. Restart
the server with the quick-start Compose command after changing values. Development
Compose supplies the database connection and frontend API URL directly, so you do
not need `POSTGRES_*`, `DATABASE_URL`, or `NEXT_PUBLIC_API_URL` in this file.

For AWS, configure parameters in the [SAM template](infrastructure/template.yaml)
instead of reusing `.env.local`:

| SAM parameter | Required? | Purpose |
| --- | --- | --- |
| `ClientOrigin` | Yes | Deployed frontend origin allowed by the API |
| `ClientGatewayUrl` | Yes | Reachable deployed client integration API URL |
| `ClientGatewayToken` | If the client requires authentication | Bearer token; defaults to empty |
| `BedrockModelId` | Yes | Bedrock model or inference profile ID |
| `TableName` | No | DynamoDB table name; defaults to `psa-production` |

The template sets `PERSISTENCE_BACKEND=dynamodb`, the table name, all five AI
providers to `bedrock`, `BEDROCK_REGION` to the deployment region, and
`ENABLE_SOURCE_SCHEDULER=false`. AWS credentials come from the Lambda execution
role. See [operations](docs/operations.md) for additional runtime settings.

## System overview

```text
evidence → relevance filtering → signal interpretation
→ authoritative entity grounding → disruption mapping and validation
→ human review → immutable experiment → client simulation

eligible signals + confirmed hypotheses → reconciled scenario draft
→ human composition → baseline simulation → plan simulations
→ deterministic ranking → human decision
```

The platform owns evidence, assessments, signals, reviews, scenarios, experiments, planning snapshots, operator prompt overrides, and non-authoritative result copies. The client integration owns operational data, entity identifiers, model state, capability contracts, simulation logic, and authoritative results. Experiment results use dedicated copy rows; planning results remain inside the planning-cycle snapshot.

## Documentation

Start with the [documentation index](docs/README.md), or jump directly to:

- [Getting started](docs/getting-started.md) — setup, migrations, tests, and code-reading path
- [Architecture](docs/architecture.md) — ownership, gateways, providers, validation, and persistence
- [Ingestion and review](docs/ingestion-and-review.md) — sources, evidence, filtering, grounding, and lifecycle
- [Simulation and planning](docs/simulation-and-planning.md) — scenarios, contingency plans, ranking, and experiments
- [AI and workflow](docs/ai-and-workflow.md) — provider boundaries, structured output, prompt configuration, and human decisions
- [API reference](docs/api-reference.md) — endpoints grouped by capability
- [Operations](docs/operations.md) — configuration, deployment, logs, rebuilds, and shutdown
- [DynamoDB data model](docs/dynamodb-data-model.md) — keys, indexes, access patterns, chunking, and concurrency
- [Client integration contract](docs/client-integration-contract.md) — authoritative client endpoints and wire formats

## Repository layout

```text
client/                 Next.js application
server/app/api/         FastAPI route handlers
server/app/services/    application workflows
server/app/repositories/ Storage-neutral contracts and backend composition
server/app/domain/      domain models and validation
server/app/integrations/ Client gateway and provider contracts
server/alembic/         database migrations
server/tests/           executable behavior documentation
docs/                   topic-focused project documentation
```
