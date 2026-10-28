# Proposed architecture

Status: Design only. No services have been deployed or tested.

## Boundaries and responsibilities

| Component | Problem it solves | Boundary / tradeoff |
| --- | --- | --- |
| Scenario generator | Reproducible lifecycles and controlled faults | Domain-aware synthetic data; no external feed dependency |
| Kafka | Buffering and independently replayable consumer positions | Finite retention; not permanent business authority |
| Spark Structured Streaming | Incremental transformation and stateful processing | Micro-batch latency and state/checkpoint operations must be measured |
| Bronze | Retain original evidence for audit and rebuild | Duplicate observations are allowed; storage grows |
| Silver | Validated canonical operations and lifecycle projections | Business semantics require more than JSON validation |
| Gold | Merchant totals and reconciliation results | Outputs must declare snapshot/rules version and freshness |
| Airflow | Batch dependencies, intervals, retries, backfills | Runs finite batch jobs; does not manage a long-running stream as a daily task |
| Read-only API | Stable product access and bounded queries | Serving engine remains undecided; API does not execute arbitrary client SQL |

## Layer design

Bronze retains original event bytes, receipt timestamp, topic, partition, offset, parser version, and processing run. Do not put raw financial payloads in logs. Unparseable bytes still need durable evidence and a transport identity.

Silver contains canonical events, conflict records, and a rebuildable payment projection. Proposed canonical event identity is `(provider_id, event_id)`. A second uniqueness rule on `(provider_id, payment_id, entity_version)` detects different events claiming the same lifecycle position. Neither rule alone is enough.

Apply lifecycle transitions only across contiguous, valid versions. A missing version produces an explicit incomplete state; it must not be guessed from a later event. A contradictory version blocks publication of newer lifecycle state until investigated. Retain the last valid prefix and expose the problem.

Gold contains aggregates and reconciliation results built from named input snapshots. Initially publish each table atomically where the selected format supports it, but make no cross-table transaction claim. An API response identifies the publication/run it used so users can detect stale or mixed versions. Choose the publication manifest protocol before implementing multi-table serving.

## Settlement and batch/stream convergence

Airflow ingests original files and a manifest containing provider, file ID, revision, business date, checksum, row count, and gross total. A repeated file ID/revision with a different checksum is a conflict. A correction has a new revision and explicitly supersedes its predecessor; preserve both raw versions.

Trusted rows are unique by `(provider_id, settlement_row_id)` within the selected effective file set. Multiple rows for the same operation are flagged rather than summed blindly. Matching uses `(provider_id, operation_id)`; merchant, payment, currency, operation type, and signed gross amount must also agree.

Reconciliation pins both event and settlement input snapshots, the business date/timezone, rules version, cutoff, and run ID. Classifications are `matched`, `event_without_settlement`, `settlement_without_event`, `amount_mismatch`, `attribute_mismatch`, and `duplicate_or_conflict`. Missing counterparts are provisional until the proposed D+2 12:00 UTC close for a UTC business day. This is a fictional provider policy for owner review, not a payments-industry guarantee. Later arrivals create a revised run; they do not erase prior results.

An incomplete payment history must be reported as incomplete evidence, not a conclusively reconciled result. Identical pinned inputs and rules produce identical business results, although execution metadata such as run IDs and timestamps may differ.

## Delivery and recovery

Proposed ingestion assumes duplicates can occur. Broker delivery, checkpoint recovery, lakehouse commits, and API publication are separate boundaries. We will test each boundary before making guarantees. Transactional table writes alone do not establish end-to-end exactly-once behavior.

Checkpoint restart handles normal continuation. Business replay reads durable Bronze into a new projection version, validates totals, and switches readers only after verification. Do not reset offsets into existing sinks and assume it is safe. Before replay, define the retained input horizon and whether all necessary reference versions still exist.

The proposed streaming lateness horizon is 24 hours for bounded state; owner approval and measurement are pending. Older valid events remain in Bronze and enter a batch repair path. A watermark is not permission to discard financial evidence. A durable identity strategy must cover replay beyond the streaming state horizon; evaluate table uniqueness, merge behavior, and concurrent writers when choosing the format. Start with one logical writer per canonical table.

Quarantine is for invalid/conflicting data with a reason, evidence pointer, and reprocessing status. Transport retries are bounded with backoff and jitter; transient infrastructure faults should fail/retry the job rather than classify otherwise-valid business events as invalid. Define poison-message isolation before implementation.

## Partitioning and scale

Propose Kafka key `(provider_id, payment_id)` to colocate a payment's deliveries. This does not reorder late events or establish global order. Provider versions define business order. Merchant analytics requires a later repartition, with attention to merchant skew.

Choose the initial partition count after measuring local capacity. A single-broker Compose deployment cannot demonstrate broker high availability. At 10x, measure consumer lag, input skew, state size, shuffle, storage commits, and small files before adding compute. At 100x, revisit hot keys, table layout, retention cost, serving isolation, and regional requirements using workload evidence.

## Local and AWS choices still open

Local: Docker Compose profiles for Kafka, Spark, object storage, batch orchestration, and API serving. Bring services up by milestone rather than requiring the entire stack for a unit test. Confirm laptop memory/CPU before choosing versions and defaults.

Production candidates: managed Kafka, S3, managed or containerized Spark, managed Airflow, and a managed API runtime. Compare operational effort, cost, compatibility, recovery behavior, and IAM needs in an AWS ADR. Do not create billable infrastructure during Phase 1.

## Security and observability

Use synthetic data, ignored local credentials, and separate read/write roles. Production design should specify private networking, encrypted transport/storage, per-service least privilege, secret rotation, retention, and restricted raw/quarantine access. An API's read-only role must not grant ingestion or table mutation rights. Ignore rules reduce accidents but are not a security control by themselves.

Track source freshness, broker lag, processing delay, duplicate/conflict counts, missing versions, late-data repairs, quarantine age, batch completeness, and reconciliation discrepancies. Separate provider/business problems from infrastructure incidents. Each metric needs an actionable owner and runbook before becoming an alert.
