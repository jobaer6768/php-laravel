# Lesson 23 — Configuration and Environments

> **Phase 2: Laravel Core** | You'll be here: ~30 min

## The Senior's Question

A junior has one `.env` file, one database, one Stripe key — and the same code somehow doesn't work on staging or production. A senior has **disciplined environment management**: dev, staging, production each have their own settings, secrets are never committed, and the code behaves correctly in each environment by configuration, not by accident.

---

## Theory: The Three Layers of Configuration

Laravel configuration flows in three layers:

```
1. .env (file) — environment-specific secrets and values
        ↓
2. config/*.php (files) — reads from .env, exposes typed values
        ↓
3. code — calls config() to read
```

| Layer | Lives in | Committed to git? |
|---|---|---|
| `.env` | Project root | **NEVER** |
| `.env.example` | Project root | Yes (template) |
| `config/*.php` | `config/` directory | Yes |
| Code calling `config()` | Everywhere | Yes |

> **The rules:**
> - Never commit `.env`
> - Always commit `.env.example` as a template
> - Read `config()` in code, not `env()` (except inside config files)
> - Read `env()` only in config files

---

## Theory: The `.env` File

```bash
# .env
APP_NAME="Coaching SaaS"
APP_ENV=local
APP_KEY=base64:abc123...           # unique per install
APP_DEBUG=true                      # false in production
APP_URL=http://localhost:8000

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_DATABASE=coaching_saas
DB_USERNAME=root
DB_PASSWORD=secret

MAIL_MAILER=smtp
MAIL_HOST=smtp.mailtrap.io
MAIL_PORT=2525
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=tls

CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis
SESSION_LIFETIME=120

STRIPE_KEY=pk_test_xxxxxxxx
STRIPE_SECRET=sk_test_xxxxxxxx
STRIPE_WEBHOOK_SECRET=whsec_xxxxxxxx

# Custom app values
BILLING_CURRENCY=USD
TRIAL_DAYS=14
SUPPORT_EMAIL=support@example.com
```

> **Notice:** no comments needed for what each value means. The names are self-explanatory.

---

## Theory: `.env.example` — The Template

```bash
# .env.example
APP_NAME="Coaching SaaS"
APP_ENV=local
APP_KEY=
APP_DEBUG=true
APP_URL=http://localhost:8000

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=
DB_USERNAME=
DB_PASSWORD=

MAIL_MAILER=log
MAIL_HOST=
MAIL_PORT=
MAIL_USERNAME=
MAIL_PASSWORD=

STRIPE_KEY=
STRIPE_SECRET=
STRIPE_WEBHOOK_SECRET=

BILLING_CURRENCY=USD
TRIAL_DAYS=14
SUPPORT_EMAIL=
```

> **`.env.example` is committed to git with empty values.** New developers copy it to `.env` and fill in real values.

```bash
cp .env.example .env
php artisan key:generate
```

---

## Theory: `config/` Files

Each config file is **a PHP file that reads from `.env` and provides a structured array:**

```php
// config/services.php
return [
    'stripe' => [
        'key' => env('STRIPE_KEY'),
        'secret' => env('STRIPE_SECRET'),
        'webhook_secret' => env('STRIPE_WEBHOOK_SECRET'),
    ],
    
    'mailgun' => [
        'domain' => env('MAILGUN_DOMAIN'),
        'secret' => env('MAILGUN_SECRET'),
    ],
];
```

Use in code:

```php
config('services.stripe.secret');      // 'sk_test_xxxxxxxx'
config('services.stripe.webhook_secret'); // 'whsec_xxxxxxxx'
```

> **Always use `config()`, never `env()`, in your application code.** Reason: `php artisan config:cache` caches config files at boot for performance. If you `env()` in code, you get the cached value, not the live one. **This is a classic bug.**

The only place to use `env()` is **inside config files themselves.**

---

## Theory: Environments (`APP_ENV`)

Laravel recognizes these environments:

- `local` — your laptop
- `development` — alias for local
- `staging` — pre-production
- `production` — live users
- `testing` — running PHPUnit

### Behavior by environment

```php
// In a service provider
public function boot(): void
{
    if (app()->environment('local')) {
        // Show debug info, query log, whoops errors
    }
    
    if (app()->environment('production')) {
        // Enforce HTTPS, disable debug bar
    }
    
    if (app()->environment(['staging', 'production'])) {
        // Same behavior on both
    }
}
```

### Detecting environment in code

```php
if (app()->environment('production')) {
    // critical alert code path
}
```

> **Senior rule:** feature flags and environment-specific behavior are usually better than `if (app()->environment())`. Use config:

```php
// config/features.php
return [
    'new_billing_ui' => env('FEATURE_NEW_BILLING_UI', false),
    'maintenance_mode' => env('APP_MAINTENANCE_MODE', false),
];

// In code
if (config('features.new_billing_ui')) { ... }
```

### In Blade

```blade
@env('local')
    <div class="debug-info">Running locally</div>
@endenv

@production
    <!-- Only shown in production -->
    <script>analytics.init()</script>
@endproduction
```

---

## Theory: `php artisan env` — See Your Environment

```bash
php artisan env
```

Shows: `The application environment is: local`

Plus other useful commands:

```bash
php artisan about       # Show app config summary
php artisan config:show cache.default  # Show one config value
```

---

## Theory: Config Caching (Production Optimization)

In production, config files get cached for performance:

```bash
php artisan config:cache
```

This combines all `config/*.php` files into a single `bootstrap/cache/config.php`. Subsequent requests skip loading dozens of files.

> **Critical:** when you change `.env` in production, you must run `php artisan config:cache` again, or Laravel will use old values.

```bash
php artisan config:clear    # Remove the cached config
php artisan config:cache    # Re-cache (production deploy step)
```

> **Senior deploy script always includes this.**

---

## Theory: Secrets Management (Beyond `.env`)

`.env` is fine for small projects but **not ideal for secrets in production** because:

- It's on every developer machine (in some setups)
- It's in your CI logs sometimes
- It's plain text on the server

Production-grade alternatives:

### For Laravel Forge / VPS hosts

Use **environment variables** at the OS level. Your `.env` is empty for sensitive values, but `$_SERVER` has them set by the host.

```bash
# On the server (one-time)
echo "STRIPE_SECRET=sk_live_xxxxx" >> /etc/environment
```

### For Kubernetes / Docker

Use **Kubernetes Secrets** or **Docker secrets**, mounted as env vars.

### For AWS

Use **AWS Secrets Manager** or **SSM Parameter Store.** Laravel has a `envoyer-aws-secrets` package or custom provider to fetch at boot.

```php
// AppServiceProvider boot()
public function boot(): void
{
    if (app()->environment('production')) {
        config([
            'services.stripe.secret' => \AWS::getSecretValue('stripe_secret'),
        ]);
    }
}
```

### For Laravel Vapor (serverless on AWS)

Vapor has built-in secrets management. Add via `vapor.yml`:

```yaml
environments:
  production:
    secrets:
      STRIPE_SECRET: $STRIPE_SECRET_LIVE
```

> **The principle:** secrets should be encrypted at rest, fetched on demand, and never in your codebase. `.env` works for solo/small teams but isn't enterprise-grade.

---

## Theory: 12-Factor App Principles

The configuration approach in Laravel follows [The Twelve-Factor App](https://12factor.net/), a famous methodology for SaaS apps:

> "Store config in the environment."

This means:

✅ Code is identical across environments (no `if dev` branches in code)
✅ Configuration is environment variables
✅ Different envs override via env vars, not code changes

> **Senior rule:** if you find yourself writing environment-specific code branches, ask "should this be a config value instead?"

---

## Theory: Multi-Tenant App Settings

In your SaaS, **per-tenant settings** are different from environment settings:

| Type | Example | Where it lives |
|---|---|---|
| App-wide config | `BILLING_CURRENCY=USD` | `.env` |
| Per-tenant setting | "ABC Coaching uses BDT" | `tenants.settings` JSON column |
| User-specific | "Bilal's language is Bangla" | `users.settings` JSON column |

You access them like this:

```php
// config('app.billing_currency')  — app-wide
// tenant('billing_currency')      — per-tenant (you write a helper)
```

A senior writes helpers for these patterns:

```php
// app/helpers.php (or in AppServiceProvider boot())
if (!function_exists('tenant_setting')) {
    function tenant_setting(string $key, mixed $default = null): mixed
    {
        $tenant = app('current_tenant');
        return data_get($tenant?->settings, $key, $default);
    }
}
```

Use:

```php
$currency = tenant_setting('currency', 'USD');
$language = tenant_setting('language', 'en');
```

---

## Why It Matters

Configuration mistakes are **the #1 cause of "works in dev, breaks in prod"** bugs. A senior:

- Commits `.env.example`, never `.env`
- Uses `config()` in code, not `env()`
- Caches config in production
- Plans for secrets management upgrade as they grow
- Documents required env vars in their README

---

## Lab: Add Custom Configuration

### Step 1 — Add custom config keys to `.env`

Edit `G:\php-laravel\coaching-saas\.env`:

```bash
BILLING_CURRENCY=BDT
TRIAL_DAYS=14
SUPPORT_EMAIL=support@coaching-saas.com
FEATURE_NEW_BILLING_UI=false
MAX_STUDENTS_PER_TENANT=500
```

### Step 2 — Create a custom config file

Create `config/coaching.php`:

```php
<?php
return [
    'billing' => [
        'currency' => env('BILLING_CURRENCY', 'USD'),
        'trial_days' => (int) env('TRIAL_DAYS', 14),
    ],
    
    'support_email' => env('SUPPORT_EMAIL', 'support@example.com'),
    
    'features' => [
        'new_billing_ui' => filter_var(env('FEATURE_NEW_BILLING_UI', false), FILTER_VALIDATE_BOOLEAN),
        'attendance_module' => filter_var(env('FEATURE_ATTENDANCE_MODULE', true), FILTER_VALIDATE_BOOLEAN),
    ],
    
    'limits' => [
        'max_students_per_tenant' => (int) env('MAX_STUDENTS_PER_TENANT', 1000),
        'max_teachers_per_tenant' => (int) env('MAX_TEACHERS_PER_TENANT', 100),
    ],
];
```

### Step 3 — Use it in code

```php
// Anywhere in your app
config('coaching.billing.currency');         // "BDT"
config('coaching.billing.trial_days');        // 14
config('coaching.support_email');             // "support@coaching-saas.com"
config('coaching.features.new_billing_ui');   // false
config('coaching.limits.max_students_per_tenant'); // 500
```

In a Blade view:

```blade
@if (config('coaching.features.new_billing_ui'))
    <div>New billing UI is enabled!</div>
@endif

<p>Need help? Email {{ config('coaching.support_email') }}</p>
```

In a controller:

```php
public function store(StoreTenantRequest $request): RedirectResponse
{
    $trialDays = config('coaching.billing.trial_days');
    
    $tenant = Tenant::create([
        // ...
        'trial_ends_at' => now()->addDays($trialDays),
    ]);
    
    return redirect()->route('tenants.show', $tenant);
}
```

### Step 4 — Verify `php artisan config:show`

```bash
php artisan config:show coaching
```

Shows the full `coaching` config tree.

### Step 5 — Update `.env.example`

Edit `.env.example`:

```bash
BILLING_CURRENCY=USD
TRIAL_DAYS=14
SUPPORT_EMAIL=
FEATURE_NEW_BILLING_UI=false
MAX_STUDENTS_PER_TENANT=500
```

> **Commit `.env.example` so the next developer knows what to fill in.**

### Step 6 — Verify `.env` is gitignored

```bash
cd G:\php-laravel\coaching-saas
cat .gitignore | grep .env
```

You should see `.env` in there. If not, **add it.**

### Step 7 — Try config caching

```bash
php artisan config:cache
```

Notice: a new `bootstrap/cache/config.php` file is created.

Change a value in `.env` and run:

```bash
php artisan config:clear
php artisan tinker --execute="echo config('coaching.billing.currency');"
```

The new value is now picked up.

> **This is the production behavior:** cache config for performance, clear when env changes.

---

## Quiz

**Q1.** Why is `env()` forbidden in your application code, but allowed inside config files? What's the specific bug it causes?

**Q2.** A junior commits their `.env` file to git. The team lead is furious. Walk through the recovery: what files need editing, what commands need running, and why is "just delete the file from git history" not enough?

**Q3.** You have a feature flag `FEATURE_NEW_BILLING_UI`. A junior reads `env('FEATURE_NEW_BILLING_UI')` in a Blade file. After `php artisan config:cache`, the flag stops responding to changes. Why? What's the fix?

**Q4.** Your app grows past 50 customers. You need real secrets management. Compare your options: Laravel Forge env vars vs AWS Secrets Manager vs Kubernetes Secrets. Which would you pick for a small VPS, which for AWS, which for K8s?

**Q5.** (Stretch) The 12-factor principle says "store config in the environment." Your SaaS has per-tenant settings (e.g. each institution sets its own currency). Are these environment config or application data? How do you reconcile?

---

## What You Now Have

✅ Understanding of the 3-layer config model
✅ `.env` / `.env.example` / `config/*.php` discipline
✅ Environment detection and feature flags
✅ Config caching for production
✅ Per-tenant settings pattern
✅ Plan for evolving secrets management as you scale

## What's Next

**Phase 2 complete!** You now understand Laravel's foundation.

**Phase 3: Architecture** begins with the most important architectural decision for your SaaS — **multi-tenancy**. How do you isolate customer data so Tenant A can never see Tenant B's records?

📖 **[Phase 3 starts: Lesson 24 — Multi-Tenancy: Shared Schema Strategy](phase-3-architecture/24-multi-tenancy-shared-schema-strategy.md)**
