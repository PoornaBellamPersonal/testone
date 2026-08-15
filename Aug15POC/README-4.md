# CMS Payout Cycle POC

A standalone, single-machine technical POC for the CMS payout-cycle architecture discussed for the client demo.

## Purpose

This project is intentionally **not the final CMS application**. It is a reference implementation used to prove that the major parts work together with deterministic sample transaction data:

- JSP + Bootstrap user interface
- embedded Camunda 7 payout BPMN
- Java workload partitioner and supervisor classes
- file-backed H2 for the POC
- ActiveMQ Artemis / JMS batching
- separate CMS batch-worker JVMs
- separate rule-engine JVMs
- named DMN rule sets
- sequential stage barriers
- transaction journey / decision audit
- runtime-node visibility
- performance / batch timing visibility (queue, DB read, REST, engine, DB write, total)

The later integrated client demo should move the reusable concepts into the **existing Spring MVC CMS**, use its JSP shell/security, replace the POC H2 data provider with **Oracle**, and use selective realistic CMS data.

## Runtime shape on one laptop

```text
Browser
  |
  v
CMS application :8080
Spring Boot POC + JSP + Bootstrap + embedded Camunda 7
  |
  | Java Commission/Tax/Withholding Workload Partitioner
  v
ActiveMQ Artemis :61616
cms.payout.batch.queue
  |
  +---------------------+
  |                     |
  v                     v
WORKER-01 :8085      WORKER-02 :8086
  |                     |
  | load transaction rows from H2
  | build REST request
  +----------+----------+
             |
        deterministic batch routing
             |
       +-----+------+
       |            |
       v            v
    RE-01 :8091  RE-02 :8092
    Camunda DMN  Camunda DMN
    no CMS DB    no CMS DB

File H2:
./data/cms-poc.mv.db
exposed to CMS + workers through H2 TCP :9093
```

## Maven modules

| Module | Responsibility |
|---|---|
| `common-contracts` | JMS and rule-engine REST DTOs |
| `cms-data` | Shared POC JPA entities/repositories |
| `poc-infrastructure` | H2 TCP/web server + embedded ActiveMQ Artemis broker |
| `rule-engine-service` | Stateless DMN REST service; run the same JAR as RE-01 and RE-02 |
| `cms-batch-worker` | JMS consumer; loads transactions; calls rule engine; saves audit/results |
| `cms-application` | JSP CMS POC + embedded Camunda + payout BPMN + partitioner/supervisor classes |

## Important boundary

The rule-engine module has **no CMS datasource dependency**. The flow is:

```text
JMS BatchCommand with transaction IDs
        |
        v
CMS Batch Worker
        |
        +--> read CMS/H2 transaction rows
        |
        +--> POST resolved business inputs to Rule Engine
        |
        +<-- decisions
        |
        +--> persist batch status + decision audit
```

This is deliberate. It maps cleanly to the future Oracle-backed CMS.

## Quick start

See [RUNBOOK.md](RUNBOOK.md) for detailed instructions.

At a minimum:

```bat
scripts\00-build.bat
scripts\07-start-all.bat
```

Then open:

```text
http://localhost:8080/login
```

Credentials:

```text
admin / admin
```

Recommended first cycle:

```text
Transactions: 5000
Batch size:   1000
```

That creates five batches in each rule stage, making worker and rule-engine distribution visible.

## POC BPMN

`cms-application/src/main/resources/processes/cms-payout-cycle.bpmn`

Main path:

```text
Start
 -> Initialize Cycle
 -> Commission Workload Partitioner.java
 -> Commission Workload Supervisor.java
      -> wait/recheck until every Commission batch completes
 -> Tax Workload Partitioner.java
 -> Tax Workload Supervisor.java
      -> wait/recheck until every Tax batch completes
 -> Withholding Workload Partitioner.java
 -> Withholding Workload Supervisor.java
      -> wait/recheck until every Withholding batch completes
 -> Generate Statements
 -> Complete Cycle
```

A 1-second BPMN timer is used between supervisor checks. No Camunda execution thread is blocked while batch work is running.

## Java files requested by the architecture

Concrete stage classes are intentionally visible:

```text
CommissionWorkloadPartitioner.java
CommissionWorkloadSupervisor.java
TaxWorkloadPartitioner.java
TaxWorkloadSupervisor.java
WithholdingWorkloadPartitioner.java
WithholdingWorkloadSupervisor.java
```

They reuse:

```text
AbstractWorkloadPartitioner.java
AbstractWorkloadSupervisor.java
```

This keeps the BPMN/business terminology explicit while avoiding duplicated implementation.

## POC database

Physical file:

```text
./data/cms-poc.mv.db
```

CMS and worker JVMs do not open the same file directly. They connect through H2 TCP:

```text
jdbc:h2:tcp://localhost:9093/./cms-poc
```

H2 browser console:

```text
http://localhost:8082
```

JDBC URL in the console:

```text
jdbc:h2:tcp://localhost:9093/./cms-poc
```

User:

```text
sa
```

Password: blank.

## Sample rule sets

- `COMMISSION_RULES`
- `TAX_RULES`
- `WITHHOLDING_RULES`

Both RE-01 and RE-02 run the **same JAR** and therefore load the same DMN definitions.

## What is intentionally not in this increment

- Kafka
- Docker / Kubernetes
- Oracle
- client authentication/SSO
- production RBAC/maker-checker
- production HA/XA transaction design
- service discovery/API gateway
- 117K-rule realistic DMN corpus
- full merge of the earlier rich Rule Management UI

The current POC proves orchestration, messaging, batching, worker distribution, rule-engine distribution, stage barriers, audit and end-to-end execution. See `INTEGRATION_NOTES.md` for the path into the existing CMS.
