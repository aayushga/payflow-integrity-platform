# Draft data contract v1

Status: Proposed. These rules and [examples](../examples/happy-path.json) are review material, not an implemented validator or finalized JSON Schema.

## Payment event envelope

| Field | Meaning / proposed constraint |
| --- | --- |
| `schema_version` | Integer `1`; unsupported versions retained and quarantined |
| `provider_id` | Fictional source namespace; required non-empty string |
| `event_id` | Stable source event identity; reused for retries, never regenerated on replay |
| `payment_id` | Stable payment identity within provider |
| `merchant_id` | Fictional merchant identity, immutable within payment |
| `entity_version` | Positive integer; contiguous per-payment sequence starting at 1 |
| `event_type` | `payment.authorized`, `payment.declined`, `payment.captured`, `payment.refunded` |
| `occurred_at` | Business event timestamp, RFC 3339 UTC with `Z` |
| `emitted_at` | Source emission timestamp, RFC 3339 UTC with `Z`; delivery time is recorded separately |
| `amount_minor` | Positive integer cents; never binary floating point |
| `currency` | `USD` in this release |
| `operation_id` | Required for capture/refund; absent for authorization/decline |
| `related_operation_id` | Required only on refund; identifies the capture being refunded |

Record platform receipt time and transport coordinates outside the source envelope. They must not change the business-content comparison used for duplicate detection.

## Lifecycle invariants

1. Version 1 is either authorization approval or decline. A decline is terminal in v1.
2. Capture requires an approved authorization and equals its full amount.
3. Refund requires a capture and equals its full amount; it references that capture's operation ID.
4. Merchant and currency stay constant within a payment; at most one capture and one refund exist.
5. Operation IDs are unique within a provider across payment, operation type, and time.
6. Do not infer ordering from arrival time, lexical event IDs, or timestamps. Apply a complete version prefix.
7. Business times may arrive late. Implausible future times and clock skew need an explicit validation tolerance before implementation.

JSON Schema can enforce field shape and some conditional requirements. Cross-event transitions, reference existence, uniqueness, and amount conservation require stateful/business validation.

## Identity, duplicates, and conflicts

- Same `(provider_id, event_id)` and same canonical business content: duplicate delivery.
- Same identity with different business content: conflict; preserve both observations, alert, and block affected projection advancement.
- Different event IDs with the same payment/version: lifecycle-position conflict, even if values look similar.
- Same operation ID attached to different payments or operations: business-identity conflict.

Define canonical serialization and its field set before implementing hashes. A hash helps compare payloads; it is not the primary business identity. Retain source bytes separately for audit. Field-order differences in JSON must not turn an identical event into a business conflict.

## Settlement row

Required fields: `provider_id`, `settlement_row_id`, `payment_id`, `merchant_id`, `operation_id`, `operation_type` (`capture` or `refund`), `settlement_date` (UTC date), `currency`, and `gross_amount_minor` (signed integer cents).

Capture gross is positive; refund gross is negative. The examples contain a 10,000-cent capture and a -10,000-cent refund, each reconciled separately. Their net is zero, but that must not hide an unmatched operation. File-level counts and gross totals are recorded in the ingestion manifest; source settlement rows remain separate from payment events.

## Schema evolution proposal

An added optional field can remain on the same major version only after consumer compatibility is tested. Renaming/removing a required field, changing its meaning, changing amount units, or changing identity semantics requires a new major version and a migration plan. Do not automatically evolve a trusted financial table merely because an input schema changed.

For v1, known fields must be validated explicitly; unknown fields are retained in raw evidence and excluded from trusted projections until reviewed. The duplicate comparison policy for future fields must be defined with each contract change. Preserve producer, parser, and rules versions so historical replay has explainable semantics.

## Open questions

- Is the single authoritative provider/version sequence a reasonable first-release assumption?
- Should the initial release allow refunds, or introduce them after capture reconciliation works?
- Is the proposed late-data horizon and fictional D+2 settlement close appropriate for the exercise?
- What clock-skew tolerance should a validator allow?
