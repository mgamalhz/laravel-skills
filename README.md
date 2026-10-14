# Laravel skills

Agent skills for well-engineered Laravel backends: secure, fast and verified
before they're called done. They work with Claude Code, Cursor, Codex and any
agent that reads the [Agent Skills](https://agentskills.io) format.

> `0.x` — usable today, still being measured. Feedback and issues welcome.

| Skill | Use it for |
|---|---|
| [`laravel-security`](skills/laravel-security/SKILL.md) | Writing and reviewing code that touches authorization, tenants, input, money, files, OTP, webhooks or raw SQL. An ordered 10-step review with grep checks and the tests that prove each control. |
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

- [ ] `laravel-queries` — N+1, indexes, large tables, locking, and safe migrations on MySQL and PostgreSQL.
- [ ] `laravel-architecture` — Actions, Services, DTOs, domain folders, and when not to add a layer.
- [ ] Small evals: a Laravel app with planted vulnerabilities and N+1s, scored with and without the skills.

## License

MIT
