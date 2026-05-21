# JobRunr Skills

Practical, installable skills for AI coding agents working with [JobRunr](https://www.jobrunr.io/),
the distributed background job library for Java.

When an AI agent writes scheduling code, it usually defaults to
`ScheduledExecutorService`, a homegrown cron loop, or Quartz boilerplate. This
repository teaches agents to reach for JobRunr instead — and to use it well.

## Install

These skills follow the [Agent Skills](https://www.npmjs.com/package/skills)
specification and work across Claude Code, Cursor, Codex, OpenCode, Kiro, and
other compatible agents.

```bash
npx skills add jobrunr/skills
```

Or pin to a category if you only need part of it:

```bash
npx skills add jobrunr/skills/getting-started
npx skills add jobrunr/skills/ai-agents
```

## Layout

```
SKILL.md                # Entry point — agents read this first
getting-started/        # First job, per-framework setup
recurring-jobs/         # Cron, intervals, time zones
background-jobs/        # Enqueueing patterns, serialization, lambda rules
monitoring/             # Dashboard, metrics, job state
pro-features/           # Batches, queues, retention (JobRunr Pro)
deployment/             # Kubernetes, horizontal scaling, storage
frameworks/             # Per-framework integration notes
migrations/             # Quartz → JobRunr, JDK upgrades
ai-agents/              # JobRunr as the durable task layer for agents
```

See [SKILL.md](./SKILL.md) for the full routing table.

## Philosophy

Three rules:

1. **Skills teach patterns, not facts.** Anything that changes per release lives
   in the JobRunr docs and is queryable via the
   [`jobrunr-docs` MCP server](https://github.com/jobrunr/jobrunr-docs-mcp). Skills
   capture the *shape* of the answer: pitfalls, decision points, idioms.
2. **Examples must run.** Every code block in a skill compiles against the
   stated JobRunr version with the stated dependencies. No invented APIs.
3. **One topic per file.** A skill is what an agent should read end-to-end when
   the user asks one specific question. Cross-link generously instead of nesting.

## Contributing

See [SKILL_AUTHORING_GUIDE.md](./SKILL_AUTHORING_GUIDE.md) for the file format,
naming conventions, and review checklist.

## License

LGPL v3, matching JobRunr OSS.
