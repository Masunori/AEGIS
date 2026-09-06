# Simulation and planning

Scenarios and plans are platform-owned, simulation-agnostic envelopes. The connected
client defines entities, disruptions, interventions, lifecycle state, and authoritative
simulation results. A disruption changes simulated conditions; an intervention is a
proposed response.

```text
eligible signals + confirmed hypotheses
→ risk proposal
→ client validation and state reconciliation
→ frozen scenario
→ baseline simulation
→ planner proposal(s)
→ client intervention validation
→ intervention simulations
→ deterministic ranking
→ human approval or rejection
```

The broader signal and experiment path is described in
[Architecture](architecture.md#signal-and-experiment-path). This document covers the
planning path, its API, ranking policy, and browser workflow.

## Planning lifecycle

The platform loads accepted, mapped, current signals whose validity overlaps the
planning horizon. Risk generation may select immutable signal-version IDs and propose
hypothetical disruptions. Users can also confirm browser-local hypotheses. The cycle
freezes the complete scenario, client context/state versions, objectives, constraints,
and provenance before submitting a baseline.

The client reconciles every disruption against frozen state. `ALREADY_REFLECTED` items
remain in the audit scenario but are excluded from simulator inputs;
`APPLY_IN_SIMULATION` items are applied; `UNKNOWN` stops submission. This avoids
double-counting observed conditions.

Baseline completion is required before planner invocation. A single planner or a
1–5-agent panel proposes typed interventions. Each intervention is validated by the
client and simulated with the same frozen scenario and versions. Completed results are
ranked deterministically, and only a recommendation can cross the human decision
boundary. Approval records a decision; it does not execute an operational change.

Planning cycles store exact client run/version links and unchanged result dictionaries
as non-authoritative snapshots. They do not create experiment-package or
simulation-result-copy rows.

## Risk, planning, and provider boundaries

Provider roles, implementations, validation, and selection are documented in
[AI and workflow](ai-and-workflow.md). Risk and planner providers cannot write records,
submit simulations, rank plans, or make decisions. Their IDs, rationale, assumptions,
and model metadata are provenance until deterministic checks and client validation
accept them.

The service validates the client's restricted schemas, rejects unknown entity IDs and
types, enforces signal relationships (`REQUIRES`, `MUTUALLY_EXCLUSIVE`, and
`SUPERSEDES`), and uses canonical frozen inputs for idempotency. Risk generation is
bounded to 20 proposals and 20 disruptions per proposal. Empty provider responses are
valid and produce no scenario or plan.

## Ranking defaults

The current `lexicographic-v1` policy minimizes `late_shipments`,
`average_delay_hours`, then `total_cost`; missing metrics sort as infinity. Infeasible
plans sort after feasible plans, with proposal ID as the stable tie-breaker. Failed or
incomplete runs remain in history but are excluded from ranking. Provider rationale and
deterministic ranking explanations are stored separately.

These metric names are demo defaults, not a required client vocabulary. A client with
different performance measures needs an explicit ranking adaptation. Lifecycle values
are `PROPOSED`, `VALIDATED`, `SUBMITTED`, `RUNNING`, `EVALUATED`, `FAILED`,
`RECOMMENDED`, `APPROVED`, and `REJECTED`.

## API and failure behavior

Cycle-level endpoints under `/api/planning/cycles` create, read, advance, approve, and
reject cycles. Granular baseline, proposal, intervention, and ranking endpoints remain
available for compatibility and recovery. Operations are refreshable and do not hold
requests open while a client run is queued.

Provider or client failures stop the current operation without fabricated proposals or
results. Gateway errors are sanitized; stale context/state and failed simulations are
retained as safe codes and messages. Reusing canonical inputs produces the same
idempotency key. Planning-provider quota failures return `429`; other provider API
failures return sanitized `502` responses.

## Browser workflow

The Planning workspace displays client readiness and frozen versions. The server loads
eligible signals; the browser confirms or removes hypotheses and compiled risk signals
before the cycle is created. **Simulate** freezes that composition and submits the
baseline. The UI polls queued runs through the cycle advance operation, displays
baseline and intervention results, and exposes approval only after deterministic
ranking recommends a plan.

The browser stores unconfirmed hypotheses in `localStorage` under
`AEGIS.planning.hypotheses.v1`; they are never signal records. It renders numerical
values only from client results, treats planner rationale as qualitative, and shows
unknown metrics as unavailable.

The UI uses strict lifecycle and result types in `client/types/planning.ts`. Presentation
rules gate simulation, ranking, and decisions independently. A failed cycle remains
readable and directs the user to correct the connected operational system before
starting a new frozen cycle.

For client-side verification, run `npm test`, `npm run lint`, and `npx tsc --noEmit`
from `client`.
