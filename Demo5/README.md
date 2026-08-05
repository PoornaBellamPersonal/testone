# Commission Rules RFP Showcase — POC #5

A single Spring Boot + Thymeleaf application focused on the commission-processing RFP.

## What this version proves

- 200+ registered business attributes without a fixed Java field per attribute.
- New monthly attributes can be added as metadata through the UI.
- Rules are sparse: each rule stores only the attributes it uses.
- The primary runtime is a compiled `SparseIndexedCommissionEngine`, not a 200-column `if` chain.
- Equality/IN conditions are compiled into bitmap-like `BitSet` indexes; residual conditions (ranges, comparisons, null checks) are evaluated only on the reduced candidate set.
- Deterministic winner policy: priority > specificity > lowest rule ID.
- File-based H2 stores attributes, versions, sparse rules, transactions, execution metrics and optional detailed results.
- Thymeleaf UI demonstrates attribute management, rule browsing/creation, test/explain, rule versions, batch execution and metrics.
- Actuator health/metrics are enabled.

## Technology

- JDK 21
- Spring Boot 3.5.16
- Spring MVC + Thymeleaf
- Spring JDBC
- File H2
- Micrometer / Actuator
- No React build and no external UI runtime

> Spring Boot 3.5.16 is intentionally used for continuity with the current Spring 6/JDK 21 POC line. It is the final OSS 3.5.x release; a later productionization step should evaluate Boot 4.x.

## Important modeling decision

`ATTRIBUTE_DEFINITION` is the schema registry. `COMMISSION_RULE` stores rule metadata and `RULE_CONDITION` stores only the conditions a rule actually uses.

Example:

```
Rule 10852
  CARRIER       EQ       ABC
  PRODUCT       EQ       TERM
  PREMIUM       BETWEEN  100000 500000
  PRODUCER_TIER EQ       GOLD
  ATTR_217      EQ       SPECIAL
  -> commission 3.25%
```

The other ~215 registered attributes do not appear on the rule.

## Transaction representation in this POC

`TRANSACTION_RECORD.ATTRIBUTES_PAYLOAD` is a compact processing projection (`attributeId=value|...`) rather than a fixed 220-column Java object. This deliberately demonstrates that source columns can change without recompiling the engine. In a client Oracle implementation, a source adapter can project the attributes needed by the active rule version from the client's wide source table.

## Start with a quick demo

Prerequisites: JDK 21 and Maven 3.9+.

```bat
scripts\seed-quick.bat
scripts\run-showcase.bat
```

Open:

- http://localhost:8080/
- H2 console: http://localhost:8080/h2-console
- Actuator health: http://localhost:8080/actuator/health

H2 URL:

```
jdbc:h2:file:./data/rfp-showcase
user: sa
password: <blank>
```

## Scale profiles

### Quick

```bat
scripts\seed-quick.bat
```

- 5K rule rows
- 50K transactions
- 220 attributes

### Medium

```bat
scripts\seed-medium.bat
```

- 50K rule rows
- 200K transactions
- 220 attributes

### RFP target

```bat
scripts\seed-target-200k-2m.bat
```

- 200K sparse rule rows
- 2M persisted transactions
- 220 registered attributes
- 40K logical rule groups, five competing variants per group
- 5% deliberate no-match transactions
- priority overrides and exact ties for ambiguity metrics

Then start the UI:

```bat
scripts\run-showcase.bat
```

Go to **Batch Execution**, select workers/batch size, and start the metrics-enabled run.

## UI demo flow (about 15 minutes)

1. **Dashboard** — show active version, 220 attributes, rule/condition/transaction counts, last SLA run.
2. **Attributes** — add `NEW_CAMPAIGN_SCORE`. No code or table alteration is required.
3. **Create Rule** — choose the new attribute from the dropdown and save a sparse rule.
4. **Test / Explain** — enter a transaction, show candidate count, winning rule and condition-by-condition explanation.
5. **Batch Execution** — run the persisted dataset and show live processed/matched/no-match/ambiguous/error metrics.
6. **Execution Detail** — show DB read, engine evaluation, DB write, processing wall time, throughput, heap and SLA headroom.
7. **Rule Versions** — clone the active monthly version and show immutable versioning intent.
8. **Architecture** — show the metadata → sparse rules → indexed matcher → results flow.

## Pages

| Page | Purpose |
|---|---|
| `/` | Management dashboard |
| `/attributes` | Dynamic attribute registry |
| `/rules` | Paginated rule explorer + CSV import/export |
| `/rules/new` | Business-friendly sparse rule editor |
| `/tester` | Evaluate one transaction with explanation |
| `/executions` | Start benchmark and view run history |
| `/executions/{id}` | Live/final execution metrics |
| `/versions` | Rule-set versions |
| `/architecture` | Architect-facing explanation |

## Metrics captured per execution

- rule count and attribute count
- transaction count / processed
- matched / no-match / ambiguous / incorrect
- average and maximum candidate count
- engine compile/load time
- cumulative DB read time
- cumulative rule evaluation time
- cumulative DB write time
- processing wall time and total wall time
- throughput
- heap after engine initialization
- SLA seconds and headroom (shown on dashboard)

## RFP scope intentionally not included yet

- enterprise SSO/RBAC
- maker/checker approval workflow
- Oracle-specific tuning
- distributed execution / Kafka / Flink
- multi-tenancy
- production XLSX rule designer
- Drools adapter for complex chained rules

Those are extension points after the RFP architecture is accepted.

## Notes for the demo dataset

The generator makes five rule variants per logical program: exact, two sparse/wildcard variants, broad range, and either priority override / exact tie / fallback. Transactions are generated from the logical program, with 5% intentionally made no-match. `EXPECTED_RULE_ID` is retained only for POC correctness verification.

## Clean generated data

```bat
scripts\clean-data.bat
mvn clean
```
