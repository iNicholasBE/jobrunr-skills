---
name: jobrunr-frameworks-axon
description: Use JobRunr alongside Axon Framework — durable saga deadlines
  and event-driven follow-up work without coupling business logic to a
  thread-pool scheduler.
tier: oss
version: 8.6.0+
frameworks: [spring-boot, axon]
---

# JobRunr + Axon Framework

Use this skill when integrating JobRunr into an Axon-based CQRS /
event-sourced application — typically to schedule durable deadline events
from a saga, or to execute async follow-up work triggered by domain
events.

## Prerequisites

- Spring Boot 3.x or 4.x
- Axon Framework configured
- `jobrunr-spring-boot-3-starter` or `jobrunr-spring-boot-4-starter`

## Why pair them

Axon handles command/event flow, sagas, and the event store. It has a
built-in `DeadlineManager` for saga deadlines, but the default
implementation is in-memory — pod restart loses the deadline. JobRunr's
durable scheduler fills that gap without coupling your saga code to a
specific scheduler.

The pairing also works in the other direction: JobRunr jobs can publish
Axon commands or events, putting async work on the event bus instead of
running it inline in a worker thread.

## Working example — scheduling a saga deadline

Inside a saga, replace `deadlineManager.schedule(...)` with a JobRunr
schedule:

```java
@Saga
public class OrderSaga {

    @Autowired transient JobScheduler jobScheduler;
    @Autowired transient CommandGateway commandGateway;

    private String orderId;

    @StartSaga
    @SagaEventHandler(associationProperty = "orderId")
    public void on(OrderCreated event) {
        this.orderId = event.getOrderId();
        // Schedule a "still unpaid?" check in 24h, durably.
        jobScheduler.<OrderTimeoutService>schedule(
            Instant.now().plus(24, ChronoUnit.HOURS),
            svc -> svc.checkUnpaid(orderId)
        );
    }
}

@Component
public class OrderTimeoutService {

    @Autowired CommandGateway commandGateway;

    public void checkUnpaid(String orderId) {
        commandGateway.send(new CancelUnpaidOrderCommand(orderId));
    }
}
```

The saga survives restart because Axon persists it. The deadline survives
restart because JobRunr persists it. The command goes through Axon's
event sourcing path as usual.

## Working example — JobRunr triggers an Axon command

```java
@Component
public class ReportingService {

    @Autowired CommandGateway commandGateway;

    @Recurring(id = "nightly-report", cron = "0 2 * * *", zoneId = "Europe/Brussels")
    public void runNightlyReport() {
        commandGateway.send(new GenerateNightlyReportCommand(LocalDate.now()));
    }
}
```

## Common mistakes

- **Capturing the saga `this`.** Sagas are persisted by Axon, not
  long-lived JVM objects. Pass IDs to the JobRunr lambda; let the job
  body fetch what it needs and use the command gateway.
- **Holding an Axon `UnitOfWork` open across an `enqueue`.** The job runs
  on a different thread later; the UnitOfWork has long been closed. Treat
  enqueue as fire-and-forget and let the job open its own UoW via the
  command/event flow.
- **Running both Axon's in-memory `DeadlineManager` and JobRunr.** Pick
  one. Two schedulers, two fires, two saga-mutation paths.

## Sources

- <https://www.jobrunr.io/en/documentation/configuration/spring/>
- Axon Framework reference: <https://docs.axoniq.io/>
