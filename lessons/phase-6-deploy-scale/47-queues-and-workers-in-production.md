# Lesson 47 — Queues and Workers in Production

> **Phase 6: Deploy & Scale** | You'll be here: ~30 min

## The Senior's Question

Your SaaS sends emails, processes payments, generates reports. All queued. In dev, `QUEUE_CONNECTION=sync` runs them immediately. **In prod, jobs sit in the queue forever, and customers don't get receipts.** The senior knows that queues are not "set and forget" — they need workers, monitoring, and scale plans.

---

## Theory: The Queue Pipeline

```
[Producer]                       [Broker]                  [Consumer]
                                                      ┌───→ worker 1
User pays → Job::dispatch() →  Redis (queue) ─────────┼───→ worker 2
                                                      └───→ worker 3
```

- **Producer:** your code. `dispatch()` puts a job on the queue.
- **Broker:** Redis (or database, SQS). Stores the job until consumed.
- **Consumer:** worker process. Pulls a job, runs `handle()`.

> **The broker is a buffer.** If consumers are slow, jobs wait. If consumers are fast, jobs are processed immediately.

---

## Theory: Queue Drivers

| Driver | Pros | Cons | Best for |
|---|---|---|---|
| `sync` | No broker needed | Blocks request | Dev only |
| `database` | No extra service | Slow, DB becomes bottleneck | Tiny apps |
| `redis` | Fast, supports priorities, batching | One more service | Most apps |
| `sqs` | AWS-managed, scales | AWS lock-in, latency | AWS deployments |
| `beanstalkd` | Old but reliable | Old, less popular | Legacy |

> **Senior default:** Redis for self-hosted, SQS for AWS.

---

## Theory: The Worker Process

A worker is a long-running PHP process:

```bash
php artisan queue:work
```

This process:
1. Polls the queue
2. Picks a job
3. Runs `handle()`
4. Deletes on success
5. Retries on failure (per job's `tries`)

> **A worker must be running.** If the worker process dies, no jobs are processed. **The senior uses Supervisor to keep workers alive.**

---

## Theory: Supervisor (Process Manager)

`/etc/supervisor/conf.d/laravel-worker.conf`:

```ini
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /home/forge/coaching-saas.com/artisan queue:work redis --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
user=forge
numprocs=2
redirect_stderr=true
stdout_logfile=/home/forge/coaching-saas.com/storage/logs/worker.log
stopwaitsecs=3600
```

> **Supervisor restarts the worker if it dies.** If the worker dies 5 times in a row, it stops. **No silent failures.**

Commands:

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start laravel-worker:*
sudo supervisorctl status laravel-worker:*
```

> **In Forge**, the queue manager is built in. You don't edit supervisor configs manually. But you should understand them.

---

## Theory: Job Configuration

In a job class:

```php
class SendReceiptEmail implements ShouldQueue
{
    use Dispatchable, Queueable;
    
    public $tries = 3;
    public $backoff = [60, 180, 600];  // 1 min, 3 min, 10 min
    public $timeout = 90;
    public $maxExceptions = 2;
    
    public function handle(): void
    {
        // ...
    }
    
    public function failed(\Throwable $e): void
    {
        // Called after all tries exhausted
        Log::error('receipt.email.failed', [
            'payment_id' => $this->payment->id,
            'error' => $e->getMessage(),
        ]);
    }
}
```

> **Each job decides its own retry policy.** A 1-second API call has 5 tries with 10s backoff. A 30-minute report has 1 try with 60-min backoff.

---

## Theory: Queue Priorities

```php
SendReceiptEmail::dispatch($payment)->onQueue('high');
GenerateReport::dispatch($tenant)->onQueue('low');
```

Configure workers per queue:

```bash
# High-priority worker (4 processes, queue: high)
command=php artisan queue:work redis --queue=high --tries=3

# Low-priority worker (1 process, queue: low)
command=php artisan queue:work redis --queue=low --tries=1 --timeout=600
```

> **High-priority jobs get workers first.** Receipts over reports.

---

## Theory: Laravel Horizon (Redis Only)

Horizon is Laravel's queue dashboard. It runs as a daemon with its own supervisor.

```bash
composer require laravel/horizon
php artisan horizon:install
php artisan migrate
```

Configure in `config/horizon.php`:

```php
'environments' => [
    'production' => [
        'supervisor-1' => [
            'connection' => 'redis',
            'queue' => ['default', 'high'],
            'balance' => 'auto',
            'minProcesses' => 1,
            'maxProcesses' => 10,
            'balanceMaxShift' => 1,
            'balanceCooldown' => 3,
        ],
        'supervisor-2' => [
            'connection' => 'redis',
            'queue' => ['low', 'reports'],
            'minProcesses' => 1,
            'maxProcesses' => 3,
        ],
    ],
],
```

> **`balance: auto`** scales workers up/down based on load. **Senior default.**

Run:

```bash
php artisan horizon
```

Visit `/horizon` for the dashboard:

- Throughput (jobs/min)
- Wait times
- Failure rates
- Recent jobs

> **Horizon's metrics tell you when you need more workers.** And when you have too many.

---

## Theory: Failed Jobs

When a job exhausts all retries, it goes to `failed_jobs` table.

```bash
# List failed
php artisan queue:failed

# Retry one
php artisan queue:retry 5

# Retry all
php artisan queue:retry all

# Forget one
php artisan queue:forget 5

# Flush all
php artisan queue:flush
```

> **Senior rule:** check failed jobs daily.** They tell you what's broken. A spike in failures = a deployment broke something. **Catch it before customers email you.**

In Sentry or Slack, alert on failed jobs:

```php
Schedule::call(function () {
    $count = DB::table('failed_jobs')->where('failed_at', '>', now()->subHour())->count();
    if ($count > 5) {
        Log::critical('failed_jobs.spike', ['count' => $count]);
    }
})->everyFiveMinutes();
```

---

## Theory: Job Idempotency

**The senior's #1 queue concern: jobs that run twice.**

Network blip → Stripe webhook fires → job dispatched → worker picks up → network blip before "ack" → Stripe retries → another job dispatched → two emails sent.

**Solution: idempotency keys.**

```php
class SendReceiptEmail implements ShouldQueue
{
    use Dispatchable, Queueable;
    
    public function __construct(public Payment $payment) {}
    
    public function uniqueId(): string
    {
        return "receipt-{$this->payment->id}";
    }
}
```

Add `ShouldBeUnique`:

```php
use Illuminate\Contracts\Queue\ShouldBeUnique;

class SendReceiptEmail implements ShouldQueue, ShouldBeUnique
{
    // ...
}
```

> **Unique jobs** are deduplicated. The second dispatch is ignored if the first is still queued/running.

> **Senior rule:** every payment-related job is `ShouldBeUnique`. Every email job. **Never risk double-sending.**

---

## Theory: Job Batching

For groups of related jobs:

```php
use Illuminate\Bus\Batch;
use Illuminate\Support\Facades\Bus;

Bus::batch([
    new SendReceiptEmail($payment1),
    new SendReceiptEmail($payment2),
    new SendReceiptEmail($payment3),
])->then(function (Batch $batch) {
    // All jobs done
    Log::info('batch.complete', ['id' => $batch->id]);
})->catch(function (Batch $batch, \Throwable $e) {
    // First failure
})->finally(function (Batch $batch) {
    // Always called
})->dispatch();
```

> **Batches** let you track "are all 1000 emails sent?" without 1000 callbacks.

---

## Theory: Job Chaining

```php
Bus::chain([
    new GenerateReport($tenant),
    new EmailReport($tenant),
    new NotifyAdmins($tenant),
])->dispatch();
```

> **Chain** runs jobs in order, stops on first failure. **Pipeline pattern.**

---

## Theory: Rate Limiting Jobs

If your API has rate limits, respect them in the job:

```php
use Illuminate\Queue\Middleware\RateLimited;

public function middleware(): array
{
    return [new RateLimited('sms-api')];
}
```

In `AppServiceProvider`:

```php
RateLimiter::for('sms-api', function ($job) {
    return Limit::perMinute(60);
});
```

> **Rate-limited jobs** wait and retry, never exceed the limit.

---

## Theory: Job Monitoring (The Senior's Setup)

### What to monitor

- **Throughput** (jobs/min) — drops = workers stuck
- **Wait time** (avg ms) — high = not enough workers
- **Failure rate** (%) — high = bugs or external service down
- **Oldest pending job** — should be < 1 minute
- **Failed jobs count** — should be 0

### Tools

- **Horizon** dashboard for Redis queues
- **Sentry** for job exceptions
- **Slack alerts** for spikes
- **Datadog/New Relic** for advanced APM

### Alerting

```php
// In a scheduled command, runs every 5 min
public function handle()
{
    $oldest = DB::table('jobs')->min('created_at');
    
    if ($oldest && now()->diffInMinutes($oldest) > 5) {
        Log::critical('queue.backed_up', [
            'oldest_job_age_min' => now()->diffInMinutes($oldest),
        ]);
    }
}
```

---

## Theory: Scaling Workers

### Vertical scale

More `numprocs` in Supervisor. More processes = more jobs at once.

### Horizontal scale

Run workers on separate servers. They're stateless. **You can run 100 workers on 10 servers.**

```bash
# On each worker server
php artisan queue:work redis --queue=default
```

> **Workers don't need the app code locally** — they just need to talk to Redis. **A worker server is tiny — 1GB RAM is enough.**

### Auto-scale

Cloud-based:

- **AWS:** ECS with auto-scaling on SQS queue depth
- **Forge:** manual; need to migrate to ECS/K8s
- **Vapor:** automatic
- **Kubernetes:** HPA based on custom metrics

---

## Theory: When to Migrate from Forge to K8s/Vapor

| Symptom | Action |
|---|---|
| Queue wait time > 1 min | Add workers (more processes) |
| Adding workers doesn't help | Multiple servers for workers |
| Single server can't handle app + workers | Separate worker servers |
| Workers waste money at night | Use SQS + Lambda (Vapor) |
| 100k+ jobs/day | K8s with auto-scaling |

> **Senior rule:** start simple. Add workers on the same box. Separate when needed. K8s when you have 10+ servers.**

---

## Theory: Common Queue Pitfalls

### 1. Queue connection in `.env` wrong

```env
QUEUE_CONNECTION=redis  # but Redis is not in production
```

Result: every job throws immediately. Check with `php artisan tinker`.

### 2. No worker running

The queue fills up forever. Customer payments process but no receipt emails.

```bash
# Check
php artisan queue:size  # should be 0
supervisorctl status laravel-worker
```

### 3. Worker stuck

```bash
php artisan queue:restart
# All workers finish current job then exit. Supervisor restarts them.
```

### 4. Job timeout too long

```php
public $timeout = 30;  // 30 seconds
```

If the job runs longer, worker kills it. Without `--timeout`, worker hangs.

### 5. Memory leak in long-running worker

```bash
php artisan queue:work --max-time=3600
```

After 1 hour, the worker restarts. **Prevents memory leaks.**

> **Senior rule:** workers restart every hour.** It's boring but reliable. **No "the worker is slow after 6 hours" bugs.**

### 6. Database connection from old worker

If a job is running, the worker holds a DB connection. If deploy happens and you don't `queue:restart`, the worker holds a connection to old code.

**Always run `queue:restart` in deploy script.**

---

## Theory: The Queue Health Check

Add to your health endpoint:

```php
public function queueHealth(): array
{
    $size = Queue::size();
    $failed = DB::table('failed_jobs')->where('failed_at', '>', now()->subDay())->count();
    $oldest = DB::table('jobs')->min('created_at');
    
    return [
        'queue_size' => $size,
        'size_status' => $size < 1000 ? 'ok' : 'warning',
        'failed_today' => $failed,
        'failed_status' => $failed < 10 ? 'ok' : 'warning',
        'oldest_job_min' => $oldest ? now()->diffInMinutes($oldest) : 0,
        'oldest_status' => !$oldest || now()->diffInMinutes($oldest) < 5 ? 'ok' : 'warning',
    ];
}
```

> **If any status is 'warning' for >5 min, page someone.**

---

## Why It Matters

Queues are the **infrastructure of trust.** They:
- Send the receipt after payment
- Email the password reset
- Generate the report the boss needs at 8 AM
- Notify the admin of failed payments

If they break:
- Silent failures
- Customer churns
- Trust is gone

> **A senior's queues are monitored, scaled, retried, and idempotent.** A junior's "the email didn't send" is a queue issue.

---

## Lab: Set Up Production Queues

### Step 1 — Switch to Redis

`.env`:

```env
QUEUE_CONNECTION=redis
```

### Step 2 — Install Horizon (Redis only)

```bash
composer require laravel/horizon
php artisan horizon:install
php artisan migrate
```

### Step 3 — Configure Horizon

`config/horizon.php`:

```php
'environments' => [
    'production' => [
        'supervisor-default' => [
            'connection' => 'redis',
            'queue' => ['default'],
            'balance' => 'auto',
            'minProcesses' => 1,
            'maxProcesses' => 10,
            'balanceMaxShift' => 1,
            'balanceCooldown' => 3,
            'tries' => 3,
        ],
        'supervisor-high' => [
            'connection' => 'redis',
            'queue' => ['high', 'emails'],
            'balance' => 'auto',
            'minProcesses' => 2,
            'maxProcesses' => 8,
            'tries' => 5,
        ],
        'supervisor-low' => [
            'connection' => 'redis',
            'queue' => ['low', 'reports'],
            'minProcesses' => 1,
            'maxProcesses' => 3,
            'tries' => 1,
            'timeout' => 600,
        ],
    ],
],
```

### Step 4 — Run Horizon in production

Add to Supervisor or Forge's daemon manager:

```bash
php artisan horizon
```

### Step 5 — Configure the dashboard

In `app/Providers/HorizonServiceProvider.php`:

```php
protected function gate(): void
{
    Gate::define('viewHorizon', function ($user = null) {
        // Only admins in production
        return in_array(optional($user)->email, [
            'admin@coaching-saas.com',
        ]);
    });
}
```

### Step 6 — Make jobs unique

For payment-related jobs:

```php
use Illuminate\Contracts\Queue\ShouldBeUnique;

class SendReceiptEmail implements ShouldQueue, ShouldBeUnique
{
    public function uniqueId(): string
    {
        return "receipt-{$this->payment->id}";
    }
    
    public $uniqueFor = 600;  // 10 minutes
}
```

### Step 7 — Add monitoring

`app/Console/Commands/QueueMonitor.php`:

```php
public function handle()
{
    $size = Queue::size();
    $oldest = DB::table('jobs')->min('created_at');
    $oldestMin = $oldest ? now()->diffInMinutes($oldest) : 0;
    $failedToday = DB::table('failed_jobs')->where('failed_at', '>', now()->subDay())->count();
    
    if ($size > 5000) {
        Log::warning('queue.size_high', ['size' => $size]);
    }
    
    if ($oldestMin > 5) {
        Log::critical('queue.backed_up', ['oldest_min' => $oldestMin]);
    }
    
    if ($failedToday > 20) {
        Log::critical('queue.failures_high', ['count' => $failedToday]);
    }
    
    $this->info("Queue size: $size, oldest: {$oldestMin}m, failed today: $failedToday");
}
```

Schedule:

```php
$schedule->command('queue:monitor')->everyFiveMinutes();
```

### Step 8 — Test the queue

```bash
php artisan tinker
>>> dispatch(new \App\Jobs\SendReceiptEmail(\App\Models\Payment::first()));
```

Watch the Horizon dashboard. The job should appear, run, and disappear.

### Step 9 — Test failure

Create a failing job:

```php
class FailingJob implements ShouldQueue
{
    public function handle(): void
    {
        throw new \Exception('This always fails');
    }
}
```

```php
dispatch(new FailingJob);
```

Watch the failed job count increase. After 3 tries, it goes to `failed_jobs`. Check with `php artisan queue:failed`.

### Step 10 — Add failed job alert to Slack

`app/Listeners/NotifyFailedJob.php`:

```php
public function handle(\Illuminate\Queue\Events\JobFailed $event): void
{
    Log::channel('slack')->critical('Queue job failed', [
        'job' => $event->job->resolveName(),
        'queue' => $event->job->getQueue(),
        'error' => $event->exception->getMessage(),
    ]);
}
```

Register in `EventServiceProvider`:

```php
Event::listen(
    \Illuminate\Queue\Events\JobFailed::class,
    \App\Listeners\NotifyFailedJob::class
);
```

### Step 11 — Test queue:restart

```bash
# In production
php artisan queue:restart
# All workers finish current job and exit. Supervisor restarts them.
```

> **Add this to your deploy script.** It must run on every deploy.

### Step 12 — Add a queue health check endpoint

`app/Http/Controllers/HealthController.php`:

```php
public function queue()
{
    $size = Queue::size();
    $oldest = DB::table('jobs')->min('created_at');
    $oldestMin = $oldest ? now()->diffInMinutes($oldest) : 0;
    
    $healthy = $size < 1000 && ($oldestMin < 5 || $oldestMin === 0);
    
    return response()->json([
        'healthy' => $healthy,
        'queue_size' => $size,
        'oldest_job_minutes' => $oldestMin,
    ], $healthy ? 200 : 503);
}
```

> **Uptime monitors ping this every 30 seconds.** If it returns 503 three times, you get paged.

---

## Quiz

**Q1.** A customer pays. They don't get a receipt. Walk through your debugging steps. What are the 5 most likely causes?

**Q2.** Why is `ShouldBeUnique` important for payment jobs? Give a concrete example of the bug it prevents.

**Q3.** You have 1000 jobs in the queue, 1 worker. Wait time is 5 minutes. You add 9 more workers. Wait time is now 30 seconds. Why isn't it 0.5 seconds?

**Q4.** A junior runs `php artisan queue:work` in the foreground and walks away. After 6 hours, jobs are slow. Why? What's the senior's fix?

**Q5.** (Stretch) You have a job that calls a third-party API. The API rate-limits at 60 requests/min. You have 10,000 jobs. How do you process them all without hitting the rate limit?

---

## What You Now Have

✅ Queue driver selection
✅ Worker setup with Supervisor
✅ Horizon for Redis queues
✅ Job configuration (tries, backoff, timeout)
✅ Idempotency with ShouldBeUnique
✅ Batching and chaining
✅ Monitoring and alerting
✅ Scaling patterns

## What's Next

Lesson 48 covers **database at scale** — when your single MySQL starts buckling. Read replicas, connection pooling, sharding, the patterns for surviving 10x growth.

📖 **[Lesson 48 — Database at Scale](48-database-at-scale.md)**
