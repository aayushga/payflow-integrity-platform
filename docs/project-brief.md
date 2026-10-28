# Project brief

Status: Draft for owner review, 2026-10-28.

## Users and outcomes

| User | Need | Initial output |
| --- | --- | --- |
| Payment operations | Explain a payment's current lifecycle and missing events | State, last contiguous version, gap/conflict indicators, freshness |
| Finance operations | Compare captured/refunded operations with settlement | Discrepancies with evidence and reconciliation run ID |
| Platform engineer | Recover from outages without duplicating business effects | Replay procedure, lag, quarantine counts, failure tests |

## Initial business model

One fictional provider emits a payment lifecycle. Each payment has one authorization attempt, followed by either approval or decline. An approved payment may have one full capture; a captured payment may have one full refund. Expiry, voids, partial captures, partial refunds, chargebacks, and multiple authorization attempts are future extensions.

The provider assigns a monotonic version starting at 1 for each payment. This is a deliberate upstream contract: independent services cannot invent a shared sequence without coordination. The generator simulates this provider and can reorder delivery without changing the original versions.

Settlement is a separate dimension. A payment does not simply move from CAPTURED to SETTLED to REFUNDED. The capture and refund each have an operation ID that can independently match a settlement row.

Settlement files represent gross operations in USD. Capture rows are positive; refund rows are negative. Fees, net payouts, FX, ledger balances, and accounting journal entries are outside this release.

## Candidate service objectives

These are proposed development targets, not observed results or production promises.

| Property | Candidate acceptance criterion |
| --- | --- |
| Correctness | Duplicate delivery never adds an extra canonical operation or changes totals |
| Recovery | Restart after a controlled failure converges to the same canonical results as an uninterrupted run |
| Freshness | Under an agreed local workload, p95 ingestion-to-published-result delay below 30 seconds |
| Reconciliation | Re-running the same input snapshots and rules produces identical classifications and totals |
| Traceability | Every canonical operation and discrepancy points to source evidence and a processing/rules version |

Before measuring freshness, record hardware, container limits, event size, event rate, partition count, micro-batch settings, warm-up, and measurement window. Report source-to-ingestion delay separately from platform processing delay. Agree on a load and recovery-time target after inspecting local resources.

## Faults we will deliberately introduce

- Re-delivery with the same event ID and same business content.
- Same identity with conflicting business content.
- Missing, late, and reordered payment versions.
- Producer interruption around broker acknowledgment.
- Stream restart around output commit/checkpoint boundaries.
- Invalid amounts, unknown schema versions, and malformed payloads.
- Duplicate settlement rows, missing files, file corrections, and DAG retries.
- A slow consumer that accumulates lag.

## Scope boundaries

Build a reproducible data platform with meaningful failure cases. Use fictional identifiers only and generate original sample data. Include provenance and generator version; do not import enterprise artifacts or redistribute third-party datasets.

Defer customer UI, real payment integrations, a full accounting ledger, multi-region active-active deployment, multiple lakehouse formats, Kubernetes, a fraud model, and AI features. Add a technology only after recording the need it addresses.

## Phase 1 acceptance

The owner can explain and approve the authority model, ordering contract, identities, lifecycle invariants, settlement matching rules, and late-data policy. Proposed documents alone do not complete this learning checkpoint.
