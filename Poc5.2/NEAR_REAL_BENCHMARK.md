# Near-real 300 / 300K / 3M benchmark checklist

## 1. Pre-flight

- Stop the showcase application.
- Confirm JDK 21 via `java -version` and `mvn -version`.
- Keep at least 10–15 GB free disk space for a comfortable POC run; actual H2 size depends on transaction density and persisted results.
- Close unrelated memory-heavy applications.

## 2. Seed

```bat
scripts\seed-near-real-300k-3m.bat
```

Expected logical shape:

- 300 attribute definitions
- 300,000 commission rules
- 60,000 logical programs
- ~12 conditions per rule on average
- 3,000,000 transaction rows
- ~3% no-match population

## 3. Start

```bat
scripts\run-showcase-large.bat
```

## 4. First benchmark

Use:

- workers = 8
- batch = 5000
- persist detailed results = No

Capture:

- total time
- processing time
- throughput
- engine compile/load time
- DB read time
- engine time
- heap
- average/max candidates
- incorrect count

## 5. Full persistence benchmark

Repeat with `Persist detailed results = Yes` and compare DB-write cost and H2 file growth.

## 6. RFP acceptance checks

- Incorrect = 0
- all deliberate no-match transactions remain no-match
- candidate population remains bounded
- SLA headroom is comfortably >1x
- 300 registered attributes visible in UI
- add attribute #301 without source/schema changes
- new attribute can be selected in Create Rule and Test / Explain
