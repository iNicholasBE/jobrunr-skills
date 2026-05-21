---
name: jobrunr-frameworks-spring-boot
description: Spring Boot-specific patterns beyond initial setup — testing with
  the in-memory provider, transactional-enqueue safety, profile-based toggles,
  and Spring Security on the dashboard.
tier: oss
version: 8.6.0+
frameworks: [spring-boot]
---

# JobRunr + Spring Boot — patterns

Use this skill *after* `../getting-started/spring-boot-setup.md` when you
need deeper Spring Boot integration patterns.

## Prerequisites

- Spring Boot 3.x or 4.x
- `jobrunr-spring-boot-3-starter` or `jobrunr-spring-boot-4-starter`

## Testing with the in-memory provider

For unit and integration tests, swap the storage provider:

```java
@TestConfiguration
public class JobRunrTestConfig {

    @Bean
    @Primary
    public StorageProvider inMemoryStorageProvider() {
        InMemoryStorageProvider sp = new InMemoryStorageProvider();
        sp.setJobMapper(new JobMapper(new JacksonJsonMapper()));
        return sp;
    }
}
```

In `application-test.properties`:

```properties
jobrunr.background-job-server.enabled=false
jobrunr.dashboard.enabled=false
```

Then enqueue, manually trigger a worker thread or invoke the job directly
in the test, and assert. The in-memory provider is per-JVM; do not
combine with `@SpringBootTest(webEnvironment = RANDOM_PORT)` and expect
isolation across parallel tests in the same JVM.

## Enqueueing inside a `@Transactional` boundary

The standard pitfall:

```java
@Transactional
public Order placeOrder(OrderRequest req) {
    Order order = orderRepository.save(new Order(req));
    BackgroundJob.<EmailService>enqueue(svc -> svc.sendConfirmation(order.getId()));
    // If anything below throws, the order rolls back BUT the job stays
    // in the JobRunr storage and runs against a non-existent order.
    riskService.assessOrder(order);
    return order;
}
```

Two fixes, in order of preference:

**1. Enqueue after commit:**

```java
TransactionSynchronizationManager.registerSynchronization(
    new TransactionSynchronization() {
        @Override
        public void afterCommit() {
            BackgroundJob.<EmailService>enqueue(svc -> svc.sendConfirmation(order.getId()));
        }
    });
```

**2. JobRunr Pro transactional integration:**

```properties
# Pro: jobs are enqueued in the same tx; if it rolls back, no job survives.
```

See <https://www.jobrunr.io/en/documentation/pro/transactions/>.

## Disabling JobRunr in `test` / `local` profiles

Often you don't want recurring jobs firing in CI or while a dev is
debugging. Spring profile-scoped properties:

```properties
# application-test.properties
jobrunr.background-job-server.enabled=false
jobrunr.dashboard.enabled=false
jobrunr.job-scheduler.enabled=false   # also disables @Recurring registration
```

## Securing the dashboard with Spring Security

The OSS dashboard runs on its own embedded HTTP server, not the Spring
context. Spring Security can't see it directly. Options:

1. **Use the basic-auth properties** (simplest):

   ```properties
   jobrunr.dashboard.username=admin
   jobrunr.dashboard.password=${JOBRUNR_DASHBOARD_PASSWORD}
   ```

2. **Front it with a reverse proxy** that enforces Spring Security or your
   IDP. The proxy is the auth boundary; JobRunr binds to localhost only.

3. **Use JobRunr Pro dashboard** which embeds in Spring/Micronaut/Quarkus
   and integrates with your existing security chain.

## Spring `@Scheduled` vs. JobRunr `@Recurring`

Don't use both for the same task. `@Scheduled` runs in-process only — if
the pod restarts mid-task, work is lost. `@Recurring` persists to JobRunr
storage and survives restarts. Migration path: take the `@Scheduled`
method, rename if needed, change the annotation to `@Recurring`, register
the bean with the JobRunr id.

## Common mistakes

- **Forgetting to disable JobRunr in tests.** Recurring jobs fire during
  `@SpringBootTest` startup; CI shows unrelated failures.
- **In-memory provider leaking into `@DataJpaTest`.** `@DataJpaTest`
  loads a minimal context that may still trigger JobRunr autoconfig.
  Exclude it: `@DataJpaTest(excludeAutoConfiguration = {JobRunrAutoConfiguration.class})`
  (adjust class name for your starter version).
- **Running `@Scheduled` and `@Recurring` on the same logic.** Two
  schedulers, two runs, double sends.

## Sources

- <https://www.jobrunr.io/en/documentation/configuration/spring/>
- <https://www.jobrunr.io/en/documentation/pro/transactions/>
