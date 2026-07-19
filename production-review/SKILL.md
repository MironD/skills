---
name: production-review
description: Use when judging a PR's readiness to run in production — failure modes, resilience of external dependencies, deployment and migration safety, observability — not whether the code is correct in isolation.
---

# Production Readiness & Operational Quality

Load this skill alongside a review task when you're judging a PR by its **readiness to run in production**, not by whether the code is correct in isolation. It complements the review skill: this file defines WHAT to check for operational quality, while the loaded review skill supplies the output format, severity labels, and the mechanics of publishing findings.

---

## 1. Feature flags and rollback readiness

### When a feature flag is mandatory

**Missing it is `major`:**
- New business logic that changes user-visible behavior
- Integration with a new external service
- Changes to payment or billing logic
- Changes to authorization or authentication rules

**Missing it is `minor`:**
- A noticeable change to a UI/UX flow
- New background jobs or workers
- A change to the data format in queues or events

**No flag needed (don't flag it):**
- Bug fixes
- Refactors that don't change behavior
- Changes to tests, docs, or CI configuration

### Rollback readiness

For every change, ask yourself the same question: **what does it take to roll this back?**

- **Code only, no migrations** — a plain revert does the job.
- **Code plus a schema migration** — is the migration reversible? Does the reverted code still agree with the new schema if the migration isn't rolled back at the same moment? If not, that's `major`.
- **Code plus a data migration** — can the original data be recovered? Is the migration idempotent if it runs twice?
- **A breaking API change** — will older clients keep working once the code is rolled back?

---

## 2. Failure modes and resilience

### External dependencies

For **every** new or changed call to an external resource (HTTP API, database, cache, queue, file storage), check:

1. **Timeout** — is one set explicitly? A client with no explicit timeout (like a bare zero-value `http.Client{}`) effectively waits forever — that's `major`.
2. **Error handling** — what happens on a 4xx? A 5xx? A timeout? A dropped connection?
3. **Retry strategy** — are there retries at all? Is the operation idempotent enough to retry safely? Retrying on a 4xx is `major`.
4. **Circuit breaker** — on a critical path, is there any protection against a failing dependency dragging everything down?
5. **Fallback** — is there a degraded mode when the dependency is unavailable?

For the details of correct timeouts and idempotent retries in Go, see §7 and §9 of the `go` skill; what matters here is that these decisions were made deliberately rather than left at their defaults.

### Cascading failures

If service A depends on B, and B depends on C:
- What happens to A when C goes down? Is there any isolation between them?
- Can a slow C exhaust A's entire connection pool or all of its goroutines?
- Is there backpressure so A doesn't flood an already overloaded B with requests?

A cascading failure with no isolation on a critical path is `critical`.

### Resource exhaustion

- **Connection pool** — are connections always returned to the pool? Is a maximum lifetime configured?
- **Memory** — are there unbounded buffers, maps, or slices that grow without ever being trimmed?
- **File descriptors** — are files and sockets always closed (`defer Close()`)?
- **Goroutines** — does each one have a way to exit? Could a goroutine leak with no way out? See the checklist in §18 of the `go` skill for the full list of goroutine-leak signs.

---

## 3. Observability and debugging

### Logging

**Required for every new error path (missing → `minor`):**
- A log line with **context**: request ID, user ID, entity ID, operation name
- A **severity level** that matches the situation (`error` for failures, `warn` for degradation, `info` for normal operations)
- **Structured logging** — key/value pairs, not strings concatenated into the message

**Never log (present → `critical`):**
- Passwords, tokens, API keys, secrets
- Full card numbers or personal data covered by GDPR or an equivalent

The logging standard for Go (`log/slog`, context-aware methods, levels) is spelled out in §8 of the `go` skill — here you're only checking *what* ends up in the log, not *how* it's written.

### Metrics and alerting

For new business operations, assess:
- Is the **throughput** of this operation tracked?
- Is there an **error rate** metric?
- Is there a **latency** metric for external calls?
- If the operation is critical — is there an alert configured for anomalies?

Missing metrics on a critical path is `minor`. For payments or authorization, it's `major`.

### Can you debug an incident?

**The 3am test:** if this change triggers an incident in the middle of the night —
- Do the logs make it clear **what actually broke**?
- Can you tell **your own bug** apart from a **dependency failure**?
- Is there enough data to **reproduce** the problem on your own machine?
- Can you **turn this functionality off** without a new deploy (feature flag)?

---

## 4. Database and migrations

### Zero-downtime migrations

**Blockers (→ `critical`):**
- An `ALTER TABLE` that takes a blocking lock on a large table without an online mode (`ALGORITHM=INPLACE`, `CONCURRENTLY`, etc.)
- `NOT NULL` without a `DEFAULT` on a column of an existing table that already holds rows
- Dropping a column the code still uses (a deploy-ordering problem: the migration and the code don't ship at the same instant)
- `DROP TABLE` without confirming that nothing still reads from or writes to it

**Risks (→ `major`):**
- A migration with no reverse version (`down`/rollback)
- A data migration that isn't idempotent when re-run
- A column type change that risks data loss (narrowing a range, truncating a string)
- Creating an index on a large table without an online mode (for example `CONCURRENTLY` in PostgreSQL)

### Data integrity

- Are foreign keys consistent with the new schema?
- Are unique constraints present where the business rules call for them?
- Is soft delete used where data must never be lost irreversibly?
- Won't a cascade delete take out more than intended — check the real dependency graph, not just what the schema declares.

---

## 5. Deployment and configuration

### Environment variables

For every new environment variable, check:
- Is there a **default value** for local development?
- Is there **validation at startup** (fail fast if it's missing, rather than crashing somewhere deep in the middle of a run)?
- Is it documented (an example env file, README, internal docs)?
- Does the default value contain a **secret**?

A new required variable with no startup validation is `minor`. A secret in a default value or hardcoded in the source is `critical`.

### Configuration changes

- Is the new configuration **backward compatible** — does the old config still work with the new code?
- Can the configuration be changed **without a redeploy**?
- Is the configuration **validated when the process starts**, rather than at the moment of first use somewhere deep in the code?

---

## 6. Queues and asynchronous processing

### Message format

- Is the message format **backward compatible** — can the consumer still handle messages in the old format?
- Do the producer and consumer deploy **at the same time**? If not, backward compatibility is mandatory, not just nice to have.
- Does the message carry an explicit **version**?

A breaking message-format change without backward compatibility is `critical`.

### Consumer idempotency

- Is the consumer **idempotent** — is reprocessing the same message safe?
- Is there a **deduplication** mechanism?
- What happens to a **poison message** — one that fails every single time? Is there a dead letter queue?

A non-idempotent consumer under at-least-once delivery is `major`.

### Ordering and concurrency

- Does the consumer depend on message **ordering**?
- What happens when several workers process **in parallel** at once?
- Is there **backpressure** if the consumer is slower than the producer?

---

## Severity calibration for operational issues

| Issue | Severity |
|---|---|
| Secret in code or in logs | `critical` |
| Cascading failure with no isolation | `critical` |
| Breaking migration with no rollback on a production table | `critical` |
| Non-idempotent consumer under at-least-once delivery | `major` |
| External service call with no timeout | `major` |
| Missing feature flag for a user-facing change | `major` |
| Retry on a non-idempotent operation | `major` |
| Error path with no logging | `minor` |
| New environment variable with no validation | `minor` |
| Missing metrics for a new operation | `minor` |
| Log line without enough context | `nit` |
