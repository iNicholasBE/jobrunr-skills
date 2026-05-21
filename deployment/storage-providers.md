---
name: jobrunr-deployment-storage
description: Choose and configure a JobRunr storage provider — relational
  options (Postgres, MySQL, Oracle, ...), NoSQL options (Mongo, DocumentDB),
  and when to use the in-memory provider.
tier: oss
version: 8.6.0+
frameworks: [spring-boot, micronaut, quarkus]
---

# Choosing a storage provider

Use this skill to pick the right backing store for JobRunr.

## Prerequisites

- A target database or NoSQL store
- The matching JDBC driver or NoSQL client on the classpath

> For the current authoritative list of supported versions, call
> `mcp__jobrunr-docs__search_jobrunr_docs` with query `storage providers`.
> The list below is accurate as of JobRunr 8.6.0 but tracks releases.

## Supported SQL providers (as of 8.6.0)

| Database | Tested version | Provider class |
|---|---|---|
| **PostgreSQL** | 15 | `PostgresStorageProvider` |
| **MySQL** | 8 | `MySqlStorageProvider` |
| **MariaDB** | latest | `MariaDbStorageProvider` |
| **Oracle** | `gvenzl/oracle-free` latest | `OracleStorageProvider` |
| **SQL Server** | latest Azure SQL Edge | `SQLServerStorageProvider` |
| **DB2** | 12.1.0.0 | `DB2StorageProvider` |
| **H2** | 2.3.232 | `H2StorageProvider` |
| **SQLite** | 3.47.2.0 | `SqLiteStorageProvider` |
| **CockroachDB** | 25.1+ | `CockroachStorageProvider` |

## Supported NoSQL providers

- **MongoDB** (3.4+) — `MongoDBStorageProvider`. Read from the primary; replica lag breaks the claim-by-conditional-update model.
- **Amazon DocumentDB** — `AmazonDocumentDBStorageProvider`. Same primary-read caveat.
- **In-memory** — `InMemoryStorageProvider`. Tests and single-instance demos only.

## Choosing between options

Rules of thumb:

- **Postgres** if you don't already have a strong opinion. Mature, widely
  hosted, JobRunr's most-tested provider.
- **Your existing application database** if you have one. Co-locating
  JobRunr with your business data keeps the operational footprint small.
- **MongoDB** only if you already run it. Don't add Mongo just for
  JobRunr — SQL is fine.
- **In-memory** in unit tests and a single-process demo. Never in
  production with more than one replica.

## Working example — Spring Boot with Postgres

```properties
spring.datasource.url=jdbc:postgresql://db.internal:5432/myapp
spring.datasource.username=myapp
spring.datasource.password=${DB_PASSWORD}

jobrunr.background-job-server.enabled=true
jobrunr.dashboard.enabled=true
```

The starter picks up the existing `DataSource` and creates the
`jobrunr_*` tables automatically on first boot.

## Letting JobRunr create the tables (or not)

By default JobRunr applies its schema migrations on startup, like Flyway.
If your DBA insists on no DDL from the app:

1. Generate the SQL scripts:

   ```
   java -cp jobrunr-8.6.0.jar \
     org.jobrunr.storage.sql.common.DatabaseSqlMigrationFileProvider postgres
   ```

2. Apply them manually.

3. Tell JobRunr to skip:

   ```properties
   jobrunr.database.skip-create=true
   ```

## Schema / table prefix

Spring Boot example for a dedicated schema:

```properties
jobrunr.database.table-prefix=jobrunr_internal.
```

Note the trailing `.` — JobRunr concatenates literally, so include the
delimiter yourself.

## Common mistakes

- **Using the in-memory provider in production.** Each replica's memory
  is private; workers don't see each other's jobs. The dashboard counts
  drift. Only use for tests and single-process tools.
- **Read-replica DataSource.** JobRunr writes on every state transition.
  Pointing it at a read replica makes startup fail; pointing it at a
  read-write proxy that routes by query type usually also fails because
  the conditional updates go to the wrong host.
- **MongoDB cluster reading from a secondary.** Replica lag causes the
  worker to see stale state and trigger `ConcurrentModificationException`.
  Force `readPreference=primary` for the JobRunr client.
- **Sharing the DB with high-write OLTP traffic without tuning the pool.**
  JobRunr polls every 15s per worker. On a busy production DB, the
  combined polling + DDL on startup can starve other connections. Give
  JobRunr its own DataSource if the workload warrants it (`jobrunr.database.datasource=`).

## Sources

- <https://www.jobrunr.io/en/documentation/installation/storage/>
