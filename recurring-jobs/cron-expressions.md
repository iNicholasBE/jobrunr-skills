---
name: jobrunr-recurring-cron
description: Schedule recurring background jobs with cron expressions in JobRunr,
  including time-zone handling, the @Recurring annotation, and stable job ids.
tier: oss
version: 8.6.0+
frameworks: [spring-boot, micronaut, quarkus]
---

# Recurring jobs with cron expressions

Use this skill to schedule a job that runs on a cron schedule (every Monday
at 9am, every hour, the first of every month, etc.).

## Prerequisites

- JobRunr installed and the background job server enabled
- A stable, unique id for each recurring job

## Working example

**Annotation form (Spring Boot, Micronaut, Quarkus):**

```java
@Component
public class ReportingService {

    @Recurring(id = "daily-sales-report", cron = "0 9 * * *", zoneId = "Europe/Brussels")
    @Job(name = "Daily sales report")
    public void runDailySalesReport() {
        // business logic
    }
}
```

The `@Recurring` annotation has four attributes: `id`, `cron`, `interval`,
and `zoneId`. The public docs page for recurring jobs doesn't currently
showcase `zoneId`, but it exists on the annotation and is the right way to
fix the "JVM-default-UTC" pitfall in containers.

The starter / feature / extension scans for `@Recurring` at startup and
registers the job. Subsequent restarts with the same id will *update* the
schedule, not create duplicates.

**Programmatic form:**

```java
BackgroundJob.scheduleRecurrently(
    "daily-sales-report",
    "0 9 * * *",
    ZoneId.of("Europe/Brussels"),
    () -> reportingService.runDailySalesReport()
);
```

There is also a `Cron` helper for the common shapes — `Cron.daily()`,
`Cron.daily(int hour)`, `Cron.hourly()`, `Cron.every5minutes()`,
`Cron.everyHalfHour()`, `Cron.weekly(DayOfWeek.MONDAY, 9)`, etc. — and a builder:

```java
jobScheduler.createRecurrently(aRecurringJob()
    .withId("daily-sales-report")
    .withCron(Cron.daily(9))
    .withZoneId(ZoneId.of("Europe/Brussels"))
    .withJobLambda(() -> reportingService.runDailySalesReport()));
```

## Time zones

Always pass `ZoneId` explicitly. If you omit it, the JVM default applies —
in containers that almost always means UTC. A "9am daily report" defined
without a zone runs at 9am UTC, which is *11am* in Brussels during summer
time. This is the most common production surprise with cron jobs.

## Cron resolution and timing

- JobRunr enqueues recurring jobs based on the poll interval (default 15s),
  so the actual fire time can drift a few seconds from the cron moment.
- The smallest allowed interval is **every 5 seconds**. JobRunr rejects
  anything finer to protect your storage provider.
- The cron interval must be **longer than `pollIntervalInSeconds`**.
  Otherwise the server enqueues multiple instances of the same recurring
  job per poll cycle.

## Deleting

```java
BackgroundJob.deleteRecurringJob("daily-sales-report");
```

Or use the dashboard. The method is idempotent — no error if the id is
unknown.

## Common mistakes

- **Implicit job id.** Calling `scheduleRecurrently(Cron.daily(), () -> ...)`
  without an id derives one from the lambda class/method. If the lambda
  changes shape (refactor, rename), JobRunr sees a *new* recurring job and
  the old one keeps firing forever. Always pass an explicit id.
- **Same id used twice.** Recurring jobs are upserted by id. Two
  `@Recurring(id = "x")` on different methods means the second one wins
  silently at startup.
- **No `zoneId`.** Use `ZoneId.of("Europe/Brussels")` (or your civil zone),
  not the JVM default.
- **OSS 100-job limit.** JobRunr OSS supports up to 100 recurring jobs. If
  you need more, see <https://www.jobrunr.io/en/pricing/>.
- **Changing the schedule without cleaning future-scheduled instances.**
  When you change a recurring job's cron, JobRunr OSS keeps any
  already-scheduled instances. Either delete them via the dashboard, or use
  JobRunr Pro which detects the change and cleans them up automatically.

## Sources

- <https://www.jobrunr.io/en/documentation/background-methods/recurring-jobs/>
- Cron expression generator: <https://www.jobrunr.io/en/tools/cron-expression-generator/>
- Java scheduling guide: <https://www.jobrunr.io/en/blog/java-cron-jobs-guide/>
