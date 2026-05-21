---
name: jobrunr-frameworks-grails
description: Use JobRunr inside a Grails application — Spring Boot starter
  underneath, GORM session handling in jobs, and Groovy-specific lambda
  considerations.
tier: oss
version: 8.6.0+
frameworks: [grails]
---

# JobRunr + Grails

Use this skill when adding JobRunr to a Grails 6.x application. Grails 6
runs on Spring Boot 3, so the `jobrunr-spring-boot-3-starter` works
unmodified — but there are GORM and Groovy details worth knowing.

## Prerequisites

- Grails 6.x (Spring Boot 3 under the hood)
- A supported storage provider — most Grails apps share JobRunr's storage
  with their main DataSource

## Working example

**Dependency** (`build.gradle`):

```groovy
implementation "org.jobrunr:jobrunr-spring-boot-3-starter:${jobrunrVersion}"
```

**Configuration** (`grails-app/conf/application.yml`):

```yaml
jobrunr:
  background-job-server:
    enabled: true
  dashboard:
    enabled: true
```

**A Grails service handling the job:**

```groovy
import org.springframework.stereotype.Component

@Component
class EmailService {

    def sendWelcomeEmail(Long userId) {
        User.withTransaction {
            def user = User.get(userId)
            // send mail
            user.welcomeEmailSent = true
            user.save(flush: true)
        }
    }
}
```

**Enqueueing from a controller:**

```groovy
import org.jobrunr.scheduling.JobScheduler
import org.springframework.beans.factory.annotation.Autowired

class UserController {

    @Autowired JobScheduler jobScheduler

    def register(Long userId) {
        jobScheduler.enqueue(EmailService) { svc -> svc.sendWelcomeEmail(userId) }
        render "queued"
    }
}
```

JobRunr's ASM analyzer handles Groovy closures the same way it handles
Java lambdas, as long as you use the type-token form (`enqueue(EmailService) { ... }`).

## GORM session handling

Workers run jobs outside of any web request — there is no Hibernate
session bound to the thread by default. Wrap DB access in
`Domain.withTransaction { ... }` or `Domain.withSession { ... }` to
materialise lazy fields:

```groovy
def sendWelcomeEmail(Long userId) {
    User.withTransaction {
        def user = User.get(userId)
        def email = user.emailAddress   // safe inside withTransaction
    }
}
```

Accessing lazy properties of a detached entity outside a session throws
`LazyInitializationException`. This is the most common Grails-on-JobRunr
issue.

## Companion guide and example repo

JobRunr publishes a dedicated Grails guide and a runnable example:

- <https://www.jobrunr.io/en/guides/jvm-frameworks/grails/>
- <https://github.com/jobrunr/example-grails>

## Common mistakes

- **Groovy closure capturing the GORM `Domain.list()` result.** Pass IDs,
  not domain objects — see `../background-jobs/parameter-serialization.md`.
  Domain objects with proxies and lazy fields don't round-trip cleanly.
- **No session in the job body.** Always open one explicitly with
  `withTransaction` or `withSession`.
- **Mixing Grails' `Quartz`/`grails-async` jobs with JobRunr `@Recurring`.**
  Two schedulers fire twice. Migrate gradually and disable the old one.

## Sources

- <https://www.jobrunr.io/en/guides/jvm-frameworks/grails/>
- <https://github.com/jobrunr/example-grails>
- <https://www.jobrunr.io/en/documentation/configuration/spring/>
