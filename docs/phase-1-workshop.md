# Phase 1 workshop

Start with one decision at a time. This is an exercise for the project owner; the mentor should review the reasoning before accepting ADRs or implementing services.

## Exercise 1: Authority and recovery

Read [ADR 0001](decisions/0001-authority-and-projections.md). Imagine a transformation bug incorrectly counted every capture twice for a day. Kafka retention has already expired for that day.

Choose A, B, or C. Explain what survives, how you would rebuild, and how users would know which result is corrected. Also explain why a settlement file is evidence to compare rather than a reason to overwrite payment events.

Owner response: **Pending**.

## Exercise 2: Delivery versus business order

Use [the synthetic example](../examples/happy-path.json). The three events are delivered in this order: version 2, version 1, version 2 again, version 3.

For each arrival, write the retained raw observation count, canonical event count, last contiguous version, published lifecycle state, and gap/conflict indicator. Then change the second copy of version 2 to a different amount while keeping its event ID.

Questions to defend:

- Why does preserving every raw observation not imply counting every observation?
- Why is a changed amount with the same ID more serious than ordinary re-delivery?
- What can consumers trust while a gap or conflict remains unresolved?

Owner response: **Pending**.

## Exercise 3: Reconciliation

Reconcile capture and refund independently. Try removing the capture settlement row, duplicating the refund row, and changing the capture settlement amount by one cent.

Explain why a zero net total is insufficient evidence of correctness. Specify when an unmatched row is provisional, when it becomes actionable, and how a later correction affects an earlier published run.

Owner response: **Pending**.

## Decisions needed before Phase 2

- Accept or revise the authority model and partition/identity proposal.
- Accept or simplify the v1 lifecycle and monetary model.
- Set the late-data horizon, settlement close policy, and clock-skew tolerance.
- Establish local CPU/memory budget and an initial measurable workload.
- Finalize executable schema and fixture expectations before writing a producer.

Delta versus Iceberg, serving engine, AWS service selection, and Kubernetes are not prerequisites for the first generator/Kafka experiment. Decide the table format before the first durable lakehouse implementation.

## Mentoring agreement

For each component: explain the problem, compare choices, ask for consequential decisions, give a small implementation task, review the owner's code with explanations, and record interview questions. If the owner explicitly asks the mentor to implement a component, keep the change small and walk through its behavior and failure cases.
