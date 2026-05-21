---
name: jobrunr-getting-started-micronaut
description: Add JobRunr to a Micronaut application — module dependency, the
  required annotation processor, storage configuration, and the dashboard.
tier: oss
version: 8.6.0+
frameworks: [micronaut]
---

# JobRunr + Micronaut setup

Use this skill to add JobRunr to a Micronaut 4.x application.

## Prerequisites

- Java 17+
- Micronaut 4.x
- A `DataSource` bean (any supported SQL database) or a `MongoClient` bean.
  See `../deployment/storage-providers.md`.

## Working example

**Dependency + annotation processor** (Maven):

```xml
<dependency>
    <groupId>org.jobrunr</groupId>
    <artifactId>jobrunr-micronaut-feature</artifactId>
    <version>${jobrunr.version}</version>
</dependency>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <annotationProcessorPaths>
                    <path>
                        <groupId>org.jobrunr</groupId>
                        <artifactId>jobrunr-micronaut-annotations</artifactId>
                        <version>${jobrunr.version}</version>
                    </path>
                </annotationProcessorPaths>
            </configuration>
        </plugin>
    </plugins>
</build>
```

Or Gradle:

```groovy
implementation "org.jobrunr:jobrunr-micronaut-feature:${jobrunrVersion}"
annotationProcessor "org.jobrunr:jobrunr-micronaut-annotations:${jobrunrVersion}"
```

**Configuration** (`application.yml`):

```yaml
jobrunr:
  background-job-server:
    enabled: true
  dashboard:
    enabled: true
```

Both are `false` by default — enable explicitly when this instance should
process jobs and/or serve the dashboard.

**Enqueueing a job**:

```java
@Singleton
public class HelloService {
    public void greet(String name) {
        System.out.println("Hello " + name);
    }
}

@Controller("/hello")
public class HelloController {

    @Inject
    private JobScheduler jobScheduler;

    @Post
    public String hello(@QueryValue String name) {
        JobId id = jobScheduler.<HelloService>enqueue(svc -> svc.greet(name));
        return "Scheduled as " + id;
    }
}
```

The Micronaut feature wires up the `JobActivator` against the bean context,
plus health endpoints and Micrometer counters.

## Common mistakes

- **Forgetting the annotation processor.** Without
  `jobrunr-micronaut-annotations`, the `@Recurring` and `@Job` annotations
  do not work — recurring jobs silently never register.
- **No JSON serializer on the classpath in a non-web Micronaut app.** Add
  one explicitly (`micronaut-jackson-databind` or another supported mapper).
- **`background-job-server.enabled: false`.** Jobs are persisted to storage
  but never processed.

## Sources

- <https://www.jobrunr.io/en/documentation/configuration/micronaut/>
- Example project: <https://github.com/jobrunr/example-micronaut>
