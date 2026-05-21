---
name: jobrunr-deployment-scaling
description: Scale JobRunr workers horizontally — how worker coordination,
  polling, and database load behave as you add nodes.
tier: oss
version: 8.6.0+
frameworks: [spring-boot, micronaut, quarkus]
---

# Horizontal scaling

Use this skill when you need to run JobRunr on more than one JVM and want
to know what scales linearly and what doesn't.

## How JobRunr coordinates across workers

Every `BackgroundJobServer` polls the shared storage provider every
`pollIntervalInSeconds` (default 15s). When a worker sees an `ENQUEUED`
job that's ready to run, it claims the row (via a conditional update) and
moves it to `PROCESSING`. Because the claim is conditional on the previous
state, only one worker wins.

There is **no leader election** for OSS workers. Every worker is equal.
Scaling out is just adding replicas.

## Working example — running 5 workers

Kubernetes:

```bash
kubectl scale deployment/jobrunr-worker --replicas=5
```

Or just `replicas: 5` in the manifest. Nothing JobRunr-side to configure
— each worker picks up where it can.

## Worker count vs replica count

Two knobs:

- **Replicas** (pods / JVMs running): horizontal scaling
- **`worker-count`** per JVM: how many concurrent jobs *that* JVM runs

```properties
jobrunr.background-job-server.worker-count=8
```

Default with platform threads: `availableProcessors`. With virtual threads
(JDK 21+, on by default in JobRunr 7+): `availableProcessors * 16`.

Total concurrency = replicas × worker-count.

## Polling and database load

Every worker polls the storage provider every `poll-interval-in-seconds`.
Adding workers adds DB load proportionally — at 5 replicas with default
15s polling, that's one `ENQUEUED` query every 3 seconds on average.

For Postgres / MySQL: tune your connection pool to handle replicas ×
worker-count + polling overhead. A common starting point is
`worker-count + 4` connections per JVM.

For MongoDB clusters: **read from the primary**. Replica lag causes
JobRunr to see stale state and trigger `ConcurrentModificationException`.

## When horizontal scaling stops helping

- **DB-bound jobs.** Adding workers means more contention for the same
  rows; throughput plateaus. Scale the DB instead.
- **Single dominant job.** A 10-minute job blocks one worker for 10
  minutes. Five workers run five copies in parallel; the 6th queues. If
  you have one slow job class starving fast ones, use JobRunr Pro
  priority queues (`../pro-features/multiple-queues.md`).
- **Recurring-job concurrency.** OSS doesn't allow concurrent execution of
  the same recurring job — the second instance is skipped if the first is
  still running. Pro adds `maxConcurrentJobs`.

## Common mistakes

- **Forgetting to scale the DB connection pool.** Workers go to 100% CPU
  waiting on connection acquisition rather than actually working.
- **Mixing OSS and Pro workers against the same store.** Don't. Different
  feature sets, different state machines.
- **In-memory storage in a multi-replica deployment.** Each replica has
  its own memory. Workers don't see each other's jobs. The `InMemoryStorageProvider`
  is for tests and single-instance use only.
- **Assuming jobs run in enqueue order.** They don't, across workers.
  Order is best-effort and varies with polling jitter. If order matters,
  use chains (Pro) or sequence by data.

## Sources

- <https://www.jobrunr.io/en/documentation/configuration/spring/> (poll
  interval, worker count)
- <https://www.jobrunr.io/en/documentation/configuration/virtual-threads/>
- <https://www.jobrunr.io/en/documentation/installation/storage/>
