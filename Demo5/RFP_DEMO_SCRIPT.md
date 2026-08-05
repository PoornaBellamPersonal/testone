# 15-minute RFP Demo Script

## Before the meeting

For a fast UI rehearsal:

```bat
scripts\seed-quick.bat
scripts\run-showcase.bat
```

For the scale demonstration, pre-seed the target dataset before the meeting:

```bat
scripts\seed-target-200k-2m.bat
scripts\run-showcase.bat
```

Open `http://localhost:8080/`.

## 1. Management dashboard — 2 minutes

Show:

- active monthly rule version
- 220 registered attributes
- active rule/condition counts
- persisted transaction volume
- latest execution, throughput and 2-hour SLA headroom

Message: **the business requirement is visible as measurable data, not hidden in console logs.**

## 2. Monthly attribute change — 2 minutes

Open **Attributes** and add:

- Code: `NEW_CAMPAIGN_SCORE`
- Display name: `New Campaign Score`
- Type: `DECIMAL`
- Source key: `NEW_CAMPAIGN_SCORE`

Message: **no Java class, DRL, or rule-table DDL changed.**

## 3. Business rule authoring — 2 minutes

Open **Create Rule** and create a small rule using the newly added attribute plus existing attributes.

Message: **a rule contains only the attributes it uses; the platform can have 200+ attributes without a 200-column rule object.**

## 4. Test and explain — 3 minutes

Open **Test / Explain**, enter values that match a seeded rule or the new demo rule, then evaluate.

Show:

- commission percentage
- winning rule
- candidate count
- number of matching rules
- priority / specificity
- evaluation latency
- condition-by-condition explanation

Message: **business users can understand why a commission was selected.**

## 5. Metrics-enabled batch — 3 minutes

Open **Batch Execution** and start the persisted dataset.

Show live:

- processed transactions
- current throughput
- matched / no-match / ambiguous / incorrect
- progress bar

When complete show:

- engine compile/load time
- DB read time
- engine evaluation time
- DB write time
- wall-clock processing time
- candidate reduction
- heap usage
- SLA headroom

Message: **performance is an auditable application feature, not a one-off benchmark claim.**

## 6. Versioning and architecture — 3 minutes

Open **Rule Versions** and show monthly version intent. Then open **Architecture**.

Message:

`dynamic metadata -> sparse conditions -> compiled indexes -> candidate reduction -> deterministic winner -> auditable result`

## Close

The RFP POC answers five questions:

1. Can attributes change monthly without code changes? **Yes.**
2. Can business users configure rules? **Yes.**
3. Can the platform explain the selected commission? **Yes.**
4. Can 200K rules / 2M transactions be measured against the SLA? **Yes.**
5. Can architects see where time and memory are spent? **Yes.**
