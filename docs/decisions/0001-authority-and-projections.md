# ADR 0001: Preserve source evidence and rebuild derived state

Status: **Proposed — owner decision pending**

Date: 2026-10-28

## Problem

After an incorrect transformation or a failed deployment, what evidence lets us reconstruct a payment's history and explain a reconciliation discrepancy?

## Options

| Option | Benefit | Cost / failure mode |
| --- | --- | --- |
| A. Current-state table is sufficient authority | Simple reads; small retained footprint | Overwrites can destroy evidence; logic corrections may be impossible to replay |
| B. Kafka is permanent authority | Simple offset-based replay story | Requires deliberate retention, durability, and archival guarantees beyond the initial broker setup |
| C. Preserve original events/files; derive versioned state | Auditability, reproducible rebuilds, independent reprocessing | More storage, provenance tracking, retention policy, and publication coordination |

## Recommendation

Choose C. Bronze is the durable record of what the platform observed, including duplicates and invalid inputs. Silver is the usable, validated projection; it is not an independent replacement for source history. Settlement files are a separate source's claims and must not silently overwrite event history.

This is an analytical platform, not the payment processor's transactional system of record. Raw evidence can itself be wrong. Conflicts require explicit resolution records and provenance; preserving a value does not certify its truth.

Rebuild into a new projection version, compare counts/invariants, then publish a new version. Never silently mutate the historical evidence to make reconciliation pass.

## Tradeoffs and failure modes

- Retention limits constrain how far back we can rebuild.
- A replay can reproduce a bug unless parser/rules versions are explicit.
- Loss before Bronze persistence is outside Bronze recovery; broker durability and upstream gap detection need separate tests.
- Readers can see inconsistent tables without a publication protocol.
- Incomplete upstream histories must remain visible as incomplete.

## Scaling

At 10x, measure file size, write amplification, replay duration, and retention cost. At 100x, consider isolated replay compute, tiered retention, and controlled publication windows. Neither scale factor is a measured capability today.

## Owner response

Pending: choose A, B, or C and explain the recovery case that matters most. Acceptance requires updating this status and the Interview Story with the owner's reasoning.
