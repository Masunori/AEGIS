# Getting started

Use the deployed demo client URL and token in the [root README](../README.md#quick-start),
or start your own standalone demo client on port 8100. For your own client, configure the host platform with
`CLIENT_GATEWAY_URL=http://localhost:8100/integration/v1`; from the server container use
`http://host.docker.internal:8100/integration/v1` or a shared-network service name.

Local Compose uses PostgreSQL, Gemini, and globally enabled scheduled collection.
From the repository root, copy `.env.local.example` to `.env.local` and set
`GEMINI_API_KEY`. Enable scheduling separately for each desired source in the UI.
For provider choices, credentials, and deployment settings, see
[Operations](operations.md).

```bash
docker compose --env-file .env.local -f compose.dev.yml up -d --build --wait
docker compose --env-file .env.local -f compose.dev.yml exec server python -m app.seed
```

Open the client-connection page at <http://localhost:3000>, evidence at `/evidence`, and
sources at `/sources`. Review, planning, and prompt workspaces are available at
`/review`, `/planning`, and `/prompts`. Seeding is repeatable and creates only a platform-owned manual
evidence source.

The Sources UI manages scraper creation, editing, enablement, immediate collection, and
deletion when no retained evidence references the source. The Evidence UI supports
manual creation, upload, editing, archive/restore, raw-content redaction, and permanent
deletion when audit protections allow it. Primary navigation is shared in the top bar.

Backend verification (from the repository root, with Docker and host Python):

```bash
python3 -m venv .venv
.venv/bin/python -m pip install -r server/requirements-dev.txt
scripts/test-backend.sh
```

The script starts DynamoDB Local and runs the shared repository contracts. The live
Bedrock smoke test remains opt-in. Development Compose runs Alembic migrations
automatically when the server starts.

Frontend verification:

```bash
cd client
npm test
npm run lint
npx tsc --noEmit
npm run build
```
