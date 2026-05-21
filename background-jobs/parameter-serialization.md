---
name: jobrunr-bg-serialization
description: How JobRunr serializes job parameters with Jackson/Gson/JSON-B/Kotlin
  Serialization, which types are safe, and how to diagnose serialization failures.
tier: oss
version: 8.6.0+
frameworks: [spring-boot, micronaut, quarkus]
---

# Job parameter serialization

Use this skill when a job won't enqueue, fails on dequeue, or behaves with
stale data when it runs.

## How JobRunr serializes

When you enqueue a job, JobRunr serializes the method signature and every
argument to JSON, then stores the JSON in the storage provider. When the
job runs, the JSON is deserialized back into argument values.

JobRunr auto-detects which JSON library is on the classpath, in this
preference order:

1. Kotlin Serialization
2. Jackson 3
3. Jackson 2
4. Gson
5. JSON-B

If you don't have a JSON library transitively (common in non-web apps),
you must add one explicitly — `jackson-databind`, `gson`, or the
equivalent.

## What is safe to pass

- **Primitives** (`int`, `long`, `boolean`, etc.)
- **`String`**, `UUID`, `Instant`, `Duration`, `LocalDate`, `LocalDateTime`
- **Java collections** (`List`, `Set`, `Map`) of the above
- **Plain POJOs** with a no-arg constructor (can be `private`) and either
  public fields or getters/setters

## What is NOT safe to pass

- **JPA / Hibernate entities** — proxies, lazy fields, and the persistence
  context don't survive serialization. The deserialized object is a
  detached snapshot, often missing relationships.
- **Spring/Micronaut/Quarkus beans by value** — pass the bean *type* (so
  the worker resolves it via the IoC container), not an instance.
- **`DataSource`, `Connection`, anything resource-like.** These can't be
  serialized and shouldn't escape the JVM that owns them anyway.
- **Lambdas, `Runnable`, `Function`.** JobRunr analyzes one outer lambda
  at enqueue time; nested ones can't be reconstructed.
- **Large JSON payloads (>tens of KB).** Inflates your storage provider
  and slows polling. Pass an id and re-load.

## Working example — pass the id, not the entity

```java
// Wrong
User user = userRepository.findById(id).orElseThrow();
BackgroundJob.enqueue(() -> emailService.sendTo(user));

// Right
BackgroundJob.<EmailService>enqueue(svc -> svc.sendTo(id));
```

Inside the job:

```java
@Component
public class EmailService {
    @Inject private UserRepository userRepository;

    public void sendTo(UUID userId) {
        User user = userRepository.findById(userId).orElseThrow();
        // ...
    }
}
```

This pattern handles "the entity changed between enqueue and run" naturally
and keeps job rows small.

## Custom types

A custom POJO works if your chosen JSON library can round-trip it:

```java
public class Mail {
    private final String from;
    private final String to;
    private final String subject;
    private final String body;

    private Mail() { this(null, null, null, null); } // no-arg ctor for Jackson
    public Mail(String from, String to, String subject, String body) { ... }
    // getters
}

Mail mail = new Mail("a", "b", "Hi", "Body");
BackgroundJob.<EmailService>enqueue(svc -> svc.send(mail));
```

If your stack uses Kotlin Serialization, annotate the class with
`@Serializable`. With Jackson, ensure the type isn't behind a generic
wildcard or a sealed hierarchy your mapper can't navigate.

## Diagnosing failures

- **Job stays `ENQUEUED` and never runs**: usually means deserialization
  failed silently or the worker can't resolve the receiver. Check the
  background-job-server logs.
- **`NotSerializableException` / Jackson error at `enqueue(...)` time**:
  one of the lambda arguments isn't representable. Fix by simplifying the
  argument or passing an id.
- **Stale data inside the job**: the argument was an entity snapshot from
  enqueue time. Switch to id + re-load.

## Common mistakes

- **Switching JSON mapper between deploys.** JobRunr does not guarantee
  cross-mapper interoperability. Jobs persisted with Jackson 2 are not
  guaranteed to deserialize cleanly with Gson.
- **Method signature changed but old jobs still in queue.** When the
  receiver method's signature changes (parameter added/removed/renamed),
  enqueued jobs against the old shape will fail. Drain the queue or write
  a compatibility shim before redeploying.
- **No JSON library on the classpath.** Pure CDI / Spring Cloud Function
  apps may not pull Jackson transitively. Add it explicitly.

## Sources

- <https://www.jobrunr.io/en/documentation/serialization/>
- <https://www.jobrunr.io/en/documentation/background-methods/passing-arguments/>
- <https://www.jobrunr.io/en/documentation/background-methods/best-practices/>
