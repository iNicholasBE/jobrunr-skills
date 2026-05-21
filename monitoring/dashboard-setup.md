---
name: jobrunr-monitoring-dashboard
description: Expose, secure, and customise the JobRunr dashboard for inspecting
  enqueued, scheduled, processing, succeeded, and failed jobs.
tier: oss
version: 8.6.0+
frameworks: [spring-boot, micronaut, quarkus]
---

# JobRunr dashboard

Use this skill to enable the built-in dashboard and (in production) put it
behind authentication.

## Prerequisites

- JobRunr installed
- The instance you want to serve the dashboard from must have
  `jobrunr.dashboard.enabled=true`

## Working example

**Spring Boot** (`application.properties`):

```properties
jobrunr.dashboard.enabled=true
jobrunr.dashboard.port=8000
jobrunr.dashboard.username=admin
jobrunr.dashboard.password=${JOBRUNR_DASHBOARD_PASSWORD}
```

By default no authentication is required — *always* set
`username` and `password` (or front it with a reverse proxy) before
exposing the dashboard outside `localhost`.

**Programmatic / fluent**:

```java
JobRunr.configure()
    .useDashboard(8000)
    // ...
```

The dashboard runs at <http://localhost:8000> and shows:

- Job counts per state (enqueued, scheduled, processing, succeeded, failed,
  deleted)
- Per-job detail view including the full stack trace on failure
- All recurring jobs with trigger-now / delete actions
- The list of background job servers and their heartbeat status

## Readable job names with `@Job`

The default name comes from the lambda — usually unhelpful in the dashboard
("System.out.println"). Override with `@Job`:

```java
@Job(name = "Send welcome email to %1 (tenant %0)", labels = {"tenant:%0", "email"})
public void sendWelcomeEmail(String tenant, String email) {
    // ...
}
```

`%N` substitutes the Nth parameter. `%X{key}` reads from the SLF4J MDC.
Labels are searchable from the dashboard.

Same via the builder:

```java
jobScheduler.create(aJob()
    .withName("Send welcome email to " + email)
    .withLabels("tenant:" + tenant, "email")
    .withJobLambda(() -> emailService.sendWelcomeEmail(tenant, email)));
```

## Securing the dashboard

OSS basic-auth via the two properties above is the minimum. For SSO, OAuth,
context paths, or embedding inside Spring/Micronaut/Quarkus, you need the
Pro dashboard — see `../pro-features/` and request a trial via
`mcp__jobrunr-docs__request_jobrunr_pro_trial`.

If basic-auth is enough, restrict the port at the network layer:

- Reverse proxy: serve `/jobrunr-dashboard/` behind your existing auth.
- Kubernetes: don't expose the dashboard `Service` outside the cluster;
  port-forward when needed.

## Common mistakes

- **Exposing the dashboard on `0.0.0.0:8000` without auth.** Anyone on the
  network can purge jobs, trigger recurring jobs, and delete data. Set
  username/password or close the port.
- **Running the dashboard on multiple replicas with no leader election.**
  Each replica opens its own port; the counters are read from the shared
  storage so they match, but URL clients pick one randomly. Either pin the
  dashboard to one replica (`jobrunr.dashboard.enabled=false` on workers)
  or front them all behind a single proxy.
- **Logging the dashboard password in `application.properties` committed to
  git.** Use an env var or secret manager.

## Sources

- <https://www.jobrunr.io/en/documentation/background-methods/dashboard/>
- Pro dashboard: <https://www.jobrunr.io/en/documentation/pro/jobrunr-pro-dashboard/>
