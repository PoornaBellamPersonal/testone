# Integration Notes - From Standalone POC to Existing CMS

The standalone POC is a proving ground. The target client demo is the existing CMS, which is Spring MVC/JSP and Oracle-backed.

## What changes

| POC | Integrated CMS demo |
|---|---|
| Spring Boot CMS launcher | Existing Spring MVC CMS |
| POC JSP shell | Existing CMS JSP layout/navigation |
| In-memory `admin/admin` | Existing CMS authentication/security |
| H2 TCP/file | Existing Oracle datasource |
| `H2TransactionIdProvider` | Oracle/CMS sales-event provider |
| deterministic sample rows | selective realistic CMS data |
| local ActiveMQ | organisation/demo ActiveMQ/JMS configuration |
| localhost workers/REs | deployable processes/VMs as required |

## What should remain conceptually the same

### BPMN

Keep the stage-level process pattern:

```text
Partition -> asynchronous batches -> Supervisor barrier -> next stage
```

Do not create one Camunda process instance per transaction.

### Java stage classes

Reusable targets:

```text
CommissionWorkloadPartitioner
CommissionWorkloadSupervisor
TaxWorkloadPartitioner
TaxWorkloadSupervisor
WithholdingWorkloadPartitioner
WithholdingWorkloadSupervisor
```

They are Spring beans/Java delegates; they do not require the CMS itself to be Spring Boot.

In the existing Spring MVC application they can be discovered through the existing Spring application context/component scan or declared as beans.

### Transaction data provider

POC:

```java
public interface TransactionIdProvider {
    List<Long> findEligibleTransactionIds(int maximum);
}
```

Current implementation:

```text
H2TransactionIdProvider
```

For CMS create an Oracle-backed implementation that applies the real cycle/business filters against existing sales-event tables.

The BPMN delegate should not care whether the source is H2 or Oracle.

### Batch message

Keep IDs, not complete 300-column rows.

```json
{
  "cycleId": "...",
  "stageKey": "COMMISSION",
  "ruleSetKey": "COMMISSION_RULES",
  "batchId": "...",
  "batchNo": 1,
  "transactionIds": [1001,1002,1003]
}
```

### Worker boundary

Worker:

```text
JMS -> IDs -> CMS/Oracle DB read -> Rule Engine REST -> result persistence
```

Rule Engine:

```text
REST business inputs -> decision evaluation -> result
```

Rule Engine should not be given direct CMS Oracle access.

### Rule-engine nodes

The same rule-engine deployable should run multiple instances.

POC:

```text
RE-01 localhost:8091
RE-02 localhost:8092
```

Demo/production:

```text
RE-01 VM/node A
RE-02 VM/node B
...
```

The worker-facing contract remains unchanged.

## JSP migration

The POC views demonstrate the screens, not the final CMS visual shell.

When integrating:

1. use the existing CMS header/navigation/footer;
2. copy/adapt cycle list/detail sections;
3. link `View Process` directly to the existing Process Visualizer;
4. preserve the existing CMS security/session model;
5. keep the pages looking like native CMS pages.

## Database migration

Do not translate every H2 DDL statement mechanically.

First map:

```text
CMS_TRANSACTION
```

to existing CMS business tables.

New operational tables likely still needed:

```text
CMS_CYCLE
CMS_STAGE
CMS_BATCH
CMS_BATCH_ITEM (or a more scalable batch membership representation)
CMS_DECISION_AUDIT
```

For the real 2M+ transaction workload, review whether persisting every `CMS_BATCH_ITEM` is necessary. The POC stores it because it makes verification/retry/debugging straightforward.

## Spring Boot removal from CMS

The main reusable classes use normal Spring/JPA/JMS/Camunda APIs.

In the existing CMS:

- remove `CmsApplication`;
- use the CMS datasource;
- use the CMS transaction manager conventions;
- wire the delegate/services into the existing Spring context;
- configure JMS with the existing Spring MVC application's XML/Java config;
- deploy the BPMN through the CMS's existing Camunda deployment mechanism.

The separate rule-engine and batch-worker applications can remain Spring Boot if that is operationally acceptable; the main CMS does not have to become Boot.

## POC-to-demo sequence

1. Prove 5K end-to-end on one laptop.
2. Run 50K benchmark and capture batch/node timings.
3. Connect a larger/realistic DMN corpus.
4. Tune worker count, batch size and rule-engine JVM resources.
5. Integrate the BPMN/delegates into CMS.
6. Implement Oracle `TransactionIdProvider` / transaction loader.
7. Reuse CMS JSP shell.
8. Link existing Process Visualizer.
9. Run selective realistic business data through all stages.
10. Re-run performance tests in the demo environment.
