# ADR 0002: Payment key, stable identities, and source versions

Status: **Proposed — owner decision pending**  
Date: 2026-10-28

## Problem

We need to recognize retries and reconstruct a valid payment lifecycle when delivery order differs from business order.

## Options

| Choice | Benefit | Tradeoff |
| --- | --- | --- |
| Key Kafka by merchant | Groups merchant traffic | A large merchant can dominate a partition; unnecessary ordering across its payments |
| Key by payment within provider | Keeps one payment's deliveries together | Merchant aggregates need another grouping/shuffle |
| Random key | Spreads arrivals | Related events may need more coordination downstream |

## Recommendation

Key by `(provider_id, payment_id)`. Use stable source event IDs for retry detection and a provider-assigned `entity_version` for lifecycle order. Retain transport coordinates for observation identity, not business identity.

Kafka arrival order alone cannot repair an event that was delivered late. Business versions make gaps explicit. The proposed upstream provider serializes versions; we are not assuming unrelated microservices can generate a reliable shared counter for free.

Use event identity, payment/version uniqueness, and operation identity together. Same ID plus different content is a conflict, not a duplicate to ignore. Keep the last valid contiguous lifecycle prefix and publish gap/conflict metadata.

## Tradeoffs and failure modes

- Versions require an upstream authority or a different causal ordering design.
- Missing versions can stall a payment indefinitely; repair needs evidence and a runbook.
- Increasing topic partition count may change key assignment; ordering during migration needs a plan.
- State eviction cannot erase the obligation to detect historical replay duplicates.
- Payment partitioning does not eliminate hot merchant aggregation keys.

## Scaling

At 10x, measure partition skew, consumer lag, shuffle, and state size. At 100x, revisit aggregate sharding, backfill isolation, and the upstream ordering assumption. Do not shard a payment's lifecycle until its actual workload requires a different model.

## Owner response

Pending: defend the partition key, explain why offsets cannot identify a retried business event, and describe how version 3 arriving before version 2 should affect state.
