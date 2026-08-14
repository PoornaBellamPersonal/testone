# POC #5.4 Verification Checklist

Use the existing 300K/3M H2 database if available. Back it up first.

1. Start with `scripts\run-showcase-large.bat`.
2. Dashboard should show 5 named rule sets and retain the previous commission benchmark metrics/history.
3. Open **Rule Sets**. Commission should point to the existing near-real active version; tax/withholding/bonus/adjustment should have active demo versions.
4. Open **Tax Rules** and verify the small sample rules are visible.
5. Open **Compose Rule**, switch to Bonus Rules and add a rule using dynamic attributes.
6. Open **Test & Explain**, choose Tax Rules, enter `ISSUE_STATE=CA` and `TRANSACTION_TYPE=NEW_BUSINESS`; the resident-tax sample should win over the fallback.
7. Create a payout cycle such as `PAYOUT-2026-08`. Open it and verify all active rule-set versions are pinned.
8. Open **Batch Execution** from that cycle and confirm Rule Set + Payout Cycle + worker/batch controls are present.
9. Start a small/appropriate stage run or use the existing benchmark dataset. The execution page should show rule-set key/version, planned batch count and cycle context.
10. If audit is persisted, open Decision Audit and explain a transaction.
11. After two or more rule-set stages are run under the same payout cycle, open the cycle and search the same transaction ID to see its stage-by-stage journey.
12. With the app running, execute `scripts\test-tax-batch-api.bat` to verify the BPMN-friendly batch API.

Expected API result for the first sample transaction: `TAX_RULES` selects the higher-priority resident-tax rule with result value `12.50`.


### Batch-level execution audit
POC #5.4 records a lightweight `BATCH_EXECUTION` row for every actual transaction chunk processed by a rule-set execution. For a 3M transaction run with batch size 5,000 this is roughly 600 audit rows, not millions. The execution detail screen shows transaction range, worker/partition, matched/no-match counts and read/engine/write/total timings per batch. Legacy runs remain valid and simply show no batch-level rows.
