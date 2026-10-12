# Lesson 30 — Queues, Jobs, and Background Work

> **Phase 3: Architecture** | You'll be here: ~35 min

## The Senior's Question

A user clicks "Generate Annual Report." The report takes 3 minutes to compile. If you do this in the request, the user stares at a spinner for 3 minutes. If the request times out at 30 seconds, they get an error. What's the senior's answer?

---

## Theory: The Two Models

### Synchronous (block the request)

```
Request → Controller → Do work → Response (3 minutes later)
```

Bad. User waits. Connection can drop. Server can time out. Doesn't scale.

### Asynchronous (queue the work)

```
Request → Controller → "OK, we'll do that" → Response (instant)
                                       ↓
                                  Queue
                                       ↓
                            Worker (background)
                                       ↓
                                  Do work
                                       ↓
                              Notify user when done
```

Good. User gets instant response. Worker does the work. User is notified when complete.

---

## Theory: Queue Drivers

Laravel supports many queue backends:

| Driver | Use case | Setup |
|---|---|---|
| `sync` | No queue, runs in same process (default) | None |
| `database` | Simplest. Uses your DB as queue. | `php artisan queue:table` |
| `redis` | Fastest, recommended for production | Redis server |
| `sqs` | AWS Simple Queue Service | AWS account |
| `beanstalkd` | Old-school reliable queue | Self-hosted |
| `null` | Disable queueing (for testing) | None |

> **For dev:** `database` driver (zero infra). **For production:** `redis` (fast, scalable) or `sqs` (serverless, no ops).

Set in `.env`:

```
QUEUE_CONNECTION=database
```

Then:

```bash
php artisan queue:table
php artisan migrate
```

---

## Theory: Writing a Job

```bash
php artisan make:job GenerateAnnualReport
```

```php
<?php
namespace App\Jobs;

use App\Models\Tenant;
use App\Notifications\ReportReady;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class GenerateAnnualReport implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
    
    public int $tries = 3;
    public int $backoff = 60;  // seconds between retries
    public int $timeout = 600;  // 10 minutes
    
    public function __construct(public Tenant $tenant) {}
    
    public function handle(): void
    {
        // The work
        $report = $this->compileReport();
        $path = $this->savePdf($report);
        
        // Notify the admin
        $this->tenant->admin->notify(new ReportReady($path));
    }
    
    private function compileReport(): array
    {
        // Heavy computation
        return [
            'students_count' => $this->tenant->students()->count(),
            'attendance_avg' => $this->tenant->attendances()->avg('status'),
            // ... lots more
        ];
    }
    
    private function savePdf(array $data): string
    {
        $path = "reports/{$this->tenant->id}-" . now()->timestamp . ".pdf";
        \PDF::loadView('reports.annual', $data)->save(storage_path("app/$path"));
        return $path;
    }
    
    public function failed(\Throwable $e): void
    {
        // Called after all retries exhausted
        \Log::error("Report generation failed for tenant {$this->tenant->id}", [
            'error' => $e->getMessage(),
        ]);
    }
}
```

> **The `ShouldQueue` interface is the magic.** Laravel sees it and runs the job through the queue, not synchronously.

---

## Theory: Dispatching Jobs

### Fire and forget

```php
GenerateAnnualReport::dispatch($tenant);
```

The job goes on the queue. The current request continues immediately.

### Delayed

```php
GenerateAnnualReport::dispatch($tenant)->delay(now()->addMinutes(10));
```

### On a specific queue

```php
GenerateAnnualReport::dispatch($tenant)->onQueue('reports');
```

### With a specific connection

```php
GenerateAnnualReport::dispatch($tenant)->onConnection('redis');
```

### After a successful event (using listeners)

```php
class SendWelcomeEmail implements ShouldQueue
{
    public function handle(UserRegistered $event): void
    {
        Mail::to($event->user)->send(new WelcomeEmail($event->user));
    }
}
```

> **Listeners that implement `ShouldQueue` are jobs.** Same mechanics, different trigger.

---

## Theory: Running Workers

A worker is a long-running process that pulls jobs from the queue and runs them:

```bash
php artisan queue:work
```

In production, you run this as a service (systemd, supervisord, etc.):

```ini
# /etc/supervisor/conf.d/laravel-worker.conf
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/coaching-saas/artisan queue:work --sleep=3 --tries=3
autostart=true
autorestart=true
numprocs=4
redirect_stderr=true
stdout_logfile=/var/www/coaching-saas/storage/logs/worker.log
```

> **`numprocs=4` means 4 worker processes.** They share the queue. Adjust based on your server.

For Laravel Forge, this is set up automatically. For Vapor, it's serverless — no worker process needed.

---

## Theory: Job Failure Handling

Jobs can fail. Network down. Database unreachable. Disk full. Laravel's mechanism:

```php
public int $tries = 3;          // try up to 3 times
public int $backoff = 60;       // wait 60 seconds between tries
public int $maxExceptions = 2;  // give up after 2 unhandled exceptions

public function handle(): void
{
    // If this throws, Laravel retries
}

public function failed(\Throwable $e): void
{
    // Called when all retries exhausted
    // Send admin an alert, log, etc.
}
```

> **Senior rule:** every job should have a `failed()` method. Even if it's just logging. You'll thank yourself when something breaks at 3 AM.

### Exponential backoff

```php
public function backoff(): array
{
    return [60, 300, 900];  // 1 min, 5 min, 15 min
}
```

> **Better than fixed backoff.** First retry quickly, then slow down.

---

## Theory: Job Batching (Run Many Together)

```php
use Illuminate\Bus\Batch;
use Illuminate\Support\Facades\Bus;

Bus::batch([
    new CompileAttendance($tenant, 'January'),
    new CompileAttendance($tenant, 'February'),
    new CompileAttendance($tenant, 'March'),
    // ...
])->then(function (Batch $batch) {
    // All succeeded
})->catch(function (Batch $batch, \Throwable $e) {
    // First failure
})->finally(function (Batch $batch) {
    // Always runs
})->dispatch();
```

> **Use for:** end-of-month reports, bulk emails, batch imports. You get progress tracking, all-or-nothing handling, and notification on completion.

---

## Theory: Chaining Jobs

```php
Bus::chain([
    new PullDataFromApi($tenant),
    new TransformData($tenant),
    new SaveToDatabase($tenant),
    new SendNotification($tenant),
])->dispatch();
```

If any step fails, the rest don't run. **Senior's pattern for multi-step async workflows.**

---

## Theory: Rate Limiting a Queue

```php
public function handle(): void
{
    // ...
}

// Don't run more than 10 per minute
public function middleware(): array
{
    return [(new \Illuminate\Queue\Middleware\RateLimited('reports'))->dontRelease()];
}
```

> **Useful for:** respecting external API rate limits (Stripe allows 100 req/s, etc.).

---

## Theory: Common Job Patterns for Your SaaS

| Job | Queue | Priority |
|---|---|---|
| `SendWelcomeEmail` | notifications | high |
| `SendPaymentReceipt` | notifications | high |
| `GenerateReport` | reports | low |
| `CompileMonthlyBilling` | billing | normal |
| `SendAttendanceDigest` | notifications | normal |
| `ProcessBulkImport` | imports | low |
| `SendInvoiceReminder` | notifications | normal |
| `CleanupOldFiles` | maintenance | low |
| `PruneAuditLogs` | maintenance | low |

> **Queue prioritization** lets important work run first. Low-priority work waits.

---

## Why It Matters

Without queues:
- User waits 3 minutes for a report
- Email send fails the whole request
- Image upload blocks other users
- You have N+1s of background work slowing down everything

With queues:
- User gets instant response
- Background work happens in parallel
- Failures are retried automatically
- You can scale workers independently
- Your request servers stay fast

A senior's mantra: **"if it can take more than 1 second, queue it."**

---

## Lab: Build and Run a Real Queue

### Step 1 — Set up the database queue driver

Edit `.env`:

```
QUEUE_CONNECTION=database
```

```bash
php artisan queue:table
php artisan migrate
```

> **A new `jobs` table is created.** This is the queue.

### Step 2 — Generate a job

```bash
cd G:\php-laravel\coaching-saas
php artisan make:job SendWelcomeEmail
```

Edit `app/Jobs/SendWelcomeEmail.php`:

```php
<?php
namespace App\Jobs;

use App\Models\User;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Mail;
use Illuminate\Support\Facades\Log;

class SendWelcomeEmail implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
    
    public int $tries = 3;
    public int $backoff = 60;
    public int $timeout = 30;
    
    public function __construct(public User $user) {}
    
    public function handle(): void
    {
        Mail::raw(
            "Welcome to {$this->user->tenant->name}, {$this->user->name}!",
            function ($message) {
                $message->to($this->user->email)
                        ->subject('Welcome!');
            }
        );
        
        Log::info("Welcome email sent to {$this->user->email}");
    }
    
    public function failed(\Throwable $e): void
    {
        Log::error("Welcome email failed for {$this->user->email}", [
            'error' => $e->getMessage(),
        ]);
    }
}
```

### Step 3 — Update the User observer to dispatch the job

```bash
php artisan make:observer UserObserver --model=User
```

Edit `app/Observers/UserObserver.php`:

```php
<?php
namespace App\Observers;

use App\Jobs\SendWelcomeEmail;
use App\Models\User;

class UserObserver
{
    public function created(User $user): void
    {
        if ($user->hasVerifiedEmail()) {
            SendWelcomeEmail::dispatch($user);
        }
    }
}
```

Register in `app/Providers/AppServiceProvider.php`:

```php
public function boot(): void
{
    User::observe(UserObserver::class);
}
```

### Step 4 — Configure mail to use log driver (for testing)

In `.env`:

```
MAIL_MAILER=log
```

> **Mails get written to `storage/logs/laravel.log` instead of actually sending.** Perfect for development.

### Step 5 — Dispatch a job in tinker

```bash
php artisan tinker
```

```php
$user = \App\Models\User::first();
\App\Jobs\SendWelcomeEmail::dispatch($user);
```

> **The job goes on the queue. The dispatch returns immediately.**

### Step 6 — Check the queue

```bash
php artisan queue:size
```

> Returns the number of pending jobs. (Should be 1.)

### Step 7 — Run a worker

In another terminal:

```bash
php artisan queue:work --once
```

> **`--once` processes one job and exits.** Useful for testing.

Or run continuously:

```bash
php artisan queue:work
```

### Step 8 — Verify the email was sent

```bash
tail -f storage/logs/laravel.log
```

You should see the email content in the log.

### Step 9 — Build a "Generate Report" job

```bash
php artisan make:job GenerateTenantReport
```

```php
<?php
namespace App\Jobs;

use App\Models\Tenant;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Storage;

class GenerateTenantReport implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
    
    public int $tries = 2;
    public int $backoff = 30;
    public int $timeout = 120;
    
    public function __construct(public Tenant $tenant) {}
    
    public function handle(): void
    {
        Log::info("Generating report for tenant {$this->tenant->id}");
        
        // Simulate heavy work
        $data = [
            'tenant' => $this->tenant->name,
            'students_count' => $this->tenant->students()->count(),
            'classes_count' => $this->tenant->schoolClasses()->count(),
            'generated_at' => now()->toDateTimeString(),
        ];
        
        $path = "reports/{$this->tenant->slug}-" . now()->timestamp . ".json";
        Storage::disk('local')->put($path, json_encode($data, JSON_PRETTY_PRINT));
        
        Log::info("Report saved to $path");
    }
    
    public function failed(\Throwable $e): void
    {
        Log::error("Report generation failed for tenant {$this->tenant->id}", [
            'error' => $e->getMessage(),
        ]);
    }
}
```

### Step 10 — Dispatch from a controller

```php
public function generateReport(Tenant $tenant): RedirectResponse
{
    $this->authorize('manage-billing');
    
    GenerateTenantReport::dispatch($tenant);
    
    return back()->with('success', 'Report generation started. You will be notified when ready.');
}
```

> **The user gets instant feedback.** The actual work happens in the background.

### Step 11 — Schedule the worker

In production, you need a long-running worker. In dev, you can just keep `php artisan queue:work` running.

Add a "queue monitor" route to your dashboard (for admins):

```php
public function queueStatus(): View
{
    $pending = \DB::table('jobs')->count();
    $failed = \DB::table('failed_jobs')->count();
    
    return view('admin.queue', compact('pending', 'failed'));
}
```

### Step 12 — Set up a scheduler to retry failed jobs

Edit `routes/console.php`:

```php
use Illuminate\Support\Facades\Schedule;

// Retry failed jobs (e.g., after 24 hours)
Schedule::command('queue:retry all')
    ->hourly()
    ->withoutOverlapping();
```

> **You now have a production-ready queue system.** The senior's last lesson for the architecture phase.

---

## Quiz

**Q1.** Why is queueing work better than running it synchronously? Give two concrete benefits.

**Q2.** You have a job that sends a receipt email. It sometimes fails because the SMTP server is down. How do you make it robust? (Hint: `tries`, `backoff`, `failed()`.)

**Q3.** A junior writes `$pdf = PDF::loadView(...)->save(...);` in a controller. A senior says "queue it." Why?

**Q4.** When would you use `Bus::chain([...])` vs separate dispatches? Give a concrete example.

**Q5.** (Stretch) Read `Illuminate\Queue\Middleware\RateLimited`. How does it work, and when would you use it? What's the production use case?

---

## What You Now Have

✅ Queue drivers: database, Redis, SQS
✅ Jobs with retries, backoff, timeouts
✅ Queued listeners (events → background work)
✅ Worker processes (supervisor, systemd)
✅ Job batching and chaining
✅ Failure handling

## What's Next

**Phase 3 complete!** You now have a full Laravel SaaS architecture.

**Phase 4: Quality** begins with **testing** — the senior's discipline for shipping code that doesn't break. Pest, TDD, and the testing pyramid.

📖 **[Phase 4 starts: Lesson 31 — Testing Mental Model and the Testing Pyramid](phase-4-quality/31-testing-mental-model-and-the-testing-pyramid.md)**
