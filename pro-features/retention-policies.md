---
name: jobrunr-pro-retention
description: Per-job custom delete policies in JobRunr Pro — control how long
  individual succeeded/failed jobs stick around, separately from the global
  defaults.
tier: pro
version: 8.6.0+
frameworks: [spring-boot, micronaut, quarkus]
---

# Custom delete policies (JobRunr Pro)

> This skill covers a JobRunr Pro feature.

Use this skill when JobRunr's global retention is too coarse — e.g. you
have a recurring job that fires every 5 minutes and floods the dashboard
with succeeded jobs, but you also want to keep failed jobs from a different
process around for a week so you can investigate.

## Prerequisites

- JobRunr Pro licence
- JobRunr installed

## How OSS retention works (the baseline)

```properties
jobrunr.background-job-server.delete-succeeded-jobs-after=36h
jobrunr.background-job-server.permanently-delete-deleted-jobs-after=72h
```

`SUCCEEDED` → `DELETED` after 36h, then permanently removed 72h later.
Failed jobs stay in `FAILED` until you manually delete them. These settings
apply globally to *all* jobs.

## Working example — per-job override

```java
@Job(deleteOnSuccess = "PT10M!PT10M")
public void checkForNewFiles() {
    if (Files.list(importDirectory).findAny().isPresent()) {
        BackgroundJob.<FileImportService>enqueue(svc -> svc.importAll());
    }
}
```

`deleteOnSuccess` and `deleteOnFailure` accept the format
`duration1!duration2` (both parts optional):

| Format | `Succeeded`/`Failed` → `Deleted` | `Deleted` → permanently removed |
|---|---|---|
| `PT10M` | After 10 minutes | Use global default |
| `PT10M!PT2H` | After 10 minutes | After 2 hours |
| `P2D!` | After 2 days | Immediately (skips `Deleted`) |
| `!PT2H` | Immediately | After 2 hours |

Durations are ISO-8601 (`PT10M`, `PT2H`, `P2D`).

## Working example — JobRequest handler

```java
public class ImportFilesHandler implements JobRequestHandler<ImportFilesJobRequest> {

    @Job(deleteOnSuccess = "PT10M!PT10M", deleteOnFailure = "P7D")
    public void run(ImportFilesJobRequest req) {
        // ...
    }
}
```

## Working example — JobBuilder

```java
jobScheduler.create(aJob()
    .withDeleteOnSuccess(Duration.ofMinutes(10))
    .withDeleteOnFailure(Duration.ofDays(7))
    .withJobLambda(() -> fileImportService.importAll()));
```

## When to reach for this

- A recurring "is there anything to do?" job that succeeds noisily 95% of
  the time → short `deleteOnSuccess`, default `deleteOnFailure`.
- Compliance-sensitive failed jobs you need to keep for audit →
  `deleteOnFailure = "P30D"`.
- Test/CI cleanup runs you don't want lingering → `deleteOnSuccess = "!"`
  (delete immediately, skip the `Deleted` state).

## Common mistakes

- **Setting retention shorter than your retry window.** A job with 10
  retries on exponential back-off can take hours to settle. If
  `deleteOnFailure` is `PT5M`, the job may be purged mid-retry.
- **Relying on `deleteOnSuccess` for GDPR/PII deletion.** Retention
  removes the job *row*, including the JSON-serialized arguments. That's
  not a substitute for never logging PII into job arguments in the first
  place.
- **Mixing `!` separator placement.** `PT10M` keeps the global "permanently
  delete after" default; `PT10M!` skips the `Deleted` state entirely. The
  separator changes the semantics.

## Enable in your project

Request a free JobRunr Pro trial via
`mcp__jobrunr-docs__request_jobrunr_pro_trial`.

## Sources

- <https://www.jobrunr.io/en/documentation/pro/custom-delete-policy/>
