# Ingestion and review

Sources and uploads become canonical `evidence`. Evidence is assessed, interpreted,
grounded through the client, and presented for human review before it can become an
accepted signal.

```text
source or upload
→ canonical evidence
→ assessment
→ interpretation
→ client grounding and mapping
→ human review
→ accepted or rejected signal
```

## Evidence and sources

The platform accepts website and feed collection, APIs, structured records, uploads,
and manual entries. Content hashes deduplicate raw content while preserving
provenance. Website collection applies bounded same-site, robots, depth, page, path,
and keyword controls.

Automatic collection requires both the global scheduler and a per-source opt-in.
Manual collection remains available when scheduling is disabled. Defaults and
Lambda/Compose behavior are documented in [Operations](operations.md).

## Assessment and interpretation

The filter classifies evidence as `ACCEPT`, `REVIEW`, `REJECT`, or `QUARANTINE`; only
accepted evidence proceeds to interpretation. The interpreter proposes a signal type,
classification, time window, probability, severity, confidence, and textual entity
mentions. Signal types must match the connected client's current disruption catalog.

Provider implementations, structured-output validation, retries, and prompt behavior
are documented in [AI and workflow](ai-and-workflow.md). Provider output remains a
proposal: deterministic orchestration and the client gateway perform grounding,
normalization, and semantic validation.

## Review lifecycle

A signal attempt is immutable and retains its evidence provenance, mapping outcome, and
provider metadata. Human review is required before activation. Accepted signals retain
the client context and disruption-catalog versions used for grounding.

If every attempt for an evidence item is rejected, the Evidence workspace enables
**Reprocess** (or `POST /api/evidence/{id}/process`). This creates a new pending attempt
whose `retry_of_signal_id` points to the latest rejected attempt; it never reopens or
overwrites history. Pending or accepted attempts block another retry. Duplicate
occurrences cannot be processed through this endpoint.

Mapping is synchronous. Unexpected mapping errors are recorded as terminal
`MAPPING_FAILED` / `PROCESSING_FAILED` outcomes with a retryable message. If provider
capacity is exhausted, the current run's new evidence is deferred; use **Process /
retry** after capacity recovers without scraping the source again.

## Entity handling

Interpretation separates `entity_mentions` (all operational entities mentioned in the
evidence) from `target_entity_mentions` (entities directly affected). All mentions are
grounded and retained for context, while only targets compatible with the selected
disruption contract become mapping targets. Unresolved related entities therefore remain
visible without blocking a valid target mapping.

## Retention, deletion, and duplicates

Permanent deletion is blocked while evidence is referenced by signal history, duplicate
provenance, or legal hold. The Evidence workspace previews blockers and offers archive
or raw-content removal as retention-safe alternatives.

Canonical evidence cannot be deleted while duplicate records point to it. Enable
**Include duplicates**, remove eligible dependent duplicates, then retry the canonical
item. **Delete unprotected duplicates** previews eligible and protected records before
confirmation. PostgreSQL performs cleanup transactionally; DynamoDB deletes sequentially
and may stop partway through an error. Protected records and provenance are preserved.

Collection retains lightweight duplicate-occurrence records by default and never
replaces canonical content during cleanup.
