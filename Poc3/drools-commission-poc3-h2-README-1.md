# Drools Commission POC #3 — File-based H2 end-to-end batch

This POC extends the successful **rules-as-facts** design from POC #2 into a more realistic persisted batch flow.

## Target profile

- **200,000 commission rules** stored in a file-based H2 database
- **200,000 input transactions** stored in the same database
- **24 matching inputs**
- one generic compiled Drools rule
- 32 partitioned stateful `KieSession`s
- paged JDBC reads
- Drools evaluation
- batched JDBC result inserts
- complete reconciliation with expected/matched rule IDs

An additional script is included for the original SLA volume of **2,000,000 transactions**.

## Architecture

```text
File H2
  ├── COMMISSION_RULE          200K business rule rows
  ├── COMMISSION_TRANSACTION   200K or 2M input rows
  └── COMMISSION_RESULT        persisted evaluation results

COMMISSION_RULE
       |
       | load once by partition
       v
32 stateful Drools sessions
       ^
       |
JDBC transaction batches
       |
       v
Drools matching
       |
       v
JDBC batch result writes
```

The 200K commission rows are **data/facts**, not 200K compiled DRL rules.

## Prerequisites

- JDK 17+; JDK 21 recommended
- Maven 3.9+
- Windows scripts are included; one Linux/macOS target script is also included

## Run order

### 1. Smoke test

```bat
scripts\run-h2-smoke.bat
```

Profile:

- 20K rules
- 20K transactions
- 8 partitions
- 4 workers
- separate database: `data/commission-smoke.mv.db`

### 2. Realistic 2-lakh target

```bat
scripts\run-h2-target.bat
```

Profile:

- 200K rules
- 200K transactions
- 32 partitions
- 8 workers
- 2K JDBC read/write batches
- database: `data/commission-benchmark.mv.db`

The script recreates and reseeds the database.

### 3. Repeat only the processing path

```bat
scripts\run-h2-reuse.bat
```

This keeps the existing database and reruns:

1. load 200K rule rows into Drools sessions
2. clear the result table
3. read 200K transactions in batches
4. evaluate
5. write 200K result rows

Use this number as the closest representation of a recurring month-end execution after input data is already staged.

### 4. Optional original full SLA volume

```bat
scripts\run-h2-full-2m.bat
```

Profile:

- 200K rules
- 2M transactions
- 32 partitions
- 8 workers
- 5K read/write batches
- database: `data/commission-2m.mv.db`

This database will consume substantially more disk space than the 200K-transaction test.

## Inspect the seeded rules and transactions

### Console sample output

```bat
scripts\inspect-h2.bat
```

It prints table counts, ten sample rules, ten transactions, ten results and the H2 database file size.

### H2 browser console

Make sure no benchmark Java process is still using the embedded database, then run:

```bat
scripts\open-h2-console.bat
```

Use:

```text
JDBC URL : jdbc:h2:file:./data/commission-benchmark
User     : sa
Password : leave blank
```

Useful SQL is provided in:

```text
queries/sample-queries.sql
```

## Database tables

### `COMMISSION_RULE`

Contains the 200K business definitions, including all exact inputs, range boundaries and commission percentage.

### `COMMISSION_TRANSACTION`

Contains persisted transaction inputs and `EXPECTED_RULE_ID`, which is synthetic POC verification metadata.

### `COMMISSION_RESULT`

Contains:

- transaction ID
- expected rule ID
- matched rule ID
- commission percentage
- `MATCHED`, `NO_MATCH`, or `INCORRECT` status
- processed timestamp

A real production table would retain rule-set version, batch ID, calculation explanation and audit metadata as well.

## Metrics

The run reports these separately:

- rule-table seed time
- transaction-table seed time
- generic KieBase build time
- H2-to-Drools rule-fact load time
- retained heap after 200K rules
- processing wall time
- end-to-end transactions/sec
- cumulative JDBC read time across workers
- cumulative Drools time across workers
- cumulative JDBC write time across workers
- batch p50/p95/p99
- input row count
- result row count
- matched/no-match/incorrect counts
- H2 file size
- recurring monthly runtime path
- complete run including seed
- projected time for 2M transactions

Cumulative read/engine/write times are sums across parallel workers, so they may be larger than wall-clock time.

## Data and disk cleanup

Generated database files are stored under `data/` and are excluded from Git.

Delete the 200K benchmark database:

```bat
scripts\clean-h2-data.bat
```

Compact it after deleting or replacing many rows:

```bat
scripts\compact-h2.bat
```

Maven output can be removed with:

```bat
mvn clean
```

## Important limitations

This version is intentionally still a controlled performance POC:

- seeded transactions are generated to match exactly one rule
- no wildcard/ANY conditions yet
- no overlapping rules or priority resolution yet
- H2 is a local stand-in, not the expected production database
- rules and transactions are generated, but now persist in tables and can be inspected
- the result percentage is selected, not yet multiplied into a full commission amount

The next realism increment should introduce wildcard density, overlaps, deterministic priority/hit policy, skewed partitions and actual business-shaped import files.

For a complete cleanup of all generated POC data, including the optional 2M database:

```bat
scripts\clean-all-generated.bat
```
