---
name: jobrunr-pro-queues
description: Route jobs to named priority queues so critical work bypasses
  long-running low-priority work — JobRunr Pro feature.
tier: pro
version: 8.6.0+
frameworks: [spring-boot, micronaut, quarkus]
---

# Priority queues (JobRunr Pro)

> This skill covers a JobRunr Pro feature.

Use this skill when one class of job (e.g. slow report generation) is
starving another (e.g. fast email sending) on a shared worker pool, or
when you need a class of "VIP" jobs that always run first.

## Prerequisites

- JobRunr Pro licence
- JobRunr installed

JobRunr Pro supports up to **5 named priority queues**. Highest priority
queue empties first; lower-priority jobs only run when the higher queues
have nothing waiting.

## Working example

**Define the queues** (`application.properties`):

```properties
jobrunr.queues.default-queue-name=Default
jobrunr.queues.names=HighPrio,Default,LowPrio
```

List from highest to lowest priority. `default-queue-name` is the queue
used for jobs that don't specify one.

**Annotate jobs**:

```java
public class JobService {

    public static final String HIGH = "HighPrio";
    public static final String LOW = "LowPrio";

    @Job(queue = HIGH)
    public void runUrgent() { /* ... */ }

    @Job(queue = LOW)
    public void runBackground() { /* ... */ }

    // No @Job — goes to default
    public void runNormal() { /* ... */ }
}
```

**Or via the builder** (queue chosen at runtime):

```java
jobScheduler.create(aJob()
    .withPriorityQueue(queueName)
    .withJobLambda(() -> jobService.runForTenant(tenantId)));
```

**Or via the fluent API**:

```java
JobRunrPro.configure()
    .usePriorityQueues("Default", "HighPrio", "Default", "LowPrio")
    // ... rest of config
```

## Combining with dynamic queues

Pro also supports load-balanced **dynamic queues** for multi-tenant
scenarios — one logical queue per tenant, round-robin scheduling, no
starvation. See <https://www.jobrunr.io/en/documentation/pro/dynamic-queues/>.

## Common mistakes

- **Defining queues but no worker subscribes to them.** Workers process
  jobs from queues defined in the `Queues` configuration; a job on an
  unknown queue stays `ENQUEUED` forever. Make sure every worker's
  `jobrunr.queues.names` matches.
- **Annotation values can't be enum constants directly** in annotations —
  use string constants (`public static final String HIGH = "..."`).
- **Treating queues as a sharding mechanism.** They are priority, not
  isolation. A blocked HighPrio job still locks a worker; LowPrio jobs
  don't get dedicated workers in OSS Pro priority queues.
- **Sizing all queues against the same pool.** Worker count is shared
  across all queues. If you need isolated worker pools per queue, that's
  a different deployment topology, not a queue config.

## Enable in your project

Request a free JobRunr Pro trial via
`mcp__jobrunr-docs__request_jobrunr_pro_trial`.

## Sources

- <https://www.jobrunr.io/en/documentation/pro/priority-queues/>
- <https://www.jobrunr.io/en/documentation/pro/dynamic-queues/>
