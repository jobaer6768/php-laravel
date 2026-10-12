# Lesson 21 — Service Container and Dependency Injection

> **Phase 2: Laravel Core** | You'll be here: ~35 min

## The Senior's Question

Laravel can automatically "wire up" complex objects for you. You ask for a `BillingService` and Laravel knows how to construct one, including all of *its* dependencies, recursively. This is the **service container**, and it's the single biggest reason Laravel controllers stay small.

---

## Theory: The Problem

Imagine you have a `BillingService` that depends on `StripeClient` and `Log`. And `StripeClient` depends on a config value. And `Log` depends on a logger...

```php
// Without DI: you'd write this in every controller
public function charge(Request $request) {
    $config = config('services.stripe');
    $logger = new Logger();
    $stripe = new StripeClient($config['secret']);
    $billing = new BillingService($stripe, $logger);
    
    $billing->charge($request->amount);
}
```

Imagine doing that in every controller method. Now imagine your coworker wants to test this method — they'd have to construct all these dependencies too.

---

## Theory: The Service Container Solves This

```php
// With DI: just type-hint what you need
public function charge(Request $request, BillingService $billing) {
    $billing->charge($request->amount);
}
```

Laravel sees `BillingService` type-hinted, **constructs it for you**, with all its dependencies. **This is automatic dependency injection.**

---

## Theory: How the Container Resolves Dependencies

When Laravel sees `BillingService $billing`:

1. Looks up `BillingService` in the container
2. Not registered? Tries to **build it automatically** using reflection
3. Inspects the constructor: `__construct(StripeClient $stripe, Logger $logger)`
4. Recursively resolves each parameter
5. Constructs the final object

This is called **autowiring.** It works for almost any class with type-hinted dependencies.

> **Senior rule:** if you follow type-hint conventions, you rarely need to register anything manually. Laravel figures it out.

---

## Theory: When to Register Manually

Sometimes you need to tell Laravel how to build something:

### Scoped bindings (singleton)

```php
// In a Service Provider's register() method
$this->app->singleton(BillingService::class, function ($app) {
    return new BillingService(
        $app->make(StripeClient::class),
        config('services.stripe.secret'),
    );
});
```

Now `BillingService` is **a singleton** — one instance per request, reused everywhere.

### Interface-to-implementation binding

```php
$this->app->bind(
    \App\Contracts\SmsSender::class,
    \App\Services\TwilioSmsSender::class,
);
```

Now when a controller type-hints `SmsSender`, Laravel injects `TwilioSmsSender`. **You can swap implementations without changing controllers.**

### Contextual binding (different impl in different places)

```php
$this->app->when(AdminController::class)
    ->needs(SmsSender::class)
    ->give(TwilioSmsSender::class);

$this->app->when(StudentController::class)
    ->needs(SmsSender::class)
    ->give(LocalLogSmsSender::class);  // dev mode
```

---

## Theory: Constructor Injection vs Method Injection

### Method injection (what we did above)

```php
public function charge(Request $request, BillingService $billing) {
    $billing->charge($request->amount);
}
```

> **Good for:** classes that don't *own* the dependency (controllers, jobs, listeners).

### Constructor injection (preferred for owned deps)

```php
class OrderProcessor
{
    public function __construct(
        private BillingService $billing,
        private NotificationService $notifications,
    ) {}
    
    public function process(Order $order): void {
        $this->billing->charge($order->total);
        $this->notifications->send($order->user, 'Order processed');
    }
}
```

> **Good for:** services, jobs, listeners, commands — anything that owns its dependencies for its lifetime.

> **Senior rule:** controllers can use method injection freely (they're short-lived). Service classes should use constructor injection (their lifetime spans many operations).

---

## Theory: The Container as a Service Locator (Anti-pattern)

The container also exposes `make()`, `bind()`, and `instance()`:

```php
$billing = app(BillingService::class);  // or app()->make(...)
```

> **This works** but it's an anti-pattern. It hides dependencies. If you reach for `app()` inside a method, you're doing it wrong. **Inject instead.**

> **Exception:** resolving from the container *inside* a service provider is fine — that's literally its job.

---

## Theory: Real Example — A Configurable Billing Service

```php
// app/Services/BillingService.php
namespace App\Services;

use App\Contracts\PaymentGateway;
use App\Models\Subscription;
use Illuminate\Support\Facades\Log;

class BillingService
{
    public function __construct(
        private PaymentGateway $gateway,    // injected: Stripe or Mock
        private string $currency = 'USD',    // injected: configured
    ) {}
    
    public function chargeSubscription(Subscription $sub): bool
    {
        try {
            $result = $this->gateway->charge(
                amount: $sub->amount,
                currency: $this->currency,
                customer: $sub->customer_id,
            );
            
            if ($result->isSuccessful()) {
                $sub->markAsPaid();
                return true;
            }
            
            $sub->markAsFailed($result->errorMessage());
            return false;
        } catch (\Throwable $e) {
            Log::error('Billing failed', [
                'subscription_id' => $sub->id,
                'error' => $e->getMessage(),
            ]);
            throw $e;
        }
    }
}
```

Register in `AppServiceProvider`:

```php
public function register(): void
{
    $this->app->bind(PaymentGateway::class, function ($app) {
        return new \App\Services\StripeGateway(
            config('services.stripe.secret')
        );
    });
    
    $this->app->singleton(BillingService::class, function ($app) {
        return new BillingService(
            $app->make(PaymentGateway::class),
            config('app.billing_currency', 'USD'),
        );
    });
}
```

Use in a controller:

```php
public function charge(Subscription $sub, BillingService $billing): RedirectResponse
{
    if ($billing->chargeSubscription($sub)) {
        return back()->with('success', 'Charged successfully');
    }
    return back()->with('error', 'Payment failed');
}
```

> **Notice:** the controller doesn't know about Stripe. It doesn't know about currency config. It doesn't construct anything. **It just trusts the container.**

---

## Theory: Why This Matters for Testing

This is the **real** win of DI. With `new`, you must construct everything in your test:

```php
// Test WITH DI
public function test_charges_subscription() {
    $gateway = new FakePaymentGateway();  // implement PaymentGateway contract
    $billing = new BillingService($gateway, 'USD');
    
    $result = $billing->chargeSubscription($this->fakeSubscription());
    
    $this->assertTrue($result);
    $this->assertEquals(1, $gateway->chargeCount);
}
```

You can swap implementations in tests without touching production code. **This is how unit testing becomes easy.**

---

## Why It Matters

- **Decoupling:** your code depends on interfaces, not implementations
- **Testability:** inject fakes in tests
- **Configurability:** bind different implementations in different environments
- **Readability:** controllers don't construct things, they use them

A senior writes:
```php
public function __construct(
    private BillingService $billing,
    private NotificationService $notifications,
) {}
```

A junior writes:
```php
public function process() {
    $billing = new BillingService(new StripeGateway(config('services.stripe.secret')), 'USD');
    $notifications = new NotificationService(new TwilioSms(...), new SmtpMail(...));
    // ... 30 lines of construction
}
```

Same behavior. Massive quality difference.

---

## Lab: Wire Up a Real Service

### Step 1 — Create the interface

Create `app/Contracts/PaymentGateway.php`:

```php
<?php
namespace App\Contracts;

interface PaymentGateway
{
    public function charge(float $amount, string $currency, string $customerId): \stdClass;
}
```

### Step 2 — Create a fake implementation (for development)

Create `app/Services/FakePaymentGateway.php`:

```php
<?php
namespace App\Services;

use App\Contracts\PaymentGateway;

class FakePaymentGateway implements PaymentGateway
{
    public int $chargeCount = 0;
    public float $totalCharged = 0;
    
    public function charge(float $amount, string $currency, string $customerId): \stdClass
    {
        $this->chargeCount++;
        $this->totalCharged += $amount;
        
        return (object) [
            'success' => true,
            'transaction_id' => 'fake_' . uniqid(),
            'amount' => $amount,
        ];
    }
}
```

### Step 3 — Create a real implementation

Create `app/Services/StripePaymentGateway.php`:

```php
<?php
namespace App\Services;

use App\Contracts\PaymentGateway;

class StripePaymentGateway implements PaymentGateway
{
    public function __construct(private string $secretKey) {}
    
    public function charge(float $amount, string $currency, string $customerId): \stdClass
    {
        // Pretend we call Stripe API
        \Log::info("Would charge $amount $currency to $customerId");
        
        // In production:
        // \Stripe\Stripe::setApiKey($this->secretKey);
        // return \Stripe\Charge::create([...]);
        
        return (object) [
            'success' => true,
            'transaction_id' => 'pending_real_stripe',
            'amount' => $amount,
        ];
    }
}
```

### Step 4 — Bind the implementations

Edit `app/Providers/AppServiceProvider.php`:

```php
public function register(): void
{
    $this->app->bind(
        \App\Contracts\PaymentGateway::class,
        function ($app) {
            // In production, use Stripe. In dev (without key), use fake.
            $key = config('services.stripe.secret');
            
            if ($key && $key !== 'fake') {
                return new \App\Services\StripePaymentGateway($key);
            }
            
            return new \App\Services\FakePaymentGateway();
        }
    );
}
```

### Step 5 — Use it in a controller

Edit `app/Http/Controllers/StudentController.php`:

```php
public function charge(
    \App\Models\Student $student,
    \App\Contracts\PaymentGateway $gateway,
): \Illuminate\Http\JsonResponse
{
    $result = $gateway->charge(
        amount: 5000.00,
        currency: 'USD',
        customerId: 'cust_' . $student->id,
    );
    
    return response()->json([
        'success' => $result->success,
        'transaction_id' => $result->transaction_id,
    ]);
}
```

> **The controller doesn't know if it's calling Stripe or a fake.** It just calls the interface.

### Step 6 — Test it

```bash
php artisan serve

curl -X POST http://127.0.0.1:8000/students/1/charge
```

You should see JSON with `success: true`. Without a Stripe key, it uses the fake. With a key, it would use Stripe.

### Step 7 — Verify the binding works

In tinker:

```php
$gateway = app(\App\Contracts\PaymentGateway::class);
get_class($gateway);
// "App\Services\FakePaymentGateway" (or StripePaymentGateway if .env has a key)
```

---

## Quiz

**Q1.** Why is constructor injection preferred for service classes, while method injection is fine for controllers?

**Q2.** A junior writes `new StripeClient(config('services.stripe.secret'))` inside a controller. Give three specific problems this causes for testing, flexibility, and readability.

**Q3.** What's the difference between `$this->app->bind()` and `$this->app->singleton()`? When would you use each?

**Q4.** You have an interface `PaymentGateway`. You bind `StripePaymentGateway` to it. In your test, you want to use a fake. How do you swap implementations for testing? (Hint: `$this->app->bind()` again.)

**Q5.** (Stretch) Read Laravel's `Illuminate\Database\DatabaseManager` (you can `cat vendor/laravel/framework/src/Illuminate/Database/DatabaseManager.php`). Why is this class registered as a singleton in the container? What's the senior's reasoning?

---

## What You Now Have

✅ Container fluency: bind, singleton, contextual, autowire
✅ Constructor vs method injection patterns
✅ Interface-to-implementation decoupling
✅ Real service wiring with config-driven implementation
✅ The mental model for testability

## What's Next

Lesson 22 covers **Artisan commands** — custom CLI commands for one-off jobs, deployment tasks, and maintenance. The senior's superpower.

📖 **[Lesson 22 — Artisan: The CSGO of Laravel](22-artisan-the-csgo-of-laravel.md)**
