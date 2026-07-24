---
name: code-review
description: Use when asked to perform a local code review of a diff, branch, or PR and produce a written review report.
---

You perform a local code review and save the result as a markdown file.

**Mandatory dependencies:** this skill requires **[../security/SKILL.md](../security/SKILL.md)**, **[../production-review/SKILL.md](../production-review/SKILL.md)**, and **[../code-quality/SKILL.md](../code-quality/SKILL.md)**. Load and read all three in full as part of loading this skill, before starting the review — none is optional and none is something to skip for small diffs. Additionally, load **[../architecture/SKILL.md](../architecture/SKILL.md)** whenever the diff changes architecture — new packages/modules, layer boundaries, coupling between components, design patterns, API/interface contracts, or the shape of an abstraction; skip it only when the change is confined to logic inside an existing structure (a bug fix, an added case, a tweaked query) with no structural shift. See §"Architecture Analysis", §"Code Quality Analysis", §"Security Analysis", and §"Production Readiness Analysis" below for how their findings integrate into this review's severity and decision rules.

---

## Context Collection (before starting the review)

Unlike platform-based review tools where context (task tracker, PR description, wiki) is pulled automatically, in a local review **you collect context interactively**. Without full context, a quality review is impossible — it is better to spend a minute asking questions than to miss the point of the changes.

### Step 1: Automatic Collection

Collect what is available without developer input:
- Current branch name (`git rev-parse --abbrev-ref HEAD`)
- Commit messages (`git log main..HEAD --oneline`, or `master` if no `main`)
- List of changed files (`git diff main...HEAD --name-only`)
- If there are task files, documentation, or TODOs in commit messages — take note
- **Diff size** — count lines changed: `git diff main...HEAD --stat | tail -1`. Record in the report header. If changed lines exceed 400, note that review quality is statistically reduced and explicitly identify the 2-3 highest-risk files to focus scrutiny on. If changed lines exceed 1000, note in the report: "This PR exceeds recommended review size. Review quality may be degraded. Consider splitting into smaller changes."

### Step 2: Ask the Developer

After automatic collection, **always ask** the developer for what is missing for full context. Formulate one message with all questions:

1. **Task description** — "What problem do these changes solve? Is there a link to a task (issue tracker) or acceptance criteria?"
2. **Decision context** — "Why was this particular approach chosen? Are there constraints, dependencies, or contracts that influenced the decision?"
3. **Additional materials** — "Is there anything that would help understand the changes better? (design documents, diagrams, discussions, related PRs in other repos)"
4. **Scope** — "Should I review all changes relative to main/master or specific files?"
5. **AI assistance** — "Was any part of this code generated or suggested by an AI tool (Copilot, Cursor, ChatGPT, Claude, etc.)?"

**Context collection rules:**
- If the developer provided a link to a task with acceptance criteria, use them to check implementation completeness
- If the developer says "no context, just look at the code", conduct the review based on the code, but note in the report that a review without task context may miss business requirements
- If the branch name and commits make the context obvious (e.g. `fix/null-pointer-in-payment-service`), confirm your understanding and ask only what is missing
- If the code is AI-generated or AI-assisted, apply heightened scrutiny to: security vulnerabilities (AI code has 15–18% more per LOC than human-written code), tests that test the implementation rather than the specification, unnecessary dependencies, and logic that looks correct but is subtly wrong. Note "AI-assisted" in the report header.
- Do not start code analysis until you receive a response or explicit refusal to provide context

### Review Scope — Only Code Changed in PR/Branch

Comments are only allowed **on lines/files changed in this PR/branch**. Do not comment on code that existed before the changes and was not modified — even if you see a problem.

If a problem is found **outside the scope of changes** (in existing code, in another module, etc.):
1. **Do not** include comments on unchanged code in the main report sections.
2. **Do not lower the score** due to out-of-scope issues — if everything changed is fine, give a positive verdict.
3. Describe the problem in a **separate section** "Out-of-Scope Recommendations" as a task:

```
💡 **Found outside scope — recommendation to create a task:**

**Title:** [brief problem description]
**Component:** [module/service]
**Description:** [what is wrong, where it is, what the risk is]
**Recommendation:** [how to fix]
```

---

## Pre-Analysis Phase

**Before writing any comments**, complete full preparation. The goal — build a mental model of the changes and only then comment. Reviewing without context is superficial pattern matching, not professional analysis.

### 1. Understand the Big Picture

- **Read the task description** — the developer's response to context collection, branch name, commit messages. Understand *what* the author wanted to do and *why*.
- **Review the list of changed files** from `git diff` — how many files, what categories (migrations, tests, configs, source), what commits.
- **Determine the type of changes** — new functionality, bug fix, refactoring, infrastructure. The type determines what to focus on.

### 2. Read Full Files, Not Just the Diff

The diff shows a fragment. Errors are often in the surrounding context.

- For **each changed file** open the full version — understand how the change fits into existing code.
- Pay attention to **imports**, **constructor/initialization**, **public interface**.
- If a file is large — read at least the class/module where the change occurred in full.

### 3. Find and Read Related Tests

Tests are a specification of behavior. Without them it is impossible to understand what is considered correct.

- For each changed file find the corresponding test file (by language convention: `*Test.java`, `test_*.py`, `*_test.go`, `*.test.ts`, `*.spec.ts`).
- Check: are tests **current** after the changes, do they cover **new logic**, are there **missing edge cases**.
- If there are **no tests** — this is already a potential finding.

### 4. Check Calls and Usage

- For each changed function/method find **all callers** via grep/search in the codebase.
- If the signature changed — check that callers are updated.
- If the behavior (contract) changed — check that callers expect the new behavior.

### 5. Review Blame Context

- Run `git blame` on the changed sections — understand who wrote the code being changed and why.
- Code written long ago by another person — higher risk of breaking a non-obvious dependency.

### 6. Build a Mental Model

Only after steps 1-5 start commenting. Formulate: what these changes do, why, how, and what the risks are.

### 7. Architectural Checklist (MANDATORY before writing comments)

Before writing any comment, answer these questions. If the answer to any is negative, it is a **mandatory** comment with severity 🔵 or above:

1. **Layer of responsibility** — is all code in the correct layer? Business logic not in controller/handler, SQL not in domain, presentation not in service?
2. **Implicit contracts** — did implicit dependencies between components appear? Call order, shared mutable state, magic strings, conventions without enforcement?
3. **Coupling delta** — does the PR increase coupling between modules? New imports between packages that did not depend on each other before?
4. **Cognitive load** — does this PR make the system harder to understand? New concepts, new invariants, new "gotchas" to remember?
5. **Future extensibility** — can a new feature be added 6 months from now without rewriting? Is the code "closed" for extension?
6. **3AM debuggability** — can on-call debug this code at 3 AM? Enough logs? Clear error messages? Can you tell "my service broke" from "a dependency is not responding"?
7. **Degradation check** — does this PR make the codebase **better** or **worse**? If worse — it is not ℹ️, it is at minimum 🔵, and if structural — 🟡.

### 8. Cognitive Bias and Fatigue Check (before writing comments)

Research shows these biases reliably degrade code review quality — actively counteract them:

- **File position bias** — files listed first in the diff receive more scrutiny than later ones. Do not review files in diff order. Instead, review in this order: (1) highest-risk files first (auth, payments, data writes, migrations), (2) core business logic, (3) utilities and config, (4) tests and docs.
- **Authority bias** — code authored by a senior developer or tech lead tends to receive less scrutiny. Apply identical standards regardless of author seniority.
- **Anchoring bias** — the first impression of the PR (good or bad) colors analysis of all subsequent files. Commit to completing the Pre-Analysis Phase fully before forming an overall judgment.
- **Confirmation bias** — once a mental model is formed, contradicting evidence is discounted. If a file does not match expectations from the PR description, treat that as a signal worth investigating rather than dismissing.
- **Review fatigue** — defect detection drops significantly after 60–90 minutes of continuous review, and at a pace faster than ~500 LOC/hour. If the review session exceeds 60 minutes or the diff is large, note in the report that later-reviewed files received reduced scrutiny and name them explicitly.

---

## Review Approach

### 1. Full Context First

Before any criticism, collect and understand:
- **What problem the code solves** — description from the developer, branch name, commit messages, changes in code
- **Why this particular implementation was chosen** — what constraints, dependencies, contracts
- **Expected behavior and edge cases** — what happens on errors, null data, boundary values

### 2. Think Before Criticizing

First understand the author's intent. Only then propose improvements. Do not criticize an approach if you do not understand the context of the decision.

### 3. Architecture — Always First

Architectural analysis is **always the first priority** of a review. The codebase should become better with each PR, not degrade from quick fixes and patches.

**Anti-degradation principle:** the reviewer is responsible for ensuring each change either maintains or improves the architecture. Accepting a "quick fix" that worsens the structure is consciously creating technical debt.

Evaluate the code as part of the system:
- Does the code solve the task **completely**, not partially
- Does it **not duplicate** existing functionality in the codebase
- Does it **not overcomplicate** the system unnecessarily
- Does it match the **overall architecture**
- Is there a **simpler approach** that achieves the same result
- Does the code look like **part of the system**, not a temporary patch
- Can the system be changed to **simplify** the implementation

**If an architectural improvement is outside the scope:**
Write one comment describing the current problem, desired state, and proposed approach. End with a recommendation to create a separate technical task (ticket) for this improvement. Format:

> **Architectural recommendation:** {description of problem and approach}. I recommend creating a technical task for improvement, as this goes beyond the current change scope.

### 4. Execution Control and Determinism

Functions should return predictable results. Check:
- Early return for invalid/edge cases — happy path not wrapped in nested ifs
- All execution branches terminate logically — no "falling through" without a result
- Code is deterministic and readable without additional explanations

### 5. Tests — Mirror of Reliability

Tests are not a formality. They are the **main indicator** that changes are correct and will not break the system in the future.

Check:
- New functionality is **always** covered by tests — this is the core rule, no exceptions
- Tests check **real behavior**, not implementation details
- No duplicate checks — each test tests something unique
- Tests separated by logical layers: unit (business logic), integration (HTTP, DB), contract (inter-service)
- Edge cases and boundary values covered
- Test names are informative — from the name it is clear what is being checked and what the expected result is
- Tests are easy to extend — adding a new case does not require rewriting
- When business logic changes, a test should "turn red" — if logic changed but tests are green, they check nothing

---

## Deep Analysis Methodology

This section defines **how to think** during a review — not just check patterns, but trace logic end-to-end, model failures, and find what is not in the PR but should be.

### Data Flow Analysis

For **each changed function/method** trace data from input to output:

- **Where does data come from** — user input, DB, external API, queue, cache?
- **What transformations happen?** Can any transformation produce an invalid state?
- **Where does data go** — DB write, API response, queue publish, log?
- **If data arrives in unexpected format** (null, empty string, wrong type, extremely large size) — what happens at **each** step?
- **If a function mutates an object** — who else holds a reference to that object? Can the mutation break another consumer?

If a function accepts external data, trace each transformation to the final write/response.

### Failure Mode Analysis

For **each external dependency** touched by the changes (DB, HTTP API, cache, queue, file system, third-party service), check:

- What will happen if the dependency **goes down**? (connection refused, timeout)
- What will happen if the dependency is **slow**? (10x latency)
- What will happen if the dependency **returns unexpected data**? (empty response, 500, malformed JSON, partial data)
- What will happen if the dependency is **overloaded**? (disk full, queue overflow, connection pool exhaustion)
- Is there a **graceful degradation path**, or does the entire request fail?

For each external dependency in the changes ask: what happens if it [goes down / is slow / returns unexpected data / is overloaded]?

### Reasoning About Absence (What Should Be There But Is Not)

After analyzing what **is** in the PR, explicitly check what is **NOT** included but should be:

- If a **write operation** is added — is there a corresponding read/list/delete? Is there a migration?
- If a **new config/env** is added — is there startup validation? Default value? Documentation?
- If **behavior changes** — is there a feature flag for gradual rollout? Rollback plan?
- If a **new endpoint** is added — is there rate limiting? Authentication? Input validation?
- If **error handling** is added — is there alerting/monitoring for the new error type?
- If a **new entity/model** is added — is there a lifecycle? Who creates, updates, deletes it?

After analyzing PR changes, explicitly check what is NOT included but should be.

### State and Transition Analysis

When code manages **state** (order status, payment state, user role, feature toggle):

- **List all possible states**
- Check that **each transition between states** is explicitly handled
- Check for **impossible/invalid combinations** of states
- Check that **concurrent state changes** are safe (two requests change one entity simultaneously)
- Check that a **failed transition** does not leave the system in an inconsistent state

If code manages state (statuses, stages, roles), build a mental map of all states and transitions.

### Context Beyond the Diff

- Read the **full file**, not just changed lines — understand how the change fits into existing code
- Check **who calls** the changed function (find usages) — will callers break?
- Check if **similar logic** already exists in the codebase — should it be reused?
- If the change is a fix — **what caused** the original bug? Is the root cause fixed, or just the symptom?

For each changed file read the file in full. For each changed function find all its callers.

---

## Preparation (after context collection)

1. Form the report file path:
   ```
   reviews/YYYY-MM-DD_HH-MM_{branch}.md
   ```
   where `YYYY-MM-DD_HH-MM` is the current time, `{branch}` is the branch name (replace `/` with `-`).
2. If the `reviews/` folder is missing — create it.
3. Determine review scope:
   - If the developer specified specific files — review those.
   - Otherwise: `git diff main...HEAD --name-only` (or `master` if no `main`).

---

## Duplicate Detection and Reuse (mandatory)

For **each new** function, method, class, helper, or utility added in the changes:

1. **Check if an equivalent already exists** — via grep in the codebase, find elements similar in name or purpose:
   - A service that does the same or very similar thing
   - A utility/helper with similar function (formatting, validation, transformation, filtering)
   - A method in a base class or interface that already implements the needed logic
   - A DTO/Value Object with a similar structure
   - Constants or enums that duplicate existing ones

2. **If an equivalent is found**, severity 🟡:
   - Indicate **where exactly** the existing equivalent is located (file, class, method)
   - Explain **how it is similar** to the new code
   - Suggest **reusing** the existing one or **extending** it instead of duplicating

3. **Check project patterns** — new code should follow existing conventions:
   - If the project has a base class for services — new service should extend it
   - If there is centralized error handling — do not create a new one
   - If there is a utility namespace — helpers should be there, not inline

4. **Check dependencies** (if there are changes in dependency files like `package.json`, `go.mod`, `requirements.txt`, etc.):
   - Is the new dependency justified? Does an existing library already cover this functionality?
   - Are there known vulnerabilities in the added packages?
   - Does the new dependency duplicate an existing one?

---

## Design Review: Degradation Control and Solution Quality

This section is the **most important** for review quality. It defines how to evaluate **solution design**, not just code correctness.

### Principle: Every PR Should Make Code Better

Every change either maintains or improves the architecture. Accepting changes that worsen the structure is consciously creating technical debt. **Degradation is not a nit.** If changes make the codebase worse — it is 🟡.

### Implicit Contracts and Hidden Coupling

The most dangerous bugs are those that appear from **implicit dependencies** between components:

- **Call order** — if method B must be called after method A, but this is not enforced anywhere, severity 🟡. Recommend State Machine or Builder pattern.
- **Shared mutable state** — two components mutate one object without coordination, severity 🔴 if race condition, 🟡 if just fragile code.
- **Magic strings and conventions** — if code depends on the format of a file name, array key, or method name without compile-time checks, severity 🔵.
- **Temporal coupling** — code only works if external events arrive in the right order, but the order is not guaranteed, severity 🟡.

### Cognitive Load

Code should be understandable **without explanations**. If changes add a new "gotcha" to remember:

- **New invariants** — if a rule "X must always be true when Y" appeared, but it is not enforced anywhere, severity 🔵 at minimum.
- **Non-obvious behavior** — if a function does something not obvious from its name/signature, severity 🔵.
- **Scattered logic** — if one business operation is scattered across 5+ files without a clear entry point, severity 🟡.

### Extensibility and Maintainability

- **Closed for extension** — if adding a new variant requires changing 5+ files, severity 🔵 with Strategy/Plugin pattern recommendation.
- **Premature abstraction** — unnecessary abstractions for a single use case, severity ℹ️.
- **Missing abstraction** — 3+ places with copy-paste logic that differs only in details, severity 🔵.

### 3AM Debuggability Test

For **each** new error path / failure scenario:

- Is there a log with **sufficient context** to understand what happened? (user ID, request ID, input data)
- Can you tell "my service broke" from "an external dependency is not responding"?
- Is there a **timeout** on each external call?
- Is the error **not silently swallowed**?

If on-call cannot diagnose the problem in 30 minutes with available logs — minimum 🔵.

### Code Incentive Structure

Good code makes **correct usage easy** and **incorrect usage difficult or impossible**.

- If an API makes it easy to make a mistake (e.g., confusing arguments of the same type), severity 🔵.
- If the only way to use an API is documented only in a comment, severity 🔵.
- If incorrect usage silently works (without error) but gives wrong results, severity 🟡.

---

## Analysis Priority

The code-quality skill's baseline (see §"Code Quality Analysis") isn't a separate step below — it's the standard every item in this list is checked against.

Check in this order — stop at 🔴 if there are blockers:

1. **Architecture and systemics** — separation of concerns, layers, coupling, SRP; code **does not duplicate** existing functionality (check via grep); no temporary patches/hacks; solution matches overall architecture; controllers are thin; early return; anti-degradation of codebase; new code follows existing project patterns; implicit contracts; cognitive load; 3AM debuggability. If the diff changed architecture, run the architecture skill (see §"Architecture Analysis")
2. **Security** — run the security skill; its 🔴/🟡/🔵/ℹ️ findings map directly onto this scale (a security 🔴 is a 🔴 here). **Privacy** — PII handling, data retention, consent, logging of sensitive data
3. **Correctness and states** — errors ignored, race conditions, non-atomic operations, invalid state transitions, inconsistent state after failure
4. **Idempotency and edge cases** — double execution, retry safety, NULL/empty/boundary inputs, race conditions on shared state
5. **Reliability, failure modes, and production readiness** — run the production-review skill: feature flags and rollback readiness, timeouts/retry/circuit-breaking on external dependencies, cascading failure isolation, resource exhaustion, zero-downtime migrations, deployment/config safety, queue idempotency and ordering; for each dependency: what if it goes down / is slow / returns garbage
6. **Resource leaks and memory** — unclosed resources, memory leaks, connection pool exhaustion, blocking without timeout, backpressure
7. **Performance** — N+1 queries, missing indexes, queries without LIMIT, loading large collections, synchronous external calls without timeout, caching
8. **Observability** — logging with context, metrics, tracing propagation, alerting; sensitive data not logged
9. **Acceptance criteria** — requirements fully implemented; acceptance criteria checked item by item
10. **Inter-service contract** — API compatibility when changing external interaction
11. **Tests** — new functionality always covered; edge cases tested; tests meaningful, separated by layer, no duplicate checks; when logic changes, test should "turn red"
12. **Documentation** — docstrings, README updated with functional changes; consistency of existing documentation with code
12b. **Accessibility** — semantic HTML, ARIA correctness, keyboard navigation, focus management, color contrast, alt text (UI changes only)
12c. **Internationalization** — translation keys, locale-aware formatting, pluralization, encoding, layout flexibility (multi-locale apps only)
13. **Dead code** — unused imports, commented code, unreachable code, uncalled functions

---

## Architecture Analysis

Read and apply **[../architecture/SKILL.md](../architecture/SKILL.md)** in full when the diff changes architecture — new packages/modules, layer boundaries, coupling between components, design patterns, API/interface contracts, or the shape of an abstraction. Skip it when the change is confined to logic inside an existing structure (a bug fix, an added case, a tweaked query) with no structural shift — the "Architectural Checklist" and "Architecture — Always First" sections above already cover that baseline for every review, applicable or not.

> When applicable, its structural findings are this review's first analysis priority (see "Analysis Priority" below) and use the same 🔴/🟡/🔵/ℹ️ scale as the rest of this report.

## Code Quality Analysis

Read and apply **[../code-quality/SKILL.md](../code-quality/SKILL.md)** in full. It covers the language-agnostic baseline this review sits on top of — readability, DRY, error handling, logging, config/secrets, git/PR hygiene, backward compatibility, performance, and concurrency basics — distinct from the language-specific idioms a matching language skill (e.g. `../go/SKILL.md`) adds on top.

> Findings from the code-quality skill use its own CRITICAL/MAJOR/MINOR calibration (§12 there); map them onto this skill's 🔴/🟡/🔵/ℹ️ scale as CRITICAL→🔴, MAJOR→🟡, MINOR→🔵.

## Security Analysis

Read and apply **[../security/SKILL.md](../security/SKILL.md)** in full. It covers OWASP Top 10, additional high-value CWEs, secrets, injection, auth/authz, data protection, infrastructure security, severity calibration, and a complete review checklist.

> Code examples in the security skill are written in **Go**. Translate patterns to the language and packages used in the codebase being reviewed — the vulnerability class and fix principle are the same regardless of stack.

> A **🔴** security finding — **Request Changes**, no exceptions. A **🟡** security finding follows the same rule as any other 🟡: Request Changes unless explained. **🔵**/**ℹ️** security findings are reported but do not block on their own — don't escalate a defense-in-depth recommendation to 🔴 just because it came from the security skill.

---

## Production Readiness Analysis

Read and apply **[../production-review/SKILL.md](../production-review/SKILL.md)** in full. It covers feature flags and rollback readiness, failure modes and resilience of external dependencies, cascading failures, resource exhaustion, observability (logging, metrics, alerting, the "3am test"), zero-downtime database migrations, deployment and configuration safety, and queue/async processing correctness — the operational-readiness lens on a change, distinct from the correctness and architecture checks elsewhere in this skill.

> Findings from the production-review skill use the severity calibration defined there; map them onto this skill's 🔴/🟡/🔵/ℹ️ scale using the equivalent `critical`/`major`/`minor`/`nit` levels.

---

## Data Privacy and PII Compliance (mandatory)

For any change that touches user data, storage, logging, or external transmission, check the following. Skip this section only if the PR demonstrably has no contact with user data.

### PII Identification

- **What counts as PII:** name, email, phone, address, IP address, device ID, location, biometrics, government IDs, payment data, health data, any combination that identifies a person.
- Identify every field, column, log statement, and API response in the PR that may contain PII.
- If PII is handled in a new way (new field, new log, new endpoint response), all items below apply.

### Storage

- PII stored in a new DB column: is the column encrypted at rest, or does the existing table encryption cover it?
- Is there a data retention policy? Does PII have a defined TTL, or is it stored indefinitely? Severity 🟡 if no retention policy for new PII storage.
- Is there a deletion path? If a user requests erasure (GDPR Article 17), can this PII be deleted without manual intervention? Severity 🟡 if no deletion path.
- Data minimization: is only the minimum necessary data being stored? Collecting fields "just in case" is 🔵.

### Transmission

- PII transmitted to a third-party service: is it documented and consented to in the privacy policy?
- PII sent over HTTP (not HTTPS): 🔴.
- PII included in URLs (query params, path segments) where it may appear in server logs: 🟡.

### Logging

- PII appearing in any log statement (even at DEBUG level): 🔴. This includes user IDs in combination with sensitive fields, email addresses, tokens, and payment data.
- Ensure error messages that reference user data mask or omit the sensitive parts.

### Consent and Regulatory

- If a new type of data is being collected that was not previously disclosed: flag as 🟡 — legal/privacy review required before merge.
- Cross-border data transfer (user data from EU stored in non-EU systems): flag as 🟡 — legal review required.
- If the codebase has a consent management layer, verify new data collection flows through it.

### Privacy Severity

| Issue | Severity | Blocks? |
|-------|----------|---------|
| PII in logs (any level) | 🔴 | yes |
| PII transmitted over HTTP | 🔴 | yes |
| New PII collected without consent/disclosure | 🟡 | yes |
| No deletion path for new PII storage | 🟡 | yes |
| No retention policy for new PII storage | 🟡 | yes |
| PII in URL query params / path segments | 🟡 | yes |
| Cross-border data transfer without review | 🟡 | yes |
| Data collected beyond what is necessary | 🔵 | no |

---

## Accessibility (a11y) — UI Changes Only

Skip this section if the PR has no UI changes (no HTML, JSX/TSX, Vue templates, or CSS). Apply to any code that renders user-facing content.

### Semantic Structure

- Interactive elements use native elements (`<button>`, `<a href>`, `<input>`) rather than `<div>` or `<span>` with click handlers. Non-semantic elements with click handlers require `role`, `tabindex="0"`, and keyboard event handlers: 🟡.
- Form inputs have associated `<label>` elements (via `for`/`id` or wrapping). Inputs without labels are inaccessible to screen readers: 🟡.
- Heading hierarchy is logical (`h1` → `h2` → `h3`), not used for visual styling alone: 🔵.
- Page/section landmarks present (`<main>`, `<nav>`, `<header>`, `<footer>`, ARIA landmark roles): 🔵.

### ARIA Usage

- ARIA attributes are used only when native semantics are insufficient (prefer native HTML first).
- `aria-label` / `aria-labelledby` present on icons, icon buttons, and controls without visible text labels: 🟡.
- Dynamic content updates use `aria-live` regions so screen readers are notified: 🔵.
- `aria-hidden="true"` not applied to elements that are keyboard-focusable: 🔴.
- ARIA roles are valid and used correctly (e.g., `role="dialog"` on modals with focus trap): 🟡.

### Keyboard and Focus

- All interactive elements reachable and operable via keyboard (Tab, Enter, Space, Arrow keys): 🟡.
- Focus is never trapped unintentionally. Modal dialogs must trap focus within themselves and restore focus on close: 🟡.
- Custom components (dropdowns, date pickers, tabs) implement standard keyboard patterns per ARIA Authoring Practices Guide: 🔵.
- `:focus-visible` styles are not suppressed (`outline: none` without a replacement is 🔵).

### Visual

- Color is not the sole means of conveying information (e.g., error state indicated by color AND icon/text): 🟡.
- Text contrast ratio ≥ 4.5:1 for normal text, ≥ 3:1 for large text (WCAG AA). Cannot verify contrast from code alone — flag if hardcoded color values look marginal: 🔵.
- Images have descriptive `alt` text. Decorative images have `alt=""`: 🟡.
- Videos/audio have captions or transcripts: 🟡.
- No content flashes more than 3 times per second (seizure risk): 🔴.

### a11y Severity Summary

| Issue | Severity | Blocks? |
|-------|----------|---------|
| Content flashing > 3 times/sec | 🔴 | yes |
| `aria-hidden` on focusable element | 🔴 | yes |
| Interactive element not keyboard-reachable | 🟡 | yes |
| Input without label | 🟡 | yes |
| Color as sole information conveyor | 🟡 | yes |
| Missing `alt` on meaningful image | 🟡 | yes |
| Focus trap not implemented on modal | 🟡 | yes |
| Missing `aria-label` on icon button | 🟡 | yes |
| Missing ARIA live region for dynamic updates | 🔵 | no |
| Focus outline suppressed without replacement | 🔵 | no |
| Suboptimal heading hierarchy | 🔵 | no |

---

## Internationalization (i18n) and Localization (l10n) — Multi-Locale Apps Only

Skip this section if the application is confirmed to be single-locale only. Apply to any code that renders user-facing text, formats dates/numbers/currency, or handles user input.

### Strings and Text

- Hardcoded user-facing strings (not loaded from a translation key/message catalog): 🟡. Every visible string must go through the project's i18n API (e.g., `t('key')`, `intl.formatMessage`, `i18n.t`).
- String concatenation used to form sentences: 🟡. Concatenation breaks in languages with different word order. Use message templates with named placeholders instead (`"Welcome, {name}!"` not `"Welcome, " + name + "!"`).
- Pluralization handled with a simple `s` suffix or ternary: 🔵. Use the project's pluralization API (ICU format, `Intl.PluralRules`, etc.) to handle language-specific plural rules.
- Assumed fixed text direction (LTR only) in layout or string manipulation: 🔵 if the app supports RTL locales.

### Formatting

- Dates formatted with `.toLocaleDateString()` or locale-aware API: check that locale is passed explicitly, not assumed from system default: 🔵.
- Numbers, currencies, or percentages formatted without `Intl.NumberFormat` or equivalent locale-aware API: 🔵.
- Timezone handling: dates stored and transmitted as UTC, converted to user timezone only at display layer: 🔵. Storing local times is a 🟡 if the app has multi-timezone users.
- Hardcoded currency symbols (`$`, `€`) in UI code: 🔵.

### Layout and Visual

- Fixed-width containers that cannot accommodate text expansion: German and Finnish strings can be 30–50% longer than English equivalents. Rigid pixel widths on text containers are 🔵.
- Icon or image assets with embedded text (not translatable): 🔵 — recommend SVG with separate text layer or text overlay.

### Input and Encoding

- Character encoding assumptions (ASCII-only validation, fixed-length DB columns that don't account for multi-byte UTF-8 characters): 🟡.
- Input validation that rejects valid Unicode characters in names, addresses, or other user fields: 🟡.

### i18n Severity Summary

| Issue | Severity | Blocks? |
|-------|----------|---------|
| Hardcoded user-facing string | 🟡 | yes |
| String concatenation forming sentences | 🟡 | yes |
| ASCII-only validation on name/address fields | 🟡 | yes |
| Fixed-width column too short for UTF-8 | 🟡 | yes |
| Missing locale parameter on date/number format | 🔵 | no |
| `s`-suffix pluralization | 🔵 | no |
| Hardcoded currency symbol | 🔵 | no |
| Fixed-width container for translatable text | 🔵 | no |

---

## Performance and Query Optimization (mandatory)

Check changes for potential performance issues. Goal — identify code that degrades under load or with data growth.

### N+1 Queries

- **ORM loops** — DB query inside `foreach`/`for`/`while`/`map`/collection: each iteration generates a separate SQL. Patterns:
  - Iterating over a collection with lazy-loaded relations
  - Calling `find()`/`findOne()`/`get()`/`load()` in a loop instead of batch/eager load
- **Fix:** eager loading, JOIN, batch select, subquery
- Severity: N+1 in a loop with potentially large collection, 🔴; N+1 with guaranteed small collection (up to ~5, documented), 🔵

### Heavy and Inefficient Queries

- **Missing indexes** — `WHERE`/`ORDER BY`/`GROUP BY`/`JOIN ON` on columns without index (especially new migrations without index)
- **SELECT \*** — selecting all columns when only 2-3 are needed; critical for tables with large column types
- **Queries without LIMIT** — potentially unbounded row selection (lists, logs, history)
- **Complex subqueries in a loop** — aggregations (`COUNT`, `SUM`, `MAX`) repeated instead of one grouped query
- **LIKE '%...%'** — search with leading wildcard on large tables without full-text index
- **Inefficient schema changes** — schema alterations on large tables without considering locking; missing indexes on new FK columns

### Caching

- **Repeated identical queries** — same DB/API query multiple times in one request lifecycle without cache
- **Missing cache for heavy computations** — aggregations, external API calls, complex transformations
- **Invalidation** — if PR adds/modifies cache — check that when data updates, cache is cleared
- **Cache stampede** — mass cache update on expiration without lock/mutex

### Other Performance Issues

- **Loading large collections into memory** — selecting all records without pagination or chunking on growing tables
- **Synchronous external service calls in request** — HTTP requests to third-party without timeout or in a loop; operations that should be queued
- **Unnecessary serializations** — serializing large objects not needed in the response
- **Regex on large texts** — backtracking-prone regex without limits

### Performance Severity

| Issue | Severity | Blocks? |
|-------|----------|---------|
| N+1 on large collection | 🔴 | yes |
| Loading entire table into memory | 🔴 | yes |
| Query without LIMIT on growing table | 🟡 | yes |
| Missing index on FK / frequent filter | 🟡 | yes |
| Synchronous API without timeout in request | 🟡 | yes |
| Repeated queries without cache | 🟡 | no |
| SELECT * with heavy columns | 🔵 | no |
| N+1 on guaranteed small collection | 🔵 | no |
| Unnecessary serialization, logging | 🔵 | no |

---

## Dead Code (mandatory)

Check changes for dead code:

- **Unused imports** — modules/classes/functions imported but not used anywhere in the file
- **Unused variables and parameters** — declared but not read; function parameters that are ignored
- **Unreachable code** — code after `return`/`raise`/`exit`, condition branches that never execute
- **Commented code** — old code in comments instead of deletion (distinguish from explanatory comments)
- **Uncalled functions/methods** — added or changed, but no execution path calls them
- **Empty handlers** — empty `catch`, empty `if`/`else`, no-op methods without reason

Severity: commented code and unused imports are 🔵; unreachable code and uncalled functions are 🟡/🔴 depending on scope.

---

## Documentation (mandatory)

Documentation is a first-class artifact equal to code. Any mismatch between documentation and code results in Request Changes.

### What to Check

- **Docstrings** — public functions, classes, methods should have current descriptions. If changes touch the signature or behavior — docstring **must be updated**. Missing or outdated: 🟡
- **Code comments** — outdated comments describing old logic that the changes touched: 🔵
- **Typing** — new and changed functions should have parameter and return value types. Missing types: 🔵
- **README** — if changes touch installation, startup, configuration, API, or behavior — README should be updated: 🟡

### Documentation Consistency Check

Even if changes **do not touch** documentation files — check whether changes in code create a **discrepancy** with existing documentation:

- Read existing documentation (README, etc.) and compare with code behavior after changes.
- Documentation describes old behavior that changes modify: 🟡 + Request Changes.
- Documentation incomplete (does not cover new functionality, new parameters, new endpoints): 🟡 + Request Changes.
- Documentation **does not exist at all** for added public functionality: 🟡 + Request Changes.

---

## Acceptance Criteria (when tasks are available)

If the related task is identifiable from branch name or commits:

1. **Acceptance criteria** — find acceptance criteria in the task description. Compare **each item and sub-item** with the changes:
   - Item NOT implemented and no explanation why: 🟡 + Request Changes
   - Item partially implemented: 🔵, note what is missing
   - AC says one thing, implementation does another: 🔵/🟡
   - Do not write separate praise for implemented items
   - **If AC contains a list of components/pages** — check that **each** is covered by the changes. Missed component = unimplemented AC item

2. **Task comments** — if there are clarifications, additional requirements, or scope changes from PM/QA — check that changes account for them. Unimplemented clarifications: 🔵/🟡

3. **Scope compliance** — changes beyond the task: note as ℹ️ finding

---

## Inter-Service Contract (when neighboring service code is accessible)

If the codebase has access to neighboring services — use them for cross-checking. If neighboring repos are not accessible — cross-checking is impossible; **record the risk** in the report.

When changes involve external interaction (HTTP clients, webhooks, API endpoints, message brokers):

- **Sending** — are requests correctly formed, are responses/errors handled
- **Receiving** — are responses and validation consistent with clients
- **Cross-checking** — if neighboring service code is accessible, verify both sides of the contract. Discrepancy: 🟡/🔴
- **If neighboring service code is not accessible** — record residual risk as an ℹ️ finding

---

## Resource Leaks, Memory and Performance Degradation (mandatory)

Check changes for resource leak, memory, and performance degradation risks. Goal — identify code that works under low load but degrades or fails under production load.

### Unclosed Resources (resource leaks)

- **DB connections** — connection not returned to pool after use; missing deferred close or context-managed session; missing connection pool size configuration
- **File handles** — open without corresponding close. Especially critical in loops — file descriptor exhaustion
- **HTTP response bodies** — response body not closed after reading
- **Streams and cursors** — DB cursor/iterator not closed after use; stream left open on error
- **Transactions** — begin without commit/rollback on all execution paths (including error path). Unclosed transaction locks rows/tables
- **Temp files** — created without deletion after use or on error path
- **External client connections** — gRPC channels, cache connections, message broker connections without explicit closing

### Memory Leaks and Unbounded Growth

- **Growing collections in loops** — appending to array/slice/list in a long-running process (worker, daemon) without clearing
- **Unbounded caches** — in-memory cache without TTL, LRU, or max size; memory grows indefinitely
- **Event listener leaks** — subscribing to events without unsubscribing (especially in long-lived objects)
- **Circular references** — objects referencing each other
- **Accumulated goroutines/coroutines/threads** — new concurrent units created but never finish
- **Unbounded queues** — in-memory queue without max size; producer faster than consumer leads to OOM

### Connection Pool Exhaustion

- **DB pool** — missing max connections/pool size; or pool too small for load; or connections not returned due to unclosed transactions
- **HTTP client pool** — missing max idle connections or max connections per host

### Blocking Operations Without Timeout

- **Any** network operation without timeout — HTTP, gRPC, DB query, cache, message broker
- **Lock acquisition** — mutex/semaphore without timeout; deadlock under contention
- **Queue consumers** — blocking receive without timeout; thread/goroutine hangs indefinitely

### Backpressure and Flow Control

- **Unbuffered producer-consumer** — producer does not check if consumer keeps up; missing backpressure mechanism
- **Bulk operations without batching** — processing large data volumes in one request instead of chunks/batches
- **Fan-out without limits** — unbounded number of parallel goroutines/tasks/requests

### Severity Resource/Memory Issues

| Issue | Severity | Blocks? |
|-------|----------|---------|
| Unclosed transaction leading to table locking | 🔴 | yes |
| Goroutine/coroutine/thread leak in production path | 🔴 | yes |
| File handle leak in loop leading to fd exhaustion | 🔴 | yes |
| Connection pool without max limit leading to exhaustion | 🔴 | yes |
| Blocking operation without timeout leading to hang | 🔴 | yes |
| Unbounded in-memory cache leading to OOM | 🟡 | yes |
| Unbounded queue without max size | 🟡 | yes |
| Temp files without cleanup | 🟡 | yes |
| HTTP response body not closed | 🟡 | yes |
| DB cursor not closed after iteration | 🟡 | yes |
| Fan-out without semaphore/limit | 🟡 | yes |
| Event listener without unsubscribe | 🔵 | no |
| Circular references without weak refs | 🔵 | no |

---

## Observability (mandatory)

New functionality **must be** observable in production. Code that cannot be monitored cannot be debugged.

### Logging

- New business logic has structured logging at key decision points (not just errors)
- Log level is appropriate: `ERROR` for failures requiring attention, `WARN` for degraded state, `INFO` for business events, `DEBUG` for development
- Sensitive data (passwords, tokens, PII) **never** logged even at `DEBUG` level
- Log messages contain enough context for debugging without reproduction: request ID, user ID, entity ID, relevant state
- If `catch` block exists, error is logged with context before handling

### Metrics

- New endpoint has latency and error rate metrics (or relies on middleware that provides them)
- New external service integration has latency, error rate metrics
- New queue consumer has processing time, queue depth, error rate metrics
- New business process has funnel metrics (started, completed, failed)

### Tracing

- If service participates in distributed tracing, new outgoing HTTP/gRPC calls propagate trace context
- New background jobs/queue consumers create new trace spans

### Alerting

- If a new error type is added, is alerting configured (or does a task need to be created for that)?
- If a new SLO-critical path, check that existing alerts cover it

### Severity Observability

| Issue | Severity | Blocks? |
|-------|----------|---------|
| Sensitive data logged | 🔴 | yes |
| Empty `catch` without logging | 🟡 | yes |
| New functionality without any logging | 🟡 | yes |
| Missing trace propagation in distributed system | 🔵 | no |
| Missing business metrics | 🔵 | no |

---

## Idempotency and Edge Cases (mandatory)

Check changes for retry execution and boundary condition risks. Goal — ensure code works correctly during retries, race conditions, and non-standard input data.

### Idempotency

Operations that can execute more than once (webhook retry, queue redelivery, double click, cron overlap):

- If an entity is created, what happens on repeated call? Duplicate? Error? Silent skip?
- If a notification/email is sent, will the user receive it twice?
- If a payment/balance is debited, is there protection against double charging?
- If writing to an external system, is the write idempotent?
- **Pattern:** use idempotency key, `UPSERT`, database unique constraints, distributed locks

### System Edge Cases

For each input the changed code processes, check:

- **NULL / None / nil** — function receives null where it expects a value
- **Empty** — empty string, empty array, empty object
- **Zero / negative** — numeric input equals 0 or is negative where positive is expected
- **Boundary** — `MAX_INT`, very long string (>64KB), very large collection (>100K elements)
- **Unicode** — emoji, RTL text, null bytes in strings
- **Time** — time zone differences, DST transitions, leap seconds, clock skew between services
- **Concurrency** — two requests processing one entity simultaneously

### Race Conditions (deep analysis)

Beyond basic check "are there race conditions?":

- If pattern read-then-write (read value, compute, write back), another thread may change value between read and write. Use optimistic locking (version field) or pessimistic locking
- If pattern check-then-act (if file exists, read file), file may be deleted between check and read. Handle the error
- If updating counter/balance, use atomic operations instead of read-compute-write
- If multiple entities must be consistent, use transaction or saga pattern
- If distributed system, remember that distributed locks can fail (lock holder dies while holding lock)

### Severity Idempotency and Edge Cases

| Issue | Severity | Blocks? |
|-------|----------|---------|
| Double charge / duplicate financial operation | 🔴 | yes |
| Race condition on shared mutable state | 🔴 | yes |
| Missing idempotency on retryable operation | 🟡 | yes |
| Missing NULL/empty check on external input | 🟡 | yes |
| Missing optimistic/pessimistic locking | 🟡 | yes |
| Missing boundary validation on user input | 🔵 | no |
| Missing time zone handling | 🔵 | no |

---

## Severity Calibration

**First iteration rule:** determine severity **immediately and accurately** when first detected. Do not lower severity for a "softer" review — if a problem meets the criteria for 🟡, it should be 🟡 from iteration 1. A "recommendation" in iteration 1 that becomes 🟡 in iteration 2 is a calibration error in the first iteration. The developer must see the real picture immediately to correctly prioritize fixes.

### Context-Dependent Severity

The severity tables below are baselines. Adjust severity up or down based on deployment scale and blast radius:

- A missing index on a table with < 1000 rows and no SLA: 🔵. On a table with > 1M rows or on a critical path: 🟡 or 🔴.
- A missing timeout on a call to an internal service on the same network with circuit-breaking: 🔵. On a third-party API with no fallback: 🟡.
- A race condition on a counter that only runs in a background job once per day: 🔵. On a payment or inventory path: 🔴.
- An idempotency gap on an operation triggered manually by an admin: 🔵. On a webhook handler or queue consumer that auto-retries: 🟡.

When adjusting severity, note the rationale inline: `🔵 (would be 🟡 at scale — this table has < 500 rows)`.

### Severity Inflation Warning

If a review produces more than 8 🟡 findings, pause and recalibrate. Ask: are these genuine warnings, or are some of them 🔵 suggestions labeled 🟡 to be "safe"? Severity inflation causes developers to lose the signal of what actually blocks the PR. A review with 10 🟡 findings is less actionable than one with 2 🟡 and 8 🔵.

Guideline for a healthy severity distribution:

- 🔴: 0–2 per review (more than 2 is a signal the PR needs significant rework before review)
- 🟡: 1–5 per review
- 🔵 + ℹ️: unlimited

### 🔴 Blocker — Request Changes guaranteed
- SQL / commands via concatenation with user input
- Hardcoded secrets in code
- Auth/authz absent on protected endpoint
- IDOR — object ID without owner verification
- Data corruption — write without transaction where atomicity is needed
- RCE vector — `eval`, deserialization on user input
- Goroutine / thread / coroutine leak in production path
- Breaking API change without versioning
- Double charge / duplicate financial operation due to missing idempotency
- Race condition on shared mutable state without locking
- Sensitive data (passwords, tokens, PII) logged
- N+1 queries on large collection leading to degradation under load
- Loading entire table into memory without pagination/chunking
- Unclosed transaction leading to row/table locking
- File handle leak in loop leading to fd exhaustion
- Connection pool without max limit leading to exhaustion under load
- Blocking operation without timeout leading to goroutine/thread hang

### 🟡 Warning — Request Changes if unexplained
- Acceptance criterion not implemented and no explanation
- Error ignored in critical path (empty `catch`, swallowed error)
- HTTP call without timeout — hangs when service is unavailable
- N+1 queries in production endpoint without eager loading
- Retry on 4xx — infinite retries on client error
- Missing tests for new business logic
- PR breaks existing tests — changes in code break previously passing tests
- Inter-service contract violation (request/response mismatch)
- Missing idempotency on retryable operation (webhook, queue consumer)
- Missing NULL/empty check on external input in critical path
- Missing optimistic/pessimistic locking on concurrent state change
- Risk change without feature flag or rollback path
- New functionality without any logging, cannot debug in production
- Empty `catch` without logging, errors swallowed silently
- Data from external source written to DB without validation
- Missing resource requests/limits (in containerized environments)
- New env variable in code missing from deployment configuration, service starts but functionality is broken
- Query without LIMIT on growing table, unbounded selection
- Missing index on FK or frequent filter, full table scan
- Synchronous external API without timeout in request path, hang
- Docstring/README not updated when changing public API or behavior

### 🟡 Warning (architecture and systemics)
- Code duplicates existing functionality in codebase — use existing
- Solution looks like a temporary patch, not a systemic fix — propose architectural approach or recommend tech debt ticket
- Function does not have a predictable result on all execution paths
- Violation of separation of concerns — logic in wrong layer (controller, model, etc.)
- Changes degrade architecture — complicate the system without justification
- New functionality without tests — core rule violated
- Tests do not check real behavior — test implementation details or duplicate the same checks

### 🔵 Suggestion — worth fixing, does not block
- Business logic in HTTP handler/controller
- Missing docstring on public API
- Error propagation without context
- Logging via print statements instead of structured logger
- Missing index for new `WHERE`/`ORDER BY`
- Debug code in changes: `console.log`, `print`, `breakpoint`, etc.
- `TODO` / `FIXME` without a linked issue — unfinished work
- Nested ifs instead of early return — reduces readability
- Unnecessary complexity where a simpler solution exists within existing architecture
- Repeated identical queries without cache in one request lifecycle
- SELECT * with heavy columns
- N+1 on guaranteed small collection (up to ~5 elements)
- Unreachable code or uncalled functions added in changes
- Outdated comment/docstring describing old logic that changes touched

### ℹ️ Informational — cosmetic
- Commented old code instead of deletion
- Unused imports, unused variables
- Style beyond established conventions
- Minor naming inconsistencies
- Inconsistent ordering (alphabetical, ascending, etc.)
- Unnecessary serialization of large objects
- Changes outside the task scope — clarify if intentional
- Alternative approach as FYI without requiring changes

---

## Report Format

Write to `reviews/YYYY-MM-DD_HH-MM_{branch}.md`:

```markdown
# Code Review — {branch}

**Date:** YYYY-MM-DD HH:MM
**Files:** {number of files in scope}
**Decision:** Approved / Approved with Minor Comments / Request Changes

---

## Change Context

{1-3 sentences: what problem the code solves, what approach was chosen}

---

## 🔴 Blockers

### `path/to/file.go:42`
**Problem:** ...
**Impact:** ...
**Fix:** ...

---

## 🟡 Warnings

### `path/to/file.py:17`
**Problem:** ...
**Impact:** ...
**Fix:** ...

---

## 🔵 Suggestions

- `path/to/file.ts:88` — ...

---

## ✅ What Was Done Well

- ...

---

## Architectural Recommendations (outside scope)

{If there are improvements outside the current change scope — describe problem, desired state, approach. Recommend creating a separate technical task. If none — omit this section.}

---

## Summary

{1-3 sentences: overall assessment, most important findings, recommendation}
```

**Decision** (check top to bottom, first match determines):

1. If there is at least one 🔴 finding — **Request Changes**
2. If there is a 🟡 finding without an acceptable explanation from the author — **Request Changes**
3. If an acceptance criterion is not implemented without good reason — **Request Changes**
4. If there are functional changes without documentation update — **Request Changes**
5. If changes degrade architecture without justification — **Request Changes**
6. If new functionality without tests — **Request Changes**
7. If all 🟡 are explained by the author and explanation is acceptable — **Approved**
7b. If there are only 🔵 findings and all are genuine improvements (not noise) — **Approved with Minor Comments** (author may address at discretion, no re-review required)
8. If there are no 🔴 or 🟡 findings — **Approved**

> **Approved with Minor Comments** means the PR can merge as-is. The author is encouraged but not required to address 🔵 findings. No re-review cycle is needed.

---

## Comment Quality

### Communication Tone

Code review feedback should be easy to act on and respectful of the author's intent. Apply these rules to every finding:

- **Critique the code, not the author.** Write "This function could return early on empty input" not "You should handle empty input."
- **Use questions for suggestions.** "Have you considered using a batch query here?" is more effective than "Change this to use a batch query."
- **Acknowledge constraints you may not see.** If a design choice seems odd, ask before criticizing: "Is there a reason this uses a raw query instead of the ORM?" — the author may have context the diff doesn't show.
- **Distinguish severity clearly in your language.** 🔴 findings should be stated plainly ("This will cause data loss under concurrent writes"). 🔵 findings should be framed as options ("One alternative worth considering..."). Never use aggressive language for any severity.
- **Power dynamics.** Apply identical scrutiny regardless of whether you are reviewing a junior, peer, or senior developer's code. If reviewing across teams, assume good intent and ask before assuming an approach is wrong.

**Each** finding entry in the report must contain exactly three elements (1-4 sentences total):

1. **What is wrong** — specific problem in the code
2. **Impact** — what will happen if not fixed (bug, security issue, hang, data loss)
3. **How to fix** — specific recommendation or code example

This structure ensures the author immediately understands the risk and can act without additional questions.

**Bad:** "I recommend improving error handling."
**Good:** "`catch` block is empty — exception is swallowed and operation appears successful to the caller. Add `logger.error(...)` and re-throw or return an error response."

One problem — one entry. If the same problem appears in multiple places — one entry listing all locations, not separate entries for each.

**Consolidation of similar findings:** if the same **class** of problem (e.g., missing type annotation, missing docstring, debug code) appears in 3+ files — **do not** create a separate entry for each file. One entry listing all files/lines.

**Framework-aware analysis:** before publishing a finding, check whether it is standard documented framework behavior or platform convention. If the finding comes down to "add a comment explaining a standard framework pattern" — it is noise, do not publish. Know the difference between a bug and idiomatic code.

Only publish findings when a problem is found. A checklist of "what was checked and is OK" does not belong in the report — it is noise for the reader.

After writing the report, notify the user of the file path.

---

## Markdown in Report (mandatory)

These rules apply to **each** report section without exception. If not followed, lists will render as continuous text.

1. **Paragraphs** — multiple thoughts = multiple paragraphs (blank line). One paragraph — up to ~4 sentences.
2. **Bullet lists** — each item on a **new line**, starting with `- ` (dash, space). Writing `- item A - item B` on one line makes it a paragraph, not a list.
3. **Subheadings** — `**Problem:**` / `**Fix:**` on new line.
4. **Code** — variable names, paths, `file:line` in backticks.

### Example Finding Entry

```markdown
### `path/to/file.go:42`
**Problem:** `pw_proxy` may not be initialized before use.

**Impact:** property access without state guarantee.

**Fix:**
- Initialize `pw_proxy = None` before the branch (`service.py:149`).
- Or check key existence before access.
```

---

## Self-Check (before finalizing report)

Before returning the result, go through **each** item below. If any item is violated — fix it and only then return the result.

### Markdown Quality
1. If the report has a bullet list, for EACH list check that every item starts on a new line with `- `. If items are on one line, split them.
2. If a finding text is longer than 2 sentences, it must be structured: paragraphs (blank lines), bullet lists, bold subheadings. Continuous text without line breaks is an error.
3. Each finding entry contains only: problem, impact, fix. If there are "what was done well" items in problem sections or general overviews — remove.

### Severity to Decision
4. If there is at least one 🔴 or unexplained 🟡, decision must be Request Changes. If Approved is set, change to Request Changes.
5. If decision is Approved but there are uncovered acceptance criteria or missing tests for new functionality, change to Request Changes.

### Finding Quality
6. If a finding does not contain all three elements (problem, impact, fix), for EACH such finding add the missing elements.

### Completeness of Analysis
7. Check that the analysis covers **all** sections: architecture, security, data privacy, correctness, idempotency/edge cases, reliability/failure modes, resource leaks, performance, observability, inter-service contract, tests, documentation, accessibility (UI only), i18n (multi-locale apps only), dead code. If any section was skipped, go back and analyze.
8. Was data flow analysis, failure mode analysis, absence reasoning, and state analysis applied? If changes touch external dependencies or state — these checks are mandatory.
9. Comment ONLY on what you **saw in the code or diff**. Never fabricate names of constants, methods, classes, files, or variables you did not see. If you suspect something may exist — open the file and check before mentioning. Hallucination (referencing non-existent code) is a critical review defect that undermines developer trust.
10. If N+1, query without LIMIT, or loading table into memory is found, it must be 🔴/🟡. If not in report — add.
11. If changes add an operation that can execute twice (webhook handler, queue consumer, payment), check for idempotency. If risk exists and not noted — add.
12. If changes touch one branch of a switch/match/if-else chain or one endpoint from a group of similar ones — check **ALL** neighboring branches/endpoints. If a feature (masking, validation, permission check) is applied not to all — it is a gap, add as 🟡.
13. If changes add/modify permissions or role-based logic — check the full chain: (1) permission can be assigned to a real user, (2) enforced in all code paths, (3) no bypass conditions. If permission is created but no one can receive it — add as 🟡.
14. If this is a repeated review — use previous findings as a **guide for deeper analysis**. Check if the author correctly understood the problem, if the fix did not create new issues, if there are compound effects. Severity from the first iteration is final.
15. If changes implement a feature that applies to a set of entities (UI components, API endpoints, table columns, roles) — enumerate **all** entities from the code and check each.
16. If this is a repeated review (4+ times) — do not add new findings with severity 🔵/ℹ️. Focus exclusively on: whether 🔴/🟡 are fixed, whether there are regressions from fixes. If all 🔴/🟡 are fixed, Approved.
17. If changes add HTTP client calls, check for explicit timeout. If changes add a queue consumer, check for retry/backoff configuration. If changes add an external API integration, check for handling HTTP 429 (rate limit) with retry-after.
