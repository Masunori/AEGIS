# AEGIS documentation

AEGIS coordinates evidence review and simulation-based planning around a client-owned
operational system. The client supplies the industry model and simulator; these docs
explain how the platform connects to it and what the current implementation supports.

## New-contributor path

1. [Getting started](getting-started.md)
2. [Architecture](architecture.md)
3. [Ingestion and review](ingestion-and-review.md)
4. [Simulation and planning](simulation-and-planning.md)

## Find documentation by task

| I want to… | Read |
| --- | --- |
| Run the project or tests | [Getting started](getting-started.md) |
| Understand ownership and system boundaries | [Architecture](architecture.md) |
| Implement a connected client | [Client integration contract](client-integration-contract.md) |
| Work on uploads, sources, or evidence | [Ingestion and review](ingestion-and-review.md) |
| Understand scenarios, plans, and ranking | [Simulation and planning](simulation-and-planning.md) |
| Work on providers or orchestration | [AI and workflow](ai-and-workflow.md) |
| Find an endpoint | [API reference](api-reference.md) |
| Configure or deploy the stack | [Operations](operations.md) |
| Implement DynamoDB persistence | [DynamoDB data model](dynamodb-data-model.md) |

## Suggested code-reading path

```text
server/app/main.py
→ server/app/api/<capability>.py
→ server/app/services/<capability>_service.py
→ server/app/repositories/contracts.py
→ server/app/repositories/postgres/<capability>.py
→ server/app/domain/ or server/app/integrations/contracts.py
→ server/app/models.py
→ server/tests/test_<capability>.py
```

The repository contains two connected workflows: reviewed signals can become immutable
`experiment_packages`, while risk and mitigation evaluation is retained in
`planning_cycles`. Read [Architecture](architecture.md) before changing either workflow
or the platform/client ownership boundary.
