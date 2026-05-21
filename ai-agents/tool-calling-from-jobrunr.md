---
name: jobrunr-ai-tool-calling
description: Pattern for calling LLM tools (Spring AI, LangChain4j) from inside
  a JobRunr background job — bounded cost, retries, idempotency, and how to
  avoid double-firing expensive tool calls.
tier: oss
version: 8.6.0+
frameworks: [spring-boot, micronaut, quarkus]
---

# Calling LLM tools from a JobRunr job

Use this skill when the body of a background job needs to make an LLM
call or invoke an LLM-callable tool. Typical setups: Spring AI's
`ChatClient`, LangChain4j's `ChatLanguageModel`, or a direct Anthropic /
OpenAI SDK call.

## Why this is a JobRunr-shaped problem

LLM calls are:

- **Slow** — seconds to tens of seconds per call.
- **Flaky** — rate limits, transient timeouts, occasional 5xx from the
  provider.
- **Expensive** — every retry costs tokens.

All three argue for moving them off the request thread into a background
job with explicit retry semantics and an idempotency key.

## Working example — Spring AI inside a JobRunr job

```java
@Component
public class SummarizationService {

    @Autowired private ChatClient chatClient;
    @Autowired private SummaryRepository summaryRepository;

    @Job(retries = 5, name = "Summarize document %0")
    public void summarize(UUID documentId) {
        // Idempotency check — bail if already done
        if (summaryRepository.existsByDocumentId(documentId)) {
            return;
        }

        String content = loadDocument(documentId);

        String summary = chatClient.prompt()
            .user("Summarize in 3 bullet points:\n\n" + content)
            .call()
            .content();

        summaryRepository.save(new Summary(documentId, summary));
    }
}
```

Enqueue from anywhere:

```java
BackgroundJob.<SummarizationService>enqueue(svc -> svc.summarize(documentId));
```

What this gives you:

- The HTTP request that triggered summarization returns in milliseconds.
- Up to 5 retries with exponential back-off if Anthropic/OpenAI returns
  a 429 or 5xx.
- Idempotency at the job entry — even if JobRunr retries after a
  successful save (network blip between save and ack), the second run
  short-circuits.

## Bounded cost — retry budget

The default retry count is 10. For LLM jobs that's *expensive* if every
attempt costs $0.10. Override per job:

```java
@Job(retries = 3)
public void expensiveLlmCall(...) { ... }
```

Or globally:

```properties
jobrunr.jobs.default-number-of-retries=3
```

JobRunr Pro adds per-exception retry policies — e.g. retry timeouts
aggressively, give up on `InvalidRequest` immediately. See
`../pro-features/` and request a trial via
`mcp__jobrunr-docs__request_jobrunr_pro_trial`.

## Idempotency by job id

If you don't want to dedupe via the application database, JobRunr Pro
gives you `JobIdentifier`:

```java
jobScheduler.<SummarizationService>enqueue(
    JobId.fromIdentifier("summarize-" + documentId),
    svc -> svc.summarize(documentId)
);
```

The second enqueue with the same identifier is a no-op while the first is
in flight. OSS doesn't support this — use a database uniqueness check
inside the job instead.

## Streaming LLM responses

Don't try to stream tokens through a JobRunr job back to the user — the
job result isn't accessible to the originating HTTP request once
`enqueue(...)` returned. Two reasonable patterns:

1. **Persist + poll**: job writes the result to DB; client polls a status
   endpoint.
2. **Persist + push**: job writes the result and pushes via SSE /
   WebSocket / Webhook from a separate handler.

If you must stream live to the user, do the LLM call inline in the
request handler — but accept that the request thread is blocked for
seconds.

## Common mistakes

- **Embedding the full prompt as a method argument.** Prompts are 10s of
  KB; inflates the job row and pollutes the storage provider. Pass an id
  and rebuild the prompt inside the job.
- **Retrying without idempotency.** A second `messageSender.send(...)`
  after a partial success means the user sees the message twice.
- **Catching the LLM SDK's exception and swallowing it.** JobRunr can only
  retry what it sees fail. Catch only what you can recover from inline;
  rethrow the rest.
- **Forgetting that the LLM client may be stateful.** Spring AI's
  `ChatClient` is fine to inject. A LangChain4j chain that holds memory
  state across calls is not — pass the conversation key, re-construct
  the chain inside the job.

## Sources

- <https://www.jobrunr.io/en/documentation/background-methods/dealing-with-exceptions/>
- <https://www.jobrunr.io/en/documentation/background-methods/best-practices/>
- Spring AI: <https://docs.spring.io/spring-ai/reference/>
- LangChain4j: <https://docs.langchain4j.dev/>
