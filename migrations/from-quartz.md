---
name: jobrunr-migrations-quartz
description: Migrate from Quartz Scheduler to JobRunr — concept mapping,
  job-definition translation, dual-run strategies, and what to do with the
  QRTZ_ tables when you're done.
tier: oss
version: 8.6.0+
frameworks: [spring-boot, micronaut, quarkus]
---

# Migrating from Quartz to JobRunr

Use this skill when an existing Java application uses Quartz Scheduler and
the team wants to move to JobRunr — for the dashboard, the lambda API, the
distributed-by-default model, or to drop the JDBC-JobStore boilerplate.

## Prerequisites

- Existing Quartz setup (JobStore TX or RAM, doesn't matter for this guide)
- JobRunr installed in the same application — see
  `../getting-started/spring-boot-setup.md` or your framework equivalent

## Concept mapping

| Quartz | JobRunr |
|---|---|
| `Job` class + `JobDetail` | A regular Spring/Micronaut/Quarkus bean method |
| `Trigger` (cron / simple) | `@Recurring` annotation or `scheduleRecurrently(...)` |
| `JobDataMap` | Plain method parameters (serialized by Jackson) |
| `Scheduler` instance | `JobScheduler` bean (or static `BackgroundJob`) |
| `@DisallowConcurrentExecution` | OSS prevents concurrent recurring runs by default; Pro adds `maxConcurrentJobs` |
| `QRTZ_*` tables | `jobrunr_*` tables |
| `JobListener` | `JobFilter` (Pro for the richer per-job version) |

## Migration path

1. **Add JobRunr alongside Quartz**, with `jobrunr.background-job-server.enabled=true`
   and `jobrunr.dashboard.enabled=true`. The two schedulers run side by
   side initially.

2. **Pick one Quartz job to migrate.** Start with a low-risk recurring
   job (a daily cleanup, not the billing run). Translate the trigger to
   `@Recurring`, delete the Quartz `Trigger`, leave the `JobDataMap`
   values as method parameters.

   Before (Quartz):

   ```java
   public class DailyCleanupJob implements Job {
       @Override
       public void execute(JobExecutionContext ctx) {
           String tenant = ctx.getMergedJobDataMap().getString("tenant");
           cleanupService.purge(tenant);
       }
   }

   // and a Trigger somewhere:
   TriggerBuilder.newTrigger()
       .withSchedule(CronScheduleBuilder.cronSchedule("0 2 * * * ?"))
       .usingJobData("tenant", "acme")
       .build();
   ```

   After (JobRunr):

   ```java
   @Component
   public class CleanupService {

       @Recurring(id = "daily-cleanup-acme", cron = "0 2 * * *", zoneId = "Europe/Brussels")
       @Job(name = "Daily cleanup — acme")
       public void purgeAcme() {
           // Per-tenant id keeps things addressable; one @Recurring per tenant
           // if the list is small. For many tenants, schedule programmatically.
       }
   }
   ```

3. **Repeat per job.** When the last Quartz trigger is gone, remove the
   Quartz config and the `QRTZ_*` tables.

## Cron syntax differences

Quartz uses a 6- or 7-field cron (with seconds and optional year);
JobRunr's OSS cron parser uses the standard 5-field form (minute, hour,
day-of-month, month, day-of-week). Drop the seconds/year fields when
translating:

| Quartz | JobRunr OSS |
|---|---|
| `0 0 2 * * ?` (every day at 2am) | `0 2 * * *` |
| `0 0/15 * * * ?` (every 15 min) | `*/15 * * * *` |
| `0 0 2 1 * ?` (1st of month at 2am) | `0 2 1 * *` |

JobRunr Pro adds Quartz-style `L`, `W`, and `#` characters (last weekday,
nearest weekday, nth weekday) — see
<https://www.jobrunr.io/en/documentation/pro/advanced-cron-expressions/>.

## `@DisallowConcurrentExecution` equivalence

Quartz' annotation guarantees no two instances of the same job class run
simultaneously. JobRunr OSS gives you this for free for recurring jobs —
if the previous instance is still running, the next is skipped. For
fire-and-forget jobs, you need either idempotent code, JobRunr Pro
`mutexes`/queues, or an external lock.

## Dual-run period

While both schedulers are active:

- **Use distinct job ids** so neither system claims work the other
  scheduled.
- **Watch the dashboard.** Any unexpected fires from Quartz are visible
  because *your* business code logs it — JobRunr can only show what it
  enqueued.
- **Switch one trigger at a time.** Tempting to bulk-migrate; cheaper to
  triage one regression than ten.

## Common mistakes

- **Assuming `@DisallowConcurrentExecution` semantics transfer.** They do
  for `@Recurring` jobs; they don't for `BackgroundJob.enqueue(...)`
  calls. Add an idempotency key or use Pro mutexes if you fire the same
  job twice.
- **Leaving QRTZ_ tables in place "just in case".** They keep getting
  polled by anything that still has the Quartz library on the classpath.
  Drop them once Quartz is removed.
- **Translating Quartz' 6-field cron literally.** JobRunr OSS expects 5
  fields — leading "seconds" zeros cause silent parse errors.

## Sources

- <https://www.jobrunr.io/en/documentation/background-methods/recurring-jobs/>
- <https://www.jobrunr.io/en/documentation/pro/advanced-cron-expressions/>
- Quartz reference: <https://www.quartz-scheduler.org/documentation/>
