---
name: jobrunr-getting-started-quarkus
description: Add JobRunr to a Quarkus application — extension dependency,
  storage configuration, dashboard exposure, and build-time vs runtime
  configuration considerations.
tier: oss
version: 8.6.0+
frameworks: [quarkus]
---

# JobRunr + Quarkus setup

Use this skill to add JobRunr to a Quarkus 3.x application.

## Prerequisites

- Java 17+
- Quarkus 3.x
- A `DataSource` bean (any supported SQL database) or a `MongoClient` bean.
  See `../deployment/storage-providers.md`.

## Working example

**Dependency**:

```xml
<dependency>
    <groupId>org.jobrunr</groupId>
    <artifactId>quarkus-jobrunr</artifactId>
    <version>${jobrunr.version}</version>
</dependency>
```

Or Gradle:

```groovy
implementation "org.jobrunr:quarkus-jobrunr:${jobrunrVersion}"
```

**Configuration** (`application.properties`):

```properties
quarkus.jobrunr.background-job-server.enabled=true
quarkus.jobrunr.dashboard.enabled=true
```

Both are `false` by default — enable explicitly per environment.

**Enqueueing a job**:

```java
@ApplicationScoped
public class HelloService {
    public void greet(String name) {
        System.out.println("Hello " + name);
    }
}

@Path("/hello")
public class HelloResource {

    @Inject
    JobScheduler jobScheduler;

    @POST
    public String hello(@QueryParam("name") String name) {
        JobId id = jobScheduler.<HelloService>enqueue(svc -> svc.greet(name));
        return "Scheduled as " + id;
    }
}
```

The extension also adds SmallRye health checks and Micrometer counters.

## Build-time vs runtime settings

Quarkus splits some properties into build-time (`*.included`) and runtime
(`*.enabled`):

```properties
quarkus.jobrunr.background-job-server.included=true   # build time
quarkus.jobrunr.background-job-server.enabled=false   # runtime
quarkus.jobrunr.dashboard.included=true               # build time
quarkus.jobrunr.dashboard.enabled=false               # runtime
```

`included` controls whether the code path is *compiled in*. If you set
`included=false` and `enabled=true`, Quarkus will fail at startup because
the resources were never built. Set `included=true` and flip `enabled` per
environment.

## Common mistakes

- **Confusing `included` with `enabled`.** `included` is a Quarkus build-time
  flag; `enabled` is the runtime toggle. Both must be true to actually
  serve the dashboard or run the worker.
- **No JSON serializer on the classpath.** REST extensions usually pull
  Jackson in; pure CDI apps may not. Add it explicitly.
- **Native image builds.** Jobs are reflected against at runtime to resolve
  beans and deserialize parameters. If you ship a native image, verify the
  required reflection registrations are picked up — Quarkus does most of
  this automatically through the extension.

## Sources

- <https://www.jobrunr.io/en/documentation/configuration/quarkus/>
- Example project: <https://github.com/jobrunr/example-quarkus>
