# CMS Cycle Processing POC V2

This POC implements the architecture agreed for CMS cycle processing:

- **CMS application**: Spring Boot + embedded Camunda 7 + file-backed H2 + admin UI + Kafka producer/consumers + configurable batch workers.
- **Rule Engine Service**: separate Spring Boot application with a generic REST contract. Its temporary implementation evaluates Camunda DMN decisions.
- **No DMN dependency exists in CMS.** CMS only knows `RuleEngineClient` and the generic batch request/response contract.
- **All application configuration is in `application.properties`.** There are no Spring `application.yml` files.

## End-to-end flow

```text
Admin logs into CMS
        |
        v
Cycle Processing tab -> parameters -> GO
        |
        v
Embedded Camunda 7 starts cmsCycleProcessing (businessKey = cycleId)
        |
        v
Seed/select transactions (POC uses synthetic rows)
        |
        v
Create logical batches (default 5000, configurable)
        |
        v
Transactional outbox rows
        |
        v
Kafka cms.batch.command.v1
        |
        +--> worker thread 1 --        +--> worker thread 2 ----> generic RuleEngineClient -> REST -> Rule Engine Service -> DMN
        +--> worker thread N --/
        |
        v
CMS saves results + stage progress
        |
        v
When all batches finish, CMS correlates a stage-completed message to Camunda
        |
        v
Next stage (Commission -> Tax -> Aggregate Limit)
```

## Why the worker is inside CMS in V2

For the POC, the worker is intentionally kept in the CMS deployable. `cms.worker.concurrency` controls the number of Kafka listener consumers/worker threads inside one CMS JVM. The worker code is isolated under `messaging`, `service`, and `rule`, so it can later be extracted to an independently scalable `cms-batch-worker` application.

Because this POC uses a file-backed H2 database, it demonstrates **vertical worker concurrency**, not multiple CMS JVMs writing to the same H2 file.

## Main configuration files

- `cms-application/src/main/resources/application.properties`
- `rule-engine-service/src/main/resources/application.properties`

Important CMS properties:

```properties
cms.batch.default-size=5000
cms.batch.max-size=20000
cms.worker.concurrency=4
cms.worker.max-attempts=3
cms.worker.retry-delay-ms=2000
cms.rule-engine.base-url=http://localhost:8090
cms.rule-engine.max-items-per-request=5000
cms.kafka.partitions=12
```

Change `cms.worker.concurrency` and restart CMS to compare 1, 2, 4, 8, etc. worker threads. Kafka topic partitions should be at least as high as the desired concurrency.

The **logical batch size is chosen per cycle in the admin UI**. The UI defaults it from `cms.batch.default-size`. The rule-engine HTTP micro-batch can be limited separately using `cms.rule-engine.max-items-per-request`.

## File H2 and existing process visualizer

CMS starts an H2 TCP server before Spring creates the DataSource. The physical database remains a file:

```text
./data/cms-poc.mv.db
```

CMS connects through:

```text
jdbc:h2:tcp://localhost:9093/cms-poc
```

Your existing visualizer can use the same JDBC URL while CMS is running. Credentials in the POC are `sa` with an empty password.

H2 uses **9093**, not 9092, because Kafka uses 9092.

Change these properties if required:

```properties
cms.h2.tcp.port=9093
cms.h2.base-dir=./data
cms.h2.database-name=cms-poc
cms.visualizer.base-url=http://localhost:8085
```

## Start Kafka

A local Kafka broker is required. For a quick local broker with Docker:

```bash
docker run --rm --name cms-kafka -p 9092:9092 apache/kafka:4.3.1
```

If your organisation already provides Kafka, simply change `spring.kafka.bootstrap-servers` in CMS `application.properties`.

CMS creates the command and DLT topics through Spring Kafka `NewTopic` beans.

## Build

Requirements:

- JDK 21
- Maven 3.9+
- Kafka on `localhost:9092`

POC dependency baseline: Camunda 7.24.0 and Spring Boot 3.5.5.

From the root:

```bash
mvn clean package
```

## Run

Start the rule engine first:

```bash
mvn -pl rule-engine-service spring-boot:run
```

Then start CMS:

```bash
mvn -pl cms-application spring-boot:run
```

Open:

```text
http://localhost:8080/login
```

POC login:

```text
admin / admin
```

Go to **Cycle Processing**, select parameters and press **GO**.

For an easy first run use:

```text
Cycle Type       TRIAL
Transactions     50000
Batch Size       5000
Worker Threads   4 (from application.properties)
```

That creates 10 logical batches per stage. The same 50,000 transactions then flow through Commission, Tax and Aggregate Limit stages.

## Camunda process

BPMN file:

`cms-application/src/main/resources/processes/cms-cycle-processing.bpmn`

The BPMN deliberately operates at **stage level**, not one process instance per transaction or batch. It waits at message catch events while Kafka workers process the high-volume work outside Camunda execution threads.

Messages used:

- `COMMISSION_STAGE_COMPLETED` / `COMMISSION_STAGE_FAILED`
- `TAX_STAGE_COMPLETED` / `TAX_STAGE_FAILED`
- `AGGREGATE_LIMIT_STAGE_COMPLETED` / `AGGREGATE_LIMIT_STAGE_FAILED`

A scheduler retries pending message correlations. This closes the race where a very fast worker might finish before the BPMN process has created its message subscription.

## CMS tables

- `CMS_CYCLE`
- `CMS_TRANSACTION`
- `CMS_BATCH`
- `CMS_BATCH_ITEM`
- `CMS_STAGE_PROGRESS`
- `CMS_OUTBOX_EVENT`

Camunda tables remain the standard `ACT_*` tables in the same H2 database.

The outbox is deliberate: creating a CMS batch and recording the command to be published happen in the same database transaction. A scheduled publisher sends pending commands to Kafka and marks them published.

Batch completion and the stage-completed counter are also updated in one database transaction. On CMS restart, any batch left in `PROCESSING` by the previous JVM is reset to `RETRY`, allowing Kafka redelivery to claim it again. These two details keep the POC from getting stuck at a stage wait state after a crash.

## Retry / DLT behavior

Technical failures from the remote rule engine are thrown by `RemoteRuleEngineClient`.

Kafka retries according to:

```properties
cms.worker.max-attempts=3
cms.worker.retry-delay-ms=2000
```

After retry exhaustion the original command is sent to `cms.batch.dlt.v1`. The DLT consumer marks the CMS batch/stage/cycle failed and the BPMN failure message is correlated.

Per-transaction DMN failures are returned as item results; they are not represented as an HTTP 500 for the entire batch.

## Rule engine service

Generic endpoint:

```http
POST /api/v1/rules/batch/evaluate
```

Example request:

```json
{
  "batchId": "COMM-00001",
  "ruleKey": "COMMISSION_RULES",
  "items": [
    {
      "transactionId": "TX-1",
      "inputs": {
        "channelType": "AGENCY",
        "amount": 20000
      }
    }
  ]
}
```

The current adapter maps rule keys to three DMNs:

- `COMMISSION_RULES`
- `TAX_RULES`
- `AGGREGATE_LIMIT_RULES`

When the other team's rule engine becomes available, CMS can keep the same `RuleEngineClient` boundary. Either replace `RemoteRuleEngineClient` to call the new API, or retain this service as an adapter and replace only its internal `RuleEvaluator`.

## POC benchmark knobs

The useful test matrix is:

```text
transaction count : 50k / 100k / 500k
logical batch size: 1000 / 2500 / 5000 / 10000
worker concurrency: 1 / 2 / 4 / 8
rule micro-batch   : 1000 / 2500 / 5000
```

Record stage elapsed time, total cycle elapsed time, rule-service request latency and failed/retried batches. This will show whether batch size, rule evaluation or worker concurrency is the limiting factor.

## POC notes

- Synthetic transaction generation is only to make the POC self-contained. Replace `TransactionRepository.seed(...)` with the real CMS transaction selection logic.
- The visualizer is not duplicated. CMS stores `processInstanceId` and uses `cycleId` as Camunda business key so the existing visualizer can locate the process.
- The POC does not attempt horizontal CMS scaling with H2. That becomes appropriate when the database is replaced by Oracle/PostgreSQL/etc. and the worker is extracted as a separate deployment.
