# POC #5.4 Client Demo Flow

1. **Rule Sets** — show commission, tax, withholding, bonus and adjustment as independent named business decisions.
2. **Attributes / Parameters** — add a new dynamic attribute without code or schema changes to the rule model.
3. **Compose Rule** — choose `TAX_RULES` or `BONUS_RULES`, dynamically add conditions and save to that rule set's active version.
4. **Test & Explain** — switch rule sets and show a single transaction evaluated with condition-level explanation.
5. **Payout Cycles** — create `PAYOUT-2026-08`; explain that all active rule-set versions are pinned immediately.
6. **Batch Execution** — choose the payout cycle + `COMMISSION_RULES`, 8 workers, 5000 batch size; point out that 3M transactions imply roughly 600 batches.
7. **Decision Audit** — select a persisted execution, search a transaction and show candidate rules, failed conditions and why the winner was selected.
8. **Payout Cycle transaction journey** — after multiple rule-set stages are executed for the same cycle, search a transaction to view its stage-by-stage decisions.
9. **Architecture** — show how the future client BPMN invokes rule-set keys and sends batches while the cycle snapshot guarantees reproducibility.

The current UI intentionally avoids BPMN-specific implementation details until the client's requirement documents are reviewed.


### Batch-level execution audit
POC #5.4 records a lightweight `BATCH_EXECUTION` row for every actual transaction chunk processed by a rule-set execution. For a 3M transaction run with batch size 5,000 this is roughly 600 audit rows, not millions. The execution detail screen shows transaction range, worker/partition, matched/no-match counts and read/engine/write/total timings per batch. Legacy runs remain valid and simply show no batch-level rows.
