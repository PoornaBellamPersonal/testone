# Drools Commission POC #4 — Realistic Rule Stress

This POC extends the successful file-H2 end-to-end benchmark with deliberately difficult business-rule behavior:

- **200,000 persisted commission rows**
- **2,000,000 persisted transaction rows**
- **24 transaction inputs**
- categorical `ANY` / wildcard conditions
- overlapping numeric and date ranges
- multiple matching rows per transaction
- explicit priority and specificity
- deterministic tie breaking
- configurable no-match records
- configurable hot-partition traffic skew
- matched-candidate and ambiguity audit columns

The goal is not merely to prove that a unique synthetic rule can match. It measures whether the same architecture remains practical when realistic commission definitions overlap.

## Rule model

The 200K rows represent 40K logical commission groups. Each group contains five rows:

| Variant | Purpose | Default rank |
|---|---|---|
| `EXACT` | Fully specific commission row | priority 100, specificity 24 |
| `GEO_WILDCARD` | `ANY` state/channel plus expanded amount/date ranges | priority 80 |
| `DISTRIBUTION_WILDCARD` | `ANY` agent/registration/payment plus expanded ranges | priority 70 |
| `POLICY_WILDCARD` | `ANY` policy/producer/customer plus broad ranges | priority 60 |
| `EXACT_FALLBACK` or `EXACT_TIE` | exact overlapping fallback or deliberate equal-rank conflict | priority 90 or 100 |

Ten percent of logical groups replace `GEO_WILDCARD` with a `PRIORITY_OVERRIDE` at priority 110. This verifies that a broader business override can outrank an exact row.

A subset of groups contains an exact equal-priority/equal-specificity conflict. The runtime still produces a deterministic winner, but records `TOP_RANK_COUNT > 1` so the configuration can be flagged for business review.

For this controlled stress profile, `PLAN_CODE` acts as the logical commission-program anchor. Therefore a normal matching transaction is expected to produce exactly five candidate rows. This isolates wildcard, range, priority and ambiguity costs without allowing unrelated rule programs to create accidental correctness failures. A later candidate-density test can intentionally share program anchors.

## Winner policy

Every matching activation calls `CommissionTransaction.considerCandidate(...)`.

The deterministic order is:

1. higher `PRIORITY`
2. higher `SPECIFICITY`
3. lower `RULE_ID`

This produces repeatable results regardless of activation firing order.

## Expected data profile

The standard full script uses:

- 5% no-match transactions
- 40% of all transactions routed to partition 0
- the remaining 60% spread over the other 31 partitions
- approximately 95% multi-match transactions
- approximately 20% top-rank ambiguity among matching transactions
- approximately 10% priority-override winners among matching transactions
- average candidate count near 4.75 over all transactions
- maximum candidate count 5

Exact counts can vary slightly because transactions select logical groups randomly.

## Architecture

```text
H2 COMMISSION_RULE (200K overlapping rows)
                  |
                  v
          32 stateful KieSessions
                  ^
                  |
H2 COMMISSION_TRANSACTION (up to 2M rows)
                  |
             JDBC batches
                  |
                  v
       generic Drools candidate matcher
                  |
                  v
 priority / specificity / ID resolver
                  |
                  v
H2 COMMISSION_RESULT
```

`COMMISSION_RESULT` contains:

- expected rule ID
- matched rule ID
- commission percentage
- winning priority
- winning specificity
- candidate count
- top-rank count
- result status

Statuses are:

- `MATCHED`
- `MATCHED_AMBIGUOUS`
- `NO_MATCH`
- `INCORRECT`

## Prerequisites

- JDK 17+; JDK 21 recommended
- Maven 3.9.6+
- Enough free disk for the 2M H2 database and Maven build output

## Run order

### 1. Smoke

```bat
scripts\run-realistic-smoke.bat
```

Profile:

- 20K rule rows / 4K logical groups
- 100K transactions
- 8 sessions
- 4 workers
- 5% no-match
- 30% hot-partition share

Check that:

- `Incorrect = 0`
- matching transactions generally have 5 candidates
- no-match transactions have 0 candidates
- ambiguity and priority behavior are visible

### 2. Medium

```bat
scripts\run-realistic-medium.bat
```

Profile:

- full 200K rule rows / 40K logical groups
- 200K transactions
- 32 sessions
- 8 workers
- 40% hot-partition share

This is the first full-rule-count decision point.

### 3. Full target

```bat
scripts\run-realistic-full-2m.bat
```

Profile:

- 200K rules
- 2M transactions
- 32 sessions
- 8 workers
- 5K read/write batches
- 5% no-match
- 40% traffic in partition 0

### 4. Reuse the 2M database

```bat
scripts\run-realistic-reuse-2m.bat
```

This skips seeding, truncates previous results, reloads active rule facts and reprocesses the stored transactions.

## Inspect the rule and result data

```bat
scripts\inspect-realistic.bat
```

Or open the H2 browser console:

```bat
scripts\open-h2-console.bat
```

Use:

```text
JDBC URL: jdbc:h2:file:./data/commission-realistic-2m
User: sa
Password: leave blank
```

Useful SQL is in:

```text
queries/realistic-queries.sql
```

For example, logical group 0 is stored as rule IDs 0 through 4.

## Metrics

The benchmark reports:

- seed times
- KieBase build time
- H2-to-Drools rule load time
- retained heap
- end-to-end throughput
- database read, Drools and database write time
- batch p50/p95/p99
- matched/no-match/incorrect counts
- multiple-match transaction count
- top-rank ambiguity count
- average and maximum candidates per transaction
- H2 file size
- monthly runtime path and two-hour margin

CSV history is appended to:

```text
benchmark-results-realistic.csv
```

## Disk cleanup

Delete only POC #4 databases:

```bat
scripts\clean-realistic-data.bat
```

Remove all generated data, Maven output, CSV history and project-local heap dumps:

```bat
scripts\clean-all-generated.bat
```

Compact the full H2 database:

```bat
scripts\compact-realistic.bat
```

## Important interpretation

This POC is a controlled realistic stress profile, not yet the final production rule repository. It proves:

- wildcard rows can coexist with exact rows
- overlapping ranges can yield several candidates
- priority overrides are deterministic
- equal-rank ambiguity can be detected and audited
- no-match and partition-skew behavior can be measured

Production follow-ups should replace `double` financial values with `BigDecimal`/`DECIMAL`, add effective rule-set versioning and approvals, and test real client distributions where many logical programs may share routing attributes.
