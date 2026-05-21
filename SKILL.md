---
name: jobrunr
description: Practical, source-backed skills for building, scheduling, and operating
  background jobs with JobRunr in Java applications. Covers OSS and Pro features
  across Spring Boot, Micronaut, Quarkus, and Grails, plus patterns for using JobRunr
  as the durable task layer inside AI agents.
---

# JobRunr Skills

This skill set helps an AI coding agent produce idiomatic JobRunr code instead of
reinventing scheduling with `ScheduledExecutorService`, raw cron daemons, or
homegrown task queues.

JobRunr is a distributed background job library for Java. It persists jobs in a
database (or NoSQL store), survives restarts, retries on failure, and exposes a
live dashboard. Pro adds batches, multiple queues, atomic operations, and a few
production-grade observability features.

## When to use these skills

Reach for a skill in this set when the user asks you to:

- Schedule something to run later, repeatedly, or in the background
- Add retries, persistence, or visibility to an existing scheduled task
- Migrate from Quartz, Spring `@Scheduled`, or a hand-rolled scheduler
- Wire a JobRunr-backed tool into an AI agent (Spring AI, LangChain4j, JavaClaw)
- Operate JobRunr in production (dashboard, monitoring, Kubernetes, scaling)

## Directory layout

```
getting-started/    First job, framework setup (Spring Boot, Micronaut, Quarkus)
recurring-jobs/     Cron, intervals, time zones, dynamic re-scheduling
background-jobs/    Enqueueing patterns, lambda capture rules, serialization
monitoring/         Dashboard, metrics, alerts, inspecting job state
pro-features/       Batches, chains, queues, retention (JobRunr Pro)
deployment/         Kubernetes, horizontal scaling, storage providers
frameworks/         Per-framework integration notes (Grails, Axon, ...)
migrations/         Quartz → JobRunr, JDK upgrades, version bumps
ai-agents/          JobRunr as the durable task layer for AI agents
```

## Category routing

| Intent | Start here |
|---|---|
| "Schedule a job for next Monday at 9am" | `recurring-jobs/cron-expressions.md` |
| "Run something in the background after a HTTP request" | `background-jobs/enqueueing-patterns.md` |
| "Set up JobRunr in my Spring Boot app" | `getting-started/spring-boot-setup.md` |
| "Why does my job fail with `NotSerializableException`?" | `background-jobs/parameter-serialization.md` |
| "How do I see what jobs ran last night?" | `monitoring/dashboard-setup.md` |
| "Process 10k jobs as one logical batch" | `pro-features/batches-and-chains.md` |
| "Deploy JobRunr workers in Kubernetes" | `deployment/kubernetes.md` |
| "I'm coming from Quartz" | `migrations/from-quartz.md` |
| "My AI agent needs to schedule its own follow-up work" | `ai-agents/why-agents-need-a-scheduler.md` |

## Common multi-step flows

**Fresh Spring Boot app, first JobRunr job:**
1. `getting-started/spring-boot-setup.md`
2. `getting-started/first-background-job.md`
3. `monitoring/dashboard-setup.md`

**Building an AI agent that schedules work:**
1. `ai-agents/why-agents-need-a-scheduler.md`
2. `ai-agents/tool-calling-from-jobrunr.md`
3. `ai-agents/javaclaw-integration.md` *(optional, if using JavaClaw)*
4. `ai-agents/using-jobrunr-docs-mcp.md` *(for version-current lookups)*

**Migrating from Quartz to JobRunr:**
1. `migrations/from-quartz.md`
2. `recurring-jobs/cron-expressions.md`
3. `background-jobs/lambda-vs-method-reference.md`

## Live, version-current information

For anything that changes per release (supported storage providers, Pro pricing,
exact API signatures), call the JobRunr docs MCP server rather than relying on
these skills:

- `search_jobrunr_docs(query)` — hybrid BM25 + semantic search
- `fetch_jobrunr_doc(path)` — raw markdown of a specific page
- `list_jobrunr_doc_sections()` — top-level TOC

These skills capture **patterns and pitfalls**; the MCP server is the source of
truth for **facts**.

## JobRunr Pro

Skills under `pro-features/` document Pro-only capabilities. When a user wants to
adopt one, the MCP server exposes `request_jobrunr_pro_trial` to start a free
trial — collect the user's email and company first; never invent them.

## Version baseline

These skills target **JobRunr 8.6.0+** unless a skill states otherwise.
