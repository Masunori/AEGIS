# DynamoDB data model

This describes the current hackathon DynamoDB backend and its implementation limits. PostgreSQL
remains the behavioral reference; selecting a backend never falls back or dual-writes.

## Table and indexes

Use one on-demand table with string keys `PK` and `SK`, deletion TTL attribute `ttl`,
and an integer `version` on selected aggregates. Version checks are not universal. Resource name is parameterized.

- Primary key: aggregate-local reads and bounded child collections.
- `GSI1PK`/`GSI1SK`: type lists in stable application order.
- `GSI2PK`/`GSI2SK`: sparse operational lookups: due sources, content hashes, and
  experiment idempotency keys.

No repository method performs `Scan`. Public repository list methods accept a bounded `limit` and continuation token.
Some internal queries, including due sources, reverse references, relationships, and
risk candidates, read only one DynamoDB response page and do not follow
`LastEvaluatedKey`; they can omit records when those collections grow.

| Entity | PK | SK | Index use |
| --- | --- | --- | --- |
| Source | `SOURCE#id` | `META` | GSI1 `SOURCE` / `name#id`; GSI2 `DUE` / `next_run_at#id` only while enabled and scheduled |
| Evidence metadata | `EVIDENCE#id` | `META` | GSI1 `EVIDENCE` / `collected_at#id`; GSI2 `HASH#sha256` / `collected_at#id` |
| Evidence chunk | `EVIDENCE#id` | `CONTENT#000000` (zero-based) | none |
| Assessment | `EVIDENCE#id` | `ASSESSMENT#time#id` | none |
| Signal | `SIGNAL#id` | `META` | GSI1 `SIGNAL` / created timestamp plus ID (queried descending) |
| Signal version | `SIGNAL#id` | `VERSION#000001` | GSI2 `SIGNAL_VERSION#id` / `META` |
| Signal entities and mapping outcome | `SIGNAL#id` | `VERSION#000001` payload | Stored inside the version payload; updated during processing |
| Experiment | `EXPERIMENT#id` | `META` | GSI1 list; GSI2 `EXPERIMENT_KEY#hash` / `META` |
| Result copy | `EXPERIMENT#id` | `RESULT#run_id` | GSI2 `RESULT#run_id` / `META` |
| Planning cycle | `PLANNING#id` | `META` | GSI1 `PLANNING` / cycle ID |
| Prompt override | `PROMPT#agent` | `META` | none; known agent keys are read individually |

Scenario and plan definition items follow the same `TYPE#id`/`META` pattern and their
existing stable-ID ordering on GSI1.

## Repository operation mapping

- Sources: point get/write/delete; ordered list through GSI1; due-source query through
  the sparse `DUE` partition. Enabling scheduling adds the GSI2 keys; disabling it
  removes them. Run completion conditionally advances `next_run_at`.
- Evidence: transact source-existence check, conditional hash/id write, and chunks.
  Lists use GSI1; deletion impact queries the evidence aggregate and bounded reverse
  references. Duplicate cleanup deletes eligible records sequentially and returns protected skips
  when it completes; it is not one transaction.
- Signals: a strong lookup item locates the version payload. Candidate creation
  transactionally writes metadata, the first version, its lookup, and evidence
  references. Processing overwrites the version payload without a version condition;
  human review uses a conditional aggregate update. Version rows are not immutable
  at the storage layer.
- Experiments: a strongly read `EXPERIMENT_KEY#key` lock and conditional transaction
  deduplicate creation. Submission checks the existing run ID in application code,
  then writes without an expected-version condition. Result saving transactionally
  writes the result and completion status, checking package existence only.
- Planning cycles: point reads and conditional snapshot replacement. Large snapshots
  are split into deterministic `SECTION#name#chunk` children before the item limit.
- Prompts: point gets/writes/deletes for the fixed allow-listed agent names.

Foreign-key behavior is reproduced with explicit checks. Source deletion checks the
source-evidence reference partition; evidence deletion checks signal and duplicate
references; experiment/result records are retained. Selected creation and update paths use `TransactWriteItems`. Other paths use
separate reads and writes or batch writes, so cross-item checks do not universally
protect against concurrent changes. The transaction helper limits requests to 100 items.

## Content limits and serialization

The DynamoDB content codec accepts at most 256 KiB of UTF-8 evidence text and
splits it into binary chunks of at most 64 KiB. Evidence metadata retains the
content hash and a content reference; the adapter does not store separate byte-count
or chunk-count fields for evidence. Reads concatenate the ordered content chunks.
The text limit does not guarantee that structured content or other aggregate payloads
fit DynamoDB's item limit. Original uploaded documents are not retained in DynamoDB
or S3.

Datetimes are UTC ISO-8601 strings with `Z`; enum values are strings; floats are
converted through decimal strings to `Decimal`; sets are stored as ordered lists when
order is observable. Empty values are normalized consistently. Continuation tokens
are URL-safe base64 JSON and are validated before use.

## Executable schema and local lifecycle

`server/app/repositories/dynamodb/schema.py` is the canonical create-table definition.
It creates string `PK`/`SK` keys, `GSI1` and `GSI2` with string partition/sort keys and
`ALL` projections, and `PAY_PER_REQUEST` billing. After creation becomes active, the
lifecycle helper enables DynamoDB TTL on `ttl`. Tests create a UUID-suffixed table and
always delete it in teardown, so parallel workers never share state.

The opt-in integration test accepts only loopback endpoints or the Compose service name
`dynamodb-local`; it rejects AWS and arbitrary remote hosts before constructing an SDK
resource. Its credentials are inert literals scoped to the local emulator and it never
loads a developer AWS profile.

## Concurrency and failure semantics

Source updates, signal reviews, and planning snapshot replacements use conditional
version checks. Evidence edits/archive/redaction, prompt and definition writes, and
experiment submission do not all have equivalent protection. Concurrent writes on
those paths can overwrite one another. Evidence content changes and duplicate cleanup
span multiple writes; failures can leave partial changes.

Scheduled collection acquires a conditional `LEASE` item with `lease_until` and
`lease_owner` before fetching. The scheduler requests a 15-minute lease and releases
it afterward; there is no lease-renewal loop. AWS disables this scheduler.

The persistence error helper maps SDK errors to domain errors where adapters invoke
it; there is no automatic PostgreSQL fallback. This backend does not provide a
uniform transactional or optimistic-concurrency guarantee across all repositories.

The SAM template enables table retention, deletion protection, encryption,
point-in-time recovery, and table-level maximum throughput. It does not define
CloudWatch alarms or separate GSI throughput caps.

## Implemented item layouts

Adapters use explicit reverse and lookup items in addition to the aggregate rows:

| Purpose | PK | SK |
| --- | --- | --- |
| Source deletion protection | `SOURCE_REF#source-id` | `EVIDENCE#evidence-id` |
| Canonical hash lock | `HASH#sha256` | `META` |
| Duplicate protection | `DUP_REF#canonical-id` | `EVIDENCE#duplicate-id` |
| Evidence signal history | `EVIDENCE_SIGNAL#evidence-id` | `SIGNAL#signal-id` |
| Strong version lookup | `SIGNAL_VERSION#version-id` | `META` |
| Relationship lookup | `VERSION_REL#version-id` | `relationship-id` |

Accepted mapped versions use sparse `GSI1PK=RISK#context-version` keys. Planning
snapshots use deterministic 60-KiB `SECTION#SNAPSHOT#000000` children and a conditional
replacement transaction; snapshots above 90 chunks are rejected before writing.

Continuation tokens contain format version `1`, query identity, exclusive-start key,
and a canonical SHA-256 integrity digest. They are capped at 16 KiB, so malformed,
digest-mismatched, and cross-query tokens fail with a storage-neutral validation
error. The digest is unkeyed: it detects inconsistent contents, not deliberate
modification by someone who recomputes the digest.
