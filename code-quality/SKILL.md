---
name: code-quality
description: Use when writing or reviewing code and language-agnostic quality principles apply — readability, DRY, error handling, logging, config/secrets, git hygiene, backward compatibility, performance, and concurrency basics. Language-specific standards come from the matching language skill; output format and severity levels come from the loaded review skill.
---

Apply general code quality principles to the current task.

**Writing code** — follow every principle in this document; silently check yourself against the §12 checklist before returning a result.

**Reviewing code** — this document defines UNIVERSAL rules for any language; language-specific standards are defined by the matching language skill; output format and severity levels are defined by the loaded review skill.

---

# General Code Quality Principles

---

## 1. Readability and Clarity

Code is read 10x more often than it's written. Every style decision should optimize for reading, not writing.

### 1.1 Self-documenting code

A good variable or function name makes a comment unnecessary. If you need a comment to explain WHAT the code does, try renaming first.

**Variables describe content:**
```
# Bad                            # Good
d = 86400                        seconds_per_day = 86400
tmp = get_data()                 unverified_users = get_unverified_users()
```

**Functions describe an action (verb + noun):**
```
# Bad                            # Good
data()                           fetch_user_orders()
process()                        validate_payment_request()
handle()                         send_verification_email()
```

**Boolean variables are questions (is*, has*, can*, should*):**
```
# Bad                            # Good
active = True                    is_active = True
admin = check_role(user)         has_admin_role = check_role(user)
```

### 1.2 Magic numbers and strings

Any numeric or string value that isn't obvious from context should become a named constant.

```
# Bad                              # Good
if retries > 3:                    MAX_RETRIES = 3
if status == "P":                  STATUS_PENDING = "P"
time.sleep(300)                    RETRY_DELAY_SECONDS = 300
```

**Exception:** 0, 1, -1, empty string — usually don't need constants when their meaning is obvious from context.

### 1.3 Comments: WHY, not WHAT

Code says WHAT. A comment says WHY.

```
# Bad — repeats the code
# Increment counter by 1
counter += 1

# Good — explains the reason
# Retry counter: the payment gateway periodically returns 503 under peak load
counter += 1

# Good — explains a business rule
# 15% discount for orders over $5000 — promo running through end of quarter (JIRA-1234)
discount_rate = 0.15
```

### 1.4 Dead code

Check on review:
- Commented-out code — **remove it**. It's in git history.
- Unused imports — **remove**.
- Unreachable code after `return`, `throw`, `break` — **remove**.
- TODO/FIXME without a tracker ticket — **require a ticket number** or remove.
- Functions and variables that are never used — **remove** or justify.

---

## 2. Simplicity and KISS

The simplest solution that works is the best solution. Complexity must be justified by requirements, not predictions.

### 2.1 Premature optimization

Optimization without profiling is guessing. Watch for these signals:

| Signal | Problem |
|--------|----------|
| Custom cache instead of a standard mechanism | Premature optimization without a proven bottleneck |
| Bit manipulation instead of readable math | Saving nanoseconds, losing readability |
| Object pool for cheap objects | Added complexity with no measurable effect |
| Inlining everything "for speed" | The compiler does it better |
| Denormalizing the DB "just in case" | Data duplication without proof of need |

**Rule:** no benchmark/profiling data means no grounds for optimization.

### 2.2 Over-engineering

| Signal | What's wrong |
|--------|-----------|
| Abstract factory for a single type | Abstraction without variation |
| Interface with one implementation for no reason | Unnecessary layer of indirection |
| Event system for two subscribers | Complicates the flow with no payoff |
| Microservice instead of a module | Distributed complexity without a scale requirement |
| Generic solution for one use case | YAGNI — You Ain't Gonna Need It |
| Config-driven behavior instead of a simple if | Flexibility nobody will use |

### 2.3 Clever vs. clear code

```
# Clever — needs thought                        # Clear — instantly obvious
result = data and data[0] or default             result = data[0] if data else default
flags = (is_admin << 2) | (is_active << 1)       permissions = {"admin": is_admin, "active": is_active}
```

### 2.4 Cyclomatic complexity

A method with >10 branches (`if`, `else`, `case`, `&&`, `||`, `catch`) is a refactoring candidate.

**Signs:**
- Nesting >3 levels — **extract method** or **early return**.
- Switch/case with >7 cases — consider a strategy pattern or a lookup table.
- **Switch/case completeness:** if a PR changes one or more branches of a switch/match — check **ALL** branches. If a feature (masking, validation, a new parameter) was applied to some branches but not others, that's a gap. Especially critical for security-related changes.
- A chain of `if/else if/else if` — consider a map/dictionary dispatch.

```
# Bad — deep nesting
def process(order):
    if order:
        if order.is_valid:
            if order.has_items:
                if order.total > 0:
                    # finally, the logic
                    ...

# Good — early return (guard clauses)
def process(order):
    if not order:
        raise ValueError("Order is required")
    if not order.is_valid:
        raise InvalidOrderError(order.id)
    if not order.has_items:
        raise EmptyOrderError(order.id)
    if order.total <= 0:
        raise InvalidTotalError(order.total)
    # main logic, no nesting
    ...
```

---

## 3. DRY — but without fanaticism

Code duplication isn't always a problem. A bad abstraction always is.

### 3.1 Rule of Three

- **1st time** — just write the code.
- **2nd time** — acceptable, take note.
- **3rd+ time** — create an abstraction.

### 3.2 When duplication is a problem

- 3+ identical blocks with the same logic — **extract method/function**.
- Copy-pasted business rules (discounts, validation, calculations) — **one source of truth**.
- Duplicated SQL queries with minor differences — **parameterize**.
- Repeated error-handling boilerplate — **middleware or decorator**.

### 3.3 When duplication is acceptable

- **Tests:** each test should be self-contained; DRY in tests hurts readability and debugging.
- **Different bounded contexts:** two services have similar code but evolve independently.
- **Accidental similarity:** the code looks the same today but has different reasons to change.

### 3.4 Wrong abstraction

**Rule:** if an abstraction needs `if type ==` or `if isinstance` inside for every variant, it's not an abstraction — it's a junk drawer. Prefer separate functions with a shared interface over one god-method with type-switching.

---

## 4. Error Handling

Errors aren't an exception to the rules — they're part of the normal flow. Handle them deliberately.

### 4.1 Handle errors at the right level

| Level | Responsibility |
|--------|-----------------|
| Transport (HTTP handler, gRPC) | Mapping to HTTP codes, response format |
| Application (use case, service) | Business-logic errors, orchestration |
| Domain | Domain validation errors |
| Infrastructure (DB, API client) | Wrap with context, retry, circuit breaker |

**Anti-pattern:** catching an error at one level and silently swallowing it.

### 4.2 Don't swallow errors

```
# CRITICALLY BAD — error disappears    # CRITICALLY BAD — logged but ignored
try:                                    try:
    process_payment(order)                  process_payment(order)
except Exception:                       except Exception as e:
    pass                                    logger.error(e)  # and then what?

# GOOD — handle or propagate
try:
    process_payment(order)
except PaymentGatewayError as e:
    logger.error("Payment failed", order_id=order.id, error=str(e))
    raise OrderPaymentFailedError(order.id) from e
```

### 4.3 Error messages: actionable + context

An error message should answer two questions:
1. **What happened?** (the fact)
2. **What do I do about it?** (the next step)

```
# Bad                                   # Good — actionable + context
"Error occurred"                        "Payment failed for order #12345: gateway timeout after 30s"
"Invalid input"                         "Email validation failed: 'john@' — expected user@domain.tld"
"Something went wrong"                  "Connection pool exhausted (50/50). Increase POOL_SIZE?"
```

### 4.4 Fail fast: validate at boundaries

Validate data at the entry point (API boundary, message consumer, CLI input), not deep inside the logic.

```
# Bad — validation deep in the logic         # Good — validation at the entry point
def calculate_discount(order):               def create_order_handler(request):
    subtotal = sum(i.price for i in items)       validated = validate(request)  # fail fast
    if subtotal < 0:  # how did this happen?     order = create_order(validated)
        raise ValueError("Negative")
```

### 4.5 Fail fast vs. graceful degradation

| Strategy | When to use |
|-----------|---------------------|
| **Fail fast** | Invalid input, invariant violations, corrupted state, programmer errors |
| **Graceful degradation** | External dependencies (API down), optional features, read-only fallback |

```
# Fail fast — invalid state
if account.balance < 0:
    raise CorruptedStateError("Negative balance", account_id=account.id)

# Graceful degradation — external service
try:
    recommendations = recommendation_service.get(user_id)
except ServiceUnavailableError:
    logger.warn("Recommendation service down, showing default")
    recommendations = get_default_recommendations()
```

### 4.6 Error propagation: wrap with context

Each level adds its own context without losing the original.

```
# Bad — loses the original               # Bad — no context
except DatabaseError:                    except DatabaseError:
    raise AppError("Failed to save")         raise

# Good — wrap with context, preserve the original
except DatabaseError as e:
    raise OrderPersistenceError(f"Failed to save order {order.id}") from e
```

---

## 5. Logging Best Practices

Logs are a production-debugging tool. They must be useful, safe, and not flood storage.

### 5.1 What to log

| Event | Log level | Example |
|-------|-----------|---------|
| Service start/stop | INFO | `Service started, version=1.2.3, port=8080` |
| Business events | INFO | `Order created, order_id=123, total=1500` |
| External calls | DEBUG/INFO | `POST /charge, duration=320ms, status=200` |
| Errors | ERROR | `Payment failed, order_id=123, error=gateway_timeout` |
| Recoverable issues | WARN | `Cache miss, falling back to DB, key=user:456` |
| Security events | WARN/ERROR | `Auth failed, ip=1.2.3.4, attempts=5` |
| Slow requests | WARN | `Slow query, duration=2500ms, query=GetOrdersByUser` |

### 5.2 What NOT to log — CRITICAL

**Never log:**
- Passwords, password hashes, API keys, tokens, secrets
- PII: email, phone, full name, address, SSN, passport
- Credit card numbers (even masked — only the last 4)
- Session tokens, JWTs (only the token ID/jti)
- Request/response bodies with sensitive data
- Database connection strings with credentials

```
# CRITICALLY BAD                                # GOOD
logger.info(f"password: {password}")            logger.info("User login", username=username)
logger.debug(f"token: {api_token}")             logger.debug("API call", endpoint="/charge")
logger.info(f"card: {card_number}")             logger.info("Payment", card_last4="1234")
```

### 5.3 Structured logging

Structured logs (key=value or JSON) parse automatically. String concatenation doesn't.

```
# Bad                                             # Good
logger.info(f"Order {id} by {uid} for {t} USD")   logger.info("Order created", order_id=id,
                                                       user_id=uid, total=t, currency="USD")
```

### 5.4 Log levels

| Level | Meaning | Who reads it | Alerting |
|-------|----------|-----------|----------|
| DEBUG | Details for the developer | Dev during debugging | No |
| INFO | Business events, normal flow | Ops, Business | No |
| WARN | Something's off, but the system works | Ops | Dashboard |
| ERROR | Needs action, something broke | On-call | PagerDuty/Alert |

**Anti-patterns:**
- `ERROR` for user validation errors — that's not a system error, it's WARN or INFO.
- `INFO` for every SQL query — floods the logs, use DEBUG.
- `DEBUG` in production — turn it off unless actively debugging a specific issue.

### 5.5 Correlation IDs

Every request should carry a unique `correlation_id` (or `request_id`, `trace_id`) that flows through every service and log line. This lets you find the entire flow with one search: `grep correlation_id=abc-123`.

On review, check whether a new service or handler propagates and logs the correlation ID.

---

## 6. Configuration & Environment

Configuration is everything that changes between environments. Code isn't.

### 6.1 Config from environment

```
# BAD — hardcoded                              # GOOD — from environment
db_host = "prod-db.internal.com"                db_host = os.environ["DATABASE_HOST"]
api_key = "sk-live-abc123"                      api_key = os.environ["PAYMENT_API_KEY"]
max_retries = 3                                 max_retries = int(os.getenv("MAX_RETRIES", "3"))
```

**Look for on review:**
- Hardcoded URLs (especially prod/staging)
- Hardcoded credentials, API keys
- Hardcoded ports, timeouts, limits
- IP addresses and domain names in code

### 6.2 Secrets management

| Level | What's allowed |
|--------|-------------|
| Source code | NEVER secrets in code |
| Git history | If a secret lands in git, it's compromised — rotate immediately |
| `.env` files | Local development only, ALWAYS in `.gitignore` |
| CI/CD secrets | Encrypted variables, vault integration |
| Production | Secret manager (Vault, AWS Secrets Manager, GCP Secret Manager) |

**Check on review:**
- New `.env`, `credentials.*`, `*.pem`, `*.key` files — shouldn't be in the PR.
- Hardcoded strings that look like tokens/keys (long alphanumeric strings, base64).
- Connection strings with credentials in plain text.

### 6.3 Feature flags

Feature flags are a mechanism for:
- Gradual rollout of new functionality
- A/B testing
- Kill switch for problematic code
- Trunk-based development without feature branches

**Check on review:**
- Is there a cleanup plan for the flag? A flag with no removal date becomes tech debt.
- Is the default correct? A new feature flag should default to OFF.
- Are both states tested (flag on / flag off)?

### 6.4 Environment-specific behavior

```
# BAD                                            # GOOD
if ENV == "production":                          sender = config.get_email_sender()
    send_real_email(user)                        sender.send(user, template)
else:                                            # config returns a RealSender or MockSender
    print(f"Would send to {user.email}")
```

**Rule:** business logic shouldn't know which environment it's running in.

---

## 7. Git & PR Hygiene

Clean git history is a documentation and debugging tool.

### 7.1 Atomic commits

One commit = one logical change.

| One commit | Separate commits |
|------------|---------------|
| Bug fix + test | Refactor + new feature |
| Rename + update usages | Bug fix A + bug fix B |
| Migration + the code that uses it | Linter fixes + business logic |

### 7.2 PR scope

**One PR = one feature or one fix.**

**Check on review:**
- PR contains changes unrelated to the title/description — **ask to split it**.
- PR >500 lines changed (excluding tests/migrations) — **ask to split it**.
- PR mixes refactoring and new functionality — **ask to split it**.

### 7.3 Commit messages

```
# Bad
fix
update
wip
asdfgh

# Good
fix: prevent duplicate payment processing for same order
feat: add email verification on signup
refactor: extract payment gateway into separate service
chore: update dependencies to fix CVE-2024-1234
```

### 7.4 Unrelated changes

On review, look for changes unrelated to the PR's goal:
- Stray formatting changes in files unrelated to the PR.
- IDE config changes (.idea, .vscode) that slipped into the PR.
- Dependency updates unrelated to the main change.
- "Drive-by" fixes — ask to split them into a separate PR.

### 7.5 Migrations

- Database migrations separate from the code that uses them (when possible) — for a safe rollback.
- Deploy the migration first, then deploy the code.
- A migration should be reversible or explicitly marked irreversible.

---

## 8. Backward Compatibility

Any change that breaks existing clients is a breaking change and needs a migration plan.

### 8.1 Detecting breaking changes

**Look for on review:**
- Public API signature changes (removed parameters, changed types).
- Removed or renamed fields in an API response.
- Changed HTTP response codes.
- Changed behavior for the same input (a silent breaking change — the worst kind).
- Changed date formats, numeric values, enum values.
- Removed or changed queue/event message schema.

### 8.2 Database migration safety

**Safe (single deploy):**
- Adding a new column (nullable or with a default)
- Adding a new table
- Adding a new index (concurrently, if supported)

**Needs a multi-step deploy:**
- Renaming a column → add a new one, migrate data, drop the old one
- Removing a column → deploy code that stops using it first, then remove it
- Changing a column type → add a new one, migrate, switch over, drop the old one

**NEVER in a single deploy:**
- DROP COLUMN that's used by the current code
- Renaming a column without dual-write
- NOT NULL constraint on an existing column with no default

### 8.3 API versioning

- New breaking changes → new API version (`/v2/`)
- Old versions → deprecation notice with a sunset date
- Non-breaking additions (new optional fields) → same version

### 8.4 Feature flags for rollout

Large changes deploy behind a feature flag:
1. Deploy the code (flag OFF) — no risk
2. Enable the flag for internal users — canary
3. Enable for 5% → 25% → 50% → 100% — gradual rollout
4. Remove the flag and the old code — cleanup

### 8.5 Zero-downtime deployment

**Check on review:**
- Can the old code work with the new DB schema? (backward-compatible migration)
- Can the new code work with the old DB schema? (forward-compatible code)
- Is there a health check endpoint that responds correctly during deploy?
- Queue consumers: do they handle both the old and new message format?

---

## 9. Performance Awareness

Don't optimize prematurely, but recognize obvious performance problems.

### 9.1 N+1 queries

The most common performance problem in code that talks to a database.

```
# N+1 — 1 query for the list + N queries in a loop    # FIX — one JOIN query
for order in get_orders(user_id):                        SELECT o.*, oi.* FROM orders o
    items = get_items(order.id)  # N queries              JOIN order_items oi ON oi.order_id = o.id
                                                           WHERE o.user_id = ?
```

**Look for on review:** a loop with a DB query or HTTP call inside it.

### 9.2 Unbounded queries

**Look for on review:**
- `SELECT *` without `WHERE` or `LIMIT` — loads the entire table.
- `.all()`, `.find()`, `.get()` with no filters on large collections.
- Collections that grow unbounded in memory.

### 9.3 Missing indexes

When a PR adds:
- A new column that will be used in `WHERE`, `JOIN`, `ORDER BY` — it needs an index.
- A new filtered query — check whether the filtered columns have an index.
- A composite filter (WHERE a = ? AND b = ?) — a composite index may be needed.

### 9.4 Large payload responses

**Look for on review:**
- An API endpoint that returns a list without pagination.
- A response with nested objects several levels deep (eager-loading the whole graph).
- Binary/blob data in JSON responses (base64-encoded files).
- `SELECT *` instead of selecting the fields the API response actually needs.

### 9.5 Caching

**When caching makes sense:**
- Data that rarely changes (reference data, configuration).
- Expensive computations or aggregations.
- Frequent reads, rare writes.

**Cache invalidation — the hardest part:**
- TTL-based — simple, but can show stale data.
- Event-based — accurate, but complex.
- Always define the invalidation strategy BEFORE introducing a cache.

**Look for on review:**
- Cache with no TTL — data never expires.
- Cache with no invalidation strategy — stale data forever.
- Cache key that doesn't include all significant parameters — cache collision.

### 9.6 Memory awareness

**Look for on review:**
- Reading large files entirely into memory instead of streaming.
- Unbounded accumulation into a list/array.
- String concatenation in a loop instead of a builder/buffer.
- Holding a reference to large objects that prevents GC from collecting them.

---

## 10. Concurrency Basics

Concurrent code is the hardest to debug. Concurrency issues show up randomly and are barely reproducible.

### 10.1 Shared mutable state

If two threads/goroutines/coroutines access the same data and at least one writes, that's a data race.

**Look for on review:**
- Global variables modified from different threads.
- Shared collections (map, list, dict) without synchronization.
- Lazy initialization without thread safety (a singleton with no lock).
- Instance fields modified inside a request handler (handlers are usually shared).

### 10.2 Race conditions

Classic TOCTOU (Time of Check to Time of Use): `if balance >= amount` → (another thread changes balance) → `balance -= amount`. Fix: an atomic operation or a lock.

**Signs of a race condition:**
- Check-then-act without a lock/transaction.
- Read-modify-write without an atomic operation.
- "Happens rarely" or "only under load" — the classic description of a race condition.

### 10.3 Deadlocks

**Look for on review:**
- Two+ locks acquired in different orders in different places.
- A lock inside a lock (nested locking).
- Database transactions waiting on each other.

**Prevention:**
- Always acquire locks in the same order.
- Use a timeout on lock acquisition.
- Minimize the time a lock is held.

### 10.4 Resource cleanup

Resources (connections, files, locks) must use guaranteed cleanup: `try-finally`, `using`, `defer`, `with`. Without it, they leak on exception.

**Look for on review:**
- Resources (connections, files, locks) without guaranteed cleanup (try-finally, using, defer, with).
- Goroutine/thread leak: started with no way to stop it.
- Channel/queue with no consumer — memory leak.

### 10.5 Timeouts on external calls

**EVERY** external call must have a timeout: HTTP requests, DB queries, cache, message queues, gRPC, network I/O.

Without a timeout, one slow dependency stalls the entire service (cascading failure).

**Look for on review:** HTTP/gRPC client calls without an explicit timeout parameter.

---

## 11. Documentation Needs

Not all code needs documentation — only the parts where the code can't explain itself.

### 11.1 When code needs documentation

| Situation | What to document |
|----------|-----------------|
| Public API (library, SDK, REST) | Every public method: parameters, return value, errors, examples |
| Non-obvious business rule | Why exactly 15%? Why 3 attempts? Reference the business requirement |
| Workaround for an external bug | Reference the bug report, when it can be removed |
| Complex algorithm | High-level description of the approach, reference to the paper/source |
| Architecture decision | ADR (Architecture Decision Record): context, decision, consequences |
| Regex | Always comment a regex, even a "simple" one |

### 11.2 README updates

**Check on review — does the README need updating:**
- New environment variable → document it in the README or `.env.example`
- New dependency that needs setup → document it
- Changed startup/deploy process → document it
- New CLI command or endpoint → document it

### 11.3 Migration guides

For breaking changes, always include:
- A CHANGELOG entry describing the change.
- A migration guide: step by step, how to update.
- A deprecation warning at least one version before removal.

### 11.4 API documentation

For new API endpoints, check on review:
- OpenAPI/Swagger spec is updated.
- Request/response examples exist.
- Error responses are documented.
- Authentication requirements are stated.

---

## 12. Severity Calibration — Review Checklist

### CRITICAL — blocks merge, potential production incident

Risk: data loss, security hole, production breakage.

- [ ] SQL injection, XSS, path traversal, insecure deserialization
- [ ] Secrets (passwords, API keys, tokens) in code or logs
- [ ] Error silently swallowed (empty `catch` body, `except: pass`)
- [ ] Data loss: DELETE/UPDATE without WHERE, DROP with no backup plan
- [ ] Race condition on financial or otherwise critical data
- [ ] Unbounded query on a large table with no LIMIT
- [ ] Missing authentication/authorization on an endpoint
- [ ] PII in logs (passwords, card numbers, personal data)
- [ ] Broken backward compatibility with no migration plan
- [ ] Infinite loop or recursion with no base case

### MAJOR — needs fixing, but not a blocker

Risk: bug, degraded performance, maintenance burden.

- [ ] Missing error handling (happy path only)
- [ ] N+1 query pattern
- [ ] Missing index for a new query
- [ ] No timeout on an external call
- [ ] Resource leak (connection, file handle, goroutine)
- [ ] Missing validation at an API boundary
- [ ] Duplicated business logic (3+ places)
- [ ] PR too large (>500 lines) — ask to split it
- [ ] Missing tests for business logic
- [ ] Wrong log level (ERROR for user input validation)
- [ ] Dead code: unused functions, unreachable branches
- [ ] Missing pagination on a list endpoint
- [ ] Database migration not reversible with no explanation

### MINOR — improvement, not a blocker

Risk: reduced readability, minor tech debt.

- [ ] Naming: unclear variable or function names
- [ ] Magic numbers with no named constants
- [ ] Excessive nesting (>3 levels)
- [ ] A comment that repeats the code (WHAT instead of WHY)
- [ ] Commented-out code with no reason given
- [ ] TODO/FIXME with no ticket number
- [ ] Inconsistent style with the rest of the codebase
- [ ] Import ordering, formatting — if there's no autoformatter
- [ ] Variable declared far from its first use
- [ ] Method too long (>50 lines) — extract candidate
- [ ] Unnecessary complexity (over-engineering for a simple case)
- [ ] Missing README update for new configuration
