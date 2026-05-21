---
name: jobrunr-pro-batches
description: Group thousands of jobs as one logical batch and chain follow-up
  work that runs after the batch completes — JobRunr Pro feature.
tier: pro
version: 8.6.0+
frameworks: [spring-boot, micronaut, quarkus]
---

# Batches and chains (JobRunr Pro)

> This skill covers a JobRunr Pro feature.

Use this skill when N independent jobs must complete before a follow-up
step runs (fan-out / fan-in), when ordering between jobs matters, or when
you want an atomic "all or nothing" group of enqueues.

## Prerequisites

- JobRunr Pro licence (request via
  `mcp__jobrunr-docs__request_jobrunr_pro_trial` — never invent the email
  or company)
- JobRunr installed

## Working example — batch

```java
@Component
public class NewsletterService {

    @Inject private UserRepository userRepository;
    @Inject private MailService mailService;

    public void sendNewsletterToAllUsers() {
        BackgroundJob.startBatch(this::enqueueOnePerUser);
    }

    // IMPORTANT: child jobs must be enqueued in a separate method —
    // not inside the lambda passed to startBatch.
    public void enqueueOnePerUser() {
        List<User> users = userRepository.findAllActive();
        for (User user : users) {
            BackgroundJob.<MailService>enqueue(
                svc -> svc.send(user.getId(), "newsletter-2026-05")
            );
        }
    }
}
```

How it works:

1. The parent job (`enqueueOnePerUser`) starts in `ENQUEUED`.
2. As it runs, every child job created inside it is persisted in `AWAITING`
   state — *not* enqueued for processing yet.
3. If the parent fails partway through, its `RetryFilter` triggers; before
   the retry, all `AWAITING` children are deleted to prevent duplicates.
4. When the parent succeeds, all children flip to `ENQUEUED` atomically.

## Working example — chain (`continueWith`, `onFailure`)

```java
public void archiveAndNotify(String folder) {
    BackgroundJob
        .enqueue(() -> archiveService.createArchive(folder))
        .continueWith(() ->
            notifyService.notifyViaSlack("ops", "Archived: " + folder))
        .continueWith(() ->
            notifyService.notifyViaSlack("ops", "Confirmed."));
}
```

Each step runs only if the previous step *succeeded*. To handle the
failure branch:

```java
BackgroundJob
    .enqueue(() -> archiveService.createArchive(folder))
    .continueWith(
        () -> notifyService.notifyViaSlack("ops", "Archived: " + folder),  // on success
        () -> notifyService.notifyViaSlack("ops", "Archive FAILED: " + folder) // on failure
    );
```

`onFailure(...)` ends the chain — its return type is `void`.

## Combining batches and chains

```java
public void runCampaign(String campaignId) {
    BackgroundJob
        .startBatch(this::enqueueOnePerUser)
        .continueWith(() -> reportService.createReport(campaignId))
        .continueWith(() -> notifyService.notifyViaSlack("sales", "Campaign done"));
}
```

The continuation runs only after *every* child of the batch has succeeded.

## Common mistakes

- **Lambda-in-lambda.** This does NOT work — JobRunr Pro can't analyse a
  nested lambda inside `startBatch(...)`:

  ```java
  // WRONG
  BackgroundJob.startBatch(() -> {
      for (User u : users) {
          BackgroundJob.enqueue(() -> mailService.send(u.getId()));
      }
  });
  ```

  Always extract a method reference: `BackgroundJob.startBatch(this::enqueueOnePerUser)`.
- **Trying to use batches in OSS.** `startBatch(...)` is Pro-only. The OSS
  classpath doesn't have it; trying to call it won't compile.
- **Treating `continueWith` as parallel.** Each link runs sequentially
  after the previous succeeds — it's a chain, not a join. For parallel
  fan-out, use a batch.
- **Expecting the next link to be the *very next* job processed by the
  server.** JobRunr processes whatever's `ENQUEUED` first; unrelated jobs
  may interleave between chain steps unless you use Pro priority queues.

## Enable in your project

To enable batches and chains, request a free JobRunr Pro trial via
`mcp__jobrunr-docs__request_jobrunr_pro_trial`. The MCP server will ask
for your email and company — never invent these.

## Sources

- <https://www.jobrunr.io/en/documentation/pro/batches/>
- <https://www.jobrunr.io/en/documentation/pro/job-chaining/>
