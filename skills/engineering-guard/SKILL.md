---
name: engineering-guard
description: Use on any non-trivial change to a backend codebase (a feature, a fix touching data or permissions, a migration, a refactor) — to pick the right specialised skills while working and to run the done-checklist before saying the work is finished.
---

# Engineering guard

A short router and a finish line. It does not repeat what specialised skills
say; it decides which of them apply and makes sure the work is verified before
it is called done.

## 1. Before writing code: route to the right skills

Read the task and the files it touches, then load the skills that apply. Use
only the ones installed; skip the rest silently.

| The change touches | Load |
|---|---|
| Authorization, user input, tenants, money, files, auth/OTP, webhooks, raw SQL, API responses | `laravel-security` |
| Queries, relationships, migrations, indexes, large tables, reports | `laravel-database-optimization`, and `supabase-postgres-best-practices` on PostgreSQL |
| A status column that moves through steps | `state-lifecycle` (mgamalhz/foundation) |
| Staff, money or access actions that must be traceable | `audit-trail` (mgamalhz/foundation) |
| Any PHP / Laravel code | `spatie-laravel-php` for style and naming |

Also read the project's own `AGENTS.md` / `CLAUDE.md`. Project conventions win
over any skill; say so when they conflict.

## 2. While writing: proportion

- Match the existing structure of the codebase before inventing a new one.
- Add a layer (Action, Service, DTO, interface, repository) only when it removes
  real duplication or isolates real complexity. A CRUD endpoint does not need
  a service, and Eloquent does not need a repository in front of it.
- Prefer the framework's or a proven library's solution over custom code.
- Keep functions short and named for what they do; no dead code, no commented-out
  code, no debug output.

## 3. Before saying done: the checklist

Run each item that applies and report the result in one line each. If something
cannot run (missing tool, no database), say so; never claim it passed.

1. **Tests** — the suite passes, and the change has tests for its behaviour,
   including the failure paths (forbidden user, invalid input, duplicate request).
2. **Format** — the project formatter is clean (`vendor/bin/pint --test`).
3. **Static analysis** — Larastan / PHPStan at the project's level, no new errors.
4. **Queries** — no N+1 on the paths you touched (tests run with
   `Model::preventLazyLoading()` or `Model::shouldBeStrict()`); new filters and
   foreign keys have indexes.
5. **Migrations** — reversible, and safe on a large table (no long locks, no
   destructive change without a backfill plan).
6. **Security** — the `laravel-security` Verify section for the areas you touched;
   `composer audit` clean.
7. **Review the diff** — read `git diff` as a reviewer would. With Trail of Bits
   skills installed, run `differential-review` on security-relevant changes.
8. **Summary** — what changed, what was verified, what was not, and any risk the
   human should decide on (new state transitions, permission changes, data migrations).

Done means verified, not written.
