---
name: laravel-performance
description: Use when writing or reviewing Laravel code that reads lists or large tables, loads relationships, builds reports or exports, calls external APIs, caches results, runs queued jobs, or returns large Inertia or API payloads, and when planning a feature that will grow with data. An ordered review for runtime cost with the exact Laravel APIs, grep checks and tests that prove a page does not slow down as tables grow.
---

# Laravel performance

Business apps rarely fall over from clever algorithms. They slow down because a
list loads every row, a loop fires one query per record, a request waits on a
third-party API, or a report runs in the web process. All of these look fine
with 50 rows in development and fail with 500,000 in production. This skill
covers those failures, in the order they cost the most.

## How to use it

- **While writing:** apply the rule for each area your change touches. Assume
  every user-growable table (orders, messages, clicks, invoices) has millions
  of rows.
- **When planning** a feature: walk the steps on the plan, not only on the
  code. A missing index or an unbounded export is cheaper to fix before it exists.
- **When reviewing:** walk the ten steps in order on the changed code. For each
  finding report `file:line`, severity (critical / high / medium), what happens
  as the data grows, and the fix. Say which steps did not apply.
- **Before done:** run the checks in [Verify](#verify) and add the tests listed
  under each step you touched.

If a rule here conflicts with the project's own conventions, follow the project
and say so.

---

## 1. Unbounded reads (critical)

The most common outage: something that returns or loads "all" of a growing table.

- Every list endpoint and Inertia page paginates: `->paginate()`,
  `->simplePaginate()`, or `->cursorPaginate()` for feeds and infinite scroll.
  The page size has a server-side maximum; never trust `per_page` from the client.
- Never `Model::all()` or `->get()` on a growable table outside a test or seeder.
- Batch work iterates in chunks: `->lazyById()` or `->chunkById()`. Use the `ById`
  variants when the loop updates the rows it reads, so no row is skipped.
- Exports and reports stream (`response()->streamDownload()`) or run in a queued
  job that writes a file; they never build the whole file in memory in a request.

```php
public function index(Request $request): AnonymousResourceCollection
{
    $orders = Order::query()
        ->forUser($request->user())
        ->latest('id')
        ->paginate(min($request->integer('per_page', 25), 100));

    return OrderResource::collection($orders);
}
```

**Test:** seed more rows than one page and assert the response holds one page.

## 2. N+1 queries (critical)

A loop that touches a relationship runs one query per row.

- Eager load what the code path uses: `with()`, `load()`, `loadMissing()`.
  Load only those relations; extra eager loads waste memory.
- Counts and sums of related rows come from SQL: `withCount()`, `withSum()`,
  `withExists()`, never `$model->items->count()` inside a loop.
- API Resources and Inertia props are loops too: every relation a Resource reads
  must be loaded by the query that feeds it. Use `whenLoaded()` in Resources so a
  missing `with()` shows up as a missing field instead of a hidden query.
- Child models that read their parent (`$comment->post`) use `chaperone()` on the
  parent's `hasMany`.
- Development and tests run strict so lazy loading throws:

```php
// AppServiceProvider::boot()
Model::preventLazyLoading(! $this->app->isProduction());
```

`Model::automaticallyEagerLoadRelationships()` exists, but it hides the problem
rather than fixing the query. Prefer explicit eager loading; use it only if the
project already relies on it.

**Test:** the query count is the same for 1 row and for 10 (see [Verify](#verify)).

## 3. Indexes and query shape (high)

- Every column used in `where`, `orderBy`, a join or a foreign key is indexed in
  the migration that introduces the query. `foreignId()->constrained()` indexes
  the key; plain integer columns do not.
- Composite indexes follow the query: equality columns first, then the range or
  sort column. `where('shop_id', $id)->where('status', 'paid')->latest()` wants
  `['shop_id', 'status', 'created_at']`.
- Filters that defeat an index: a function on the column (`whereDate`,
  `whereYear`, `LOWER(col)`), a leading wildcard `like '%term%'`, and `orWhere`
  across different columns. Use a range (`whereBetween('created_at', [...])`),
  a stored normalized column, or full-text search instead.
- Check a suspect query with `->explain()` on realistic data and look for a full
  table scan.

## 4. Work belongs in SQL (high)

- Filter, aggregate and sort in the query, not in PHP. `->get()->where(...)`,
  `->get()->filter(...)`, `->get()->sum(...)` load every row to discard most of them.
- Existence and counts: `->exists()`, `->count()` on the builder, never
  `->get()->isEmpty()` or `->get()->count()`.
- Select only the columns you need for large reads: `select([...])`, `pluck()`,
  or `toBase()` when models are not needed.
- Bulk writes use one statement: `upsert()`, `insert()` with chunks, or
  `update()` on a query, not `save()` in a loop.

## 5. Slow work leaves the request (high)

The user should never wait for mail, PDFs, image processing, imports, exports
or third-party APIs they do not need the answer from.

- Dispatch it to a queue. Jobs that react to a database change dispatch after
  commit: `->afterCommit()` or `public $afterCommit = true;`, so a rolled-back
  transaction never sends an email for an order that does not exist.
- Every job sets `$tries` (or `retryUntil()`), `$timeout` and `backoff()`, and
  has a `failed()` method or a monitored failed-jobs table. Jobs are idempotent:
  running one twice does not charge, send or create twice.
- Every outbound HTTP call sets a timeout. The client waits 30 seconds by
  default, which ties up a PHP worker per slow request:
  `Http::timeout(5)->connectTimeout(2)->retry(2, 200)->get(...)`.
- Independent outbound calls run together: `Http::pool(...)`.
- Never call an external API inside `DB::transaction()`; it holds row locks for
  the length of the network call.

## 6. Caching with an invalidation path (medium)

- Cache expensive reads that change rarely: dashboards, settings, catalog, home
  content. `Cache::remember($key, $ttl, fn () => ...)`, or `Cache::flexible($key,
  [$fresh, $stale], fn () => ...)` to serve a stale value while it refreshes after
  the response.
- Every cache write has a named invalidation: `Cache::forget()` in the observer,
  event listener or service that changes the data. A cache with only a TTL is
  correct only if stale data is acceptable for the whole TTL; say so.
- Keys include everything that varies the result: tenant, user, locale, filters,
  page. A key without the tenant leaks data between tenants.
- `Cache::tags()` does not work on the `file`, `database`, `dynamodb` or `storage`
  drivers. Check the project's store before using it.
- Values read repeatedly within one request or job: `Cache::memo()`.

## 7. Contention and long transactions (medium)

- Transactions hold only the writes that must succeed together, and nothing
  slow: no HTTP, no mail, no file processing inside `DB::transaction()`.
- `lockForUpdate()` locks rows until commit; lock the fewest rows for the
  shortest time, and always in the same order to avoid deadlocks.
- Work that must not run twice at once (a sync, a payout run) uses
  `Cache::lock($key, $seconds)->block($wait, fn () => ...)` or the
  `WithoutOverlapping` job middleware. Scheduled commands use `->withoutOverlapping()`
  and, on more than one server, `->onOneServer()`.
- Hot counters (views, clicks) are not updated row by row per request. Buffer
  them (cache `increment()` flushed by a scheduled job) or insert events and
  aggregate later.

## 8. Memory in long-running processes (medium)

Queue workers and Octane keep the application in memory between jobs and requests.

- No request data in static properties or container singletons; it leaks into
  the next request and grows forever.
- Large loops use `lazyById()` / `cursor()`, not collections that accumulate.
- Workers restart on deploy (`php artisan queue:restart`) and are capped with
  `--max-jobs` / `--max-time` so a slow leak cannot take the server down.

## 9. Migrations on large tables (medium)

- Adding an index or a `NOT NULL` column to a large table can lock it for
  minutes. Add nullable columns first, backfill in chunks from a job, then add the
  constraint. On PostgreSQL build big indexes with
  `DB::statement('CREATE INDEX CONCURRENTLY ...')` in a migration that sets
  `public $withinTransaction = false;`, since `CONCURRENTLY` cannot run in a transaction.
- Backfills never run as one `UPDATE` over millions of rows inside a migration.

## 10. Payload size (medium)

- Inertia pages send only the props the page renders. Map models to the fields
  the page needs instead of passing whole models with every relation.
- Props that are slow to compute or below the fold load after the page:
  `Inertia::defer(fn () => ...)`. Props only some visits need:
  `Inertia::optional(fn () => ...)`.
- API Resources do not expose relations the client did not ask for.
- Images are resized and served from storage or a CDN, never as originals.

---

## Verify

Run what applies and include the results in your summary.

```bash
grep -rnE '::all\(\)' app
grep -rnE '\->get\(\)->(where|filter|sum|count|isEmpty|first)\(' app
grep -rnE 'Http::(get|post|put|patch|delete)\(' app        # each call needs a timeout
grep -rnE 'DB::transaction' app                          # read each: no HTTP, mail or files inside
grep -rnE 'implements ShouldQueue' app                  # each job: tries, timeout, backoff, failure path
grep -rnE 'Cache::(remember|put|forever|flexible)\(' app  # each key: invalidation + tenant in key
```

Each grep hit is a candidate, not a verdict: read it and either fix it or state
why it is safe.

**N+1 test.** The number of queries must not grow with the number of rows:

```php
public function test_orders_index_does_not_grow_queries_with_rows(): void
{
    $user = User::factory()->create();

    $count = function (int $rows) use ($user): int {
        Order::factory()->count($rows)->for($user)->hasItems(2)->create();
        DB::flushQueryLog();
        DB::enableQueryLog();

        $this->actingAs($user)->getJson('/api/v1/orders')->assertOk();

        return count(DB::getQueryLog());
    };

    $this->assertSame($count(1), $count(10));
}
```

**Index check.** For each new query on a growable table, name the index it uses.
If unsure, run `->explain()` against a seeded database with realistic volume.

## Report format

```
## Performance review: <scope>

Critical
- app/Http/Controllers/MessageController.php:24 index() returns ->get() on
  messages; a creator with 40k messages loads all of them. Fix: ->paginate(30)
  + test that a second page exists.

High
- app/Services/InvoiceService.php:88 Http::post() to the e-invoicing API has no
  timeout and runs inside DB::transaction(). Fix: move the call to a queued job
  dispatched afterCommit, with Http::timeout(10).

Checked, nothing found: 4 (SQL work), 8 (memory)
Not applicable: 9 (no migration in this change)
Verify: N+1 test added for messages index (3 queries for 1 and 10 rows).
```
