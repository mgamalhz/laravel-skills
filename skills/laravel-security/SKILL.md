---
name: laravel-security
description: Use when writing or reviewing Laravel code that touches authorization, user input, tenants, money, files, authentication or OTP, webhooks, raw queries or API responses — and always before calling such a feature done. An ordered review workflow for business applications with the exact Laravel APIs, grep checks and tests that prove each control is in place.
---

# Laravel security

Business applications are rarely broken through exotic attacks. They leak one
customer's orders to another, let a user set `role=admin` through a form, or
accept the same payment webhook twice. This skill covers those failures, in the
order they cost the most.

## How to use it

- **While writing:** apply the rule for each area your change touches.
- **When reviewing** (a diff, a feature, or "is this secure?"): walk the ten
  steps below in order on the changed code. For each finding report
  `file:line`, severity (critical / high / medium), what an attacker does, and
  the fix. Say explicitly which steps did not apply.
- **Before done:** run the checks in [Verify](#verify) and add the tests listed
  under each step you touched.

Never weaken a control to make a test pass. If a rule here conflicts with the
project's own conventions, follow the project and say so.

---

## 1. Authorization on every entry point (IDOR) — critical

Authentication says *who*; authorization says *may they touch this record*.
Most real breaches in business apps are a missing second check.

- Every route is behind `auth` (or an explicit, commented reason it is public).
- Every action on a specific record authorizes against that record:
  `Gate::authorize('update', $order)`, `$this->authorize(...)`, or `can:update,order`
  route middleware. Listing endpoints filter by what the user may see.
- Nested resources use scoped bindings so `/shops/1/orders/99` cannot load
  another shop's order: `Route::scopeBindings()->group(...)`.
- Never decide ownership from an ID in the request body (`user_id`, `shop_id`).
  Take it from the authenticated user or the bound parent.

```php
public function update(UpdateOrderRequest $request, Order $order)
{
    Gate::authorize('update', $order);
    // ...
}
```

**Test:** for every policy method, one test where a *different* user gets 403
(or 404 if the app hides existence).

## 2. Tenant and owner scoping — critical

- Multi-tenant data is filtered in one place: a global scope or a query
  method on the model, not a `where` remembered in each controller.
- Validation lookups are scoped too: `Rule::exists('products', 'id')->where('shop_id', $shopId)`.
- Background jobs carry the tenant explicitly; there is no "current user" in a queue.
- Caches include the tenant in the key.

**Test:** a user of tenant A requests tenant B's record by ID and fails; a
list endpoint for A never contains B's rows.

## 3. Mass assignment and input — critical

- Models written from requests declare `$fillable`. `$guarded = []` is only
  acceptable on models never filled from user input.
- Write from validated data only: `$request->validated()` or
  `$request->safe()->only([...])`. Never `$request->all()` / `$request->input()`
  into `create()`, `update()` or `fill()`.
- Privilege-bearing fields (`role`, `is_admin`, `status`, `price`, `balance`,
  `tenant_id`) are never fillable from a user-facing request.
- Validate shape and type: `Rule::enum(Status::class)`, `integer`, `array:key1,key2`
  for nested input, `max:` on strings and arrays.
- In development and tests: `Model::shouldBeStrict()` so silently discarded
  attributes and lazy loading fail loudly.

## 4. Data exposure in responses and logs — high

- API responses go through API Resources that list fields explicitly. Never
  `return $model` or `->toArray()` of a model with internal columns.
- Secrets and internal fields are in `$hidden`.
- `APP_DEBUG=false` outside local; exception messages never reach clients in production.
- Logs and exception context never include tokens, passwords, OTPs or full
  card/IBAN numbers.

## 5. Authentication, OTP and sessions — high

- Every login, OTP send, OTP verify and password reset route is rate limited
  with a named limiter keyed by identifier **and** IP:

```php
RateLimiter::for('otp', fn (Request $request) => [
    Limit::perMinute(3)->by('otp:'.$request->input('phone')),
    Limit::perMinute(10)->by('otp-ip:'.$request->ip()),
]);
```

- OTP codes are stored hashed (`Hash::make` / `Hash::check`), expire in
  minutes, are single-use, and lock after N failed attempts.
- Session is regenerated on login (`$request->session()->regenerate()`).
- Sanctum tokens have abilities and an expiry; logout revokes the token.
- Responses don't reveal whether an account exists ("If this number is
  registered, a code was sent").

**Test:** the N+1th attempt is throttled; an expired or used code fails.

## 6. File uploads — high

- Validate with the rule object: `File::types(['pdf', 'jpg', 'png'])->max('5mb')`.
  It checks the detected MIME type, not only the extension.
- Store on a private disk with a generated name (`$file->store('docs', 'private')`).
  Never `storeAs()` with the client's filename.
- Serve through an authorized controller or a short-lived `temporaryUrl()`.
- Treat SVG and HTML uploads as script; do not serve them inline from the app's origin.

## 7. Webhooks and outbound requests — high

- Verify the signature on the **raw body** with `hash_equals`, before parsing
  or touching the database. Reject stale timestamps when the provider sends one.
- Make processing idempotent: store the provider's event ID under a unique
  index and skip duplicates. Providers retry; attackers replay.
- Acknowledge fast and process in a queued job.
- Never fetch a URL supplied by a user without an allowlist (SSRF), and never
  follow redirects to internal addresses.

## 8. Injection and output encoding — high

- Raw SQL uses bindings only: `whereRaw('amount > ?', [$min])`. No variables
  interpolated into `DB::raw`, `whereRaw`, `orderByRaw`, `selectRaw`.
- Sort and filter columns from the request are mapped through an allowlist
  before reaching `orderBy()` / `where()`.
- Blade prints user content with `{{ }}`. `{!! !!}` only for HTML you generated
  or sanitized.
- Spreadsheet exports prefix cells starting with `=`, `+`, `-`, `@`, tab or CR
  with `'` (formula injection).
- Shell commands use `Process::run([...])` with an argument array, never a string
  built from input.

## 9. Money, stock and concurrency — high

- Amounts are integers in minor units (or decimal strings with `bcmath`), never floats.
- Prices, discounts and totals are computed on the server, never trusted from the client.
- Read-check-write on balances, stock or seats happens inside
  `DB::transaction()` with `lockForUpdate()` on the row being changed.
- Payment callbacks are idempotent (see step 7) and verify the amount and
  currency against the order, not only the status.

## 10. Configuration and dependencies — medium

- `composer audit` is clean or every advisory is understood and tracked.
- Secrets live in environment variables, never in committed config or code.
- CORS allows only the app's own origins; cookies are `secure` and `http_only`.
- Links that grant access without login use `URL::temporarySignedRoute()`.

---

## Verify

Run what applies and include the results in your summary.

```bash
composer audit
php artisan route:list --except-vendor --json   # every non-public route has auth + authorization
grep -rn 'guarded = \[\]' app/Models
grep -rnE '\$request->(all|input)\(\)' app/Http
grep -rnE '(whereRaw|orderByRaw|selectRaw|DB::raw)\(.*\$' app
grep -rn '{!!' resources/views
grep -rnE "storeAs\(.*getClientOriginalName" app
```

Each grep hit is a candidate, not a verdict: read it and either fix it or state
why it is safe.

## Report format

```
## Security review — <scope>

Critical
- app/Http/Controllers/OrderController.php:42 — update() has no authorization;
  any logged-in user can edit any order by ID. Fix: Gate::authorize('update', $order)
  + test that another user gets 403.

High
- ...

Checked, nothing found: 4 (data exposure), 6 (files), 10 (config)
Not applicable: 7 (no webhooks in this change)
Verify: composer audit clean; 2 grep hits reviewed, both safe (reasons).
```
