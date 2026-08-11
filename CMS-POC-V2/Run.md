Worked for 2m 19s

Yes. I would run this POC in a very deliberate sequence, because we want to verify not just that “the applications start,” but that the whole architecture works: Admin → Camunda → batch creation → outbox → Kafka → parallel workers → remote rule engine → DB updates → stage completion → Camunda continuation.

The current POC is configured for JDK 21, Maven 3.9+, Kafka on port 9092, CMS on 8080, rule engine on 8090, and file-backed H2 exposed over TCP on 9093. 

1. Extract and prepare the project

Extract:

Download V2 POC

You should have:

cms-cycle-processing-poc-v2/
│
├── pom.xml
├── README.md
├── cms-application/
├── rule-engine-service/
└── docs/

From a command prompt or PowerShell, verify:

java -version
mvn -version
docker --version

Use JDK 21 for this POC.

Because I could not execute Maven in my own runtime when I generated the project, I would treat the first mvn clean package as verification gate #1. If you get any compilation/dependency error, send me the complete first error and I will correct the project.


---

2. Start Kafka first

The POC expects:

spring.kafka.bootstrap-servers=localhost:9092

and creates two topics:

cms.batch.command.v1
cms.batch.dlt.v1

with 12 partitions by default. 

The simplest option is Docker:

docker pull apache/kafka:4.3.1

docker run --rm --name cms-kafka -p 9092:9092 apache/kafka:4.3.1

Apache Kafka's current 4.3 quickstart confirms that this official image can run directly on port 9092 with that command. 

Keep that terminal open.

In another terminal you can verify:

docker ps

You should see:

cms-kafka

running.


---

3. Build both applications

From the root folder:

cd cms-cycle-processing-poc-v2

mvn clean package

Expected result:

BUILD SUCCESS

and you should get approximately:

cms-application/
└── target/
    └── cms-application-1.0.0-SNAPSHOT.jar

rule-engine-service/
└── target/
    └── rule-engine-service-1.0.0-SNAPSHOT.jar

The POC README also defines mvn clean package as the root build command. 


---

4. Start the Rule Engine Service

Do this before starting CMS.

From the root:

mvn -pl rule-engine-service spring-boot:run

Alternatively, after successful packaging:

java -jar rule-engine-service\target\rule-engine-service-1.0.0-SNAPSHOT.jar

It should start on:

8090

The service is deliberately generic externally, while internally it currently maps three rule keys to the three DMN decisions. 

Verify health

Open:

http://localhost:8090/actuator/health

Expected:

{
  "status": "UP"
}


---

5. Test the Rule Engine independently

Do this before involving Kafka or Camunda. It isolates the rule-engine layer.

In PowerShell:

$body = @{
    batchId = "DEMO-BATCH-1"
    ruleKey = "COMMISSION_RULES"
    items = @(
        @{
            transactionId = "TX-1"
            inputs = @{
                channelType = "AGENCY"
                amount = 20000
            }
        },
        @{
            transactionId = "TX-2"
            inputs = @{
                channelType = "DIRECT"
                amount = 5000
            }
        }
    )
} | ConvertTo-Json -Depth 10

Invoke-RestMethod `
    -Method Post `
    -Uri "http://localhost:8090/api/v1/rules/batch/evaluate" `
    -ContentType "application/json" `
    -Body $body

The endpoint is:

POST /api/v1/rules/batch/evaluate

and the project already contains an equivalent request in docs/demo-requests.http. 

For the first transaction:

AGENCY
amount = 20,000

the commission DMN should calculate approximately:

commissionAmount = 1400

because that sample DMN applies a 7% commission to an agency transaction above 10,000.

At this point we have verified:

REST
  ↓
RuleEvaluationService
  ↓
DMN engine
  ↓
DMN result

independently of CMS.


---

6. Start CMS

Open another terminal:

mvn -pl cms-application spring-boot:run

or:

java -jar cms-application\target\cms-application-1.0.0-SNAPSHOT.jar

CMS should start on:

8080

The configured POC credentials are:

admin
admin

and the CMS is configured to start its embedded Camunda 7 engine, enable job execution, deploy BPMN automatically, and use AUDIT history. 

What to verify in the CMS startup log

You should see evidence of four things:

Spring Boot started

H2 TCP server started on 9093

Camunda process engine started / BPMN deployed

Kafka consumers started

Also check:

http://localhost:8080/actuator/health


---

7. Verify the H2 file

The CMS application uses a real file-backed database:

./data/cms-poc.mv.db

and exposes it through:

jdbc:h2:tcp://localhost:9093/cms-poc

with:

username = sa
password = <empty>

That configuration exists specifically so your separate process visualizer can read the same live Camunda database while CMS is running. 

After CMS starts, verify that this exists:

cms-cycle-processing-poc-v2/
└── data/
    └── cms-poc.mv.db

Your process visualizer should connect with:

jdbc:h2:tcp://localhost:9093/cms-poc

The POC has both the standard Camunda ACT_* tables and application tables such as CMS_CYCLE, CMS_TRANSACTION, CMS_BATCH, CMS_BATCH_ITEM, CMS_STAGE_PROGRESS, and CMS_OUTBOX_EVENT. 


---

8. Start your first CMS cycle

Open:

http://localhost:8080/login

Login:

admin / admin

Go to:

Cycle Processing

For the first test, I strongly recommend:

Parameter	Value

Cycle Type	TRIAL
Transaction Count	50,000
Batch Size	5,000
Description	First CMS Batch POC
Data Filter	ALL
Worker concurrency	4


Worker concurrency comes from:

cms.worker.concurrency=4

while batch size is selected per cycle and defaults to 5000. 

Press:

GO


---

9. First thing to verify after pressing GO

The system should generate a cycle ID similar to:

CMS-20260811-103245-A12F

That same value becomes:

CMS_CYCLE.CYCLE_ID
        =
Camunda businessKey

This is particularly useful for your visualizer.

The process definition being started is:

cmsCycleProcessing

and its BPMN is explicitly designed to operate at stage level, waiting for messages between Commission, Tax, and Aggregate Limit processing. 


---

10. What should happen for 50,000 / 5,000?

This is the most useful deterministic test.

You entered:

Transactions = 50,000
Batch Size   = 5,000

Therefore:

50,000 / 5,000 = 10 batches

Commission stage

Expected:

COMMISSION

10 total batches
0 → 10 completed
0 failed

Then Camunda receives:

COMMISSION_STAGE_COMPLETED

and continues.

Tax stage

It should then create:

TAX

10 batches

and eventually send:

TAX_STAGE_COMPLETED

Aggregate-limit stage

Then:

AGGREGATE_LIMIT

10 batches

followed by:

AGGREGATE_LIMIT_STAGE_COMPLETED

The BPMN explicitly contains the completed and failed message pairs for each of those three stages. 

So for the complete successful cycle you should ultimately see:

3 stages
×
10 batches

= 30 successfully processed logical batches


---

11. Watch the BPMN in your process visualizer

This is where your existing visualizer becomes very useful.

Immediately after GO, find the process by:

businessKey = <your CMS cycle ID>

You should see the process move roughly through:

Initialize Cycle Data
        ↓
Create & Queue Commission Batches
        ↓
Wait for Commission Stage

While workers are processing commission batches, Camunda should not be processing 50,000 records. It should be sitting at the Commission wait state.

The BPMN uses an event-based gateway and separate completed/failed message catch events for Commission. 

After all ten commission batches finish:

Commission Stage Completed
        ↓
Create & Queue Tax Batches
        ↓
Wait for Tax Stage

Then:

Tax Stage Completed
        ↓
Create & Queue Aggregate Limit Batches
        ↓
Wait for Aggregate Limit Stage

and finally:

Complete Cycle
        ↓
Cycle Completed

That complete stage progression is present directly in the BPMN. 

This is one of the most important architectural validations of the entire POC:

Camunda = orchestration

NOT

Camunda = transaction-processing engine


---

12. Watch CMS logs while processing

With:

cms.worker.concurrency=4

you should see multiple worker threads handling different batches.

Look for log lines conceptually similar to:

Worker received CMS-...-COMMISSION-00001 with 5000 items
Worker received CMS-...-COMMISSION-00002 with 5000 items
Worker received CMS-...-COMMISSION-00003 with 5000 items
Worker received CMS-...-COMMISSION-00004 with 5000 items

The POC uses configurable Kafka listener concurrency, and the topic is configured with 12 partitions specifically so tests such as concurrency 1/2/4/8 are meaningful. 


---

13. Verify the CMS database

These queries are extremely useful.

Substitute your cycle ID:

SELECT *
FROM CMS_CYCLE
WHERE CYCLE_ID = 'CMS-20260811-XXXXXX-XXXX';

During execution expect approximately:

STATUS        RUNNING
CURRENT_STAGE COMMISSION

then:

CURRENT_STAGE TAX

then:

CURRENT_STAGE AGGREGATE_LIMIT

and finally:

STATUS        COMPLETED
CURRENT_STAGE COMPLETED

Verify stage progress

SELECT
    STAGE,
    TOTAL_BATCHES,
    COMPLETED_BATCHES,
    FAILED_BATCHES,
    STATUS,
    CORRELATED,
    MESSAGE_NAME
FROM CMS_STAGE_PROGRESS
WHERE CYCLE_ID = 'your-cycle-id'
ORDER BY STAGE;

At the end of the 50K/5K happy-path test, expect conceptually:

STAGE             TOTAL   COMPLETED   FAILED   STATUS
--------------------------------------------------------
COMMISSION          10       10         0      COMPLETED
TAX                 10       10         0      COMPLETED
AGGREGATE_LIMIT     10       10         0      COMPLETED

And:

CORRELATED = TRUE

for all three stages.

That corresponds to the design where the stage completion event is correlated back to Camunda after all batches finish. 


---

14. Verify individual batches

Run:

SELECT
    STAGE,
    STATUS,
    COUNT(*) AS BATCH_COUNT
FROM CMS_BATCH
WHERE CYCLE_ID = 'your-cycle-id'
GROUP BY STAGE, STATUS
ORDER BY STAGE, STATUS;

Successful completion should show:

AGGREGATE_LIMIT   COMPLETED   10
COMMISSION        COMPLETED   10
TAX               COMPLETED   10

Then inspect actual workers:

SELECT
    BATCH_ID,
    STAGE,
    BATCH_NUMBER,
    ITEM_COUNT,
    STATUS,
    ATTEMPT_COUNT,
    WORKER_ID,
    STARTED_AT,
    COMPLETED_AT
FROM CMS_BATCH
WHERE CYCLE_ID = 'your-cycle-id'
ORDER BY STAGE, BATCH_NUMBER;

Every normal batch should have:

ITEM_COUNT = 5000
STATUS     = COMPLETED

except possibly the final batch when transaction count is not exactly divisible by batch size.


---

15. Verify the transactional outbox

Run:

SELECT
    STATUS,
    COUNT(*) AS CNT
FROM CMS_OUTBOX_EVENT
GROUP BY STATUS;

After everything has been successfully sent to Kafka, you ideally want:

PUBLISHED   30

for this test.

The POC intentionally persists the batch command in the outbox before the scheduler publishes it to Kafka, rather than directly coupling database updates and Kafka sends. 

This proves:

Create DB batch
      +
Create outbox record

same DB transaction

followed asynchronously by:

Outbox
   ↓
Kafka


---

16. Verify transaction results

First:

SELECT COUNT(*)
FROM CMS_TRANSACTION
WHERE CYCLE_ID = 'your-cycle-id';

Expected:

50000

Then:

SELECT
    COUNT(*) AS TOTAL,
    COUNT(COMMISSION_AMOUNT) AS COMMISSION_DONE,
    COUNT(TAX_AMOUNT) AS TAX_DONE,
    COUNT(LIMIT_STATUS) AS LIMIT_DONE
FROM CMS_TRANSACTION
WHERE CYCLE_ID = 'your-cycle-id';

Happy path expectation:

TOTAL            50000
COMMISSION_DONE  50000
TAX_DONE         50000
LIMIT_DONE       50000

Then:

SELECT LIMIT_STATUS, COUNT(*)
FROM CMS_TRANSACTION
WHERE CYCLE_ID = 'your-cycle-id'
GROUP BY LIMIT_STATUS;

You should see both:

APPROVED
REVIEW

because the current sample aggregate-limit DMN deliberately produces both outcomes.


---

17. Verify Camunda directly from the DB

Because history is configured as:

camunda.bpm.history-level=audit

you can inspect the process historically as well. 

For example:

SELECT
    ID_,
    PROC_INST_ID_,
    BUSINESS_KEY_,
    PROC_DEF_ID_,
    START_TIME_,
    END_TIME_
FROM ACT_HI_PROCINST
WHERE BUSINESS_KEY_ = 'your-cycle-id';

During execution:

END_TIME_ = NULL

After success:

END_TIME_ != NULL

You can also inspect active executions:

SELECT
    ID_,
    PROC_INST_ID_,
    ACT_ID_,
    IS_ACTIVE_
FROM ACT_RU_EXECUTION
WHERE PROC_INST_ID_ = 'your-process-instance-id';

And active message subscriptions:

SELECT
    EVENT_TYPE_,
    EVENT_NAME_,
    PROC_INST_ID_,
    ACTIVITY_ID_
FROM ACT_RU_EVENT_SUBSCR
WHERE PROC_INST_ID_ = 'your-process-instance-id';

While Commission is running, you should see Commission-related message subscriptions. Then those should disappear and Tax-related subscriptions appear.

That is a particularly strong proof that workers and Camunda are asynchronously decoupled.


---

18. Verify worker scaling

After the happy-path run, this should be your second major test.

Change:

cms.worker.concurrency=1

Restart CMS.

Run:

Transactions = 50,000
Batch Size   = 5,000

Record elapsed time.

Then:

cms.worker.concurrency=2

Repeat.

Then:

cms.worker.concurrency=4

Repeat.

Then:

cms.worker.concurrency=8

Repeat.

The POC explicitly supports this comparison, and the project documentation recommends 1/2/4/8 worker tests. 

Make a table such as:

Transactions	Batch	Workers	Commission	Tax	Limit	Total

50K	5000	1				
50K	5000	2				
50K	5000	4				
50K	5000	8				


This will quickly show whether worker parallelism actually improves throughput or whether the remote rule engine becomes the bottleneck.


---

19. Verify batch-size configurability

Keep:

cms.worker.concurrency=4

Then run separate cycles.

For 50,000 transactions:

Batch Size 1000  → 50 batches/stage
Batch Size 2500  → 20 batches/stage
Batch Size 5000  → 10 batches/stage
Batch Size 10000 → 5 batches/stage

This is arguably more useful than testing only the architect's proposed 5000.

The configured maximum is currently:

cms.batch.max-size=20000

and the HTTP call to the rule engine can independently be limited with:

cms.rule-engine.max-items-per-request=5000

so CMS logical batch size and rule-engine HTTP micro-batch size are independent controls. 

For example:

CMS batch size = 10,000

Rule max items/request = 5,000

should result internally in:

CMS Batch #1
     │
     ├── Rule HTTP request 1 → 5000
     └── Rule HTTP request 2 → 5000

That is a very important capability to demonstrate.


---

20. Failure test: stop the Rule Engine

This should absolutely be part of the POC demonstration.

Start a cycle:

50,000 transactions
5,000 batch
4 workers

While Commission is processing, stop:

rule-engine-service

CMS is configured with:

cms.worker.max-attempts=3
cms.worker.retry-delay-ms=2000

After retries are exhausted, the batch should be sent to:

cms.batch.dlt.v1

and the DLT consumer should mark the batch/stage/cycle failed. 

Verify:

SELECT *
FROM CMS_BATCH
WHERE CYCLE_ID = 'your-cycle-id'
  AND STATUS = 'FAILED';

Then:

SELECT *
FROM CMS_STAGE_PROGRESS
WHERE CYCLE_ID = 'your-cycle-id';

You should eventually see:

STATUS = FAILED
FAILED_BATCHES > 0

And Camunda should take:

COMMISSION_STAGE_FAILED

rather than:

COMMISSION_STAGE_COMPLETED

The BPMN explicitly models both outcomes. 

Then the process goes to:

Mark Cycle Failed
      ↓
Cycle Failed

rather than progressing to Tax. 

That is an excellent architecture-demo scenario.


---

21. Failure test: restart CMS

Another important scenario:

Start cycle
    ↓
Batches processing
    ↓
Kill CMS abruptly
    ↓
Restart CMS

The POC is intended to reset batches left in:

PROCESSING

back to:

RETRY

on restart, while the persistent H2 database preserves the cycle/batch state. The project documentation specifically includes this recovery behavior. 

This tests that the architecture isn't relying only on Java memory.


---

22. One current POC limitation to keep in mind

For this version:

Transaction Count

actually controls how many synthetic POC transactions are seeded.

Parameters such as:

Cycle Type
Effective Date
Data Cut-off
Data Filter
Description

are captured as cycle/process metadata, but we have not yet implemented the real CMS transaction-selection query driven by those fields.

That is intentional for the POC.

The next production-oriented step would replace:

Generate 50,000 synthetic transactions

with:

SELECT eligible CMS transactions
WHERE
 effective-date...
 AND cutoff...
 AND cycle-type...
 AND configured data-filter...

The POC documentation explicitly identifies the current input as synthetic data rather than the eventual real CMS query. 


---

Recommended acceptance checklist

I would consider V2 successfully demonstrated only when all of these pass:

mvn clean package succeeds.

Kafka starts on 9092.

Rule Engine health is UP on 8090.

Direct COMMISSION_RULES REST evaluation works.

CMS starts on 8080.

cms-poc.mv.db is physically created.

Your existing visualizer can connect through H2 TCP 9093.

Admin can login and press GO.

Exactly one Camunda cycle process instance is created with businessKey = cycleId.

50K/5K creates exactly 10 batches for Commission.

Four configured worker threads process batches concurrently.

Camunda waits rather than processing individual transactions.

Commission completion correlates back to Camunda.

Tax creates another 10 batches.

Aggregate Limit creates another 10.

All 50K transactions receive commission, tax and limit results.

All 30 outbox events ultimately become published.

All three CMS_STAGE_PROGRESS rows become COMPLETED.

All three stage completion messages are correlated.

CMS_CYCLE becomes COMPLETED.

Camunda process history shows an end time.

1/2/4/8-worker tests demonstrate the effect of concurrency.

1000/2500/5000/10000 batch-size tests demonstrate the effect of batching.

Stopping the rule service demonstrates retry → DLT → BPMN failure path.

Restarting CMS during processing demonstrates recovery.


If these work, we have proved far more than “Kafka integration.” We have demonstrated the core architecture:

CMS Admin
                  │
                  ▼
          Embedded Camunda 7
                  │
             orchestration
                  │
                  ▼
            CMS Batch DB
                  │
                Outbox
                  │
                  ▼
                Kafka
          ┌───────┼───────┐
          ▼       ▼       ▼
       Worker   Worker   Worker
          │       │       │
          └───────┼───────┘
                  ▼
          Rule Engine REST
                  │
                  ▼
              DMN today
                  │
                  ▼
              Results
                  │
                  ▼
             CMS/H2 DB
                  │
          Stage completed
                  │
                  ▼
          Resume Camunda

That is the architecture we ultimately want to present to the architect/team, rather than merely showing that 5,000 records can be sent to a rule engine.
