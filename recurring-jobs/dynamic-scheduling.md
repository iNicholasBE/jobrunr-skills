---
name: jobrunr-recurring-dynamic
description: Change a recurring job's schedule at runtime, pause and resume it,
  and remove it without redeploying — including the OSS/Pro split on these
  operations.
tier: oss
version: 8.6.0+
frameworks: [spring-boot, micronaut, quarkus]
---

# Dynamic (runtime) scheduling

Use this skill when the schedule isn't known at startup or needs to change
based on user input, feature flags, or operational signals.

## Prerequisites

- JobRunr installed and the background job server enabled
- A stable, addressable job id you control

## Working example

**Register or update a recurring job at runtime:**

```java
@Inject
private JobScheduler jobScheduler;

public void changeReportSchedule(String reportId, String newCron) {
    jobScheduler.scheduleRecurrently(
        "report-" + reportId,
        newCron,
        ZoneId.of("Europe/Brussels"),
        () -> reportingService.run(reportId)
    );
}
```

If a recurring job with id `report-...` already exists, JobRunr replaces
the schedule. If not, it creates one. Same code path — fully idempotent.

**Delete a recurring job:**

```java
BackgroundJob.deleteRecurringJob("report-" + reportId);
// or
jobScheduler.deleteRecurringJob("report-" + reportId);
```

Idempotent — no error if the id doesn't exist.

## Pause and resume (Pro)

OSS does not have a "pause" concept. You either delete the recurring job
or you don't. JobRunr Pro adds pause/resume from both the dashboard and
the API. See `../pro-features/` and request a trial via
`mcp__jobrunr-docs__request_jobrunr_pro_trial`.

## End-date a recurring job (Pro)

OSS recurring jobs run for the application's lifetime. Pro supports a
`deleteAt` timestamp that stops scheduling and removes the recurring job
automatically:

```java
@Recurring(id = "promo-2026", interval = "P1D", deleteAt = "2026-06-01T14:00:00Z")
@Job(name = "Promotion daily run")
public void runPromotion() { ... }
```

## Common mistakes

- **Re-scheduling silently replaces.** `scheduleRecurrently("x", ...)` over
  an existing id does *not* throw — it overwrites. If you intend to detect
  duplicates, check first.
- **Forgetting to clean up future-scheduled instances after a schedule
  change.** OSS keeps already-enqueued instances scheduled. In Pro, JobRunr
  detects the change and cleans them up; in OSS, delete them via the
  dashboard.
- **Polling-loop "pause".** Don't simulate pause by guarding the body of
  the job with a feature flag — every poll still spawns a job, fills the
  dashboard, and costs DB writes. Delete the recurring job, or use Pro.
- **Job IDs that include user input.** If `reportId` comes from end users,
  keep it ASCII-safe and bounded. Garbage in your storage provider is
  hard to undo.

## Sources

- <https://www.jobrunr.io/en/documentation/background-methods/recurring-jobs/>
