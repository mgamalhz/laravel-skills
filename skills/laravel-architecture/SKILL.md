---
name: laravel-architecture
description: Use when starting a Laravel project or module, adding a feature that spans controllers, services, models, jobs and events, introducing a status or workflow, handling money or multi-step writes, or reviewing where code lives. A small, opinionated structure for business applications (thin controllers, services over Eloquent, state machines, transaction vs queue boundaries) and a list of layers not to add.
---

# Laravel architecture

Most Laravel codebases that become hard to change did not lack patterns; they
had logic in the wrong place. Business rules in controllers, statuses changed
from five files, an email sent for an order that rolled back, or the opposite
problem: repositories, interfaces and mappers wrapped around Eloquent for no
benefit. This skill keeps the structure small and puts each kind of code in one
predictable place.

## How to use it

- **While writing:** put each piece of code where the rules below say. When a
  rule says "only when", do not add the layer until the condition is true.
- **When planning** a feature or module: decide the module, the service that
  owns the write, the statuses and who may change them, and what runs in the
  transaction versus the queue. Write that down before code.
- **When reviewing:** walk the steps in order on the changed code. For each
  finding report `file:line`, what will hurt when the code changes, and the fix.
- **Before done:** run the checks in [Verify](#verify).

The project's own conventions win. If its `CLAUDE.md` or existing code does it
differently and consistently, follow the project and say so.

---

## 1. Request flow

```
FormRequest -> Controller -> Service -> Model
                    |
              Resource / Inertia props
```

- **Controllers are traffic control.** Receive validated input, call one service
  method, return a Resource, an Inertia response or a status code. No business
  rules, no queries beyond route model binding, no `try`/`catch`, no building
  response arrays by hand.
- **Validation lives in a FormRequest**, including authorization (`authorize()`)
  or a policy call. Never `$request->validate()` in the controller.
- **Route model binding** instead of `Model::find($id)`; scoped bindings for
  nested resources.
- **Resources are the only path to JSON** for APIs. Inertia pages receive props
  shaped by a Resource or a dedicated method, never raw models with every attribute.
- **Services never receive the `Request`.** They take validated data: typed
  parameters, `$request->validated()`, or a DTO (see step 3).

```php
public function store(StoreOrderRequest $request, OrderService $orders): OrderResource
{
    $order = $orders->place($request->user(), $request->validated());

    return OrderResource::make($order);
}
```

## 2. Services hold the business logic

- One service per business concept (`OrderService`, `PayoutService`), not one per
  model and not one per controller method. It owns every write for that concept.
- Services use Eloquent directly. Reusable reads become query scopes on the
  model; a read with real complexity (geo search, ranking, reports) becomes a
  dedicated query class.
- A service method does one business operation and names it in business terms:
  `place()`, `cancel()`, `markPaid()`, not `update()` with a flag.
- Models hold relationships, casts, scopes and small derived values. Not
  workflows, not HTTP, not mail.

## 3. Add a layer only when it removes a real problem

| Layer | Add it when | Not when |
|---|---|---|
| DTO / data object | data crosses several layers, has many fields, or comes from more than one source (request, job, import) | a method takes two or three typed parameters |
| Interface + implementations | there is a real seam: payment gateway, SMS/OTP provider, matching strategy, AI provider; or a second implementation exists or is planned | there is one implementation "in case" |
| Query class | a read is complex, reused, or needs its own tests | a `where` that fits in a scope |
| Action class | one operation is shared by a controller, a job and a command, and the project already uses actions | every method becomes an action by default |

**Do not add:** repositories over Eloquent, use-case/interactor classes,
DTO-to-model mappers, a base service class, or an interface for every service.
They double the files touched per change without isolating anything that
actually changes.

## 4. Statuses are a state machine

Any model with a lifecycle (order, booking, request, invoice, subscription):

- The status is a backed enum. The allowed transitions are declared in one
  place: a method on the enum (`canTransitionTo()`) or a transition table.
- Exactly one service performs transitions. Nothing else writes the `status`
  column: no `$order->update(['status' => ...])` in controllers, jobs or listeners.
- The transition method checks the current status under a row lock, writes the
  new status and a history row (who, from, to, when, reason) in the same
  transaction, and fires a domain event.
- Illegal transitions throw a domain exception; they are not silently ignored.

```php
public function cancel(Order $order, User $by, string $reason): Order
{
    return DB::transaction(function () use ($order, $by, $reason) {
        $order = Order::query()->lockForUpdate()->findOrFail($order->id);
        $this->transition($order, OrderStatus::Cancelled, $by, $reason);

        return $order;
    });
}
```

**Test:** every allowed transition succeeds; at least one illegal transition
throws and leaves the status unchanged.

## 5. Transaction or queue: the consistency boundary

Ask of every step: "if this fails, must the whole operation fail?"

- **Yes, it is business-critical:** money movements, order and its items, stock,
  balances, ownership changes. Do it inside one `DB::transaction()`, all or nothing.
- **No, it is a side effect:** email, push, SMS, webhooks out, search indexing,
  analytics, PDFs, cache warming. Do it after commit in a queued job or a queued
  listener, idempotent and retryable.

Rules:

- Side effects dispatch after commit: `->afterCommit()` on the job, or events
  implementing `ShouldDispatchAfterCommit`, so a rollback never leaves a sent email
  for an order that was never saved.
- No HTTP calls, mail or file processing inside a transaction.
- Every money movement writes a ledger row in the same transaction as the
  balance or status change it explains. Amounts are integers in minor units or
  decimal strings, never floats.
- Consumers of side effects tolerate delays and duplicates (idempotency keys,
  upserts, "already sent" checks).

## 6. Side effects through events

- A service fires a domain event after a business operation (`OrderPlaced`,
  `OrderCancelled`). Notifications, audit entries, activity feeds and
  integrations listen to it. The core service does not know they exist.
- Listeners that do I/O implement `ShouldQueue`.
- Do not use model observers for business side effects; they fire on every
  save, including seeders and imports, and hide the flow from the service.

## 7. Errors

- Services throw domain exceptions with a stable code (`OrderNotCancellable`).
- One place renders them: `->withExceptions()` in `bootstrap/app.php` maps
  exceptions to the project's error envelope and HTTP status.
- Controllers and services do not catch exceptions to turn them into responses.
  Catch only to add context, retry, or compensate, and rethrow or log.

## 8. Module structure

- **Small app or a few modules:** Laravel's default folders (`app/Models`,
  `app/Services`, `app/Http/...`).
- **Larger app, several business areas:** group by module and keep HTTP apart:

```
app/
  Domain/<Module>/   Models/ Services/ Enums/ Events/ Listeners/ Jobs/ Data/
  Http/
    Controllers/<Module>/
    Requests/<Module>/
    Resources/<Module>/
```

- A module talks to another through its service or its events, not by writing
  to the other module's tables.
- Pick one structure per project and keep it; mixed structures are worse than
  either.

## 9. API stability

- Version the API from the first endpoint (`/api/v1`).
- Changes are additive: new fields and endpoints are fine; renaming or removing
  a field, or changing its type or meaning, needs a new version or a deprecation
  period agreed with the clients.
- One success shape and one error shape across the API, produced by Resources and
  the exception handler, not by hand.

## 10. Configuration

- `env()` is called only inside `config/*.php`. Everywhere else use `config()`;
  after `php artisan config:cache`, `env()` outside config files returns `null`.
- Every new config key has an entry in `.env.example`.
- Feature switches and provider choices (payment gateway, SMS driver, AI model)
  are config, bound in a service provider, not `if` checks spread through the code.

---

## Verify

Run what applies and include the results in your summary.

```bash
grep -rnE 'try \{' app/Http/Controllers                    # controllers do not catch
grep -rnE '\$request->validate\(' app/Http/Controllers      # validation belongs in FormRequests
grep -rnE 'Request \$request' app/Services app/Domain 2>/dev/null   # services never take the Request
grep -rnE "response\(\)->json\(" app/Http/Controllers      # JSON goes through Resources
grep -rnE "['\"]status['\"] *=>" app --include=*.php        # status writes outside the lifecycle service
grep -rnE '\benv\(' app routes                            # env() outside config
grep -rnE 'Repository|Interface\b' app --include=*.php -l  # each one needs a real seam
```

Each grep hit is a candidate, not a verdict: read it and either fix it or state
why it is right here.

If the project uses Pest, encode the rules that matter as architecture tests so
they stay true:

```php
arch('controllers stay thin')
    ->expect('App\Http\Controllers')
    ->not->toUse(['Illuminate\Support\Facades\DB', 'Illuminate\Support\Facades\Mail']);

arch('services do not depend on HTTP')
    ->expect('App\Services')
    ->not->toUse('Illuminate\Http\Request');
```

## Report format

```
## Architecture review: <scope>

Must fix
- app/Http/Controllers/OrderController.php:57 cancel() sets status and sends the
  email inline. Fix: OrderService::cancel() transitions under lock and fires
  OrderCancelled; a queued listener sends the email after commit.

Should fix
- app/Repositories/OrderRepository.php wraps Eloquent with no second
  implementation. Fix: move the two queries to scopes on Order and remove the class.

Checked, nothing found: 7 (errors), 10 (config)
Not applicable: 9 (no API change)
```
