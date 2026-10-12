# Lesson 14 — The Request Lifecycle

> **Phase 2: Laravel Core** | You'll be here: ~25 min

## The Senior's Question

Before you write a single line of Laravel code, you need to know what happens between "browser hits URL" and "HTML comes back." If you don't know this, every bug you hit will be a guess. If you do know it, every bug has a place to look.

---

## Theory: The Full Request Lifecycle

```
1. Browser sends HTTP request
        ↓
2. Web server (Nginx) receives it
        ↓
3. Nginx serves static files directly (CSS, JS, images)
        ↓
4. For dynamic requests, Nginx hands off to PHP-FPM
        ↓
5. PHP loads public/index.php
        ↓
6. public/index.php loads the Composer autoloader
        ↓
7. Composer autoloader makes all classes available
        ↓
8. public/index.php creates the Application (Kernel)
        ↓
9. Kernel bootstraps providers, facades, config
        ↓
10. Request is bound to the service container
        ↓
11. Request flows through the Global Middleware stack
        ↓
12. Router matches the URL to a route definition
        ↓
13. Route's middleware stack runs
        ↓
14. Controller method is called (or closure runs)
        ↓
15. Controller returns a Response (View, JSON, redirect, file, etc.)
        ↓
16. Response flows back through the middleware (in reverse)
        ↓
17. Response is sent back to PHP-FPM → Nginx → browser
```

That's a lot. Let's break it into **four mental layers**:

```
┌──────────────────────────────────────┐
│  LAYER 1: Web Server                 │  Nginx: serves static, routes dynamic
└──────────────────┬───────────────────┘
                   ↓
┌──────────────────────────────────────┐
│  LAYER 2: PHP Runtime                │  PHP-FPM: runs PHP files
└──────────────────┬───────────────────┘
                   ↓
┌──────────────────────────────────────┐
│  LAYER 3: Framework Bootstrap        │  Laravel: loads config, providers, routes
└──────────────────┬───────────────────┘
                   ↓
┌──────────────────────────────────────┐
│  LAYER 4: Application Code           │  Your code: routes, controllers, models
└──────────────────────────────────────┘
```

You write Layer 4. Laravel does Layer 3. PHP does Layer 2. Nginx does Layer 1.

---

## Theory: What Happens Inside `public/index.php`

The actual file is ~20 lines. Here's what it does:

```php
<?php
use Illuminate\Foundation\Application;
use Illuminate\Http\Request;

define('LARAVEL_START', microtime(true));

// Maintenance mode check
if (file_exists($maintenance = __DIR__.'/../storage/framework/maintenance.php')) {
    require $maintenance;
}

// Composer autoloader
require __DIR__.'/../vendor/autoload.php';

// Bootstrap the framework
$app = require_once __DIR__.'/../bootstrap/app.php';

// Receive the request, send the response
$app->handleRequest(Request::capture());
```

That's it. The whole entry point.

> **The three key lines:**
> 1. `require __DIR__.'/../vendor/autoload.php';` — loads all Composer packages so any class can be found
> 2. `$app = require_once __DIR__.'/../bootstrap/app.php';` — creates the Laravel application
> 3. `$app->handleRequest(Request::capture());` — the request flows in, the response flows out

---

## Theory: The Service Container

The `$app` variable is an instance of `Illuminate\Foundation\Application`. It's the **service container** — Laravel's central registry.

Think of it as a giant dictionary:

```php
// Bind a service into the container
$app->bind(StripeClient::class, function () {
    return new StripeClient(config('services.stripe.secret'));
});

// Resolve (get) a service from the container
$stripe = $app->make(StripeClient::class);
```

Almost everything in Laravel is registered in the container:
- Database connection
- Cache driver
- Mailer
- Logger
- Session manager
- Your custom services

When you type-hint a class in a controller, Laravel **automatically resolves it from the container.** This is "dependency injection." We'll cover it in Lesson 21.

> **Senior rule:** if you ever see `new SomeService()` in a controller, that's a smell. Inject it instead.

---

## Theory: Service Providers — The "Bootstrap Scripts"

When Laravel starts, it runs through all registered **service providers** in order. Each provider can:

- Register services in the container
- Boot services that depend on other services
- Load routes, views, translations, migrations

Laravel ships with ~30 providers. You can see them in `bootstrap/providers.php` (Laravel 11+) or `config/app.php` (older versions).

A provider looks like:

```php
namespace App\Providers;

use App\Services\BillingService;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->singleton(BillingService::class, function ($app) {
            return new BillingService(
                config('services.stripe.secret'),
            );
        });
    }

    public function boot(): void
    {
        // ... boot logic, e.g. register view composers
    }
}
```

> **You'll rarely write a provider from scratch.** But you'll modify `AppServiceProvider` to register your own services.

---

## Theory: Facades — The "Static Interface" to the Container

You'll see this in Laravel code all the time:

```php
Cache::put('key', 'value', 60);
Mail::to($user)->send(new WelcomeEmail());
Route::get('/students', [StudentController::class, 'index']);
```

`Cache::put()` looks like a static call, but it's actually resolving the cache service from the container. The `Cache` "class" is a **facade** — a static proxy to the container.

You can see what `Cache` resolves to:

```bash
php artisan config:show cache.default
```

> **Senior view on facades:** they're convenient but hide the dependency. Some teams prefer to always inject services explicitly. Both are valid. The Laravel community accepts facades because they're readable.

---

## Theory: The Middleware Pipeline

After the framework boots, **the request flows through middleware** before reaching your controller:

```
Request
   ↓
[TrustProxies] — for load balancer setups
   ↓
[HandleCors] — CORS headers
   ↓
[PreventRequestsDuringMaintenance] — 503 page during maintenance
   ↓
[ValidatePostSize] — form too large?
   ↓
[TrimStrings] — strip whitespace from input
   ↓
[ConvertEmptyStringsToNull] — '' becomes null
   ↓
[StartSession] — load session
   ↓
[ShareErrorsFromSession] — make old validation errors available
   ↓
[VerifyCsrfToken] — POST/PUT/DELETE must have valid CSRF token
   ↓
[SubstituteBindings] — resolve {id} in URL to a Model instance
   ↓
[Route-specific middleware: auth, tenant, throttle, etc.]
   ↓
Controller
```

After the controller, **the response flows back through the same middleware in reverse.** This is the **onion model.**

```
Response
   ↑
[SubstituteBindings] (response phase)
   ↑
[auth middleware response phase]
   ↑
[VerifyCsrfToken] (response phase)
   ↑
[StartSession] (save session)
   ↑
[TrimStrings] (response phase — usually does nothing)
   ↑
Response sent to browser
```

You can write middleware that does work **before** the controller (request phase) and **after** the controller (response phase):

```php
public function handle(Request $request, Closure $next): Response
{
    // Request phase: BEFORE controller
    Log::info("Request started: {$request->url()}");
    
    $response = $next($request);  // call the controller
    
    // Response phase: AFTER controller
    Log::info("Response sent: {$response->getStatusCode()}");
    
    return $response;
}
```

---

## Theory: The Router

The router matches the URL + method + middleware to a closure or controller method. The definitions live in `routes/web.php` and `routes/api.php`:

```php
// routes/web.php
Route::get('/', [HomeController::class, 'index']);
Route::post('/students', [StudentController::class, 'store']);
Route::put('/students/{student}', [StudentController::class, 'update']);
Route::delete('/students/{student}', [StudentController::class, 'destroy']);
```

`{student}` is a **route parameter.** If the URL is `/students/42`, Laravel tries to find a `Student` model with ID 42 and inject it.

> This is the magic of **route model binding** — a topic for Lesson 17.

---

## Why It Matters

When something goes wrong in a Laravel app, the question is "where in this lifecycle did it break?" The four layers give you a triage:

| Symptom | Where to look |
|---|---|
| 404 on a valid URL | Routes file (Layer 4) |
| 500 with "Class not found" | Composer autoloader (Layer 3) |
| 500 with "Permission denied" on a file | File system / storage permissions (Layer 2) |
| 502 Bad Gateway | PHP-FPM not running (Layer 2) |
| Blank page | PHP error suppressed (Layer 2) |
| Request hangs | Middleware loop or slow query (Layer 3-4) |
| `auth` middleware blocks valid user | Auth logic in middleware (Layer 4) |

A senior reads the error, **places it in the lifecycle, and knows where to look.** A junior panics and starts guessing.

---

## Lab: Trace a Request

### Step 1 — Boot the dev server

```bash
cd G:\php-laravel\coaching-saas
php artisan serve
```

In a separate terminal, make a request:

```bash
curl -v http://127.0.0.1:8000/
```

> **Notice the `v` flag** for verbose output. You'll see request/response headers, status codes, timing.

### Step 2 — Look at the actual response

```bash
curl -i http://127.0.0.1:8000/
```

The `-i` flag shows headers + body. You should see `200 OK` and the HTML of the welcome page.

### Step 3 — Try a URL that doesn't exist

```bash
curl -i http://127.0.0.1:8000/does-not-exist
```

You get `404 Not Found`. The router didn't match any route.

### Step 4 — See the registered routes

In a new terminal:

```bash
cd G:\php-laravel\coaching-saas
php artisan route:list
```

You'll see all routes Laravel knows about. With the default project, there's exactly one: `GET /` returns the welcome view.

### Step 5 — Look at the global middleware

Open `coaching-saas/bootstrap/app.php`. You'll see the middleware configuration. In Laravel 11+:

```php
->withMiddleware(function (Middleware $middleware) {
    // Customize middleware here
})
```

The default global middleware is auto-applied: TrimStrings, ConvertEmptyStringsToNull, etc.

### Step 6 — See all registered services

```bash
php artisan about
```

This dumps a summary of your app: environment, cache driver, DB driver, mail driver, queue driver, etc. Each is bound in the container.

### Step 7 — Trace it manually

Add a temporary route to see the full Request object:

```php
// routes/web.php — add this temporarily
Route::get('/debug', function (\Illuminate\Http\Request $request) {
    return response()->json([
        'url' => $request->url(),
        'method' => $request->method(),
        'ip' => $request->ip(),
        'user_agent' => $request->userAgent(),
        'headers' => $request->headers->all(),
        'all_input' => $request->all(),
    ]);
});
```

Visit `http://127.0.0.1:8000/debug` in your browser. You see **everything** about the incoming request.

> **Senior note:** this is how you debug. Hit a debug endpoint, dump the request, see what your code is actually receiving.

### Step 8 — Watch the lifecycle with a log

Add a tiny middleware that logs:

```bash
php artisan make:middleware LogRequest
```

Edit `app/Http/Middleware/LogRequest.php`:

```php
public function handle(Request $request, Closure $next): Response
{
    \Log::info("Request started", ['url' => $request->url()]);
    
    $response = $next($request);
    
    \Log::info("Response sent", [
        'status' => $response->getStatusCode(),
    ]);
    
    return $response;
}
```

Register it globally in `bootstrap/app.php`:

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->append(LogRequest::class);
})
```

Hit any URL. Then check the log:

```bash
cat storage/logs/laravel.log
```

You'll see the request and response lines, in order. **This is the lifecycle, captured.**

### Step 9 — Remove the debug route and middleware when done

Clean up. Don't leave debug routes in production.

---

## Quiz

**Q1.** A request comes in but the response is a 500 with "Class 'Stripe\Stripe' not found." Which layer of the lifecycle is broken, and what's the fix?

**Q2.** A POST to `/students` returns 419 "Page Expired." What middleware is responsible, and what does it check?

**Q3.** Why is `public/index.php` the only file the web server can access directly? What would happen if Nginx was misconfigured to serve `storage/`?

**Q4.** You write a controller that calls `new BillingService()`. A senior says "inject it instead." What's the senior's reasoning, and how would you refactor?

**Q5.** (Stretch) Read the four-layer diagram. If you scale to multiple servers behind a load balancer, which layer needs special consideration? Why?

---

## What You Now Have

✅ Mental model of the full request lifecycle
✅ Knowledge of the four layers
✅ Understanding of the service container, providers, facades, middleware
✅ Ability to trace a request and see it happen
✅ Pattern for debugging via log statements

## What's Next

Lesson 15 dives into **routing, controllers, and responses** — the first things you'll write real application code in.

📖 **[Lesson 15 — Routing, Controllers, and Responses](15-routing-controllers-and-responses.md)**
