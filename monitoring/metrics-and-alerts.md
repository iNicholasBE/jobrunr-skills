---
name: jobrunr-monitoring-metrics
description: Export JobRunr metrics via Micrometer (Prometheus, Datadog, JMX)
  and define alerts for stuck queues, failed jobs, and missing workers.
tier: oss
version: 8.6.0+
frameworks: [spring-boot, micronaut, quarkus]
---

# Metrics and alerting

Use this skill when JobRunr is running in production and you need
observability beyond the dashboard.

## Prerequisites

- JobRunr installed
- Micrometer on the classpath (Spring Boot / Micronaut pull this in
  transitively when actuators/metrics are enabled)
- A metrics backend (Prometheus, Datadog, JMX, ...)

## Enabling the two metric groups

**Background job server metrics** — server health (heartbeat, CPU/memory,
worker pool size). Enabled by default in Spring and Micronaut:

```properties
jobrunr.background-job-server.metrics.enabled=true
```

**Job metrics** — job counts per state. **Disabled by default** to keep
the polling cost low; enable explicitly when you want alerting:

```properties
jobrunr.jobs.metrics.enabled=true
```

If you're using a Spring Boot actuator-enabled app, the metrics show up at
<http://localhost:8080/actuator/metrics>. Without a framework, register the
Micrometer integration via the fluent API:

```java
JobRunr.configure()
    .useMicroMeter(new JobRunrMicroMeterIntegration(meterRegistry))
    // ...
```

## Exported metrics

**Server metrics** (one set per BackgroundJobServer):

```
jobrunr.background-job-server.first-heartbeat
jobrunr.background-job-server.last-heartbeat
jobrunr.background-job-server.poll-interval-in-seconds
jobrunr.background-job-server.process-cpu-load
jobrunr.background-job-server.process-free-memory
jobrunr.background-job-server.process-all-located-memory
jobrunr.background-job-server.system-cpu-load
jobrunr.background-job-server.system-free-memory
jobrunr.background-job-server.system-total-memory
jobrunr.background-job-server.worker-pool-size
```

**Job metrics**:

```
jobrunr.jobs.by-state
```

Drill in by state via a tag, e.g.
`jobrunr.jobs.by-state{state="SUCCEEDED"}`. States are `SCHEDULED`,
`ENQUEUED`, `PROCESSING`, `SUCCEEDED`, `FAILED`, `DELETED`.

## Sample Prometheus alerts

```yaml
groups:
  - name: jobrunr
    rules:
      - alert: JobRunrFailedJobsGrowing
        expr: increase(jobrunr_jobs_by_state{state="FAILED"}[15m]) > 5
        for: 5m
        annotations:
          summary: "More than 5 jobs moved to FAILED in the last 15 minutes"

      - alert: JobRunrServerHeartbeatStale
        expr: time() - jobrunr_background_job_server_last_heartbeat > 60
        for: 2m
        annotations:
          summary: "Background job server hasn't heartbeated in over a minute"

      - alert: JobRunrEnqueuedBacklog
        expr: jobrunr_jobs_by_state{state="ENQUEUED"} > 1000
        for: 10m
        annotations:
          summary: "Enqueued backlog over 1000 for 10+ minutes — workers are behind"
```

Tune thresholds to your throughput. Alerting on absolute backlog size is
fine when load is steady; alert on growth rate for variable workloads.

## Common mistakes

- **Enabling job metrics but forgetting the cost.** `jobrunr.jobs.metrics.enabled=true`
  causes the server to query the storage provider for state counts on
  every Micrometer scrape. On large deployments, raise the scrape interval
  rather than expecting free counts.
- **Alerting on `FAILED` count without considering retries.** Jobs cycle
  in and out of `FAILED` while retrying; an `increase()` over a window is
  more useful than the raw gauge.
- **Forgetting the server heartbeat alert.** The dashboard shows a green
  light per server, but humans don't watch dashboards at 3am. Heartbeat
  staleness is the single most important JobRunr alert.

## Pro: per-job timings

JobRunr Pro exports per-job duration histograms (`job.execution.time`)
labelled by job name. See <https://www.jobrunr.io/en/documentation/pro/observability/>.
Request a trial via `mcp__jobrunr-docs__request_jobrunr_pro_trial`.

## Sources

- <https://www.jobrunr.io/en/documentation/configuration/metrics/>
- <https://www.jobrunr.io/en/documentation/pro/observability/>
