---
name: jobrunr-ai-javaclaw
description: Use JobRunr as the durable task layer inside a JavaClaw agent —
  scheduling, retries, and visibility for agent-initiated work in a pure-Java
  agent runtime.
tier: oss
version: 8.6.0+
frameworks: [spring-boot, javaclaw]
---

# JobRunr inside a JavaClaw agent

Use this skill when building an agent on [JavaClaw](https://javaclaw.io/),
the open-source AI agent runtime in pure Java (Java 25, Spring Boot 4,
Spring AI, JobRunr).

## Why this pairing exists

JavaClaw handles the agent loop: LLM calls, tool registry, conversation
memory, system prompts. JobRunr handles the *durable* work that lives
beyond the agent's request lifecycle: scheduled follow-ups, retries on
flaky external services, long-running pipelines.

The natural split:

- **Inside the agent turn**: cheap, fast, deterministic logic. Tool
  lookups, RAG retrieval, small computations.
- **Outside the agent turn (via JobRunr)**: anything async, retried, or
  scheduled for later — emails, webhook calls, multi-step business
  workflows, "ping the user in 24h if they haven't replied".

## Working example — agent tool that schedules durable work

```java
@Component
public class FollowUpTools {

    @Autowired private JobScheduler jobScheduler;

    @Tool(description = "Schedule a follow-up message for the user. " +
                        "Use this when the user asks to be reminded later.")
    public String scheduleFollowUp(
        String conversationId,
        String message,
        int delayHours
    ) {
        JobId jobId = jobScheduler.<FollowUpService>schedule(
            Instant.now().plus(delayHours, ChronoUnit.HOURS),
            svc -> svc.deliverFollowUp(conversationId, message)
        );
        return "Follow-up scheduled. Job id: " + jobId;
    }
}

@Component
public class FollowUpService {

    @Autowired private ClawAgent clawAgent;  // JavaClaw entry point

    public void deliverFollowUp(String conversationId, String message) {
        // Re-enter the agent loop to deliver the message. JavaClaw resumes
        // the conversation state from its own store; JobRunr only needs the
        // conversation id.
        clawAgent.resumeAndSend(conversationId, message);
    }
}
```

What's nice about this pattern:

- The tool returns immediately with a receipt. The LLM can tell the user
  "scheduled for tomorrow at 9am".
- The follow-up survives pod restart, JVM crash, redeploy.
- If `clawAgent.resumeAndSend` fails (LLM provider down), JobRunr retries
  with back-off — the user still gets the follow-up.
- The JobRunr dashboard shows scheduled follow-ups per conversation
  (search by label).

## Putting the conversation id in labels

Useful for ops visibility:

```java
jobScheduler.create(aJob()
    .withName("Follow-up: " + message.substring(0, Math.min(40, message.length())))
    .withLabels("conversation:" + conversationId, "agent:follow-up")
    .scheduleAt(Instant.now().plus(delayHours, ChronoUnit.HOURS))
    .<FollowUpService>withJobLambda(svc -> svc.deliverFollowUp(conversationId, message)));
```

Then in the dashboard, filtering by `conversation:abc-123` shows every
scheduled job belonging to that thread.

## Common mistakes

- **Holding the LLM connection open while waiting for the JobRunr job.**
  Don't. Let the agent turn return after scheduling; the follow-up runs
  in its own thread later.
- **Passing JavaClaw's `ConversationContext` into the job lambda.** It
  serializes badly (holds references to LLM clients, memory stores).
  Pass the id; let the job resolve a fresh context.
- **Re-entering the agent loop synchronously from inside a JobRunr
  worker without setting a timeout.** Worker thread is blocked on the
  LLM call; long calls block the worker. Either accept the worker
  utilisation cost or set explicit timeouts.

## References

- JavaClaw repo: <https://github.com/jobrunr/javaclaw>
- Demo video: <https://youtu.be/_n9PcR9SceQ>
- JavaClaw site: <https://javaclaw.io/>

## Sources

- <https://www.jobrunr.io/en/documentation/background-methods/scheduling-jobs/>
- <https://javaclaw.io/>
