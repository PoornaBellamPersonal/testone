# POC Verification Checklist

## Startup

- [ ] Maven build succeeds.
- [ ] Infrastructure starts H2 TCP 9093, H2 web 8082 and ActiveMQ 61616.
- [ ] CMS starts on 8080.
- [ ] RE-01 starts on 8091.
- [ ] RE-02 starts on 8092.
- [ ] WORKER-01 starts on 8085.
- [ ] WORKER-02 starts on 8086.
- [ ] Runtime Nodes page shows every component UP.

## Rule engine smoke test

- [ ] `scripts\09-smoke-rule-engine.bat` returns RE-01.
- [ ] APAC + LIFE + GOLD + amount 25000 matches `COMM-001`.
- [ ] commission rate is 8.25.

## 5K functional cycle

Use:

```text
Transactions 5000
Batch size   1000
```

- [ ] One Camunda payout process instance starts.
- [ ] Commission creates 5 batches.
- [ ] Both worker IDs appear in batch execution.
- [ ] Both rule-engine node IDs appear.
- [ ] Tax does not create/start until all Commission batches complete.
- [ ] Withholding does not create/start until all Tax batches complete.
- [ ] Cycle reaches COMPLETED.
- [ ] Transaction 1 has Commission/Tax/Withholding audit records.
- [ ] Process page shows completed stages.
- [ ] Performance page shows RE distribution and durations.

## 50K architecture/performance run

Use:

```text
Transactions 50000
Batch size   5000
```

- [ ] 10 batches per rule stage.
- [ ] 30 rule-stage batches overall.
- [ ] no batch is left PENDING/PROCESSING after cycle completion.
- [ ] end-to-end duration captured.
- [ ] worker distribution captured.
- [ ] RE-01/RE-02 distribution captured.

## Failure demonstration (optional)

- [ ] Stop RE-02.
- [ ] Start a small cycle.
- [ ] a batch routed to RE-02 becomes FAILED.
- [ ] stage supervisor detects failed count.
- [ ] cycle becomes FAILED rather than starting later stages.

Restart RE-02 before the normal client walkthrough.

## Persistence

- [ ] Stop all applications cleanly.
- [ ] Restart infrastructure and CMS.
- [ ] previous cycle rows remain because H2 is file-backed.
