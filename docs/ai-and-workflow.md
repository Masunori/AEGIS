# AI and workflow

AI work is isolated behind seven vendor-neutral provider protocols:

| Provider | Responsibility |
| --- | --- |
| `FilterProvider` | Classify evidence for relevance and safety. |
| `InterpreterProvider` | Extract proposed disruptions and textual entity mentions. |
| `EffectMappingProvider` | Map an accepted disruption to a client effect. |
| `RelationshipProvider` | Check relationships between signals. |
| `RiskProvider` | Propose risks from eligible signals and hypotheses. |
| `HypothesisProvider` | Turn a user's planning prompt into review-only proposals. |
| `PlannerProvider` | Propose interventions for a validated scenario. |

The protocols and shared output models live in `server/app/integrations/contracts.py`
and `model_provider.py`. Provider construction is centralized in
`server/app/integrations/providers.py`.

## Implementations

Deterministic stubs are implemented in `providers.py`. Gemini adapters are in
`gemini.py`, and Bedrock Converse adapters are in `bedrock.py`. The adapters implement
the same protocols; effect mapping and relationship inference currently use stubs in
both deployments. Local Compose defaults the other workflows to Gemini, while AWS
uses Bedrock.

Providers are untrusted boundaries. They cannot persist workflow records, call the
simulator, promote lifecycle state, or manufacture trusted client identifiers.

## Provider behavior

Gemini and Bedrock responses use structured schemas and are validated before entering
deterministic orchestration. Invalid fields, probabilities, temporal windows, and
provider-supplied metadata are rejected or repaired within the configured attempt
limits. Transport failures remain explicit errors; they never trigger a different
provider or persistence backend.

Bedrock uses Converse with Pydantic-generated schemas. Its schema-correction attempts,
SDK retries, timeouts, model IDs, Regions, and credential requirements are documented
in [Operations](operations.md#bedrock-credentials-and-tuning).

The stubs provide deterministic fixtures for tests and local development. They do not
read operator prompt overrides.

## Entity grounding and workflow boundaries

The interpreter returns textual mentions, not trusted client IDs. For accepted
evidence, AEGIS supplies versioned client capabilities and disruption contracts as
reference data, then resolves mentions through `ClientGateway`. Only client-normalized
disruptions can be reviewed or submitted.

Risk and hypothesis providers receive a bounded, client-grounded `entity_scope`.
Hypothesis proposals remain in the browser until the user confirms them. Deterministic
orchestration rechecks scope, entity types, immutable signal IDs, relationships, and
client state before persistence or simulation.

The complete signal and planning sequences are documented in
[Architecture](architecture.md#signal-and-experiment-path) and
[Simulation and planning](simulation-and-planning.md#risk-and-planning-workflow).

## Selecting providers

Provider values, environment-variable precedence, credentials, and reload commands
are maintained in [Operations](operations.md#switching-local-ai-providers). The
application does not watch or automatically load `.env.local` when run directly.

## Planner panel

Each planning cycle freezes `planner_mode` as `single` or `panel`. Panel mode invokes
1–5 role-labelled planners (default 3) through a bounded `PlannerProvider` coordinator.
Bedrock retains successful partial results and reports failed roles as warnings;
Gemini propagates a failed panel call. The panel does not vote or hold free-form agent
conversations. Every draft still requires client validation and simulation.

## Prompt configuration

Gemini and Bedrock filter, interpreter, and planner adapters read their system prompts
when constructed. Built-in safe defaults can be overridden through
`/api/settings/prompts`; overrides are stored in PostgreSQL `agent_prompts` or DynamoDB
`PROMPT#agent` items. Risk and hypothesis prompts are not currently editable through
this API. Prompt overrides guide untrusted output and do not replace schema, client,
version, persistence, or human-review checks.
