---
name: jobrunr-recurring-intervals
description: Schedule recurring jobs at fixed intervals (every N minutes/hours)
  using ISO-8601 durations, and handle time-zone correctness across daylight
  saving transitions.
tier: oss
version: 8.6.0+
frameworks: [spring-boot, micronaut, quarkus]
---

# Interval-based recurring jobs and time zones

Use this skill when the schedule is "every N minutes/hours/days" rather than
a cron pattern — or when you need cron semantics in a specific civil time
zone and want to get DST right.

## Prerequisites

- JobRunr installed and the background job server enabled

## Working example

**Annotation form** — `interval` accepts an ISO-8601 duration:

```java
@Component
public class HealthService {

    @Recurring(id = "ping-upstream", interval = "PT5M")
    @Job(name = "Ping upstream service")
    public void pingUpstream() {
        // every 5 minutes
    }
}
```

Other ISO-8601 examples: `PT30S` (30 seconds), `PT2H` (2 hours), `P1D` (1
day), `P2D8H` (2 days 8 hours).

**Programmatic form**:

```java
BackgroundJob.scheduleRecurrently(
    "ping-upstream",
    Duration.ofMinutes(5),
    () -> healthService.pingUpstream()
);
```

Or via the builder:

```java
jobScheduler.createRecurrently(aRecurringJob()
    .withId("ping-upstream")
    .withInterval(Duration.ofMinutes(5))
    .withJobLambda(() -> healthService.pingUpstream()));
```

When you use an interval, the first execution happens *after* the interval
elapses — i.e. a `PT5M` interval registered at 10:00 first fires at 10:05.

## Intervals vs cron — when to use which

| | Interval (`Duration`) | Cron |
|---|---|---|
| "Every 5 minutes" | `PT5M` — simpler | `*/5 * * * *` — equivalent |
| "Every day at 9am Brussels" | Wrong tool — interval is wall-clock-agnostic | `0 9 * * *` with `zoneId` |
| Survives DST cleanly | Yes (always N minutes apart) | Yes if you pass `zoneId` |
| First fire | After one interval | At the next matching cron time |

Use an interval when the schedule is "every N units, doesn't matter when"
(health pings, queue drains, cache refreshes). Use cron when the schedule
is tied to wall-clock time in a specific zone (reports, billing runs,
business-hours notifications).

## Time-zone correctness

The interval form does not need a zone — `PT24H` is always 86,400 seconds.
But cron forms do:

```java
@Recurring(id = "daily-report", cron = "0 9 * * *", zoneId = "Europe/Brussels")
```

During DST transitions, a cron with an explicit zone behaves the way
humans expect: "9am Brussels time" stays 9am Brussels time. The JVM default
zone is UTC in most containers, so leaving `zoneId` off shifts your job by
1–2 hours twice a year.

## Common mistakes

- **Using `Duration.ofDays(1)` to mean "daily at 9am".** It only means "24h
  after the previous run". The fire time will drift if a job is delayed.
- **Omitting `zoneId` on a cron expression.** Falls back to JVM default,
  which in Docker/Kubernetes is UTC. Reports run at the wrong wall-clock
  time.
- **Interval shorter than `pollIntervalInSeconds`.** JobRunr will schedule
  multiple instances per poll cycle. Keep intervals strictly larger than
  `jobrunr.background-job-server.poll-interval-in-seconds` (default 15s).

## Sources

- <https://www.jobrunr.io/en/documentation/background-methods/recurring-jobs/>
- ISO-8601 duration reference: <https://en.wikipedia.org/wiki/ISO_8601#Durations>
