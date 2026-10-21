---
name: laravel-done-checklist
description: Use before saying any Laravel change is done, fixed or ready — a feature, a bug fix, a migration, a refactor. Runs the checks that prove it works (tests, Pint, Larastan, N+1, indexes, migrations, security, diff review) and reports what was verified and what was not.
---

# Laravel done checklist

"Done" means verified, not written. Before you tell the user a change is
finished, run every item that applies to it and report the result.

## Rules

- **Run it, don't predict it.** Only report a check as passing if you ran it in
  this session and read the output. "Should pass" is not a result.
- **Say what you could not run.** No database, missing tool, failing
  environment: report it as not verified, with the reason. Never skip silently.
- **Don't weaken the check to pass it.** No skipped tests, lowered Larastan
  level, `@phpstan-ignore` or removed assertions to get a green result —
  unless the user agrees, and then say so in the summary.
- **Fix, then re-run.** A failing item is fixed and the item run again; the
  summary reports the final run.

## The checklist

### 1. Tests

```bash
php artisan test            # or: vendor/bin/pest
```

- The whole suite passes, not only the new tests.
- The change has tests for its behaviour **and its failure paths**: a user who
  is not allowed, invalid input, a duplicate request, an empty result.
- A bug fix has a test that failed before the fix.

### 2. Formatting

```bash
vendor/bin/pint --test      # then vendor/bin/pint to fix
```

### 3. Static analysis

```bash
vendor/bin/phpstan analyse  # Larastan, at the project's configured level
```

No new errors. If the project has no Larastan, say so; don't install it unasked.

### 4. N+1 and lazy loading

The test suite should run with strict models so lazy loading fails loudly. If
the project doesn't do this yet, propose adding it to `AppServiceProvider::boot()`:

```php
Model::shouldBeStrict(! app()->isProduction());
```

For each list endpoint, report or export you touched: relationships used in the
loop are eager loaded (`with()`, `load()`, `withCount()`), and the number of
queries does not grow with the number of rows.

For lists, reports, exports, external calls or caching, also run the
**Verify** section of `laravel-performance`.

### 5. Indexes

Every new column used in `where`, `orderBy`, a join or a foreign key has an
index in the migration. Composite indexes match the order of the query's
filters.

### 6. Migrations

```bash
php artisan migrate && php artisan migrate:rollback && php artisan migrate
```

Run against a local or test database only.

- `down()` works.
- Migrations that already ran outside your machine are never edited; a change
  to them is a new migration.
- Safe on a large table: no long table locks during deploy, new `NOT NULL`
  columns have a default or a backfill, renames and type changes are split into
  steps the running code can survive.
- No data is dropped without an explicit, user-approved plan.

### 7. Security

For the areas the change touches, run the **Verify** section of
`laravel-security` (authorization, tenant scoping, mass assignment, files,
webhooks, raw SQL). Always:

```bash
composer audit
```

### 8. Diff review

```bash
git diff
git diff --staged
```

Read it as a reviewer would:

- Every changed line serves the task; no unrelated edits.
- No debug leftovers: `dd(`, `dump(`, `ray(`, `var_dump(`, `Log::debug` added for testing.
- No commented-out code, no secrets, no `.env` changes committed.
- New config keys exist in `.env.example`.

## Report

End with a short summary in this shape:

```
Done: <one line on what changed>

Verified
- Tests: 142 passed (6 new: 2 authorization, 1 invalid input, ...)
- Pint: clean
- Larastan: no new errors
- composer audit: no advisories
- Migrations: migrate / rollback / migrate OK
- N+1: orders index eager-loads customer and items

Not verified
- Webhook signature test: no provider secret in the test env

Needs your decision
- New status transition paid → refunded (business rule)
```

Leave out sections that are empty, but never leave out "Not verified" when
something was skipped.
