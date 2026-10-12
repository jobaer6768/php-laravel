# Lesson 40 — Logging, Monitoring, and Debugging

> **Phase 5: Production** | You'll be here: ~30 min

## The Senior's Question

It's 2 AM. A customer says "the app is broken." You have no idea what they're seeing. The junior starts `dd()`-ing random places. The senior looks at the logs, finds the error, fixes it, and is back in bed. **Logging and monitoring are the difference between guessing and knowing.**

---

## Theory: The Three Observability Pillars

```
1. LOGS     — discrete events ("User X paid $Y at 14:23:01")
2. METRICS  — aggregated numbers ("99th percentile latency: 230ms")
3. TRACES   — request path through services ("This request touched 4 services in 1.2s")
```

> **Senior rule:** you need all three. Logs tell you what happened. Metrics tell you how often. Traces tell you where in the code.

---

## Theory: Laravel's Logging

Laravel uses Monolog under the hood. Channels in `config/logging.php`:

```php
'channels' => [
    'stack' => [
        'driver' => 'stack',
        'channels' => ['single', 'slack'],
    ],
    'single' => [
        'driver' => 'single',
        'path' => storage_path('logs/laravel.log'),
        'level' => 'debug',
    ],
    'daily' => [
        'driver' => 'daily',
        'path' => storage_path('logs/laravel.log'),
        'days' => 14,
        'level' => 'info',
    ],
    'slack' => [
        'driver' => 'slack',
        'url' => env('LOG_SLACK_WEBHOOK_URL'),
        'level' => 'critical',
    ],
    'sentry' => [
        'driver' => 'sentry',
        'level' => 'error',
    ],
],
```

> **`stack`** writes to multiple channels. Errors go to file AND Slack AND Sentry.

---

## Theory: Log Levels

```
debug   → Detailed flow info (variable values, query counts)
info    → User actions, system events (user logged in, payment received)
notice  → Unusual but not error (cache miss, retry)
warning → Recoverable issues (deprecated API call, slow query)
error   → Caught exceptions (payment failed)
critical→ Component down (DB unreachable, queue worker crashed)
alert   → Page someone (production broken)
emergency → System unusable
```

> **In production, default level is `info` or `warning`.** Debug is for dev.

In `.env`:

```env
LOG_CHANNEL=stack
LOG_LEVEL=info
LOG_STACK=single,slack
```

---

## Theory: Structured Logging (The Senior Way)

```php
// Bad — string with context mixed in
Log::info("User {$user->id} paid {$amount}");

// Good — structured, queryable
Log::info('payment.received', [
    'user_id' => $user->id,
    'amount' => $amount,
    'currency' => 'USD',
    'invoice_id' => $invoice->id,
    'tenant_id' => $user->tenant_id,
]);
```

> **Structured logs are searchable.** `level:error AND tenant_id:1` returns just one tenant's errors. Strings require regex.

Add a context to every log automatically:

```php
// In a middleware
Log::withContext([
    'tenant_id' => app('current_tenant')->id ?? null,
    'request_id' => request()->header('X-Request-Id'),
    'user_id' => auth()->id(),
]);
```

Now every log line includes these. **No need to repeat in each call.**

---

## Theory: Error Tracking with Sentry

**Sentry** is the senior's choice for error tracking.

```bash
composer require sentry/sentry-laravel
```

In `.env`:

```env
SENTRY_LARAVEL_DSN=https://xxx@sentry.io/123
SENTRY_TRACES_SAMPLE_RATE=0.1
```

That's it. Errors are captured automatically. Stack traces, breadcrumbs, user context.

> **Sentry groups similar errors.** 1000 of the same error? One issue. The senior looks at the frequency, not 1000 copies.

Add context:

```php
\Sentry\configureScope(function ($scope) {
    $scope->setUser([
        'id' => auth()->id(),
        'tenant_id' => app('current_tenant')->id ?? null,
    ]);
});
```

---

## Theory: Application Performance Monitoring (APM)

Sentry has Performance, but you can also use:

- **Laravel Telescope** (dev, free)
- **Laravel Pulse** (production-friendly)
- **Sentry Performance**
- **New Relic**
- **DataDog**
- **OpenTelemetry + Grafana**

Telescope in dev:

```bash
composer require laravel/telescope --dev
php artisan telescope:install
php artisan migrate
```

Visit `/telescope` to see:
- Requests (with timing)
- Queries (with N+1 detection)
- Jobs
- Events
- Cache hits/misses
- Mail
- Notifications

> **Senior rule:** Telescope in dev, paid APM in production. Don't run Telescope in prod — it adds overhead.

---

## Theory: Health Checks

Your app needs to tell monitoring tools "I'm alive."

`routes/web.php`:

```php
Route::get('/health', function () {
    return response()->json([
        'status' => 'ok',
        'database' => DB::connection()->getPdo() ? 'up' : 'down',
        'redis' => Redis::connection()->ping() ? 'up' : 'down',
        'queue' => Queue::size() < 1000 ? 'healthy' : 'backed_up',
        'app' => config('app.name'),
        'env' => app()->environment(),
    ]);
});
```

> **Health checks** are pinged every 30 seconds by uptime monitors (UptimeRobot, Pingdom, etc.). If 3 in a row fail, you get a page.

---

## Theory: Queued Job Monitoring

```php
public function monitor(): array
{
    return [
        'queue_size' => Queue::size(),
        'failed_jobs' => DB::table('failed_jobs')->count(),
        'oldest_pending' => DB::table('jobs')->min('created_at'),
    ];
}
```

Set alerts:
- Queue size > 10,000 → page someone
- Failed jobs > 50 → notify team
- Oldest pending > 1 hour → page someone

> **A backed-up queue is a slow SaaS.** Email arrives 3 hours after the payment. Reports never generate.

---

## Theory: The Debugging Toolkit

### `php artisan tinker`

```bash
php artisan tinker
>>> User::find(1)
>>> App\Models\Tenant::with('students')->find(1)
>>> Cache::flush()
```

### `php artisan route:list`

See all routes — useful when "the URL doesn't work."

### `php artisan queue:failed`

See failed jobs. Then:

```bash
php artisan queue:retry all
```

### `php artisan db:show`

Table sizes, row counts.

### `php artisan about`

App version, environment, drivers, cache stats. **The senior's first command on a new box.**

### `php artisan pail` (Laravel 11+)

Real-time log tailing in the terminal. Like `tail -f` for Laravel.

```bash
php artisan pail --filter=error
```

---

## Theory: The Senior's Debugging Flow

When something is broken:

```
1. Look at the user's report — what did they actually do?
2. Check error tracking (Sentry) — is there an error?
3. Check logs — is there context?
4. Reproduce locally
5. Add a log line at the suspect point
6. Deploy and re-test
7. Fix
8. Add a regression test
```

> **Senior rule:** "works on my machine" is not debugging. If you can't reproduce, add observability. Logs, traces, screenshots. **Make the bug tell you where it lives.**

---

## Theory: Logging Anti-Patterns

### Logging sensitive data

```php
// NEVER
Log::info('User logged in', ['password' => $request->password]);
Log::info('Payment processed', ['card' => $request->card_number]);
```

> **PCI, GDPR, HIPAA violations.** Sentry's default sanitizes some fields, but you should configure yours.

In `config/logging.php`:

```php
'single' => [
    'driver' => 'single',
    'path' => storage_path('logs/laravel.log'),
    'level' => 'debug',
    'processors' => [
        \App\Logging\RedactSensitiveData::class,
    ],
],
```

`app/Logging/RedactSensitiveData.php`:

```php
class RedactSensitiveData
{
    public function __invoke(LogRecord $record): LogRecord
    {
        $record->extra['password'] = '***REDACTED***';
        $record->extra['card_number'] = '***REDACTED***';
        return $record;
    }
}
```

### Logging too much

```php
// DON'T log every query
DB::listen(fn($q) => Log::debug($q->sql));
// = 10,000 log lines per minute
```

> **In prod, log at `info` or above.** Debug is for dev. Otherwise you're paying Sentry per event.

### Logging in tight loops

```php
foreach ($students as $student) {
    Log::info('Processing student', ['id' => $student->id]);
}
```

> **Aggregate.** Log "Processed 1000 students" at the end, not per iteration.

---

## Theory: Custom Log Channels

For multi-tenant debugging, write per-tenant logs:

```php
'tenant' => [
    'driver' => 'daily',
    'path' => storage_path('logs/tenants/' . app('current_tenant')?->id . '.log'),
    'level' => 'info',
],
```

Or send specific events to a separate log:

```php
Log::channel('payments')->info('Payment processed', $context);
```

---

## Theory: Tracing with Sentry

Enable Performance in Sentry:

```env
SENTRY_TRACES_SAMPLE_RATE=0.1
```

`0.1` = sample 10% of requests. **100% would be too expensive.** Adjust up for high-value flows (checkout, payment), down for high-volume ones (page views).

In Sentry → Performance, see:
- Slow endpoints
- Slow DB queries
- Slow external API calls
- The exact waterfall of a request

---

## Theory: Custom Monitors

Track business events, not just errors:

```php
class BusinessMetrics
{
    public static function record(string $event, array $data = []): void
    {
        Log::info("metric.{$event}", $data);
        
        // Or send to a metrics service
        \Sentry\captureMessage("metric.{$event}", [
            'extra' => $data,
        ]);
    }
}

BusinessMetrics::record('student.enrolled', [
    'tenant_id' => $tenant->id,
    'plan' => $tenant->plan,
]);
```

Now you can answer:
- How many students enrolled today?
- Which plan converts best?
- Are signups increasing after the new marketing push?

> **Business metrics** are the senior's "is the company growing?" dashboard.

---

## Why It Matters

Without logging and monitoring:
- "It's broken" → "What broke?" → "I don't know" → "I can't fix it"
- Customer churns
- You're blind to the actual usage
- You find out about outages from customers

With it:
- Errors caught before users notice
- Slow queries visible
- Business trends visible
- Postmortems are quick and accurate

> **A senior's production app is observable.** You always know what's happening, why, and how often.

---

## Lab: Set Up Sentry + Telescope

### Step 1 — Install Sentry

```bash
composer require sentry/sentry-laravel
```

In `.env`:

```env
SENTRY_LARAVEL_DSN=https://xxx@sentry.io/123
SENTRY_TRACES_SAMPLE_RATE=0.1
```

Test:

```bash
php artisan tinker
>>> throw new \Exception('Test Sentry');
```

Check Sentry dashboard — exception should appear.

### Step 2 — Configure Sentry context

`app/Providers/AppServiceProvider.php`:

```php
public function boot(): void
{
    if (app()->bound('current_tenant')) {
        \Sentry\configureScope(function ($scope) {
            $scope->setTag('tenant_id', app('current_tenant')->id);
            if ($user = auth()->user()) {
                $scope->setUser([
                    'id' => $user->id,
                    'email' => $user->email,
                    'tenant_id' => $user->tenant_id,
                ]);
            }
        });
    }
}
```

### Step 3 — Install Telescope (dev only)

```bash
composer require laravel/telescope --dev
php artisan telescope:install
php artisan migrate
```

Visit `/telescope`. See requests, queries, jobs, etc.

### Step 4 — Add a health check route

`routes/web.php`:

```php
Route::get('/health', function () {
    $checks = [
        'app' => 'ok',
        'database' => 'unknown',
        'cache' => 'unknown',
        'queue' => 'unknown',
    ];
    
    try {
        DB::connection()->getPdo();
        $checks['database'] = 'ok';
    } catch (\Exception $e) {
        $checks['database'] = 'down';
    }
    
    try {
        Cache::put('health', 'ok', 1);
        $checks['cache'] = Cache::get('health') === 'ok' ? 'ok' : 'down';
    } catch (\Exception $e) {
        $checks['cache'] = 'down';
    }
    
    try {
        $size = Queue::size();
        $checks['queue'] = $size < 10000 ? 'ok' : 'backed_up';
        $checks['queue_size'] = $size;
    } catch (\Exception $e) {
        $checks['queue'] = 'down';
    }
    
    $allOk = !in_array('down', array_column($checks, null), true) 
        && !in_array('down', $checks, true);
    
    return response()->json($checks, $allOk ? 200 : 503);
});
```

### Step 5 — Add structured logging

`app/Actions/RecordPayment.php`:

```php
public function execute(FeeInvoice $invoice, float $amount, string $method): Payment
{
    Log::info('payment.recording', [
        'invoice_id' => $invoice->id,
        'tenant_id' => $invoice->tenant_id,
        'amount' => $amount,
        'method' => $method,
    ]);
    
    try {
        $payment = DB::transaction(function () use ($invoice, $amount, $method) {
            // ... existing logic
        });
        
        Log::info('payment.recorded', [
            'payment_id' => $payment->id,
            'invoice_id' => $invoice->id,
            'tenant_id' => $invoice->tenant_id,
            'amount' => $amount,
        ]);
        
        return $payment;
    } catch (\Exception $e) {
        Log::error('payment.failed', [
            'invoice_id' => $invoice->id,
            'tenant_id' => $invoice->tenant_id,
            'amount' => $amount,
            'error' => $e->getMessage(),
        ]);
        
        throw $e;
    }
}
```

### Step 6 — Add a request context middleware

```bash
php artisan make:middleware AddRequestContext
```

```php
public function handle($request, Closure $next)
{
    $requestId = $request->header('X-Request-Id') ?? Str::uuid();
    
    Log::withContext([
        'request_id' => $requestId,
        'tenant_id' => app('current_tenant')?->id,
        'user_id' => auth()->id(),
    ]);
    
    $response = $next($request);
    $response->headers->set('X-Request-Id', $requestId);
    
    return $response;
}
```

Add to global middleware in `bootstrap/app.php`:

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->append(\App\Http\Middleware\AddRequestContext::class);
})
```

### Step 7 — Configure Slack alerts for critical

`config/logging.php`:

```php
'slack' => [
    'driver' => 'slack',
    'url' => env('LOG_SLACK_WEBHOOK_URL'),
    'level' => 'critical',
    'username' => 'Laravel Logger',
    'emoji' => ':boom:',
],
```

`.env`:

```env
LOG_SLACK_WEBHOOK_URL=https://hooks.slack.com/services/xxx
LOG_STACK=single,slack
```

Test:

```php
Log::critical('Test alert', ['context' => 'manual test']);
```

### Step 8 — Add a failed job monitor command

`app/Console/Commands/MonitorFailedJobs.php`:

```php
public function handle()
{
    $count = DB::table('failed_jobs')
        ->where('failed_at', '>', now()->subHour())
        ->count();
    
    if ($count > 10) {
        Log::critical('failed_jobs.spike', [
            'count' => $count,
            'period' => '1 hour',
        ]);
        
        $this->error("Found {$count} failed jobs in the last hour");
    }
}
```

Schedule in `app/Console/Kernel.php`:

```php
$schedule->command('monitor:failed-jobs')->everyFiveMinutes();
```

### Step 9 — Add daily log cleanup

```php
$schedule->command('log:clear')->daily();
```

> **Log files pile up.** Daily rotation + weekly cleanup is a senior's discipline.

### Step 10 — Create a status page

`routes/web.php`:

```php
Route::get('/status', function () {
    return view('status', [
        'services' => [
            ['name' => 'App', 'status' => 'operational'],
            ['name' => 'Database', 'status' => 'operational'],
            ['name' => 'Email', 'status' => 'operational'],
            ['name' => 'Payments', 'status' => 'operational'],
        ],
    ]);
});
```

Or use **Instatus** / **Statuspage** (free for hobby).

---

## Quiz

**Q1.** Why use structured logging (`Log::info('event', ['key' => $value])`) instead of `Log::info("Event happened for user $id")`?

**Q2.** A junior adds `DB::listen()` to log every query. Why is this a problem in production?

**Q3.** You receive a Sentry alert: "TypeError in InvoiceController." Walk through your debugging steps.

**Q4.** Your health check returns 200 OK but the database is broken. What did you forget?

**Q5.** (Stretch) You have a slow endpoint taking 1.5 seconds. How do you find out where the time is going? Walk through using Telescope / Sentry Performance.

---

## What You Now Have

✅ Structured logging with context
✅ Sentry for error tracking
✅ Telescope for dev
✅ Health check endpoint
✅ Slack alerts for critical
✅ Business metrics tracking

## What's Next

Lesson 41 covers **search with Scout** — fast, relevant search across students, classes, payments. From `LIKE` queries to full-text search with Meilisearch or Algolia.

📖 **[Lesson 41 — Search with Scout](41-search-with-scout.md)**
