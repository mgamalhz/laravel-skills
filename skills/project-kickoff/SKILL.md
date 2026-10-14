---
name: project-kickoff
description: Use at the start of a new backend project or the first session in an unfamiliar one — to learn the stack and conventions, propose only the packages and skills the project needs now, and set up the quality gates before features are built. Also use at the end of a feature to note work worth extracting for reuse.
---

# Project kickoff

The first hour of a project decides whether every later change is consistent.
This skill makes that hour deliberate and short.

## 1. Learn the project (read, don't change)

- `composer.json`, `package.json`: framework and PHP versions, installed packages.
- `README`, `AGENTS.md`, `CLAUDE.md`, `.ai/guidelines/`: written conventions.
- `app/` layout: is code grouped by type (`Services/`, `Actions/`) or by domain
  (`Domain/Billing/...`)? Follow what exists.
- `routes/`, `config/auth.php`: guards, API vs web, multi-tenancy.
- `tests/`: framework (Pest / PHPUnit), how factories and auth are used.
- Database driver and whether CI exists.

Summarise the findings in five lines or fewer.

## 2. Propose what to add — only what is needed now

- **Packages:** if the developer keeps a catalog of reusable packages (for
  example `CATALOG.md` in `mgamalhz/foundation`), read it and propose only the
  entries this project needs now, one line of reason each. Otherwise propose
  well-maintained packages for needs the project already has. Never install
  "for later".
- **Skills:** propose the specialised skills that match the stack (see
  `engineering-guard` for the routing table).
- **Conventions:** if the project has no `AGENTS.md`, propose a short one:
  commands to test/format/analyse, folder conventions, commit message format.

**Wait for approval before installing or writing anything.**

## 3. Set up the quality gates

Once approved, make sure these run locally and in CI before the first feature:

- tests (Pest or PHPUnit) with `Model::shouldBeStrict()` in the test bootstrap
- formatter (`pint`)
- static analysis (Larastan) at a level the codebase passes today
- `composer audit`

Report what was added and the command for each.

## 4. At the end of each feature: harvest

Ask once: did this feature build something generic that another project will
need too (an audit log, an OTP flow, a webhook guard, an export)? If yes and the
developer keeps a harvest list (for example `HARVEST.md`), add one line:
what it is, which project, and whether it has now been seen twice. Do not copy
client code into shared packages; record the design, not the code.
