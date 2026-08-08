Exception and Error Handling in BPMN with Camunda 7

1. Training Objectives

By the end of this session, participants should be able to:

- distinguish a business error from a technical exception
- decide whether something should be represented as:
  - normal process flow
  - BPMN Error
  - Java exception
  - retry
  - incident
  - timeout
  - escalation
  - compensation
- understand how Camunda transactions affect exception handling
- understand "asyncBefore" and "asyncAfter"
- configure failed-job retries
- understand when and why incidents are created
- correctly use "BpmnError"
- catch errors at:
  - service-task level
  - subprocess level
  - call-activity level
  - event-subprocess level
- avoid common implementation anti-patterns
- design consistent exception-handling standards across projects

---

2. The Most Important Principle

Business error != technical exception

This should be the first slide of the training.

Camunda explicitly describes BPMN Errors as intended for business errors, rather than ordinary technical exceptions.

Consider these situations:

Situation| Classification
Customer is not eligible| Business outcome/error
Policy already exists| Business error
Insufficient account balance| Business error
Required document missing| Business outcome
Underwriting declined case| Business outcome
Product unavailable| Business error
REST server returned 503| Technical error
Database unavailable| Technical error
Connection timeout| Technical error
NullPointerException| Technical/programming error
JSON deserialization failed unexpectedly| Technical error
OutOfMemoryError| Technical/system failure

The process model normally cares about the first group.

Infrastructure and operations normally care about the second group.

---

3. The Four Main Categories

A useful enterprise standard is to classify every failure into one of four categories.

Category 1 — Expected business result

Example:

«Check whether case information is complete.»

Result:

Complete = true

or

Complete = false

"false" is not really an exception.

Model it using normal sequence flow:

Check Case Information
        |
        v
     XOR Gateway
      /       \
 Complete    Incomplete

Camunda's own error-handling guidance uses essentially this distinction: if incompleteness is the result of checking completeness, it is often better represented as data and an XOR gateway rather than an exception.

---

4. Category 2 — Exceptional Business Error

Example:

Reserve Product

The service returns:

PRODUCT_NOT_AVAILABLE

This is exceptional, but the business knows what should happen.

For example:

             +--------------------+
             | Reserve Product    |
             +--------------------+
                       |
                 success
                       |
                       v
                 Continue Order

Boundary Error:
PRODUCT_NOT_AVAILABLE
        |
        v
Offer Alternative Product

This is a good use of a BPMN Error.

---

5. Category 3 — Recoverable Technical Error

Examples:

HTTP 503
Socket timeout
Database connection temporarily unavailable
Remote service temporarily down

Usually we do not want BPMN such as:

Call API
   |
Network failed?
   |
Retry
   |
Wait 5 min
   |
Retry

on every service task.

Instead:

asyncBefore
+
Camunda Job Executor
+
Retry configuration
+
Incident after retries exhausted

is normally the better solution.

Camunda recommends asynchronous jobs and failed-job retry handling for many technical errors rather than filling the business model with infrastructure handling.

---

6. Category 4 — Non-Recoverable Technical Error

Examples:

NullPointerException
Invalid application configuration
Missing deployment class
Serialization bug
SQL syntax error

Repeated execution is unlikely to fix these.

The desired lifecycle is normally:

Technical Exception
       |
       v
Job fails
       |
       v
Retries exhausted
       |
       v
Incident
       |
       v
Operations / Development investigates
       |
       v
Fix underlying problem
       |
       v
Retry job

---

7. The Decision Tree

Teach implementation teams this decision tree.

Something unusual happened
          |
          v
Is it an expected result?
       /       \
     YES        NO
      |          |
Normal flow     Does business understand
XOR gateway     and handle this situation?
                  /        \
                YES         NO
                 |           |
             BPMN Error   Technical exception
                              |
                              v
                   Is it potentially temporary?
                         /          \
                       YES           NO
                        |             |
                     Retry          Incident/
                  asynchronously     intervention

This one decision tree can prevent a large percentage of BPMN error-handling mistakes.

---

8. Understanding Camunda Transactions

This is probably the most important technical topic in the whole training.

Camunda does not persist after every service task.

The process engine advances from one stable state/wait state to another inside a transaction.

Camunda documentation describes wait states as points where execution is persisted and the transaction can commit.

For example:

User Task
   |
   v
Service Task A
   |
   v
Service Task B
   |
   v
User Task

Suppose the first User Task has already been created.

The user completes it.

Camunda may execute:

Complete User Task
Service Task A
Service Task B
Create next User Task

inside one transaction.

If Service Task B throws an unhandled Java exception:

Service Task B
        X
        |
   exception

the entire current transaction rolls back.

The process returns to the previous persisted wait state.

Conceptually:

Previous Wait State
       |
       | transaction begins
       v
Service A
       |
Service B ---- X exception
       |
       X
ROLLBACK
       |
       v
Previous Wait State remains

Camunda states that an unhandled exception rolls the transaction back to the previous save point.

---

9. A Surprising Case: Failure While Starting a Process

Consider:

runtimeService.startProcessInstanceByKey("caseProcess");

and BPMN:

Start
 |
Service Task
 |
User Task

If the service task throws a technical exception before any wait state is reached, the process-start transaction fails.

Camunda documentation notes that if an exception occurs during "startProcessInstanceByKey", the process instance may not be persisted at all.

This surprises many developers.

They expect:

Process instance
    |
failed service task

to appear in Cockpit.

It might not.

There was no committed process instance yet.

---

10. How asyncBefore Changes Everything

Now configure:

Start
 |
 |
 v
Service Task
asyncBefore=true
 |
User Task

Execution becomes approximately:

Start Process
     |
Create job
     |
COMMIT
     |
API returns

Later:

Job Executor
    |
    v
Execute Service Task

If the service fails:

Job Executor
     |
Service Task
     |
Exception
     |
Job failure
     |
Retry

The process instance now exists.

The failure is associated with a job.

This is why asynchronous continuations are so important to reliable Camunda architecture.

Camunda describes "asyncBefore"/"asyncAfter" as mechanisms for explicitly creating transaction boundaries; asynchronous continuations are executed by the Job Executor.

---

11. asyncBefore vs asyncAfter

asyncBefore

Activity A
    |
----COMMIT----
    |
   Job
    |
Service Task B

Configuration:

<serviceTask
    id="service1"
    camunda:asyncBefore="true"
    camunda:class="com.example.MyDelegate" />

If B fails, A remains committed.

---

asyncAfter

Service Task B
     |
     B completes
     |
----COMMIT----
     |
    Job
     |
Continue

Configuration:

<serviceTask
    id="service1"
    camunda:asyncAfter="true"
    camunda:class="com.example.MyDelegate" />

Both create transaction boundaries, but at different points.

---

12. BPMN Error

A BPMN Error represents an error the business process is prepared to handle.

Examples:

CUSTOMER_NOT_ELIGIBLE
PRODUCT_NOT_AVAILABLE
DUPLICATE_APPLICATION
VALIDATION_FAILED
CREDIT_LIMIT_EXCEEDED

These should normally have stable codes.

Avoid:

ERROR1
ERROR2
ERR001

unless those codes are part of a defined enterprise error catalogue.

Better:

PRODUCT_NOT_AVAILABLE
CUSTOMER_NOT_ELIGIBLE
CASE_ALREADY_EXISTS

---

13. Throwing BPMN Error from Java

Camunda provides:

org.camunda.bpm.engine.delegate.BpmnError

Example:

public class ReserveProductDelegate implements JavaDelegate {

    @Override
    public void execute(DelegateExecution execution) {

        ProductResult result = productService.reserve();

        if (!result.isAvailable()) {
            throw new BpmnError("PRODUCT_NOT_AVAILABLE");
        }

        execution.setVariable("reservationId",
                              result.getReservationId());
    }
}

Camunda specifically provides "BpmnError" for throwing BPMN business faults from delegation code.

---

14. Translating a Domain Exception into a BPMN Error

Often the service/domain layer should not depend on Camunda.

For example:

public class ProductService {

    public Product reserve(String id) {
        ...
        throw new ProductUnavailableException(id);
    }
}

Camunda delegate:

public class ReserveProductDelegate implements JavaDelegate {

    @Override
    public void execute(DelegateExecution execution) {

        try {

            productService.reserve(
                (String) execution.getVariable("productId")
            );

        } catch (ProductUnavailableException ex) {

            throw new BpmnError(
                "PRODUCT_NOT_AVAILABLE",
                ex.getMessage()
            );
        }
    }
}

This is a good architectural separation:

Domain layer
      |
Domain Exception
      |
Process Adapter / Delegate
      |
BpmnError
      |
BPMN

Do not make the entire business/service layer dependent on Camunda's "BpmnError".

---

15. Do Not Convert Every Java Exception to BpmnError

This is a serious anti-pattern:

try {

    remoteService.call();

} catch (Exception ex) {

    throw new BpmnError("SYSTEM_ERROR");
}

Why?

Suppose the actual problem was:

Database outage
HTTP timeout
NullPointerException
OutOfMemory condition
IllegalStateException
Programming defect

You have transformed a technical failure into a business event.

Consequences:

- no normal job retry behavior
- no failed-job incident
- technical problem may look like successful process routing
- monitoring becomes harder
- business BPMN becomes coupled to infrastructure failures

Use "BpmnError" deliberately.

Camunda explicitly says it should be used for business faults while technical errors should be represented by other exception types.

---

16. Technical Exception Example

Correct:

public class SendRequestDelegate implements JavaDelegate {

    @Override
    public void execute(DelegateExecution execution) {

        remoteClient.sendRequest();
    }
}

If:

remoteClient.sendRequest();

throws:

RemoteSystemException

allow it to propagate unless you have a specific reason to transform it.

For example:

throw new RuntimeException(
    "Unable to communicate with underwriting system",
    ex
);

Then Camunda's transaction/job mechanism handles it.

---

17. BPMN Error Boundary Event

Suppose we have:

         +---------------------+
         | Reserve Product     |
         +---------------------+
                    |
                    v
                Continue

Attach:

      PRODUCT_NOT_AVAILABLE
                O
                |
         +------+------+
         | Reserve     |
         | Product     |
         +-------------+
                |
                v
       Offer Alternative

The boundary event catches the error code.

When an error boundary event catches an error, Camunda destroys the activity scope to which it is attached and continues along the boundary-event path.

So error boundary events are effectively interrupting.

---

18. Matching Error Codes

Define:

<bpmn:error
    id="Error_ProductUnavailable"
    name="Product unavailable"
    errorCode="PRODUCT_NOT_AVAILABLE" />

Catch:

<bpmn:errorEventDefinition
    errorRef="Error_ProductUnavailable" />

Java:

throw new BpmnError("PRODUCT_NOT_AVAILABLE");

Matching is based on the error code.

---

19. Catch-All Error Handler

If "errorRef" is omitted on an error catch event, Camunda can catch any BPMN Error reaching that scope.

Conceptually:

Subprocess
+----------------------------------+
|                                  |
| Task A -> Task B -> Task C       |
|                                  |
+----------------------------------+
                O
                |
          Any BPMN Error
                |
                v
        Generic Business
        Error Handler

Use this carefully.

Specific errors are usually easier to understand and govern.

---

20. Error Propagation

Errors propagate outward through BPMN scopes.

Example:

Parent Process
|
+--- Subprocess A
     |
     +--- Subprocess B
          |
          +--- Service Task
                  |
                  X
          PRODUCT_NOT_AVAILABLE

Camunda searches outward for a matching handler. Error propagation through parent scopes continues until an appropriate boundary handler is found.

Therefore you can design:

Low-level process
    |
throws business error
    |
higher-level subprocess
    |
handles error

This is very useful for reusable subprocess design.

---

21. Error Handling on a Call Activity

Suppose:

Parent Process

Start
 |
Call Activity: Underwriting
 |
Issue Policy

The called process:

Underwriting Process

Evaluate Risk
     |
     +--- Accepted ---> End
     |
     +--- Declined ---> Error End
                       UNDERWRITING_DECLINED

Parent:

              Error
     UNDERWRITING_DECLINED
                  O
                  |
        +---------+---------+
        | Call Underwriting |
        +-------------------+
                  |
                  v
          Inform Customer

This provides clean separation:

Child:
I could not successfully complete
because UNDERWRITING_DECLINED.

Parent:
I understand that outcome and
know what to do next.

Error boundary handling is particularly useful on embedded subprocesses and call activities because they establish scopes.

---

22. Error End Event

Instead of Java throwing "BpmnError", the process itself can throw one.

Example:

Evaluate Eligibility
       |
       v
      XOR
     /   \
Eligible Not Eligible
   |        |
   v        v
Continue   Error End
           CUSTOMER_NOT_ELIGIBLE

Reaching an Error End Event throws the corresponding BPMN error.

This is extremely useful because the business semantics remain entirely visible in BPMN.

---

23. Event Subprocess for Centralized Business Error Handling

Instead of placing boundary handlers on many tasks:

Main Process
+-------------------------------------+
|                                     |
| Task A -> Task B -> Task C          |
|                                     |
|   +-----------------------------+   |
|   | Error Event Subprocess      |   |
|   |                             |   |
|   | Error Start -> Handle Error |   |
|   +-----------------------------+   |
|                                     |
+-------------------------------------+

An Error Start Event can only start an Event Subprocess, not a normal process instance, and it is always interrupting.

This provides a very useful pattern for centralized business error handling within a process scope.

---

24. Capturing Error Code and Error Message

Camunda supports:

camunda:errorCodeVariable
camunda:errorMessageVariable

For example:

<errorEventDefinition
    errorRef="BusinessError"
    camunda:errorCodeVariable="errorCode"
    camunda:errorMessageVariable="errorMessage" />

After catching:

execution.getVariable("errorCode");
execution.getVariable("errorMessage");

Camunda documents both variables as available on error catch events when configured.

This can be useful for:

audit logging
notification
support information
generic error handlers

---

25. Very Important: Unhandled BPMN Errors

This behavior should definitely appear in the training.

Suppose Java throws:

throw new BpmnError("CUSTOMER_NOT_ELIGIBLE");

but there is no matching handler.

Many developers expect:

Incident

or:

Java exception

That is not necessarily what happens.

By default, Camunda can log the unhandled BPMN Error and end the affected execution. Camunda provides the configuration property:

enableExceptionsAfterUnhandledBpmnError

which changes this behavior so an unhandled BPMN Error results in a process-engine exception.

For project governance, I recommend at minimum:

Every possible BpmnError should have
an intentionally designed catcher.

And strongly consider enabling:

enableExceptionsAfterUnhandledBpmnError=true

during development and testing so accidental uncaught business errors are visible immediately.

---

26. Technical Errors and Retries

For technical failures:

Call External Service

configure:

asyncBefore

Then the Job Executor performs the work.

If execution throws an exception:

Job
 |
 X
Exception
 |
Retry counter decreases

Camunda's default failed-job behavior retries jobs three times; custom schedules can be configured.

---

27. Configuring Retry Policy

Example:

<serviceTask
    id="callExternalSystem"
    camunda:asyncBefore="true"
    camunda:class="com.example.CallExternalSystemDelegate">

    <extensionElements>

        <camunda:failedJobRetryTimeCycle>
            R5/PT5M
        </camunda:failedJobRetryTimeCycle>

    </extensionElements>

</serviceTask>

Meaning:

R5     = maximum configured retries
PT5M   = five-minute interval

Camunda documents ISO-8601 retry schedules for failed jobs.

---

28. Example Enterprise Retry Strategy

For a network service:

PT1M,
PT2M,
PT5M,
PT10M,
PT30M

Conceptually:

Attempt 1
   |
fail
   |
wait 1 min

Attempt 2
   |
fail
   |
wait 2 min

Attempt 3
   |
fail
   |
wait 5 min
...

This avoids hammering an unavailable downstream service.

Retry strategy should depend on the failure type.

---

29. What Happens When Retries Reach Zero?

Job
 |
Retry
 |
Retry
 |
Retry
 |
0 retries
 |
v
INCIDENT

Camunda defines a "failedJob" incident when automatic retries for an asynchronous job/timer have been exhausted. Administrative action is then necessary.

This is an important distinction:

Exception != Incident

An exception may happen several times before an incident exists.

Typical lifecycle:

Exception
    |
Retry 2 remaining
    |
Exception
    |
Retry 1 remaining
    |
Exception
    |
Retry 0
    |
Incident

---

30. What Is an Incident?

An Incident means:

«The process cannot continue automatically and requires administrative attention.»

Examples:

failedJob
failedExternalTask

Camunda 7 supports "failedJob" and "failedExternalTask" incident types and also permits custom incidents.

Operationally:

Cockpit
   |
Incident
   |
Inspect exception
   |
Fix root cause / variables / downstream system
   |
Increase retries
   |
Job Executor
   |
Process continues

---

31. Operators Should Not Simply Keep Clicking Retry

A failed job is a symptom.

Before retrying:

1. Read exception message
2. Read stack trace
3. Identify failed activity
4. Inspect relevant process variables
5. Determine whether problem is:
   - temporary infrastructure issue
   - invalid data
   - configuration
   - deployment mismatch
   - programming defect
6. Fix root cause
7. Retry

Otherwise:

Retry
 -> fail
 -> retry
 -> fail

only creates noise.

---

32. Idempotency Is Essential

Retries introduce another architectural requirement:

Service tasks should be idempotent wherever possible.

Imagine:

Debit Account
   |
response lost
   |
Camunda thinks call failed
   |
retry
   |
Debit Account AGAIN

That can be disastrous.

Prefer:

businessKey / transactionId
             |
             v
External service checks:
"Have I processed this request already?"

Example:

POST /payment

requestId = CASE-123-PAYMENT-01

Remote system:

if alreadyProcessed(requestId)
    return existingResult;

Every project using automatic retries should review idempotency.

---

33. External Tasks

External Tasks have a slightly different API model.

Worker:

fetch and lock
     |
execute work
     |
+----+-------------------+
|                        |
success                failure
|                        |
complete()          handleFailure()

For technical failures:

externalTaskService.handleFailure(
    externalTask,
    "CRM system unavailable",
    stackTrace,
    retries,
    retryTimeout
);

If retries reach zero, a "failedExternalTask" incident is created.

---

34. External Task Business Error

If the worker discovers a business problem:

externalTaskService.handleBpmnError(
    externalTask,
    "CUSTOMER_NOT_FOUND",
    "Customer does not exist"
);

Camunda describes "handleBpmnError" as reporting a business error, while "handleFailure" represents task execution failure.

So:

handleBpmnError()
        |
        v
BPMN Error Boundary/Event Subprocess


handleFailure()
        |
        v
Retry / Incident

This distinction should become an implementation standard.

---

35. Timeouts Are Usually Not BPMN Errors

Suppose:

Wait for Customer Documents

The customer has 5 days.

Use:

                 Timer
                   O
                   |
       +-----------+---------+
       | Wait for Documents  |
       +---------------------+
                   |
                   v
             Documents received

Timer boundary:

After 5 days
    |
Send reminder / cancel case / escalate

A business timeout is normally represented using a Timer Event, not Java exception handling.

---

36. Escalation vs Error

These are often confused.

Think:

Error

«Something happened that prevents the normal work in this scope from continuing.»

Usually interrupting.

Example:

CUSTOMER_NOT_ELIGIBLE

---

Escalation

«Something needs higher-level attention, but it is not necessarily a failure.»

Examples:

Approval amount > ₹10 lakh
VIP customer
Case needs senior review
SLA approaching limit

Conceptually:

Error:
STOP normal handling and take alternate path.

Escalation:
Notify/route something to a higher level.

Do not use BPMN Error merely because a case needs attention.

---

37. Compensation Is Different Again

Suppose the process performs:

Reserve Hotel
     |
Reserve Flight
     |
Charge Customer

Flight reservation succeeds.

Hotel reservation succeeds.

Payment later fails.

Database rollback cannot automatically:

cancel flight
cancel hotel

because those operations occurred in external systems.

You may need business compensation:

Payment Failed
      |
      v
Cancel Hotel
      |
Cancel Flight

Camunda's error-handling documentation explicitly distinguishes BPMN compensation/business transactions from technical ACID rollback.

Important:

Technical transaction rollback
             !=
Business compensation

---

38. Technical Rollback vs Compensation

Technical rollback

Example:

INSERT database record
   |
Java exception
   |
transaction rollback

Database operation may be undone automatically.

---

Compensation

Example:

Flight booking REST API
   |
successful
   |
later task fails

You cannot simply roll back the remote airline system transaction.

You need:

Cancel Flight Reservation

as a separate business operation.

---

39. Do Not Catch Exception and Ignore It

Anti-pattern:

try {

    service.call();

} catch (Exception e) {

    log.error("Something failed", e);

}

execution.setVariable("status", "SUCCESS");

Now BPMN sees:

SUCCESS

although work failed.

This is one of the worst failure-handling patterns.

If the process cannot continue correctly:

throw the technical exception

or:

throw a meaningful BPMN business error

depending on classification.

---

40. Avoid Generic SYSTEM_ERROR BPMN Paths

Bad model:

Task A ----\
Task B ----- SYSTEM_ERROR -> Error User Task
Task C ----/
Task D ---/

Eventually every service task has:

SYSTEM_ERROR

The business diagram becomes an infrastructure-monitoring diagram.

Camunda's own guidance warns against filling the business process with technical error-handling user tasks and generally prefers failed-job monitoring for such technical situations.

---

41. Do Not Model Technical Retries Everywhere

Avoid:

Call API
  |
failed
  |
Timer
  |
Retry
  |
failed
  |
Timer

unless the retry itself has business significance.

Use:

asyncBefore
+
failedJobRetryTimeCycle

for generic technical recovery.

Camunda also recommends keeping explicit retry loops for cases where seeing the retry in the business process is genuinely meaningful.

---

42. When Explicit Retry in BPMN IS Appropriate

Sometimes business needs to see it.

Example:

Request Documents
       |
wait 3 days
       |
No response
       |
Send Reminder
       |
wait 3 days
       |
No response
       |
Close Case

That is a business retry.

Model it.

Another example:

Submit claim
     |
Rejected due to missing information
     |
Correct information
     |
Resubmit

Again, business-visible retry.

---

43. When Camunda Retry Is Appropriate

Example:

Call Policy Administration System

Failures:

connection refused
HTTP 503
read timeout
database unavailable

The business does not care whether technically it took:

1 attempt

or:

3 attempts

Therefore:

Camunda Job Retry

is more appropriate.

---

44. Recommended Delegate Architecture

I recommend projects standardize on something similar to:

BPMN
 |
Delegate / Process Adapter
 |
Application Service
 |
Domain Service
 |
Infrastructure Client

Error translation occurs primarily at the adapter boundary.

Example:

@Component
public class ValidateCaseDelegate
        implements JavaDelegate {

    @Override
    public void execute(
            DelegateExecution execution) {

        try {

            validationService.validate(
                execution.getVariable("caseId", String.class)
            );

        } catch (CaseValidationException ex) {

            throw new BpmnError(
                "CASE_VALIDATION_FAILED",
                ex.getMessage()
            );
        }
    }
}

Technical exceptions remain technical.

---

45. Suggested Enterprise Error Taxonomy

Projects can standardize error codes.

Example:

CASE_001   CASE_NOT_FOUND
CASE_002   CASE_ALREADY_EXISTS
CASE_003   CASE_VALIDATION_FAILED

UW_001     UNDERWRITING_DECLINED
UW_002     ADDITIONAL_REQUIREMENTS_NEEDED

PRODUCT_001 PRODUCT_NOT_AVAILABLE

PAYMENT_001 INSUFFICIENT_FUNDS
PAYMENT_002 PAYMENT_REJECTED

But don't mix these with:

HTTP_503
DATABASE_TIMEOUT
NULL_POINTER

Those are technical.

---

46. Recommended Process Variables

If your projects need centralized auditing, standard variables can help:

errorCode
errorMessage
errorCategory
failedActivity
failureTimestamp

But do not indiscriminately copy entire Java stack traces into process variables.

Camunda jobs/incidents already provide operational failure information.

Keep business data and operational diagnostics conceptually separate.

---

47. Exception Handling Matrix

Use this as a slide.

Situation| BPMN technique| Camunda mechanism
Expected business result| XOR gateway| Process variable
Business exception| Error Event| "BpmnError"
Business timeout| Timer Event| Timer Job
Business notification| Message/Signal| Correlation
Higher-level attention| Escalation| BPMN Escalation
Temporary technical failure| None in BPMN normally| Async + retry
Persistent technical failure| None in BPMN normally| Incident
External task business problem| Error Event| "handleBpmnError()"
External task technical problem| —| "handleFailure()"
Undo completed business action| Compensation| Compensation handler
Unexpected programming bug| —| Exception + incident

---

48. A Strong Default Pattern for Service Tasks

For integration/service tasks:

                    +----------------+
                    | Previous State |
                    +----------------+
                           |
                           v

                      asyncBefore

                           |
                           v

                    +----------------+
                    | Service Task   |
                    +----------------+
                     |              |
             business error     technical error
                     |              |
                     v              v
               BPMN Error       Job retry
                     |              |
                     v              v
            Alternate flow       Incident
                                after retries

This pattern works extremely well for enterprise Camunda implementations.

---

49. Error Handling at Different Levels

Task level

Use when error handling is specific to exactly one task.

Service Task
    |
Boundary Error

Example:

Reserve Product
PRODUCT_NOT_AVAILABLE

---

Subprocess level

Use when multiple activities form one logical business scope.

+--------------------------+
| Validate Application     |
|                          |
| Task A -> Task B -> C    |
+--------------------------+
             |
          Error

This creates cleaner models.

---

Call Activity level

Use when called process exposes business failures to its caller.

Call Underwriting
       |
UNDERWRITING_DECLINED

Excellent for reusable processes.

---

Event subprocess level

Use for centralized handling across a larger scope.

Main process

+----------------------------------+
|                                  |
| Happy path                       |
|                                  |
| [Error Event Subprocess]         |
|                                  |
+----------------------------------+

---

50. Catch and Rethrow Pattern

Sometimes a lower layer needs to perform local handling and still propagate the business error upward.

Conceptually:

Child Scope
     |
Business Error
     |
Error Event Subprocess
     |
Log / cleanup
     |
Throw same BPMN Error again
     |
Parent boundary handler

Camunda explicitly supports catch-and-rethrow patterns for BPMN errors.

---

51. Listener Exceptions

This is an advanced topic worth mentioning.

"BpmnError" can be caught when thrown from many listeners participating in normal process flow, such as activity execution listeners and common task listener events.

But Camunda documents exceptions where BPMN Error propagation does not behave the same way—for example some process-level/delete/listener scenarios outside normal execution flow.

Therefore:

«Do not build complex business error-handling architecture primarily inside listeners.»

Keep business behavior visible in process activities wherever practical.

---

52. Process Engine Exceptions

Camunda itself can throw:

ProcessEngineException

These are not BPMN business errors.

Examples can include:

invalid persistence operation
invalid variable
concurrent operation
engine configuration problem

Camunda 7 also exposes exception codes so applications do not need to parse exception message strings.

For example, Camunda documents categories including:

Optimistic locking
Deadlock
Foreign-key constraint
Column size errors
Custom codes

Do not build code such as:

if (exception.getMessage()
             .contains("some text")) {
   ...
}

Messages can change.

Use exception classes/error codes.

---

53. Optimistic Locking Exception

Camunda uses optimistic locking to handle concurrent updates.

Interestingly, Camunda documentation notes that failed-job retries are not decremented for optimistic-locking exceptions because these can naturally occur with concurrent execution and are expected to resolve through retry.

This is another useful advanced topic for developers working with:

parallel gateways
parallel multi-instance
concurrent jobs

---

54. Operational Monitoring Standard

Every implementation project should monitor at least:

Number of active incidents
Incident age
Failed activity
Process definition
Process instance/business key
Retry count
Exception type
Downstream system

An ideal operations dashboard might answer:

How many incidents exist?

Which process has most incidents?

Which activity fails most often?

Which downstream system causes them?

How old is the oldest incident?

How many jobs recovered automatically through retry?

---

55. Recommended Incident Workflow

Incident detected
      |
      v
Identify technical category
      |
      +--- Temporary outage
      |       |
      |       v
      |   Restore system
      |       |
      |     Retry
      |
      +--- Bad process data
      |       |
      |    Correct data
      |       |
      |     Retry
      |
      +--- Software bug
              |
          Deploy fix
              |
            Retry

Do not cancel instances just because an incident exists.

Incidents are intentionally designed to allow recovery.

---

56. Recommended Implementation Rules

I would establish the following as project-wide standards.

Rule 1

Never use "BpmnError" for generic technical exceptions.

Rule 2

Every BPMN Error must have a documented business meaning.

Rule 3

Every thrown BPMN Error should have an intentionally designed catcher.

Rule 4

Prefer normal sequence flow for expected results.

Rule 5

Prefer retries/incidents for temporary technical failures.

Rule 6

Integration/service tasks should be idempotent if automatic retry is possible.

Rule 7

Use asynchronous continuations to establish meaningful recovery/save points.

Rule 8

Never silently swallow technical exceptions.

Rule 9

Do not model infrastructure failures repeatedly in business BPMN.

Rule 10

Use timer events for business timeouts.

Rule 11

Use compensation when previously completed external business actions must be undone.

Rule 12

Always test failure paths—not only the happy path.

---

57. Recommended Failure Tests

Every important service task should have tests for:

1. Successful execution

2. Business failure
   -> correct BPMN Error handler

3. Unexpected technical exception
   -> transaction rollback

4. async job failure
   -> retry count decreases

5. retries exhausted
   -> incident created

6. retry after root cause fixed
   -> process continues

7. duplicate/repeated execution
   -> idempotent behavior

8. downstream timeout

9. invalid process variable

10. unhandled BPMN Error

Exception paths deserve the same testing discipline as successful paths.

---

58. Suggested Hands-On Training Lab

Demo 1 — Normal result

Process:

Start
 |
Check Case
 |
XOR
/ \
Valid Invalid

Show why "Invalid" may simply be a result.

---

Demo 2 — BPMN Error

Start
 |
Reserve Product
 |
End

Boundary:
PRODUCT_NOT_AVAILABLE
 |
Manual Selection

Java:

throw new BpmnError(
    "PRODUCT_NOT_AVAILABLE"
);

Run both success and business-error scenarios.

---

Demo 3 — Technical Exception Without async

Start
 |
Failing Service Task
 |
User Task

Java:

throw new RuntimeException(
    "Database unavailable"
);

Observe transaction rollback.

Explain why an incident may not appear.

---

Demo 4 — Same Exception With asyncBefore

Change:

Failing Service Task

to:

asyncBefore=true

Run again.

Observe:

job
retry
exception
incident

This is probably the most valuable demonstration in the training.

---

Demo 5 — Retry Configuration

Configure:

<camunda:failedJobRetryTimeCycle>
    R3/PT1M
</camunda:failedJobRetryTimeCycle>

Observe retry lifecycle.

---

Demo 6 — Error Propagation Through Subprocess

Parent
 |
Subprocess
+----------------+
| Service Task   |
| throws ERROR_A |
+----------------+
        |
Boundary Error
ERROR_A

Show scope propagation.

---

Demo 7 — Call Activity

Parent:

Call Child Process

Child:

Task
 |
ERROR END

Parent catches error on Call Activity.

---

Demo 8 — Compensation

Book Hotel
 |
Book Flight
 |
Payment
 |
Failure
 |
Compensate reservations

Demonstrate that compensation is fundamentally different from Java rollback.

---

59. Training Questions to Ask the Audience

Give them scenarios and ask:

«BPMN Error, technical exception, normal result, timer, or compensation?»

Scenario A

Customer is 17 but minimum eligibility is 18.

Answer:

Business result/error

Depending on the process semantics, normal XOR or BPMN Error.

---

Scenario B

Oracle database is temporarily unavailable.

Answer:

Technical exception
+
retry/incident

---

Scenario C

Customer does not submit documents within 10 days.

Answer:

Timer Event

---

Scenario D

Payment succeeded but subsequent order creation fails.

Answer:

Potential compensation requirement

---

Scenario E

Remote API responds HTTP 503.

Answer:

Technical retry

---

Scenario F

Underwriting declines the application.

Answer:

Business result/error

---

60. The Golden Architecture

The ideal architecture can be summarized as:

                    FAILURE
                       |
             +---------+---------+
             |                   |
         BUSINESS             TECHNICAL
             |                   |
      +------+-----+        +----+-----+
      |            |        |          |
 expected      exceptional temporary permanent
 result           |          |          |
   |              |          |          |
 XOR          BPMN Error    Retry     Incident
                            async

And when previously completed external work needs to be undone:

Compensation

---

61. Ten Things Participants Should Remember

1. BPMN Error is primarily a business concept.

2. Java exception is primarily a technical execution concept.

3. Expected business outcomes often belong on normal sequence flows.

4. Unhandled technical exceptions cause transaction rollback.

5. "asyncBefore"/"asyncAfter" create transaction boundaries and jobs.

6. Jobs enable retries.

7. When retries are exhausted, an incident can be created.

8. Do not transform every technical exception into "BpmnError".

9. Retries require idempotent service design.

10. Compensation is not the same thing as database rollback.

---

62. One-Slide Cheat Sheet

BUSINESS RESULT
    ↓
Sequence Flow / XOR

BUSINESS EXCEPTION
    ↓
BpmnError
    ↓
Boundary Error / Event Subprocess

TECHNICAL TRANSIENT FAILURE
    ↓
Java Exception
    ↓
Async Job
    ↓
Retry

TECHNICAL PERSISTENT FAILURE
    ↓
Retries exhausted
    ↓
Incident
    ↓
Operations

TIME-BASED BUSINESS CONDITION
    ↓
Timer Event

HIGHER-LEVEL ATTENTION
    ↓
Escalation

UNDO PREVIOUS BUSINESS ACTION
    ↓
Compensation

---

63. Recommended Project Review Checklist

When reviewing a BPMN implementation, ask:

BPMN

- Are expected outcomes modeled as normal flows?
- Are BPMN Errors really business errors?
- Are error codes meaningful?
- Is every error intentionally handled?
- Is error handling placed at the appropriate scope?
- Are timers used for timeouts?
- Is compensation used where distributed business operations need reversal?

Java

- Are technical exceptions allowed to propagate?
- Are domain exceptions translated to "BpmnError" only where appropriate?
- Are exceptions logged with sufficient context?
- Are exceptions being swallowed?
- Are remote calls idempotent?

Camunda configuration

- Are transaction boundaries intentional?
- Should the service task have "asyncBefore"?
- Should it have "asyncAfter"?
- Is retry configuration appropriate?
- What happens when retries reach zero?
- Is incident monitoring configured?

Operations

- Who monitors incidents?
- Who owns each downstream integration?
- How are retries performed?
- Are incidents correlated with business keys?
- Is there an alert for old incidents?
- Is there a documented recovery procedure?

---

64. Final Principle

The strongest Camunda implementations separate three concerns:

BPMN
"What should the business do?"

Application code
"How is the work performed?"

Camunda runtime/operations
"How do we recover when technology fails?"

When those three responsibilities are mixed together, BPMN diagrams become complicated and operational recovery becomes unreliable.

When they are separated correctly:

Business problem
      -> BPMN

Technical temporary problem
      -> Retry

Technical persistent problem
      -> Incident

Already-completed business action to undo
      -> Compensation

That is the exception-handling model I would recommend standardizing across implementation projects.
