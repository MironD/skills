# skills

A collection of [Claude Code](https://docs.claude.com/en/docs/claude-code) skills for writing and reviewing code — one universal quality baseline, plus focused skills for architecture, security, production readiness, testing, Go, and full PR review.

## Skills

| Skill | Use when |
|---|---|
| [`code-quality`](code-quality/SKILL.md) | Writing or reviewing code and language-agnostic quality principles apply — readability, DRY, error handling, logging, config/secrets, git hygiene, backward compatibility, performance, and concurrency basics. |
| [`architecture`](architecture/SKILL.md) | Reviewing or writing code where the concern is structural — layer boundaries, coupling, SOLID violations, design patterns, API contracts, complexity — rather than syntax or line-level correctness. |
| [`security`](security/SKILL.md) | Reviewing code for security vulnerabilities — authentication, authorization, injection, secrets, data protection, infrastructure and dependency risks. |
| [`production-review`](production-review/SKILL.md) | Judging a PR's readiness to run in production — failure modes, resilience of external dependencies, deployment and migration safety, observability. |
| [`testing`](testing/SKILL.md) | Reviewing the quality of tests in a PR — coverage, structure, test doubles, flakiness, and severity calibration for missing or weak tests. |
| [`go`](go/SKILL.md) | Writing or reviewing Go code — language idioms, error handling, concurrency, HTTP clients/servers, testing conventions, and project layout. |
| [`code-review`](code-review/SKILL.md) | Performing a full local code review of a diff, branch, or PR and producing a written review report. |

## How they fit together

`code-quality` is the language-agnostic base: it's meant to be applied on every write-or-review task, silently on writes and as the universal rulebook on reviews. Language-specific skills (`go`, with more to follow) layer language idioms on top of it. Domain skills (`architecture`, `security`, `production-review`, `testing`) each own one lens and can be invoked standalone.

`code-review` is the orchestrator: given a diff, branch, or PR, it runs the review end to end and treats `security`, `production-review`, and `code-quality` as mandatory sub-analyses — their findings map directly onto its own 🔴/🟡/🔵/ℹ️ severity scale. It additionally loads `architecture` whenever the diff changes architecture (new packages, layer boundaries, coupling, design patterns, API contracts), and folds `testing` concerns into the same report; a matching language skill is consulted for language-specific idioms wherever the diff calls for it.

## Installation

Claude Code loads skills from `.claude/skills/` — either at the project level or globally under `~/.claude/skills/`. Clone this repo once and symlink in whichever skills you want:

```bash
git clone https://github.com/MironD/skills.git

mkdir -p ~/.claude/skills
for skill in code-quality architecture security production-review testing go code-review; do
  ln -s "$(pwd)/skills/$skill" ~/.claude/skills/"$skill"
done
```

Use a project's `.claude/skills/` directory instead of `~/.claude/skills/` to scope a skill to one repo.

## Adding a skill

Each skill is a directory containing a `SKILL.md` with YAML frontmatter:

```markdown
---
name: my-skill
description: Use when <the specific situation that should trigger this skill>.
---

<skill body>
```

The `description` is what Claude matches against the current task, so lead with "Use when" and state the trigger precisely — vague descriptions cause skills to fire (or fail to fire) unpredictably.

## License

[MIT](LICENSE)
