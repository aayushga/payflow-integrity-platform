# ADR 0001: Preserve source evidence and rebuild derived state

Status: **Accepted — architecture decision; implementation pending**

Date: 2026-10-28

Accepted: 2026-11-12

## Problem

After an incorrect transformation or a failed deployment, what evidence lets us reconstruct a payment's history and explain a reconciliation discrepancy?

## Options

| Option | Benefit | Cost / failure mode |
| --- | --- | --- |
| A. Current-state table is sufficient authority | Simple reads; small retained footprint | Overwrites can destroy evidence; logic corrections may be impossible to replay |
| B. Kafka is permanent authority | Simple offset-based replay story | Requires deliberate retention, durability, and archival guarantees beyond the initial broker setup |
| C. Preserve original events/files; derive versioned state | Auditability, reproducible rebuilds, independent reprocessing | More storage, provenance tracking, retention policy, and publication coordination |

## Decision

Choose C with hybrid replay. Bronze is the durable record of what the platform observed, including duplicates and invalid inputs. Kafka provides recent offset-based recovery while the needed offsets remain available; retained raw events/files support historical rebuilds. Silver is the usable, validated projection; it is not an independent replacement for source history. Settlement files are a separate source's claims and must not silently overwrite event history.

Persist raw observations continuously while Kafka still retains them. This is overlapping retention, not a delayed move to files when Kafka history expires. Track durable archival progress per topic/partition and alert before ingestion lag approaches actual broker retention. An offset outside retained history requires an explicit gap investigation; never silently skip to the latest offset. Kafka retention duration, size limits, raw retention, and recovery budgets remain open. The owner's examples of one, six, or 24 hours are candidate windows, not an approved configuration.

Use recent Kafka recovery for suitable operational failures. A transformation bug may require rebuilding from Bronze even when the data remains in Kafka. Choose the recovery source from evidence availability, affected processing stage, and correctness needs, rather than event age alone. Both paths must use the same reviewed business rules and stable identities.

This is an analytical platform, not the payment processor's transactional system of record. Raw evidence can itself be wrong. Conflicts require explicit resolution records and provenance; preserving a value does not certify its truth.

Rebuild into a new projection version, compare counts/invariants, then publish a new version. Never silently mutate the historical evidence to make reconciliation pass.

## Identity and verification obligations

The owner identified stable unique IDs as necessary for recognizing duplicates across retries and replay. An ID alone does not repair transformation bugs or decide how an already-counted aggregate should be corrected. Duplicate handling, conflicting-payload detection, and projection publication still require explicit rules. The exact identity fields, partition key, and ordering contract remain proposed in ADR 0002.

For a capture-double-counting incident, rebuild with corrected, versioned logic into an isolated projection. Compare unique operation counts, per-merchant/currency capture and refund totals, lifecycle invariants, and independently specified expected fixtures. Reconcile individual operations against settlement evidence, accounting for pending/missing settlements. Do not rely only on a net total, the defective previous output, or running the same faulty logic twice. Publish after checks pass and retain the previous result and repair provenance. This is the planned acceptance procedure, not a completed recovery experiment.

## Tradeoffs and failure modes

- Retention limits constrain how far back we can rebuild.
- A replay can reproduce a bug unless parser/rules versions are explicit.
- Loss before Bronze persistence is outside Bronze recovery; broker durability and upstream gap detection need separate tests.
- Archival lag can overrun Kafka retention; monitoring must compare durable per-partition coverage with the earliest available broker offsets.
- Readers can see inconsistent tables without a publication protocol.
- Incomplete upstream histories must remain visible as incomplete.

## Scaling

At 10x, measure file size, write amplification, replay duration, and retention cost. At 100x, consider isolated replay compute, tiered retention, and controlled publication windows. Neither scale factor is a measured capability today.

## Owner response

The owner selected option C on 2026-11-12, identified unique IDs as necessary for duplicates, delayed deliveries, and replay, and proposed Kafka for recent offset replay plus retained raw files for older history. The mentor clarified continuous archival overlap and the distinction between deduplication and repairing buggy transformations. Acceptance covers the authority/replay architecture; identity details and retention values remain undecided. The owner's rebuild-verification explanation is still an open workshop follow-up.
