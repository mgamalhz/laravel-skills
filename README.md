# Skills

Agent skills for well-engineered Laravel backends. They work with Claude Code,
Cursor, Codex and any agent that reads the [Agent Skills](https://agentskills.io) format.

> `0.x` — usable today, still being measured. Feedback and issues welcome.

| Skill | Use it for |
|---|---|
| [`laravel-security`](skills/laravel-security/SKILL.md) | Writing and reviewing Laravel code that touches authorization, tenants, input, money, files, OTP, webhooks or raw SQL. An ordered 10-step review with grep checks and the tests that prove each control. |
| [`engineering-guard`](skills/engineering-guard/SKILL.md) | Any non-trivial change: routes the agent to the right skills while working and runs a done-checklist (tests, format, static analysis, N+1, migrations, security) before it says "done". |

## Install

**Claude Code**

```bash
/plugin marketplace add mgamalhz/skills
/plugin install laravel-engineering@mgamalhz
```

**Any agent** (via [skills.sh](https://skills.sh))

```bash
npx skills add mgamalhz/skills
npx skills add mgamalhz/skills --skill laravel-security   # just one
```

**Manually:** copy a folder from `skills/` into your agent's skills directory
(e.g. `.claude/skills/`).

## Works well with

These are maintained elsewhere — install them from their source:

- [`spatie/guidelines-skills`](https://github.com/spatie/guidelines-skills) — Laravel/PHP code style.
- [`trailofbits/skills`](https://github.com/trailofbits/skills) — `differential-review`, `static-analysis`, `fp-check` for security review.
- [`mgamalhz/foundation`](https://github.com/mgamalhz/foundation) — `state-lifecycle` and `audit-trail` skills ship with its packages.

`engineering-guard` uses whichever of these are installed and skips the rest.

## Roadmap

- [ ] Evals for `laravel-security`: a Laravel app with planted vulnerabilities, scored with and without the skill.
- [ ] `laravel-architecture`: when to use Actions, Services, DTOs and domain folders — and when not to.

## License

MIT
