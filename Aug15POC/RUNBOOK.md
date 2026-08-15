# CMS Payout Cycle POC - Detailed Runbook

This runbook assumes Windows because the supplied convenience scripts are `.bat` files. The same JAR/WAR commands can be used from Linux/macOS shells.

---

## 1. Prerequisites

Install:

1. **JDK 17 or later**
   - Target bytecode is Java 17.
   - JDK 17 or JDK 21 is fine.
2. **Maven 3.9+**
3. Internet access for the first Maven dependency download only.
4. A browser.

Verify:

```bat
java -version
mvn -version
```

Recommended memory for the first functional POC:

```text
16 GB+ machine: comfortable
8 GB machine: use one worker and/or one rule engine if needed
```

### Ports that must be free

| Port | Process |
|---:|---|
| 8080 | CMS JSP application |
| 8082 | H2 web console |
| 8085 | WORKER-01 |
| 8086 | WORKER-02 |
| 8091 | RE-01 |
| 8092 | RE-02 |
| 9093 | H2 TCP |
| 61616 | ActiveMQ Artemis |

On Windows you can check a port, for example:

```bat
netstat -ano | findstr :8080
```

---

# 2. Extract the project

Example:

```text
C:\work\cms-payout-cycle-poc
```

Open Command Prompt or PowerShell in that directory.

---

# 3. Build everything

From the project root:

```bat
mvn clean package
```

or:

```bat
scripts\00-build.bat
```

Expected deployables:

```text
poc-infrastructure\target\poc-infrastructure-1.0.0-SNAPSHOT.jar
cms-application\target\cms-application-1.0.0-SNAPSHOT.war
rule-engine-service\target\rule-engine-service-1.0.0-SNAPSHOT.jar
cms-batch-worker\target\cms-batch-worker-1.0.0-SNAPSHOT.jar
```

The CMS is intentionally packaged as an executable WAR because the POC UI uses JSP.

---

# 4. Recommended first run - start each process manually

For the first run, use separate terminals so you can see what each component is doing.

## Terminal 1 - Infrastructure

Run:

```bat
scripts\01-start-infrastructure.bat
```

Equivalent command:

```bat
java -jar poc-infrastructure\target\poc-infrastructure-1.0.0-SNAPSHOT.jar
```

This starts:

```text
H2 TCP       localhost:9093
H2 Web       http://localhost:8082
ActiveMQ     tcp://localhost:61616
```

Expected console message includes:

```text
CMS POC infrastructure is UP
```

The database is still a **file database**. The physical file will be:

```text
data\cms-poc.mv.db
```

The TCP process only makes that same file safely accessible to multiple POC JVMs.

---

## Terminal 2 - CMS application

Only after infrastructure is running:

```bat
scripts\02-start-cms.bat
```

Equivalent:

```bat
java -jar cms-application\target\cms-application-1.0.0-SNAPSHOT.war
```

CMS starts on:

```text
http://localhost:8080
```

On first startup it will:

1. connect to the file-backed H2 through TCP;
2. create/update the POC JPA tables;
3. let Camunda create/update its `ACT_*` tables;
4. deploy `cms-payout-cycle.bpmn`;
5. seed deterministic sample transaction rows up to the configured count.

Default:

```properties
poc.seed.transaction-count=50000
```

Wait until the CMS startup completes before starting workers on the first-ever run, because the CMS creates the POC schema.

Open:

```text
http://localhost:8080/login
```

Credentials:

```text
admin / admin
```

Do **not** start a payout cycle yet. Start rule engines and workers first.

---

## Terminal 3 - Rule Engine RE-01

```bat
scripts\03-start-rule-engine-1.bat
```

Equivalent:

```bat
java -jar rule-engine-service\target\rule-engine-service-1.0.0-SNAPSHOT.jar ^
  --server.port=8091 ^
  --rule-engine.node-id=RE-01
```

Verify:

```text
http://localhost:8091/api/v1/node
```

---

## Terminal 4 - Rule Engine RE-02

```bat
scripts\04-start-rule-engine-2.bat
```

Equivalent:

```bat
java -jar rule-engine-service\target\rule-engine-service-1.0.0-SNAPSHOT.jar ^
  --server.port=8092 ^
  --rule-engine.node-id=RE-02
```

Verify:

```text
http://localhost:8092/api/v1/node
```

Important: these are **two independent JVMs running the same rule-engine JAR**.

They do not read the CMS H2 database.

---

## Terminal 5 - Batch Worker WORKER-01

```bat
scripts\05-start-worker-1.bat
```

Equivalent:

```bat
java -jar cms-batch-worker\target\cms-batch-worker-1.0.0-SNAPSHOT.jar ^
  --server.port=8085 ^
  --cms.worker.id=WORKER-01
```

It connects to:

```text
ActiveMQ :61616
H2 TCP   :9093
RE nodes :8091,:8092
```

Verify:

```text
http://localhost:8085/api/v1/node
```

---

## Terminal 6 - Batch Worker WORKER-02

```bat
scripts\06-start-worker-2.bat
```

Equivalent:

```bat
java -jar cms-batch-worker\target\cms-batch-worker-1.0.0-SNAPSHOT.jar ^
  --server.port=8086 ^
  --cms.worker.id=WORKER-02
```

Verify:

```text
http://localhost:8086/api/v1/node
```

WORKER-01 and WORKER-02 are **competing consumers of the same JMS queue**.

The broker delegates each queue message to one available worker.

---

# 5. Verify the complete runtime before processing

In CMS open:

```text
Runtime Nodes
```

Expected:

```text
CMS             UP
H2-TCP          UP
ActiveMQ        UP
WORKER-01       UP
WORKER-02       UP
RE-01           UP
RE-02           UP
```

You can also run:

```bat
scripts\08-verify-nodes.bat
```

Smoke-test RE-01 directly:

```bat
scripts\09-smoke-rule-engine.bat
```

Expected result contains:

```text
"nodeId":"RE-01"
"matchedRule":"COMM-001"
"commissionRate":8.25
```

---

# 6. First end-to-end functional cycle

Go to:

```text
Payout Cycles
```

Use:

```text
Cycle period   current month
Transactions   5000
Batch size     1000
```

Press:

```text
GO - Start Cycle
```

## What should happen

Camunda starts one stage-level process instance.

### Commission

`CommissionWorkloadPartitioner.java`:

1. finds 5,000 eligible transaction IDs;
2. splits them into five batches of 1,000;
3. writes batch/batch-item records;
4. publishes five JMS messages containing transaction IDs.

Example conceptual message:

```json
{
  "cycleId": "...",
  "stageKey": "COMMISSION",
  "ruleSetKey": "COMMISSION_RULES",
  "batchId": "...-COMMISSION-0001",
  "batchNo": 1,
  "transactionIds": [1,2,3]
}
```

The real message contains all IDs for that batch, not full transaction rows.

### Workers

WORKER-01 and WORKER-02 compete for the five JMS messages.

For each message a worker:

1. loads the selected transaction rows from H2;
2. builds business input maps;
3. chooses a rule-engine node;
4. calls the rule-engine REST API;
5. writes the decision audit;
6. marks the batch complete.

### Rule-engine distribution

The POC node selector deliberately uses batch number:

```text
batch 1 -> RE-01
batch 2 -> RE-02
batch 3 -> RE-01
batch 4 -> RE-02
batch 5 -> RE-01
```

This makes the distribution easy to explain in the demo.

### Commission supervisor

`CommissionWorkloadSupervisor.java` checks:

```text
total batches
completed batches
failed batches
```

If not all five are complete, BPMN waits on a 1-second timer and invokes the supervisor again.

Only when all Commission batches are complete does the BPMN move to Tax.

Tax repeats the same pattern, and only after every Tax batch completes does Withholding start.

Then statement generation and cycle completion occur.

---

# 7. What to verify in the CMS UI

## Cycle Details

Open the cycle.

Verify:

- status changes from RUNNING to COMPLETED;
- stages appear one after another;
- every stage reaches 100%;
- Batch Execution table shows different workers;
- Batch Execution table shows RE-01 and RE-02.

The running cycle page refreshes periodically.

## Process Visualizer

Choose:

```text
View Process
```

Verify the stage progression and barrier behavior.

The POC includes a simple integrated process view. In the final CMS demo this action can link into the existing Camunda Process Visualizer while keeping the same CMS header/navigation.

## Transaction Journey

After completion, enter:

```text
1
```

in Transaction Journey.

Expected stages:

```text
COMMISSION
TAX
WITHHOLDING
```

Each row/card should show:

- named rule set;
- matched rule;
- rule-engine node;
- result JSON.

## Decision Audit

Open:

```text
Decision Audit
```

Search:

```text
Transaction ID = 1
```

Verify the same transaction has separate decision records for each business stage.

## Performance

Open:

```text
Performance
```

Verify:

- end-to-end cycle duration;
- queue wait per batch;
- CMS DB read time;
- REST round-trip time;
- rule-engine internal evaluation time;
- audit/DB write time;
- total batch time;
- worker IDs;
- rule-engine IDs;
- transaction count per RE node.

---

# 8. Recommended performance-oriented POC run

After the 5K functional run, start another cycle with:

```text
Transactions   50000
Batch size     5000
```

That means:

```text
10 Commission batches
10 Tax batches
10 Withholding batches
```

for 30 batch executions in total.

This is better for basic architectural throughput measurement.

Important: the supplied DMNs contain only a few sample rules. This run proves the **batch/distribution/orchestration architecture**, not the final 117K-rule DMN performance.

The realistic rule corpus/benchmark should be connected after the architecture is stable.

---

# 9. Compare one rule-engine node vs two

Keep both worker JVMs.

For a one-RE baseline, stop WORKER-01 and WORKER-02 and restart them with:

### WORKER-01

```bat
java -jar cms-batch-worker\target\cms-batch-worker-1.0.0-SNAPSHOT.jar ^
  --server.port=8085 ^
  --cms.worker.id=WORKER-01 ^
  --cms.rule-engine.nodes=http://localhost:8091
```

### WORKER-02

```bat
java -jar cms-batch-worker\target\cms-batch-worker-1.0.0-SNAPSHOT.jar ^
  --server.port=8086 ^
  --cms.worker.id=WORKER-02 ^
  --cms.rule-engine.nodes=http://localhost:8091
```

Run:

```text
50,000 / 5,000
```

Record the cycle duration.

Then restart both workers normally:

```text
scripts\05-start-worker-1.bat
scripts\06-start-worker-2.bat
```

Now both are configured with:

```text
http://localhost:8091,http://localhost:8092
```

Run another identical cycle.

Compare the two cycle durations in Performance.

Do not expect perfectly linear scaling on one laptop because the JVMs share the same physical CPU, memory and disk. The purpose is to prove distribution and collect real scaling evidence.

---

# 10. Fast start after you understand the processes

After a successful build:

```bat
scripts\07-start-all.bat
```

This opens all six roles in separate command windows.

Still verify the Runtime Nodes page before starting a cycle.

---

# 11. Stop the POC

For the first version, stop each terminal with:

```text
Ctrl+C
```

Recommended shutdown order:

```text
workers
rule engines
CMS
infrastructure last
```

Do not terminate infrastructure while CMS/workers are writing H2 data.

---

# 12. Reset the POC database

Stop every POC process first.

Then:

```bat
scripts\10-reset-poc-data.bat
```

or manually delete:

```text
data\
```

The next CMS startup recreates schema/sample data.

---

# 13. H2 console

Start infrastructure and CMS first.

Open:

```text
http://localhost:8082
```

Use:

```text
Driver Class: org.h2.Driver
JDBC URL:     jdbc:h2:tcp://localhost:9093/./cms-poc
User:         sa
Password:     <blank>
```

Useful tables:

```text
CMS_CYCLE
CMS_STAGE
CMS_BATCH
CMS_BATCH_ITEM
CMS_TRANSACTION
CMS_DECISION_AUDIT
```

Camunda:

```text
ACT_RU_*
ACT_HI_*
ACT_RE_*
ACT_GE_*
```

---

# 14. Useful configuration

CMS:

```text
cms-application/src/main/resources/application.properties
```

Worker:

```text
cms-batch-worker/src/main/resources/application.properties
```

Rule engine:

```text
rule-engine-service/src/main/resources/application.properties
```

Infrastructure:

```text
poc-infrastructure/src/main/resources/application.properties
```

### Change sample transaction count

```properties
poc.seed.transaction-count=50000
```

### Change rule-engine nodes seen by workers

```properties
cms.rule-engine.nodes=http://localhost:8091,http://localhost:8092
```

### Change JMS queue

```properties
cms.jms.batch-queue=cms.payout.batch.queue
```

---

# 15. Troubleshooting

## CMS says connection refused on port 9093

Cause:

```text
poc-infrastructure is not running.
```

Fix:

```bat
scripts\01-start-infrastructure.bat
```

before starting CMS.

## Worker cannot start because CMS tables do not exist

On the very first run, start CMS and let it finish schema creation before starting worker JVMs.

## Runtime Nodes shows RE-01/RE-02 DOWN

Start:

```bat
scripts\03-start-rule-engine-1.bat
scripts\04-start-rule-engine-2.bat
```

## Batches fail immediately

Look at:

```text
WORKER console
Rule Engine console
CMS Batch Execution table -> error status
```

Most likely causes:

- RE process not running;
- wrong port;
- rule-engine REST call failed.

Because the supervisor detects a failed batch, the sample BPMN marks that cycle failed rather than moving silently to the next stage.

## Port already in use

Example:

```bat
netstat -ano | findstr :8091
tasklist /FI "PID eq <pid>"
```

Either stop the conflicting process or launch the POC component on a different port and update the configured node list.

## H2 file appears locked

Do not connect CMS and workers directly using separate `jdbc:h2:file:` URLs.

This POC intentionally uses:

```text
jdbc:h2:tcp://localhost:9093/./cms-poc
```

for all JVMs while still storing data in a file.

---

# 16. Before moving into the existing CMS

Do not copy the Spring Boot launcher/configuration blindly.

Use `INTEGRATION_NOTES.md`.

The important reusable concepts are:

- BPMN stage design;
- Partitioner/Supervisor Java logic;
- JMS BatchCommand contract;
- worker flow;
- rule-engine REST contract;
- named rule-set keys;
- audit/batch model;
- transaction data-provider abstraction.

The integrated CMS remains Spring MVC + JSP + Oracle.
