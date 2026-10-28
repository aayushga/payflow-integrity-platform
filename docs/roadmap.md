# Roadmap

Phases are learning checkpoints, not delivery-date commitments. Phase 0 is complete: the owner selected payments and settlement reconciliation. Phase 1 is in progress; all ADRs are proposed.

| Phase | Small deliverable | Evidence required to exit |
| --- | --- | --- |
| 1. Domain and contracts | Brief, event examples, owner-reviewed decisions | Owner explains authority, identity, ordering, lifecycle, late-data, and reconciliation policies |
| 2. Event backbone | Generator, Kafka producer, Compose profile | Deterministic seed reproduces scenarios; same retry keeps its event ID; consumer-group/partition experiment recorded |
| 3. Raw ingestion | Spark to Bronze using one selected table format | Persist original evidence; prove restart behavior and raw completeness under a controlled failure |
| 4. Trusted state | Validation, canonical events, lifecycle projection | Tests for duplicates, conflicts, gaps, reorder, replay, state horizon, and invalid transitions |
| 5. Batch reconciliation | Airflow file ingestion and comparison | Idempotent DAG retry; manifests; correction revisions; pinned input snapshots and discrepancy fixtures |
| 6. Products and API | Merchant aggregates and bounded read-only endpoints | Results declare freshness/completeness; pagination, error handling, and authorization boundaries verified |
| 7. Reliability | Failure harness, metrics, runbooks | Broker/worker interruption, slow processing, late file, and repair experiments with measured outcomes |
| 8. Engineering quality | CI, dependency checks, reproducible integration suite | Fresh-clone instructions verified; tests exercise meaningful failure boundaries; no unsupported badges/claims |
| 9. AWS design and IaC | AWS ADR, scoped Terraform, cost/recovery model | Format/validation checks; least-privilege review; deployment remains optional and separately scoped |
| 10. Portfolio evidence | Benchmark report and interview narratives | Actual hardware/workload/results, limitations, architecture walkthrough, resume bullets grounded in evidence |
| Optional 11. AI investigation | Governed read-only discrepancy assistant | Core platform is complete; query allowlist, access limits, and evaluation cases demonstrated |

Add contract tests with the first implementation and integration tests with each new boundary. Phase 8 consolidates CI and reproducibility; it is not the first time we test the system.

Before Phase 3, compare Delta Lake and Iceberg using our actual Spark versions, local object store, merge/idempotency needs, catalog overhead, query serving, and AWS deployment options. Select one and record the rejected alternative.

Each phase ends with code review, a failure discussion, an Interview Story update, and a small next task for the owner. Do not pre-generate later services to fill out the repository tree.
