# Lesson 36 — Caching with Redis and Cache Invalidation

> **Phase 5: Production** | You'll be here: ~35 min

## The Senior's Question

Your SaaS works on your laptop. Then 5,000 coaching centers sign up. The dashboard takes 8 seconds to load. The junior adds more indexes. The senior adds a cache. But **caching is the source of half the bugs in production** — stale data, forgotten invalidation, "works on my machine" surprises. Let me show you how a senior thinks about caching.

---

## Theory: What Caching Solves

```
Without cache:
   User → App → Database → 200ms

With cache:
   User → App → Redis → 5ms (cache hit)
                 → Database → 200ms (cache miss)
```

> **Cache = trade memory for speed.** You store a computed result so you don't have to compute it again. The cost: keeping the cache in sync with the source of truth.

---

## Theory: The Two Hard Things in CS

There are two hard things in computer science:

1. Cache invalidation
2. Naming things
3. Off-by-one errors

**Cache invalidation** is hard because:
- When do you refresh the cache? (on every change? on a timer?)
- What if the change happens outside your code? (DB trigger? admin SQL?)
- What if you have multiple servers?
- What if the cache key depends on a relationship?

> **Senior rule:** don't cache what you don't have to. The simplest cache is no cache. The second simplest is a short TTL. **Get fancy only when measurements demand it.**

---

## Theory: Cache Drivers in Laravel

Laravel supports multiple cache backends:

| Driver | Speed | Use case |
|---|---|---|
| `array` | Fastest | Test only, dies between requests |
| `file` | Slow | Single-server dev |
| `database` | Slow | Avoid — DB caching defeats the purpose |
| `redis` | Fast | **Production default** |
| `memcached` | Fast | Alternative to Redis |
| `dynamodb` | Medium | AWS serverless |

Configure in `.env`:

```env
CACHE_DRIVER=redis
```

In `config/cache.php`:

```php
'default' => env('CACHE_DRIVER', 'redis'),
```

> **Senior choice:** Redis. It's fast, supports tags, supports atomic operations, has a great Laravel integration.

---

## Theory: Installing Redis

### Windows (with Laravel Herd or Docker)

```bash
docker run -d --name redis -p 6379:6379 redis:7-alpine
```

### macOS

```bash
brew install redis
brew services start redis
```

### Ubuntu

```bash
sudo apt install redis-server
sudo systemctl enable redis-server
```

### Predis (pure PHP, no extension)

```bash
composer require predis/predis
```

### PhpRedis (C extension, faster)

```bash
# Install extension
pecl install redis
# Add to php.ini:
extension=redis.so
```

```bash
composer require phpredis
```

> **Predis is simpler to install.** PhpRedis is faster. For most apps, the difference is negligible.

Configure in `config/database.php`:

```php
'redis' => [
    'client' => env('REDIS_CLIENT', 'predis'),
    'default' => [
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'password' => env('REDIS_PASSWORD'),
        'port' => env('REDIS_PORT', '6379'),
        'database' => env('REDIS_DB', '0'),
    ],
    'cache' => [
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'password' => env('REDIS_PASSWORD'),
        'port' => env('REDIS_PORT', '6379'),
        'database' => env('REDIS_CACHE_DB', '1'),
    ],
]
```

> **Separate Redis DB for cache and sessions.** When you flush one, you don't kill the other.

---

## Theory: The Cache Facade — Basic Operations

```php
use Illuminate\Support\Facades\Cache;

// Get
$value = Cache::get('key');
$value = Cache::get('key', 'default');

// Check existence
if (Cache::has('key')) { ... }

// Store
Cache::put('key', 'value', 3600);  // 1 hour
Cache::put('key', 'value', now()->addMinutes(10));

// Store forever (until manually deleted)
Cache::forever('key', 'value');

// Delete
Cache::forget('key');
Cache::flush();  // wipe ALL cache — dangerous

// Atomic increment/decrement
Cache::increment('counter');
Cache::decrement('counter');
```

> **`Cache::flush()` is a nuclear option.** It clears the entire cache. Never call it in production code paths. Only in console commands.

---

## Theory: `remember()` — The Senior's Best Friend

`remember()` is the most common cache pattern. It tries to get; on miss, it computes and stores.

```php
$students = Cache::remember('students:all', 3600, function () {
    return Student::with('class')->get();
});
```

The flow:

```
Cache::remember('key', 3600, fn):
  1. Get('key') → exists? Return it.
  2. Run the closure.
  3. Store the result with TTL 3600s.
  4. Return the result.
```

> **Senior rule:** `Cache::remember` is the default. Use it instead of manual `get` + `put`.

Variants:

```php
// Forever (no expiry)
Cache::rememberForever('key', fn() => ...);

// With tags (group invalidation)
Cache::tags(['students', 'tenant:1'])->remember('students:all', 3600, fn() => ...);

// Get-or-set with default
$value = Cache::get('key', fn() => expensive());
```

---

## Theory: Cache Keys — The Naming Discipline

A senior names cache keys hierarchically:

```
{entity}:{scope}:{identifier}

Examples:
students:list:tenant:1
students:list:tenant:1:class:5
student:1:detail
student:1:fees
dashboard:tenant:1
attendance:today:tenant:1
```

> **Hierarchical keys** make invalidation easier. If a student changes, you can `forget("student:{$id}:*")` — but Redis doesn't support wildcards natively (more on that below).

```php
// Helper for consistent keys
class CacheKey
{
    public static function student(int $id): string
    {
        return "student:{$id}";
    }
    
    public static function studentList(int $tenantId, ?int $classId = null): string
    {
        $key = "students:list:tenant:{$tenantId}";
        if ($classId) {
            $key .= ":class:{$classId}";
        }
        return $key;
    }
}
```

---

## Theory: Cache Invalidation — The Hard Part

There are three strategies:

### Strategy 1: Time-based (TTL)

```php
Cache::remember('dashboard:stats', 300, fn() => $this->computeStats());
```

- Refreshes every 5 minutes
- Stale up to 5 minutes
- **Simple. The senior's default.**

### Strategy 2: Event-based (write-through)

```php
// On student create/update
event(new StudentUpdated($student));

// Listener clears cache
class ClearStudentCache
{
    public function handle(StudentUpdated $event): void
    {
        Cache::forget("student:{$event->student->id}");
        Cache::forget("students:list:tenant:{$event->student->tenant_id}");
    }
}
```

- Always fresh
- Risk: forget to clear after some change
- **The most reliable when done right.**

### Strategy 3: Tag-based (Redis only)

```php
// Store with tag
Cache::tags(['students', "tenant:{$tenant->id}"])
    ->put("student:{$id}", $student, 3600);

// Clear all keys with tag
Cache::tags(['tenant:1'])->flush();
```

- Group invalidation
- **Most powerful, requires Redis.**

> **Senior rule:** use TTL as the safety net, event-based for instant invalidation, tags for group operations.

---

## Theory: Cache Tags — When to Use

Cache tags work only with Redis and Memcached:

```php
// Store
Cache::tags(['students', "tenant:{$tenantId}"])
    ->remember("student:{$id}", 3600, fn() => Student::find($id));

// Invalidate everything tagged 'students'
Cache::tags(['students'])->flush();
```

> **Tags add overhead.** Each tagged key takes more memory. Use them only when you need group invalidation.

Common patterns:
- `tenant:{id}` — all data for a tenant
- `student:{id}` — all data for a student
- `report:{type}` — all data of a report type

---

## Theory: The N+1 Problem in Caching

If you cache a list, the closure might re-trigger the N+1 problem when it expires:

```php
// BAD: Without eager loading, this fires 1 + N queries when cache misses
$students = Cache::remember('students:all', 3600, function () {
    return Student::all();  // ← N+1 when accessed
});
```

```php
// GOOD: Eager load inside the closure
$students = Cache::remember('students:all', 3600, function () {
    return Student::with(['class', 'guardian'])->get();
});
```

> **Inside a cache closure, always eager load.** The next access won't re-run the query.

---

## Theory: Cache Stampede — The Hidden Killer

Imagine 1000 users hit an expired cache key at the same instant. All 1000 run the closure. **Database dies.**

Solutions:

### Solution 1: Atomic locks

```php
$students = Cache::lock('students:all', 10)->block(5, function () {
    return Cache::remember('students:all', 3600, fn() => Student::all());
});
```

> **Only one process computes; others wait.** Standard pattern.

### Solution 2: Stale-while-revalidate

```php
$students = Cache::remember('students:all', 3600, function () {
    // Always serve stale for 1 hour, refresh in background
    return Cache::flexible('students:all', [300, 3600], fn() => Student::all());
});
```

> **Laravel 11+ has `flexible()`.** New value computed after 5 min, but old served for 1 hour.

### Solution 3: Probabilistic early expiration

```php
// At 80% of TTL, recompute
if (random_int(1, 100) <= 20) {
    Cache::put('students:all', Student::all(), 3600);
}
```

> **Spreads the recompute load.** Each request has a 20% chance of refreshing.

---

## Theory: What NOT to Cache

- **Auth checks** — they must be live (or use very short TTL)
- **Money in flight** — payments should not be cached
- **CSRF tokens** — they must rotate
- **Anything user-specific that changes constantly**
- **Large objects** (over 1MB) — Redis is fast, but big objects kill bandwidth

> **Senior rule:** cache computed results, not raw data. Cache the dashboard view, not the underlying rows.

---

## Theory: Cache + Multi-Tenancy

Every cache key MUST include the tenant. Otherwise, tenant A gets tenant B's data.

```php
// BAD — tenant leaks!
Cache::remember('students:all', 3600, fn() => Student::all());

// GOOD — scoped to tenant
Cache::remember("students:tenant:{$tenant->id}", 3600, function () use ($tenant) {
    return $tenant->students()->with('class')->get();
});

// EVEN BETTER — let the trait handle it
trait Cacheable
{
    public function getCacheKey(string $suffix = 'list'): string
    {
        $tenantId = app('current_tenant')->id ?? 'global';
        return $this->getTable() . ":tenant:{$tenantId}:{$suffix}";
    }
}
```

> **The single biggest multi-tenant bug: cross-tenant cache pollution.** Always include `tenant_id` in the key.

---

## Theory: Cache Debugging

```bash
# See all keys
php artisan tinker
>>> Cache::getRedis()->keys('*')

# See value
>>> Cache::get('students:tenant:1')

# Count keys
>>> count(Cache::getRedis()->keys('students:*'))

# Clear specific key
>>> Cache::forget('students:tenant:1')

# Clear all (CAREFUL)
>>> Cache::flush()
```

Or use a tool like **Laravel Telescope** to see cache hits/misses in dev.

---

## Why It Matters

Caching is the difference between:
- 200ms page load vs 5ms page load
- 100 users vs 10,000 users on the same hardware
- "It's slow" vs "It's instant"

But it's also:
- The source of "stale data" bugs
- The reason "it worked in dev, broken in prod"
- The cause of "the dashboard is wrong" complaints

> **A senior caches aggressively where it matters, conservatively where it doesn't, and always has an invalidation strategy.**

---

## Lab: Cache the Coaching Dashboard

### Step 1 — Install Redis

```bash
docker run -d --name redis -p 6379:6379 redis:7-alpine
```

```bash
composer require predis/predis
```

Update `.env`:

```env
CACHE_DRIVER=redis
REDIS_CLIENT=predis
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
REDIS_CACHE_DB=1
```

Test it:

```bash
php artisan tinker
>>> Cache::put('test', 'hello', 60);
=> "hello"
>>> Cache::get('test')
=> "hello"
```

### Step 2 — Add a cache key helper

`app/Support/CacheKey.php`:

```php
<?php

namespace App\Support;

class CacheKey
{
    public static function dashboard(int $tenantId): string
    {
        return "dashboard:tenant:{$tenantId}";
    }
    
    public static function studentList(int $tenantId): string
    {
        return "students:list:tenant:{$tenantId}";
    }
    
    public static function studentDetail(int $id): string
    {
        return "student:{$id}:detail";
    }
    
    public static function studentFees(int $id): string
    {
        return "student:{$id}:fees";
    }
}
```

### Step 3 — Cache the dashboard controller

`app/Http/Controllers/DashboardController.php`:

```php
<?php

namespace App\Http\Controllers;

use App\Models\Attendance;
use App\Models\Student;
use App\Support\CacheKey;
use Illuminate\Support\Facades\Cache;

class DashboardController extends Controller
{
    public function index()
    {
        $tenant = app('current_tenant');
        
        $stats = Cache::remember(
            CacheKey::dashboard($tenant->id),
            now()->addMinutes(5),
            function () use ($tenant) {
                return [
                    'total_students' => $tenant->students()->count(),
                    'active_students' => $tenant->students()->where('is_active', true)->count(),
                    'today_attendance' => Attendance::whereDate('date', today())
                        ->whereHas('student', fn($q) => $q->where('tenant_id', $tenant->id))
                        ->count(),
                    'pending_fees' => $tenant->students()
                        ->withSum('feeInvoices', 'due')
                        ->get()
                        ->sum('fee_invoices_sum_due'),
                ];
            }
        );
        
        return view('dashboard.index', compact('stats'));
    }
}
```

### Step 4 — Invalidate on changes

Create an observer:

```bash
php artisan make:observer StudentObserver --model=Student
```

`app/Observers/StudentObserver.php`:

```php
<?php

namespace App\Observers;

use App\Models\Student;
use App\Support\CacheKey;
use Illuminate\Support\Facades\Cache;

class StudentObserver
{
    public function created(Student $student): void
    {
        $this->invalidate($student);
    }
    
    public function updated(Student $student): void
    {
        $this->invalidate($student);
    }
    
    public function deleted(Student $student): void
    {
        $this->invalidate($student);
    }
    
    private function invalidate(Student $student): void
    {
        Cache::forget(CacheKey::studentDetail($student->id));
        Cache::forget(CacheKey::studentList($student->tenant_id));
        Cache::forget(CacheKey::dashboard($student->tenant_id));
        Cache::forget(CacheKey::studentFees($student->id));
    }
}
```

Register in `AppServiceProvider`:

```php
use App\Models\Student;
use App\Observers\StudentObserver;

public function boot(): void
{
    Student::observe(StudentObserver::class);
}
```

### Step 5 — Cache the student list

`app/Http/Controllers/StudentController.php`:

```php
public function index()
{
    $tenant = app('current_tenant');
    
    $students = Cache::remember(
        CacheKey::studentList($tenant->id),
        now()->addMinutes(10),
        function () use ($tenant) {
            return $tenant->students()
                ->with(['class', 'guardian'])
                ->orderBy('name')
                ->get();
        }
    );
    
    return view('students.index', compact('students'));
}
```

### Step 6 — Add a lock for safety

Protect against stampede on the dashboard:

```php
$stats = Cache::lock(CacheKey::dashboard($tenant->id) . ':lock', 10)
    ->block(5, function () use ($tenant) {
        return Cache::remember(
            CacheKey::dashboard($tenant->id),
            now()->addMinutes(5),
            fn() => $this->computeStats($tenant)
        );
    });
```

### Step 7 — Add a `php artisan cache:clear` shortcut

```bash
php artisan cache:clear
php artisan cache:clear --tags=tenant:1
```

### Step 8 — Test it

```bash
# First request — slow (cache miss)
php artisan tinker
>>> measure(fn() => app(DashboardController::class)->index());
# ~150ms

# Second request — fast (cache hit)
>>> measure(fn() => app(DashboardController::class)->index());
# ~5ms
```

### Step 9 — Verify invalidation

```bash
php artisan tinker
>>> Cache::put('test', 'value', 60);
>>> Student::factory()->count(1)->create();
>>> Cache::has('test');  // should still be true
```

### Step 10 — Monitor in production

Install Telescope in dev:

```bash
composer require laravel/telescope --dev
php artisan telescope:install
php artisan migrate
```

Visit `/telescope` to see cache hits, misses, and slow queries.

---

## Quiz

**Q1.** You cache a list of students with `Cache::remember('students:all', 3600, fn() => Student::all())`. What's the multi-tenant bug? How do you fix it?

**Q2.** A user reports "the dashboard shows old data even after I added a new student." You have TTL of 5 minutes. What's the most likely cause? How do you fix it?

**Q3.** `Cache::flush()` is dangerous. When is it acceptable to use?

**Q4.** A senior uses `Cache::remember()` instead of `get()` + `put()`. Why? What's the bug-prone alternative?

**Q5.** (Stretch) You have 1000 concurrent users hitting an expired cache key. Walk through what happens without a lock, then with a lock.

---

## What You Now Have

✅ Cache fluency: get, put, remember, forget
✅ Cache key naming discipline
✅ Three invalidation strategies (TTL, event, tags)
✅ Multi-tenant safe caching
✅ Stampede protection with locks
✅ Observers for invalidation

## What's Next

Lesson 37 covers **subscription billing with Laravel Cashier** — making tenants pay you. Stripe integration, plans, trials, webhooks, the recurring revenue engine that turns a SaaS into a business.

📖 **[Lesson 37 — Subscription Billing with Laravel Cashier](37-subscription-billing-with-laravel-cashier.md)**
