# Interview Story

Status: Design reasoning only. No implementation, benchmark, or incident evidence exists yet. Do not describe proposed decisions as delivered production experience.

## Story 001: Recoverable analytical state

- **Problem:** Fix incorrect derived payment data after original broker retention expires.
- **Options considered:** Current state only, Kafka as permanent authority, retained source evidence plus rebuildable projections.
- **Decision:** Proposed option C; owner review pending in ADR 0001.
- **Tradeoff:** More storage and publication complexity in exchange for replay and auditability.
- **Failure modes:** Lost evidence before persistence, expired retention, non-reproducible rules, mixed table versions.
- **Scaling approach:** Measure replay throughput, file layout, and storage cost before changing topology.
- **At 10x:** Tune file sizing and isolate expensive reprocessing if measurements justify it.
- **At 100x:** Revisit retention tiers, replay compute isolation, and publication coordination.
- **Evidence:** Proposed design only; later link a controlled corruption/rebuild experiment.
- **Interview prompt:** What exactly is authoritative, and how do you know a replay repaired the data?

## Story 002: Ordering is a business contract

- **Problem:** Re-delivery and out-of-order events can double-count money or publish invalid lifecycle state.
- **Options considered:** Merchant/payment/random partition keys; timestamp order versus source versions.
- **Decision:** Proposed payment key, stable event identities, operation IDs, and contiguous source versions; owner review pending in ADR 0002.
- **Tradeoff:** Per-payment ordering is useful, but introduces a sequenced-source assumption and gap repair obligations.
- **Failure modes:** Conflicting identities, missing versions, state eviction, partition migration, skewed merchant aggregates.
- **Scaling approach:** Separate lifecycle processing from merchant aggregation; inspect lag, state size, and skew.
- **At 10x:** Increase parallelism only where partitions and workload permit.
- **At 100x:** Consider skew-aware aggregation and independent backfill capacity.
- **Evidence:** Synthetic example only; later link duplicate, restart, and reorder test results.
- **Interview prompt:** Why doesn't an exactly-once sink eliminate duplicate business events?

## Entry template

For each accepted design decision, record problem, options, decision, tradeoff, failure modes, scaling approach, 10x/100x changes, and links to code/tests/measurements. Record the owner's explanation, including alternatives rejected and why. Keep proposed, implemented, and demonstrated claims separate.
