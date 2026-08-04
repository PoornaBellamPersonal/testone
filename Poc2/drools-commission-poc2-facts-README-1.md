# Drools Commission POC #2 — Rules as Facts

This POC changes the model after POC #1 showed excellent transaction execution but unacceptable memory growth while compiling thousands of generated DRL rules.

## Goal

Benchmark:

- **200,000 commission rows**
- **24 business inputs**
- **2,000,000 transactions**
- completion comfortably inside **2 hours**

The key difference is that the 200K rows are now **`CommissionRule` facts/data**, not 200K compiled DRL rules.

## Architecture

```text
                         ONE generic compiled DRL rule
                                      |
                                      v
                              shared small KieBase
                                      |
              +-----------------------+-----------------------+
              |                       |                       |
              v                       v                       v
        Stateful session 0      Stateful session 1      ... session 31
        ~6,250 rule facts       ~6,250 rule facts           ~6,250 facts
              ^                       ^                       ^
              |                       |                       |
        routed tx batches       routed tx batches       routed tx batches
```

Each partition/session is owned by only one benchmark task at a time. Commission-rule facts remain resident; transaction facts are inserted in batches, rules are fired, and transaction facts are deleted before the next batch.

Drools is therefore compiling **one rule**, while matching transaction facts against business-rule facts at runtime.

## The 24 inputs

1. carrier
2. product
3. state
4. channel
5. agentType
6. registrationType
7. commissionType
8. transactionType
9. policyType
10. currency
11. planCode
12. producerLevel
13. customerSegment
14. paymentMode
15. renewal
16. replacement
17. specialProgram
18. internalBusiness
19. premium (range)
20. faceAmount (range)
21. productionAmount (range)
22. policyDurationMonths (range)
23. issueAge (range)
24. effectiveEpochDay (range)

POC #2 also scrambles deterministic rule IDs through a reversible 50-bit permutation before decoding dimensions, so high-order categorical columns vary across the 200K synthetic rows instead of remaining mostly constant.

## Why beta range indexing is enabled

The generic rule joins many `CommissionRule` facts with transaction facts. Exact equality constraints should narrow candidates first. The remaining premium/date/etc. conditions are range joins. Drools documents an optional beta/join-node range index for workloads with many facts being joined; the benchmark therefore exposes `--beta-range-index=true|false` so it can be measured rather than assumed.

## Prerequisites

- JDK 17+ (JDK 21 recommended)
- Maven 3.9.6+

## Run order

### 1. Smoke

```bat
scripts\run-facts-smoke.bat
```

Profile:

- 50K commission facts
- 100K transactions
- 8 partitions
- 4 threads
- `-Xmx4g`

### 2. Medium — memory decision point

```bat
scripts\run-facts-medium.bat
```

This deliberately loads the **full 200K commission rows** already:

- 200K commission facts
- 500K transactions
- 32 partitions
- 8 threads
- `-Xmx8g`

If this remains comfortably below the heap cap with zero incorrect results, the original POC #1 memory problem is effectively solved at the rule-representation level.

### 3. Full target

```bat
scripts\run-facts-target.bat
```

- 200K commission facts
- 2M transactions
- 32 partitions
- 8 threads
- 5K tx/batch
- `-Xmx10g`

### 4. Compare Drools beta range indexing

```bat
scripts\run-compare-range-index.bat
```

Do not assume the range index helps; compare the same workload with it enabled and disabled.

## Metrics

The benchmark reports separately:

- generic KieBase build time
- commission-fact loading time
- max partition load time
- heap after KieBase build + GC
- heap after all commission facts are resident + GC
- processing wall time
- transactions/sec
- batch p50/p95/p99
- matched/no-match/incorrect counts
- heap and non-heap after measured run + GC
- projected 2M duration

CSV history is written to `benchmark-results-facts.csv`.

## Important interpretation

This POC is testing whether Drools can serve as a **high-performance indexed matcher** when business rules are data. It is not yet a complete business-rules product.

A successful result must satisfy both:

1. **Memory:** 200K resident commission facts fit comfortably in the chosen heap.
2. **Execution:** 2M transactions have zero correctness errors and large headroom against 2 hours.

Later POCs should add wildcard/ANY fields, priority/override behavior, overlapping rules, actual Oracle/CSV import, rule-version hot swap, audit output, and comparison with a custom bitmap/indexed matcher using the same dataset.
