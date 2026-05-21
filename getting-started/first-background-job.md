---
name: jobrunr-getting-started-first-job
description: Enqueue your first JobRunr background job — pick a method, capture
  arguments correctly, and verify it ran via the dashboard.
tier: oss
version: 8.6.0+
frameworks: [spring-boot, micronaut, quarkus]
---

# Your first JobRunr background job

Use this skill after the framework setup is done. It walks through the
minimal happy path: enqueue → execute → confirm.

## Prerequisites

- JobRunr installed and the background job server enabled (see
  `spring-boot-setup.md`, `micronaut-setup.md`, or `quarkus-setup.md`)
- Dashboard reachable at <http://localhost:8000>

## Working example

A service that does the actual work:

```java
@Component
public class EmailService {

    public void sendWelcomeEmail(UUID userId) {
        // your business code
    }
}
```

Enqueue from anywhere — controller, listener, scheduled trigger:

```java
@Inject
private JobScheduler jobScheduler;

// Form 1: type-token enqueue — the worker resolves EmailService via the IoC container
JobId id = jobScheduler.<EmailService>enqueue(svc -> svc.sendWelcomeEmail(userId));

// Form 2: instance enqueue — instance is captured at enqueue time but the
// actual instance used to run the job is still resolved from the IoC container
emailService.sendWelcomeEmail(userId);
JobId id2 = jobScheduler.enqueue(() -> emailService.sendWelcomeEmail(userId));
```

Form 1 is the idiomatic shape and the one to prefer. The `userId` is
captured *by value* and serialized into the job record; `EmailService` is
re-resolved from Spring/Micronaut/Quarkus when the job runs.

What happens internally:

1. JobRunr analyzes the lambda with ASM to extract method + arguments.
2. Arguments are serialized to JSON.
3. The job row is written to the storage provider.
4. `enqueue(...)` returns immediately.
5. A background job server polls (default every 15 seconds), claims the job,
   resolves `EmailService`, and invokes the method.

Verify in the dashboard at <http://localhost:8000> — the job moves through
`ENQUEUED` → `PROCESSING` → `SUCCEEDED`.

## Common mistakes

- **Treating `enqueue(...)` as synchronous.** It is not. The method runs on a
  worker thread, possibly seconds later, possibly on a different JVM.
- **Capturing entities or other mutable state.** Pass IDs, not domain
  objects — entity state may change before the job runs. See
  `../background-jobs/parameter-serialization.md`.
- **Forgetting to make the method re-entrant.** Jobs retry on failure;
  sending an email twice because the SMTP server flaked is not what the user
  wants. Guard with a "has this already happened?" check.
- **Catching `Throwable` inside the job.** JobRunr can only retry what it
  sees fail — never swallow the exception. If you must log, re-throw.

## Next steps

- Recurring jobs → `../recurring-jobs/cron-expressions.md`
- Lambda capture rules → `../background-jobs/lambda-vs-method-reference.md`
- Dashboard tour → `../monitoring/dashboard-setup.md`

## Sources

- <https://www.jobrunr.io/en/documentation/background-methods/enqueueing-jobs/>
- <https://www.jobrunr.io/en/documentation/background-methods/best-practices/>
