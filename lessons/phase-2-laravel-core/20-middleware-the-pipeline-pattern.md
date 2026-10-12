# Lesson 20 — Middleware: The Pipeline Pattern

> **Phase 2: Laravel Core** | You'll be here: ~30 min

## The Senior's Question

Every request passes through a series of "filters" before hitting your controller. These filters are called middleware, and they're the **right place to put cross-cutting concerns** — auth, logging, tenant scoping, rate limiting, request signing. A senior reaches for middleware before reaching for anything else.

---

## Theory: The Onion Model

```
       Request
          ↓
   ┌──────────────┐
   │  Middleware 1 │ ← request phase
   ├──────────────┤
   │  Middleware 2 │
   ├──────────────┤
   │  Middleware 3 │
   ├──────────────┤
   │  Controller  │
   ├──────────────┤
   │  Middleware 3 │ ← response phase
   ├──────────────┤
   │  Middleware 2 │
   ├──────────────┤
   │  Middleware 1 │
   └──────────────┘
          ↓
       Response
```

Each middleware can:
1. Run code **before** the next middleware (request phase)
2. Call `$next($request)` to pass the request down
3. Run code **after** the next middleware (response phase)
4. Return early without calling `$next` (short-circuit, e.g. 401 Unauthorized)

---

## Theory: Writing Middleware

```bash
php artisan make:middleware LogRequest
```

```php
<?php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Symfony\Component\HttpFoundation\Response;

class LogRequest
{
    public function handle(Request $request, Closure $next): Response
    {
        // Request phase: BEFORE controller
        Log::info('Request started', [
            'method' => $request->method(),
            'url'    => $request->url(),
        ]);
        
        $response = $next($request);  // call the next middleware / controller
        
        // Response phase: AFTER controller
        Log::info('Response sent', [
            'status' => $response->getStatusCode(),
        ]);
        
        return $response;
    }
}
```

> **The key line is `$next($request)`.** Everything before it is the request phase. Everything after is the response phase. **If you don't call `$next`, the chain stops.**

---

## Theory: Short-Circuiting (Reject the Request)

```php
public function handle(Request $request, Closure $next): Response
{
    if (!$this->isApiKeyValid($request)) {
        return response()->json(['error' => 'Unauthorized'], 401);
    }
    
    return $next($request);
}
```

> **When you return without calling `$next`, the request is rejected.** The controller never runs.

---

## Theory: Middleware Parameters

```php
public function handle(Request $request, Closure $next, string $role): Response
{
    if (!$request->user()?->hasRole($role)) {
        abort(403, "Requires role: $role");
    }
    
    return $next($request);
}
```

Use it:

```php
Route::middleware(['auth', 'role:admin'])->group(function () {
    // ...
});
```

> **Parameters come after the middleware name, separated by colons.**

---

## Theory: Where to Register Middleware

In `bootstrap/app.php` (Laravel 11+):

```php
->withMiddleware(function (Middleware $middleware) {
    // 1. Global middleware (runs on every request)
    $middleware->append(LogRequest::class);
    $middleware->prepend(TrustProxies::class);
    
    // 2. Group middleware (alias for a set)
    $middleware->appendToGroup('tenant', [
        \App\Http\Middleware\ResolveTenant::class,
        \App\Http\Middleware\EnsureTenantActive::class,
    ]);
    
    // 3. Aliases (shorthand for routes)
    $middleware->alias([
        'role' => \App\Http\Middleware\EnsureRole::class,
    ]);
})
```

> **In Laravel 10 and earlier, this was in `app/Http/Kernel.php`.** Laravel 11 collapsed it into `bootstrap/app.php` for simplicity.

---

## Theory: Three Types of Middleware

### 1. Global middleware

Runs on every HTTP request. Use for things like:
- Trust proxies
- Trim strings
- Convert empty strings to null
- CORS

These are configured in `bootstrap/app.php` or the framework's defaults.

### 2. Route middleware (named aliases)

```php
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('/dashboard', ...);
});
```

These are aliases. The actual middleware classes are registered in `bootstrap/app.php`.

### 3. Route group middleware (with parameters)

```php
Route::middleware(['auth', 'role:admin'])->prefix('admin')->group(function () {
    // ...
});
```

You can also assign middleware to a controller via the constructor:

```php
class StudentController extends Controller
{
    public function __construct()
    {
        $this->middleware(['auth', 'tenant']);
        $this->middleware('can:view,student')->only('show');
    }
}
```

> **Senior preference:** put middleware in routes, not in controllers. Routes are the URL → handler map; middleware is part of the URL contract.

---

## Theory: Real-World Middleware Examples

### Tenant resolution

```php
class ResolveTenant
{
    public function handle(Request $request, Closure $next): Response
    {
        $host = $request->getHost();              // e.g. "abc.coaching-saas.com"
        $subdomain = explode('.', $host)[0];      // "abc"
        
        $tenant = Tenant::where('slug', $subdomain)->firstOrFail();
        
        app()->instance('current_tenant', $tenant);
        
        return $next($request);
    }
}
```

> **This is the heart of multi-tenant architecture.** The tenant is set once per request and used by the global scope (Lesson 24).

### Rate limiting

```php
class ThrottleApi
{
    public function handle(Request $request, Closure $next): Response
    {
        $key = $request->user()?->id ?? $request->ip();
        
        if (RateLimiter::tooManyAttempts($key, 60)) {
            return response()->json(['error' => 'Too many requests'], 429);
        }
        
        RateLimiter::hit($key, 60);  // 60-second window
        
        return $next($request);
    }
}
```

> Laravel ships with built-in rate limiters. Use them via `throttle:60,1` middleware alias.

### Request signing (for webhooks)

```php
class VerifyStripeWebhook
{
    public function handle(Request $request, Closure $next): Response
    {
        $signature = $request->header('Stripe-Signature');
        $secret = config('services.stripe.webhook_secret');
        
        try {
            \Stripe\Webhook::constructEvent($request->getContent(), $signature, $secret);
        } catch (\Exception $e) {
            return response()->json(['error' => 'Invalid signature'], 400);
        }
        
        return $next($request);
    }
}
```

> **Why this matters:** webhooks from Stripe must be verified, or attackers can forge them. We cover this in detail in Lesson 37.

---

## Theory: Terminable Middleware (Work After Response Sent)

Sometimes you want to do work **after** the response is sent to the user (so the user doesn't wait):

```php
class LogRequestAfterResponse
{
    public function handle(Request $request, Closure $next): Response
    {
        return $next($request);
    }
    
    public function terminate(Request $request, Response $response): void
    {
        // Runs after the response is sent to the user
        Log::info('Request completed', [
            'status' => $response->getStatusCode(),
            'duration_ms' => (microtime(true) - LARAVEL_START) * 1000,
        ]);
    }
}
```

> **Use for:** logging, analytics, slow query tracking. Things the user shouldn't wait for.

---

## Theory: Common Middleware Aliases Laravel Ships With

| Alias | What it does |
|---|---|
| `auth` | User must be logged in |
| `auth.basic` | HTTP Basic Auth (for APIs) |
| `guest` | User must NOT be logged in |
| `verified` | User must have verified email |
| `signed` | URL must be signed (prevents tampering) |
| `throttle:60,1` | Rate limit: 60 requests per 1 minute |
| `can:permission,model` | Policy check |
| `password.confirm` | User must re-enter password for sensitive actions |

```php
Route::middleware(['auth', 'throttle:60,1'])->group(function () {
    // Logged-in users, max 60 requests/min
});

Route::middleware('signed')->get('/verify-email/{token}', ...);
```

> **Senior rule:** learn the built-in middleware. Don't reinvent what Laravel already gives you.

---

## Why It Matters

Middleware is **the right place for cross-cutting concerns.** If you find yourself writing the same check in every controller, it's middleware. If you find yourself doing it in service classes, it's middleware. **Middleware first.**

For your SaaS, the critical middleware will be:
- `auth` — user logged in
- `tenant` — resolve current institution
- `subscription_active` — check subscription
- `role:admin` — role check
- `throttle:60,1` — rate limit
- Custom: log, monitor, etc.

These will be in **every request.** You cannot skip learning middleware.

---

## Lab: Build Tenant Resolution Middleware

### Step 1 — Create the middleware

```bash
cd G:\php-laravel\coaching-saas
php artisan make:middleware ResolveTenant
```

### Step 2 — Fill it in

Edit `app/Http/Middleware/ResolveTenant.php`:

```php
<?php
namespace App\Http\Middleware;

use App\Models\Tenant;
use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class ResolveTenant
{
    public function handle(Request $request, Closure $next): Response
    {
        // Get tenant from subdomain: abc.coaching-saas.com → "abc"
        $host = $request->getHost();
        $parts = explode('.', $host);
        
        // For localhost, use header or query param (development convenience)
        if ($host === '127.0.0.1' || $host === 'localhost') {
            $slug = $request->header('X-Tenant', 'default');
        } else {
            $slug = $parts[0];
        }
        
        $tenant = Tenant::where('slug', $slug)->first();
        
        if (!$tenant) {
            abort(404, "Unknown institution: $slug");
        }
        
        if (!$tenant->is_active) {
            abort(403, "Institution is not active");
        }
        
        // Bind the tenant as a singleton in the container
        app()->instance('current_tenant', $tenant);
        app()->instance(Tenant::class, $tenant);
        
        return $next($request);
    }
}
```

### Step 3 — Register the alias

Edit `bootstrap/app.php`:

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->alias([
        'tenant' => \App\Http\Middleware\ResolveTenant::class,
    ]);
})
```

### Step 4 — Use it in routes

Edit `routes/web.php`:

```php
Route::middleware(['tenant'])->group(function () {
    Route::resource('students', StudentController::class);
    Route::resource('school-classes', SchoolClassController::class);
});
```

> **Now every request to `/students` first goes through `ResolveTenant`.** If the tenant doesn't exist or is inactive, the request never reaches the controller.

### Step 5 — Access the tenant in a controller

```php
public function index(): View
{
    $tenant = app('current_tenant');  // or app(Tenant::class)
    $students = $tenant->students()->latest()->paginate(20);
    
    return view('students.index', compact('students'));
}
```

> **This is how a senior wires multi-tenancy:** the middleware resolves the tenant, the controller uses it. Clean separation.

### Step 6 — Test it

In `.env`, add (for dev convenience):

```
APP_DOMAIN=localhost
```

Make sure you have a `default` tenant in your DB:

```bash
php artisan tinker
>>> \App\Models\Tenant::firstOrCreate(['slug' => 'default'], [
...     'name' => 'Default Coaching',
...     'email' => 'admin@default.com',
...     'type' => 'coaching',
... ]);
```

Visit `http://127.0.0.1:8000/students`. The middleware runs, finds the default tenant, sets it as the current tenant. The page loads.

### Step 7 — Test it rejects bad tenants

Add a header in tinker:

```bash
curl -i -H "X-Tenant: nonexistent" http://127.0.0.1:8000/students
```

You should get a 404 "Unknown institution: nonexistent".

> **The middleware is now protecting every request.** This is multi-tenant security at the framework level.

---

## Quiz

**Q1.** A junior writes `if (!auth()->check()) abort(401);` in every controller method. A senior says "use middleware." Why is the senior right?

**Q2.** What's the difference between the "request phase" and the "response phase" of middleware? Give a concrete example of work that should happen in each.

**Q3.** When would you use `terminate()` instead of putting code in the response phase of `handle()`?

**Q4.** Why is `ResolveTenant` middleware critical for multi-tenant security? What attack does it prevent?

**Q5.** (Stretch) Read Laravel's built-in `VerifyCsrfToken` middleware (in `vendor/laravel/framework/src/Illuminate/Foundation\Http/Middleware/VerifyCsrfToken.php`). Trace what it does in both phases. Why is CSRF protection something that *must* be middleware, not controller code?

---

## What You Now Have

✅ Middleware fluency: write, register, parameterize, short-circuit
✅ Understanding of the request/response phases
✅ Global, group, and route middleware
✅ Real middleware: tenant resolution, rate limiting, webhook verification
✅ The `terminate()` pattern for post-response work

## What's Next

Lesson 21 covers **the service container and dependency injection** — Laravel's "magic" that makes controllers clean, testable, and decoupled.

📖 **[Lesson 21 — Service Container and Dependency Injection](21-service-container-and-dependency-injection.md)**
