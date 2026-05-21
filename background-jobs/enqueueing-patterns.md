---
name: jobrunr-bg-enqueueing
description: Patterns for enqueueing background jobs — fire-and-forget,
  scheduled, type-token, JobRequest, and bulk Stream enqueue — with guidance
  on which to pick.
tier: oss
version: 8.6.0+
frameworks: [spring-boot, micronaut, quarkus]
---

# Enqueueing patterns

Use this skill to choose the right `enqueue` API for the situation. JobRunr
has several shapes that look similar but have meaningful differences.

## Prerequisites

- JobRunr installed and the background job server enabled

## The four shapes you'll actually use

### 1. Instance lambda — quick and obvious

```java
JobId id = BackgroundJob.enqueue(() -> emailService.send(userId));
```

The `emailService` reference must be resolvable at enqueue time. The
worker still resolves it from the IoC container when the job runs — so
this works even if the instance you captured is no longer the same object.

### 2. Type-token lambda — idiomatic for DI-heavy apps

```java
JobId id = BackgroundJob.<EmailService>enqueue(svc -> svc.send(userId));
```

You don't need an instance of `EmailService` in scope to enqueue. The
worker resolves the type from the IoC container at run time. Prefer this
shape inside frameworks (Spring/Micronaut/Quarkus).

### 3. JobRequest — decoupled job description

```java
public class SendEmailJobRequest implements JobRequest {
    private UUID userId;
    private String templateKey;

    // constructor + getters + no-arg constructor

    @Override
    public Class<? extends JobRequestHandler> getJobRequestHandler() {
        return SendEmailJobRequestHandler.class;
    }
}

@Component
public class SendEmailJobRequestHandler implements JobRequestHandler<SendEmailJobRequest> {
    @Inject private EmailService emailService;

    @Override
    public void run(SendEmailJobRequest req) {
        emailService.send(req.getUserId());
    }
}

JobId id = BackgroundJobRequest.enqueue(new SendEmailJobRequest(userId, "welcome"));
```

Use this when you want a strongly-typed, serializable command object
(easier to evolve, easier to test the handler in isolation) — or when you
need to enqueue from code that doesn't have access to the target service.

### 4. JobBuilder — when you need name, labels, or retry overrides

```java
JobId id = jobScheduler.create(aJob()
    .withName("Welcome email for " + userId)
    .withLabels("tenant:" + tenant, "email")
    .withAmountOfRetries(3)
    .<EmailService>withJobLambda(svc -> svc.send(userId)));
```

The other three forms cover ~80% of cases. Reach for the builder when you
want to set the dashboard-visible name, attach searchable labels, or
override retry behaviour per job.

## Bulk enqueue from a Stream

For thousands of jobs (newsletter sends, batch imports), use Stream
overloads — JobRunr batches the DB writes:

```java
Stream<User> users = userRepository.findAllActive();
BackgroundJob.<EmailService, User>enqueue(
    users,
    (svc, user) -> svc.sendNewsletter(user.getId())
);
```

Or with `JobRequest`:

```java
Stream<SendEmailJobRequest> reqs = users.map(u -> new SendEmailJobRequest(u.getId(), "newsletter"));
BackgroundJobRequest.enqueue(reqs);
```

This integrates well with Spring Data `Stream<T>` repository methods —
incremental processing without loading the whole result set into memory.

## Scheduled (not fire-and-forget)

Same shapes, with a time argument:

```java
BackgroundJob.<EmailService>schedule(
    Instant.now().plus(24, ChronoUnit.HOURS),
    svc -> svc.sendDay2Email(userId)
);
```

Accepts `Instant`, `ZonedDateTime`, `OffsetDateTime`, `LocalDateTime`
(converted via the JVM default zone — prefer `ZonedDateTime` or `Instant`
in production).

## Common mistakes

- **Enqueueing inside a transaction that may roll back.** The job is
  written immediately. If the outer transaction rolls back, the job still
  runs — pointing at a row that was never committed. Either enqueue *after*
  the transaction commits (use a `TransactionSynchronization.afterCommit`
  in Spring) or design the job to handle the missing row.
- **Capturing entity objects in the lambda.** Entities are serialized as
  JSON; relationships, lazy proxies, and JPA session state don't survive.
  Pass IDs. See `parameter-serialization.md`.
- **Big argument payloads.** Method arguments are serialized into the
  job row. A 5MB JSON blob inflates your storage provider, slows polling,
  and survives long after the job. Pass an ID and re-load.
- **Catching all exceptions inside the job body.** JobRunr's retry is
  triggered by uncaught exceptions. Catch what you can handle; rethrow
  the rest.

## Sources

- <https://www.jobrunr.io/en/documentation/background-methods/enqueueing-jobs/>
- <https://www.jobrunr.io/en/documentation/background-methods/scheduling-jobs/>
- <https://www.jobrunr.io/en/documentation/background-methods/best-practices/>
