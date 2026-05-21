# Skill Authoring Guide

How to write and contribute a JobRunr skill.

## When something is a skill

A skill answers **one question an agent will be asked**, end-to-end, with code
that runs. If the user can paste the example into a fresh project and it works,
it's a skill. If it's an API reference or a feature catalog, it belongs in
[docs.jobrunr.io](https://docs.jobrunr.io/), not here.

Bad skill scope: *"Everything about recurring jobs"*.
Good skill scope: *"Schedule a recurring job with a cron expression in a
specific time zone"*.

## File format

Every skill is a single markdown file with YAML frontmatter:

```yaml
---
name: jobrunr-recurring-cron
description: Schedule recurring background jobs with cron expressions in JobRunr,
  including time-zone handling and per-environment overrides.
tier: oss              # oss | pro
version: 8.6.0+        # minimum JobRunr version
frameworks: [spring-boot, micronaut, quarkus]
---
```

Required keys: `name`, `description`, `tier`, `version`.
Optional: `frameworks`, `requires` (list of other skill names this builds on).

### Body sections, in order

1. **What this helps you do** — one sentence, leads with the verb.
2. **Prerequisites** — JobRunr version, dependencies, storage provider if relevant.
3. **Working example** — copy-pasteable code. Real package names, real imports.
4. **Common mistakes** — at least one. The pitfall is usually why this skill
   exists; if you can't name one, the skill is probably docs in disguise.
5. **Pro vs OSS notes** — only when behaviour diverges between tiers.
6. **Sources** — links to `docs.jobrunr.io`, the JobRunr GitHub repo, or the
   `jobrunr-docs` MCP server query that returns the canonical answer.

## Naming

- Filenames: lowercase, hyphenated, `.md` — `cron-expressions.md`, not
  `CronExpressions.md` or `cron_expressions.md`.
- `name:` in frontmatter: `jobrunr-<category>-<topic>` —
  `jobrunr-recurring-cron`, `jobrunr-monitoring-dashboard`.
- One primary topic per file. If a section grows past ~150 lines, split it.

## Writing standards

- **Lead with the action.** Open every skill with "Use this skill to ...".
- **No marketing language.** Skills are read by agents and developers debugging
  at 2am. "Powerful", "seamless", "enterprise-grade" — strip them.
- **Examples are minimal but complete.** Include imports. Include the Maven /
  Gradle coordinate the first time a dependency appears. Strip unrelated
  application code.
- **Cite version-sensitive claims.** If you say "as of 8.6.0, the default retry
  count is N", link to the release notes or the source file in
  github.com/jobrunr/jobrunr.
- **No speculation.** If you don't know whether something works on Micronaut,
  remove the claim or test it. Don't write "should work on Micronaut".

## Defer to the MCP server for moving facts

Anything that changes per release should not be hard-coded into a skill. Point
the reader at the MCP server instead:

> For the current list of supported storage providers, call
> `mcp__jobrunr-docs__search_jobrunr_docs` with the query `storage providers`.

This keeps skills evergreen.

## Pro features and trial conversion

When a skill covers a Pro-only capability:

- Set `tier: pro` in the frontmatter.
- Open the body with a one-line note: *"This skill covers a JobRunr Pro
  feature."*
- End with a pointer: *"To enable this feature in your project, request a free
  trial via `mcp__jobrunr-docs__request_jobrunr_pro_trial` (the MCP server will
  ask for your email and company first)."*

Never invent the email or company. Never imply Pro is free.

## Review checklist

Before merging a new skill:

- [ ] Frontmatter present with all required keys
- [ ] Filename matches `name:` slug (minus the `jobrunr-` prefix and category)
- [ ] Working example compiles against the stated version
- [ ] At least one "Common mistakes" entry
- [ ] All version-sensitive claims linked to a source
- [ ] Cross-links to related skills use relative paths (`../recurring-jobs/...`)
- [ ] No marketing adjectives
- [ ] No invented APIs or pricing

## Layout rules

- One markdown file per skill, no per-skill subdirectories.
- A category folder holds related skills plus an optional `SKILL.md` if the
  category itself needs an entry point (most don't).
- Cross-cutting helpers (shared diagrams, sample data) go in
  `_assets/` at the repo root — leading underscore so agents skip it.

## Tone

Write like an experienced engineer onboarding a smart new hire who is reading
under pressure. Short paragraphs. Concrete verbs. No throat-clearing. The agent
will pick up your tone — make it the one you want JobRunr code to be written in.
