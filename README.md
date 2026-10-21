# Laravel skills

Agent skills for well-engineered Laravel backends: secure, fast and verified
before they're called done. They work with Claude Code, Cursor, Codex and any
agent that reads the [Agent Skills](https://agentskills.io) format.

> `0.x` — usable today, still being measured. Feedback and issues welcome.

| Skill | Use it for |
|---|---|
| [`laravel-security`](skills/laravel-security/SKILL.md) | Writing and reviewing code that touches authorization, tenants, input, money, files, OTP, webhooks or raw SQL. An ordered 10-step review with grep checks and the tests that prove each control. |
| [`laravel-performance`](skills/laravel-performance/SKILL.md) | Lists, large tables, relationships, reports, external APIs, caching, queues and Inertia payloads. An ordered 10-step review with grep checks and a query-count test that proves a page does not slow down as data grows. |
| [`laravel-architecture`](skills/laravel-architecture/SKILL.md) | Where code lives: thin controllers, services over Eloquent, state machines for statuses, transaction vs queue boundaries, and the layers not to add (repositories, use cases, mappers). |
| [`laravel-done-checklist`](skills/laravel-done-checklist/SKILL.md) | Before calling any change done: tests (including failure paths), Pint, Larastan, N+1, indexes, migrations, security and a diff review — with an honest report of what was and wasn't verified. |

## Install

**Claude Code**

```bash
/plugin marketplace add mgamalhz/laravel-skills
/plugin install laravel-engineering@mgamalhz
```

**Any agent** (via [skills.sh](https://skills.sh))

```bash
npx skills add mgamalhz/laravel-skills
npx skills add mgamalhz/laravel-skills --skill laravel-security   # just one
```

**Manually:** copy a folder from `skills/` into your agent's skills directory
(e.g. `.claude/skills/`).

## Works well with

General-purpose skills maintained elsewhere — install them from their source:

- [`karpathy-guidelines`](https://github.com/forrestchang/andrej-karpathy-skills) — small, surgical changes; state assumptions; no speculative abstractions.
- [`obra/superpowers`](https://github.com/obra/superpowers) — pick `systematic-debugging` and `verification-before-completion`.
- [`trailofbits/skills`](https://github.com/trailofbits/skills) — `differential-review` and `sharp-edges` for deeper security review.
- [`mgamalhz/foundation`](https://github.com/mgamalhz/foundation) — `state-lifecycle` and `audit-trail` ship with its packages.

## Roadmap

- [ ] Small evals: a Laravel app with planted vulnerabilities and N+1s, scored with and without the skills.

## License

MIT
