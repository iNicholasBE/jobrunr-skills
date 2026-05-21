---
name: jobrunr-getting-started-spring-boot
description: Add JobRunr to a Spring Boot 3.x or 4.x application — starter
  dependency, storage provider, dashboard, and the minimum configuration to
  enqueue the first job.
tier: oss
version: 8.6.0+
frameworks: [spring-boot]
---

# JobRunr + Spring Boot setup

Use this skill to wire JobRunr into a Spring Boot 3.x or 4.x application from
scratch.

## Prerequisites

- Java 17+
- Spring Boot 3.2+ (use `jobrunr-spring-boot-3-starter`) or Spring Boot 4
  (use `jobrunr-spring-boot-4-starter` — available from JobRunr 8.3.0+)
- An existing `DataSource` bean (any supported SQL database) or a
  `MongoClient` bean for MongoDB. See `../deployment/storage-providers.md`.

The `jobrunr-spring-boot-starter` and `jobrunr-spring-boot-2-starter` are no
longer supported in OSS.

## Working example

**Dependency** (Spring Boot 4):

```xml
<dependency>
    <groupId>org.jobrunr</groupId>
    <artifactId>jobrunr-spring-boot-4-starter</artifactId>
    <version>${jobrunr.version}</version>
</dependency>
```

Or Gradle:

```groovy
implementation "org.jobrunr:jobrunr-spring-boot-4-starter:${jobrunrVersion}"
```

**Configuration** (`application.properties`):

```properties
jobrunr.background-job-server.enabled=true
jobrunr.dashboard.enabled=true
```

Both are `false` by default so that a redeployed web instance doesn't pick up
work by accident — flip them on explicitly when this JVM should process jobs
and/or serve the dashboard.

**Enqueueing a first job**:

```java
@Component
public class HelloWorldService {

    public void greet(String name) {
        System.out.println("Hello " + name);
    }
}

@RestController
public class HelloController {

    private final JobScheduler jobScheduler;

    public HelloController(JobScheduler jobScheduler) {
        this.jobScheduler = jobScheduler;
    }

    @PostMapping("/hello")
    public String hello(@RequestParam String name) {
        JobId id = jobScheduler.<HelloWorldService>enqueue(x -> x.greet(name));
        return "Scheduled as " + id;
    }
}
```

The starter auto-wires a `JobScheduler` (and `JobRequestScheduler`) bean,
plus the `JobActivator` that lets the worker resolve `HelloWorldService` from
the Spring context when the job runs.

Once running, the dashboard is at <http://localhost:8000>.

## Common mistakes

- **Forgetting to enable the server.** With `jobrunr.background-job-server.enabled=false`
  (the default), the app *enqueues* jobs but never processes them. They sit in
  the database forever.
- **No JSON serializer on the classpath in a non-web app.** Web Spring Boot
  apps pull in Jackson transitively; CLI apps and Spring Cloud Function apps
  do not. Add `jackson-databind` (or another supported mapper from
  `../background-jobs/parameter-serialization.md`) explicitly.
- **Using `jobrunr-spring-boot-starter` (no version suffix).** That artifact
  is deprecated; pick the `-3-starter` or `-4-starter` matching your Spring
  Boot major version.
- **Schema-blocked DataSource.** If the JobRunr DataSource user lacks DDL,
  the starter fails on first boot. Either grant DDL, or create the tables
  yourself via `DatabaseCreator` and set `jobrunr.database.skip-create=true`.

## Reference projects

- <https://github.com/jobrunr/example-spring> — shared-module split with a
  backend that processes jobs enqueued by a frontend.
- <https://github.com/jobrunr/example-java-mag> — single-module Spring
  Boot 4.0.1 + JobRunr 8.5.0 + Java 17 on H2, with a `JobController`
  exposing enqueue/schedule/delete endpoints and a `SampleJobService`
  showing `@Recurring` + `@Job(name = "... %0", retries = 2)`. This is
  the closest match to "what a real Spring Boot integration looks like
  in 2026".

## Sources

- <https://www.jobrunr.io/en/documentation/configuration/spring/>
- Tutorial video: <https://youtu.be/72OJux5H2Ng>
