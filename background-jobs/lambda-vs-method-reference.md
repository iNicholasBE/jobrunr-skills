---
name: jobrunr-bg-lambda-capture
description: Understand what JobRunr captures from a lambda or method reference
  when enqueueing — what survives serialization, what doesn't, and the patterns
  that silently break across redeploys.
tier: oss
version: 8.6.0+
frameworks: [spring-boot, micronaut, quarkus]
---

# Lambdas vs. method references in `enqueue(...)`

Use this skill to understand the single most common JobRunr footgun: what
the framework actually captures when you call `enqueue(() -> ...)`.

## How JobRunr records a job

JobRunr does **not** serialize the lambda itself. Instead, it inspects the
lambda with ASM at enqueue time and extracts:

- The **target class** (e.g. `EmailService`)
- The **method name** and signature
- The **literal argument values** passed at the call site

Those three things are serialized to JSON and stored. When the job runs,
JobRunr asks the IoC container for a fresh `EmailService` instance and
invokes the method with the deserialized arguments.

This has three consequences:

1. The lambda must call exactly *one* method on exactly *one* receiver.
2. Anything captured from the enclosing scope is captured as a *value* at
   enqueue time — closure state is not re-evaluated when the job runs.
3. The receiver instance you "see" inside the lambda is not the one used
   to run the job. The container resolves a new one.

## Working examples — right and wrong

**Right** — type-token form, plain value arguments:

```java
UUID userId = ...;
BackgroundJob.<EmailService>enqueue(svc -> svc.send(userId, "welcome"));
```

**Right** — instance form, plain value arguments:

```java
BackgroundJob.enqueue(() -> emailService.send(userId, "welcome"));
```

**Wrong** — `Instant.now()` captured by value:

```java
BackgroundJob.enqueue(() -> emailService.send(userId, Instant.now()));
// Stores the instant at *enqueue* time, not "now" when the job runs.
// If you wanted the latter, compute it inside the method.
```

**Wrong** — calling a method on a captured argument:

```java
BackgroundJob.enqueue(() -> emailService.send(user.getId()));
// JobRunr sees this as: call emailService.send with the value of user.getId()
// evaluated NOW. Usually fine, but easy to confuse with the next case.
```

**Wrong** — entity captured by reference:

```java
User user = userRepository.findById(id).orElseThrow();
BackgroundJob.enqueue(() -> emailService.sendTo(user));
// 'user' is serialized as JSON. JPA proxies, lazy fields, the persistence
// context — all gone. By the time the job runs, the DB row may have changed.
// Pass user.getId() instead and re-load inside the job.
```

**Wrong** — multiple statements:

```java
BackgroundJob.enqueue(() -> {
    User u = userRepository.findById(id).get();
    emailService.send(u.getEmail());
});
// JobRunr only analyzes a single method call. Multi-statement lambdas
// either fail at enqueue time or capture state you didn't expect.
```

## Keep the lambda small

From the best-practices doc: the more instructions inside the lambda, the
slower ASM analysis is. Hoist computation outside the lambda:

```java
// Slower — DTO conversion happens inside the lambda
BackgroundJob.enqueue(svc -> svc.generateReport(
    dtoConverter.toDto(new ReportRequest(Instant.now(), svc.previous()))));

// Faster — DTO is a plain value argument
ReportRequestDto dto = dtoConverter.toDto(new ReportRequest(Instant.now(), svc.previous()));
BackgroundJob.enqueue(svc -> svc.generateReport(dto));
```

## Common mistakes

- **Capturing `this` in an inner class context.** When the outer class is
  redeployed and the JVM restarts, the `this` reference doesn't exist.
  Use the type-token form instead.
- **Capturing non-final locals.** Java requires effectively final, but
  even then the captured *value* is what survives — not the variable.
- **Lambda body that does multiple things.** Move work into a service
  method and call that single method from the lambda.
- **Lambda inside a lambda.** Batches and bulk enqueue analyze lambdas;
  if you nest, the analyzer can't trace it. See `../pro-features/batches-and-chains.md`.

## Sources

- <https://www.jobrunr.io/en/documentation/background-methods/best-practices/>
- <https://www.jobrunr.io/en/documentation/background-methods/passing-arguments/>
