---
name: jobrunr-monitoring-state-inspection
description: Look up a JobRunr job by id and read its state, history, and
  failure cause programmatically — from application code, the dashboard, or
  directly from the storage provider.
tier: oss
version: 8.6.0+
frameworks: [spring-boot, micronaut, quarkus]
---

# Inspecting job state

Use this skill when you have a `JobId` and need to know what happened — is
it scheduled, processing, succeeded, failed; if failed, why.

## Prerequisites

- JobRunr installed
- A `JobId` returned from `enqueue(...)` or known from the dashboard URL

## Working example

```java
@Inject
private StorageProvider storageProvider;

public JobReport inspect(JobId jobId) {
    try {
        Job job = storageProvider.getJobById(jobId);   // throws JobNotFoundException
        StateName state = job.getState();
        String name = job.getJobName();
        List<JobState> history = job.getJobStates();
        // The most recent state is at the end of history
        return new JobReport(state, name, history);
    } catch (JobNotFoundException e) {
        return JobReport.notFound(jobId);
    }
}
```

`StorageProvider.getJobById` is overloaded for both `JobId` and `UUID`. It
**throws `JobNotFoundException`** rather than returning `Optional` or
`null`.

If the job failed, the last state in `history` is a `FailedState` with the
exception class, message, and stack trace:

```java
if (job.getState() == StateName.FAILED) {
    FailedState fail = (FailedState) job.getJobStates().get(job.getJobStates().size() - 1);
    String exception = fail.getException();        // class name
    String message = fail.getMessage();
    String stackTrace = fail.getStackTrace();
}
```

## Reading state in bulk

For dashboards or admin endpoints, query by state:

```java
Page<Job> failed = storageProvider.getJobs(
    StateName.FAILED,
    PageRequest.descOnUpdatedAt(0, 50)
);
```

## Job retention — read jobs while they still exist

By default JobRunr OSS moves `SUCCEEDED` jobs to `DELETED` after 36 hours
and purges them after 72 hours. After that, `getJobById` throws
`JobNotFoundException`. Configure via:

```properties
jobrunr.background-job-server.delete-succeeded-jobs-after=36h
jobrunr.background-job-server.permanently-delete-deleted-jobs-after=72h
```

If you need long-term history of who ran what, copy the relevant fields
into your own table during the job — don't rely on the JobRunr storage as
an audit log.

## Raw SQL fallback

Sometimes the easiest way to triage at 2am is `psql`:

```sql
-- Job by id
SELECT id, state, jobAsJson
  FROM jobrunr_jobs
 WHERE id = '...';

-- Failed jobs in the last hour
SELECT id, updatedat, jobAsJson
  FROM jobrunr_jobs
 WHERE state = 'FAILED' AND updatedat > now() - interval '1 hour'
 ORDER BY updatedat DESC;
```

The full job, including history and exception, is in the `jobAsJson` column.

## Common mistakes

- **Assuming `SUCCEEDED` jobs stick around forever.** Retention deletes
  them. If you need to audit, persist the audit record yourself.
- **Reading `getState()` without checking history.** `getState()` is the
  current state; if you need "did this fail at any point and then succeed
  on retry?", walk `getJobStates()`.
- **Casting state without a state check.** The state list contains
  different subtypes (`ScheduledState`, `EnqueuedState`, `ProcessingState`,
  `SucceededState`, `FailedState`, `DeletedState`); cast only after
  checking.
- **Reading from a read replica.** Lag means you can see a job in
  `ENQUEUED` that's actually already `SUCCEEDED`. Read from the primary
  when triage matters.

## Sources

- JobRunr `StorageProvider` Javadoc: <https://repo.jobrunr.io/javadoc/releases/org/jobrunr/jobrunr/latest>
- <https://www.jobrunr.io/en/documentation/background-methods/dealing-with-exceptions/>
