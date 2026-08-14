# POC #5.4 — Named Rule Sets + Payout Cycle Integration

## Why this version exists
The client payout process invokes different business decisions at different BPMN stages. Commission, tax, withholding, bonus and adjustment rules must therefore be maintained and invoked as independent named rule sets. Transactions are processed in batches.

## Added
- First-class `RULE_SET` catalog with stable BPMN/API keys.
- Independent `RULE_SET_VERSION` history per named rule set.
- Backward-compatible migration of the existing commission version and benchmark executions.
- Default RFP rule sets:
  - `COMMISSION_RULES`
  - `TAX_RULES`
  - `WITHHOLDING_RULES`
  - `BONUS_RULES`
  - `ADJUSTMENT_RULES`
- Business-friendly rule-set list/detail/create screens.
- Rule Explorer, Composer, CSV import, versioning and Test/Explain are now rule-set aware.
- Batch Execution selects a named rule set and optional payout cycle.
- Batch count and rule-set/version metadata captured on every execution.
- Non-commission rule sets do not compare against the commission benchmark's expected winner IDs.
- `CYCLE_EXECUTION` and `CYCLE_RULESET_SNAPSHOT` model immutable rule-set versions at payout-cycle start.
- Payout Cycle UI with rule-set snapshot and transaction journey view.
- BPMN-friendly synchronous batch API:
  `POST /api/v1/rule-sets/{ruleSetKey}/evaluate-batch`
- Updated architecture view showing BPMN → batches → named rule sets → sparse indexed engine → audit.

## Existing 300K / 3M database
POC #5.4 is designed to start against the POC #5.3 H2 database. `schema.sql` uses additive tables/columns and startup migration. Existing commission rules, 3M transactions and Execution #1/#2 are retained.

On first POC #5.4 startup, legacy rule versions are linked to `COMMISSION_RULES`. Empty active demo versions are created for tax/withholding/bonus/adjustment. If the near-real 300-attribute catalog is present, a few lightweight sample rules are inserted into the auxiliary rule sets for UI demonstration.

## Not yet implemented
- The client's final BPMN process definition (waiting for requirement documents).
- Production asynchronous batch-reference API / message broker contract.
- Full maker/checker approvals and RBAC.
- Oracle production persistence and distributed workers.


### Batch-level execution audit
POC #5.4 records a lightweight `BATCH_EXECUTION` row for every actual transaction chunk processed by a rule-set execution. For a 3M transaction run with batch size 5,000 this is roughly 600 audit rows, not millions. The execution detail screen shows transaction range, worker/partition, matched/no-match counts and read/engine/write/total timings per batch. Legacy runs remain valid and simply show no batch-level rows.
