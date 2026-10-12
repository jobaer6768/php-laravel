# Lesson 37 — Subscription Billing with Laravel Cashier

> **Phase 5: Production** | You'll be here: ~40 min

## The Senior's Question

You have a SaaS. People use it. Now you need them to **pay you monthly.** The junior builds a payments table and writes Stripe API calls. The senior uses **Laravel Cashier** — a tested, maintained abstraction that handles the gnarly parts: trials, plan changes, proration, failed payments, webhooks. Let me show you how a senior monetizes.

---

## Theory: The Subscription Model

A coaching SaaS typically has tiers:

```
Free Trial (14 days, no card)
   ↓
Basic ($29/mo) — 1 admin, 50 students
   ↓
Pro ($99/mo) — 5 admins, unlimited students, reports
   ↓
Enterprise (custom) — multiple branches, API
```

Cashier handles:
- Subscribing a tenant to a plan
- Trial periods
- Swapping plans (with proration)
- Canceling
- Resuming
- Failed payment recovery
- Webhooks (the events Stripe sends you)

> **Senior rule:** never roll your own billing. Stripe + Cashier is battle-tested. Your custom code will miss edge cases (refunds, disputes, plan changes mid-cycle).

---

## Theory: Install Laravel Cashier

```bash
composer require laravel/cashier
```

Publish the config:

```bash
php artisan vendor:publish --tag="cashier-config"
php artisan vendor:publish --tag="cashier-migrations"
php artisan migrate
```

Add the Billable trait to your `Tenant` model (or User, depending on your model):

```php
use Laravel\Cashier\Billable;

class Tenant extends Model
{
    use Billable;
}
```

> **Senior decision:** bill the tenant, not the user. A tenant's subscription persists even if a specific admin leaves.

---

## Theory: Stripe Setup

1. Create a Stripe account: https://dashboard.stripe.com
2. Get your API keys: Dashboard → Developers → API keys
3. Add to `.env`:

```env
STRIPE_KEY=pk_test_xxxxxxxxxxxx
STRIPE_SECRET=sk_test_xxxxxxxxxxxx
STRIPE_WEBHOOK_SECRET=whsec_xxxxxxxxxxxx
CASHIER_CURRENCY=usd
CASHIER_CURRENCY_LOCALE=en
```

In `config/services.php`:

```php
'stripe' => [
    'key' => env('STRIPE_KEY'),
    'secret' => env('STRIPE_SECRET'),
    'webhook' => [
        'secret' => env('STRIPE_WEBHOOK_SECRET'),
    ],
],
```

> **Test mode first.** Use `pk_test_*` and `sk_test_*`. Use card `4242 4242 4242 4242` for testing.

---

## Theory: Defining Plans

In Stripe Dashboard → Products → Add product:

```
Product: Coaching SaaS Basic
  - Price: $29/month, recurring
  - Price ID: price_1ABC...  ← copy this
```

Or via Stripe CLI:

```bash
stripe products create --name="Coaching Basic"
stripe prices create --product=prod_XXX --unit-amount=2900 --currency=usd --recurring[interval]=month
```

In `config/cashier.php`:

```php
'plans' => [
    'basic' => [
        'monthly' => 'price_1ABC...',
        'yearly' => 'price_1DEF...',
    ],
    'pro' => [
        'monthly' => 'price_1GHI...',
        'yearly' => 'price_1JKL...',
    ],
],
```

> **Senior rule:** store price IDs in config, not the database. They're Stripe's identifiers; let Stripe manage them.

---

## Theory: The Subscription Flow

### Step 1 — Show pricing page

```php
public function pricing()
{
    $plans = config('cashier.plans');
    return view('billing.pricing', compact('plans'));
}
```

### Step 2 — Create checkout session

```php
public function subscribe(Request $request, Tenant $tenant)
{
    $priceId = config("cashier.plans.{$request->plan}.{$request->interval}");
    
    return $request->user()->tenant->checkout([
        $priceId => $request->quantity ?? 1,
    ], [
        'success_url' => route('billing.success'),
        'cancel_url' => route('billing.pricing'),
    ]);
}
```

> **`checkout()`** creates a Stripe Checkout session and redirects the user to Stripe's hosted page. **No PCI compliance needed on your side** — Stripe handles card collection.

### Step 3 — Handle success

```php
public function success(Request $request)
{
    return view('billing.success');
}
```

After payment, Stripe sends a webhook. Cashier processes it.

### Step 4 — Handle webhooks

In `routes/web.php`:

```php
Route::post('stripe/webhook', '\Laravel\Cashier\Http\Controllers\WebhookController@handleWebhook');
```

Cashier's `WebhookController` handles all standard events. **You don't write webhook logic.**

> **Senior rule:** ALWAYS use Cashier's webhook controller. Never write your own. Edge cases: failed payments, dispute creation, subscription updates — Cashier handles them.

---

## Theory: Checking Subscription Status

In a controller or middleware:

```php
$tenant = app('current_tenant');

if ($tenant->subscribed('default')) {
    // Active subscription
}

if ($tenant->subscription('default')->onTrial()) {
    // In trial period
}

if ($tenant->subscription('default')->ended()) {
    // Subscription canceled and grace period over
}

if ($tenant->subscription('default')->hasIncompletePayment()) {
    // Payment failed, need to update card
}
```

> **`subscribed('default')`** checks the default subscription. You can have multiple (`'default'`, `'addons'`).

---

## Theory: Subscription Middleware

Block access for non-subscribers:

```bash
php artisan make:middleware EnsureTenantIsSubscribed
```

```php
public function handle($request, Closure $next)
{
    $tenant = app('current_tenant');
    
    if (!$tenant || !$tenant->subscribed('default')) {
        return redirect()->route('billing.pricing');
    }
    
    return $next($request);
}
```

Apply to routes:

```php
Route::middleware(['auth', 'tenant', 'subscribed'])->group(function () {
    Route::resource('students', StudentController::class);
    Route::resource('classes', SchoolClassController::class);
    // ...
});
```

> **Senior rule:** billing gates the entire app. The pricing page is the only public page for unauthenticated users.

---

## Theory: Trials

### Add trial days in Stripe Checkout

```php
$checkout = $tenant->checkout([$priceId => 1], [
    'trial_days' => 14,
    'success_url' => route('billing.success'),
    'cancel_url' => route('billing.pricing'),
]);

return $checkout;
```

### Check trial status

```php
if ($tenant->onTrial()) {
    $daysLeft = $tenant->trialEndsAt()->diffInDays(now());
    return view('billing.trial-banner', compact('daysLeft'));
}
```

### Extend trial (admin promotion)

```php
$tenant->extendTrial(now()->addDays(7));
```

---

## Theory: Plan Swaps (Upgrades/Downgrades)

```php
// Upgrading Basic → Pro
$tenant->subscription('default')
    ->swap('price_1GHI_pro_monthly');

// Cashier automatically prorates
```

> **Proration:** Stripe calculates the unused portion of the old plan and credits it. Cashier does the math.

---

## Theory: Cancellation

```php
// Cancel at end of period (graceful)
$tenant->subscription('default')->cancel();

// Cancel immediately
$tenant->subscription('default')->cancelNow();

// Resume a canceled subscription
$tenant->subscription('default')->resume();
```

> **Senior rule:** always cancel at end of period, not immediately. The user can still use the service until the period ends. Less angry emails.

---

## Theory: The Incomplete Payment Problem

Sometimes a card fails. Cashier marks the subscription as `incomplete`:

```php
@if ($tenant->subscription('default')->hasIncompletePayment())
    <div class="alert alert-danger">
        Your last payment failed. 
        <a href="{{ route('billing.update-card') }}">Update your card</a>.
    </div>
@endif
```

Provide a way to update the card:

```php
public function updateCard(Request $request, Tenant $tenant)
{
    return $tenant->updatePaymentMethod();
}
```

Stripe Smart Retries handles the rest. If the card is permanently dead, the subscription is canceled.

---

## Theory: Webhook Security

Stripe signs every webhook. Cashier verifies the signature.

In production, the route must be excluded from CSRF:

```php
// In bootstrap/app.php (Laravel 11+) or App\Http\Middleware\VerifyCsrfToken
->except(['stripe/webhook'])
```

> **Critical:** if you skip CSRF, the route MUST verify the Stripe signature. Cashier does this. Don't write your own.

Test webhooks locally with Stripe CLI:

```bash
stripe login
stripe listen --forward-to localhost:8000/stripe/webhook
stripe trigger invoice.payment_succeeded
```

---

## Theory: Invoices and Receipts

Cashier auto-generates invoices. Show them:

```php
public function invoices(Tenant $tenant)
{
    $invoices = $tenant->invoices();
    return view('billing.invoices', compact('invoices'));
}
```

User can download PDF:

```php
return $tenant->downloadInvoice($invoice->id, [
    'vendor' => 'Coaching SaaS Inc.',
    'product' => 'Subscription',
]);
```

> **Senior rule:** every successful charge should generate an invoice. **Tax compliance, customer trust, refunds.** Cashier handles it.

---

## Theory: Local Development Without Stripe

For tests, fake Cashier:

```php
use Laravel\Cashier\Cashier;

Cashier::fake();

// Now subscriptions don't actually hit Stripe
$tenant = Tenant::factory()->create();
$tenant->subscription('default')->create('price_fake');
```

Or use Stripe test mode with test cards.

---

## Theory: Billing Quirks to Know

### Currency

Cashier uses one currency per Stripe account. If you have global customers, you need a multi-currency setup. **Out of scope for v1.**

### Tax

Stripe Tax calculates tax automatically. Enable in Dashboard. Add to checkout:

```php
$checkout = $tenant->checkout([$priceId => 1], [
    'automatic_tax' => ['enabled' => true],
    // ...
]);
```

### Coupons

```php
$tenant->subscription('default')->withCoupon('PROMO20')->create('price_xxx');
```

### Customer Portal

Stripe has a hosted customer portal. Let users manage their own subscription:

```php
public function portal(Tenant $tenant)
{
    return $tenant->redirectToBillingPortal();
}
```

> **Senior rule:** use Stripe's customer portal for cancel/update card. Don't build your own.

---

## Why It Matters

Billing is the **business logic** of your SaaS. Get it wrong:
- You charge customers but lose access (bad)
- Customers have access but you don't charge (worse)
- Webhooks fail and you miss payments (disaster)

Cashier is the senior's answer because:
- **Battle-tested** — used by thousands of apps
- **Handles edge cases** you don't know about (disputes, refunds, proration)
- **Updates with Stripe's API** — you don't have to chase changes

> **Never write payment code from scratch.** Even Stripe says so. Use Cashier.

---

## Lab: Add Subscriptions to Your SaaS

### Step 1 — Install Cashier

```bash
cd G:\php-laravel\coaching-saas
composer require laravel/cashier
php artisan vendor:publish --tag="cashier-config"
php artisan vendor:publish --tag="cashier-migrations"
php artisan migrate
```

### Step 2 — Set up Stripe

Create a Stripe account. Get test keys. Add to `.env`:

```env
STRIPE_KEY=pk_test_...
STRIPE_SECRET=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
CASHIER_CURRENCY=usd
```

### Step 3 — Add Billable trait

`app/Models/Tenant.php`:

```php
use Laravel\Cashier\Billable;

class Tenant extends Model
{
    use Billable, BelongsToTenant;
}
```

### Step 4 — Create plans in Stripe

In Stripe Dashboard:

```
Product: "Coaching Basic"
  - Monthly: $29 → price_basic_monthly
  - Yearly: $290 → price_basic_yearly

Product: "Coaching Pro"
  - Monthly: $99 → price_pro_monthly
  - Yearly: $990 → price_pro_yearly
```

Update `config/cashier.php`:

```php
'plans' => [
    'basic' => [
        'monthly' => env('CASHIER_BASIC_MONTHLY'),
        'yearly' => env('CASHIER_BASIC_YEARLY'),
    ],
    'pro' => [
        'monthly' => env('CASHIER_PRO_MONTHLY'),
        'yearly' => env('CASHIER_PRO_YEARLY'),
    ],
],
```

Add to `.env`:

```env
CASHIER_BASIC_MONTHLY=price_xxx
CASHIER_BASIC_YEARLY=price_xxx
CASHIER_PRO_MONTHLY=price_xxx
CASHIER_PRO_YEARLY=price_xxx
```

### Step 5 — Create the pricing page

`app/Http/Controllers/BillingController.php`:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class BillingController extends Controller
{
    public function pricing()
    {
        $plans = config('cashier.plans');
        return view('billing.pricing', compact('plans'));
    }
    
    public function subscribe(Request $request)
    {
        $request->validate([
            'plan' => 'required|in:basic,pro',
            'interval' => 'required|in:monthly,yearly',
        ]);
        
        $tenant = app('current_tenant');
        $priceId = config("cashier.plans.{$request->plan}.{$request->interval}");
        
        if ($tenant->subscribed('default')) {
            return back()->with('error', 'You are already subscribed.');
        }
        
        return $tenant->checkout([$priceId => 1], [
            'success_url' => route('billing.success') . '?session_id={CHECKOUT_SESSION_ID}',
            'cancel_url' => route('billing.pricing'),
            'trial_days' => 14,
        ]);
    }
    
    public function success(Request $request)
    {
        return view('billing.success');
    }
    
    public function portal(Request $request)
    {
        return app('current_tenant')->redirectToBillingPortal(
            route('billing.pricing')
        );
    }
}
```

### Step 6 — Add routes

`routes/web.php`:

```php
Route::middleware(['auth', 'tenant'])->group(function () {
    Route::get('billing/pricing', [BillingController::class, 'pricing'])->name('billing.pricing');
    Route::post('billing/subscribe', [BillingController::class, 'subscribe'])->name('billing.subscribe');
    Route::get('billing/success', [BillingController::class, 'success'])->name('billing.success');
    Route::get('billing/portal', [BillingController::class, 'portal'])->name('billing.portal');
});

// Cashier webhook (no auth, no CSRF)
Route::post('stripe/webhook', '\Laravel\Cashier\Http\Controllers\WebhookController@handleWebhook');
```

### Step 7 — Create the subscription middleware

```bash
php artisan make:middleware EnsureTenantIsSubscribed
```

`app/Http/Middleware/EnsureTenantIsSubscribed.php`:

```php
public function handle($request, Closure $next)
{
    $tenant = app('current_tenant');
    
    if (!$tenant) {
        return redirect()->route('login');
    }
    
    if (!$tenant->subscribed('default') && !$tenant->onTrial()) {
        return redirect()->route('billing.pricing')
            ->with('warning', 'Please subscribe to continue.');
    }
    
    return $next($request);
}
```

Register in `bootstrap/app.php` (Laravel 11+):

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->alias([
        'subscribed' => \App\Http\Middleware\EnsureTenantIsSubscribed::class,
    ]);
})
```

Apply to your routes:

```php
Route::middleware(['auth', 'tenant', 'subscribed'])->group(function () {
    Route::resource('students', StudentController::class);
    // ...
});
```

### Step 8 — Add the pricing view

`resources/views/billing/pricing.blade.php`:

```blade
@extends('layouts.app')

@section('content')
<div class="container">
    <h1>Choose Your Plan</h1>
    
    <div class="row">
        @foreach($plans as $name => $intervals)
        <div class="col-md-4">
            <div class="card">
                <div class="card-body">
                    <h3 class="card-title">{{ ucfirst($name) }}</h3>
                    
                    <form method="POST" action="{{ route('billing.subscribe') }}">
                        @csrf
                        <input type="hidden" name="plan" value="{{ $name }}">
                        
                        <button type="submit" name="interval" value="monthly" class="btn btn-primary btn-block">
                            Monthly
                        </button>
                        <button type="submit" name="interval" value="yearly" class="btn btn-outline-primary btn-block">
                            Yearly (Save 17%)
                        </button>
                    </form>
                </div>
            </div>
        </div>
        @endforeach
    </div>
</div>
@endsection
```

### Step 9 — Show a trial banner

`resources/views/layouts/app.blade.php`:

```blade
@auth
    @php $tenant = app('current_tenant'); @endphp
    
    @if($tenant && $tenant->onTrial())
        <div class="alert alert-info">
            You have {{ max(0, (int) $tenant->trialEndsAt()->diffInDays(now())) }} days left in your trial.
            <a href="{{ route('billing.pricing') }}">Subscribe now</a>
        </div>
    @endif
    
    @if($tenant && $tenant->subscription('default')?->hasIncompletePayment())
        <div class="alert alert-danger">
            Your last payment failed.
            <a href="{{ route('billing.portal') }}">Update your card</a>.
        </div>
    @endif
@endauth
```

### Step 10 — Test the flow

```bash
# Start the app
php artisan serve

# Start Stripe CLI to forward webhooks
stripe listen --forward-to localhost:8000/stripe/webhook

# In browser: visit /billing/pricing, click Subscribe
# Use card: 4242 4242 4242 4242, any future date, any CVC

# After payment, check:
php artisan tinker
>>> app('current_tenant')->subscribed('default')
=> true
```

### Step 11 — Test failed payment

Use card `4000 0000 0000 9995` (insufficient funds).

The webhook fires `invoice.payment_failed`. The subscription becomes `incomplete`. The banner shows.

### Step 12 — Test cancellation

Visit `/billing/portal` → Cancel subscription. Webhook fires `customer.subscription.deleted`. The user is unsubscribed.

---

## Quiz

**Q1.** Why use Cashier instead of calling the Stripe API directly? List 3 specific things Cashier handles for you.

**Q2.** You want to allow a 14-day trial without requiring a card. How do you configure this in Cashier?

**Q3.** A customer wants to upgrade from Basic ($29) to Pro ($99) mid-cycle. What does "proration" mean? How does Cashier handle it?

**Q4.** A customer's card fails. The subscription is `incomplete`. They can't access the app. Walk through what the user sees and what you should do.

**Q5.** (Stretch) Your SaaS is global. You have customers in the US, EU, and India. You need to charge in USD, EUR, and INR with correct tax. Walk through the Cashier + Stripe Tax setup.

---

## What You Now Have

✅ Stripe + Cashier installed
✅ Plans configured
✅ Checkout flow
✅ Webhook handling
✅ Subscription middleware
✅ Trial support
✅ Failed payment UX

## What's Next

Lesson 38 covers **email and notifications** — transactional emails, queues, mail templates, the patterns that make user communication reliable at scale.

📖 **[Lesson 38 — Email and Notifications](38-email-and-notifications.md)**
