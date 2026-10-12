# Lesson 43 — Environments: Dev, Staging, Production

> **Phase 6: Deploy & Scale** | You'll be here: ~30 min

## The Senior's Question

"It works on my machine." **Four words that have killed more SaaS companies than bad code.** The senior knows that the difference between dev, staging, and production isn't just config files — it's a discipline of **mirror images, isolation, and gatekeeping.** Let me show you how to set it up right.

---

## Theory: The Three Environments

```
Local (your laptop)
   ↓ git push
Staging (server)         ← tests the deploy, mirrors prod
   ↓ manual approval
Production (servers)     ← serves real users
```

### Local — your laptop

- **Purpose:** write code, run tests
- **Audience:** you
- **Data:** fake / fixtures
- **Cost:** $0
- **Risk:** zero

### Staging — pre-prod server

- **Purpose:** test deploys, integration testing
- **Audience:** QA, you, sometimes customers
- **Data:** anonymized production data
- **Cost:** ~$50-200/mo
- **Risk:** low

### Production — live

- **Purpose:** serve users, make money
- **Audience:** customers
- **Data:** real
- **Cost:** $200-2000+/mo
- **Risk:** $$$$ — every bug costs money

> **Senior rule:** staging must mirror production. If you skip staging, you'll find bugs in production. If you skip production-like staging, you'll find them in production too.

---

## Theory: The `.env` Discipline

`.env` files hold environment-specific config. They should NEVER be committed.

`.gitignore`:

```
.env
.env.*
!.env.example
```

> **`.env.example` is the template.** It has all keys, no real values. Committed. Shared.

`.env.example`:

```env
APP_NAME="Coaching SaaS"
APP_ENV=production
APP_KEY=
APP_DEBUG=false
APP_URL=https://coaching-saas.com

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=
DB_USERNAME=
DB_PASSWORD=

CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis

MAIL_MAILER=postmark
POSTMARK_TOKEN=

STRIPE_KEY=
STRIPE_SECRET=

AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_BUCKET=
```

> **The senior's `.env.example`** is exhaustive. No "I forgot to set this in production" surprises.

---

## Theory: Configuration That Doesn't Belong in `.env`

> **Senior rule:** if it's not secret and not environment-specific, put it in `config/`, not `.env`.

```php
// config/coaching.php
return [
    'trial_days' => env('COACHING_TRIAL_DAYS', 14),
    'max_students_per_tenant' => env('COACHING_MAX_STUDENTS', 1000),
    'currency' => env('COACHING_CURRENCY', 'USD'),
    'support_email' => env('COACHING_SUPPORT_EMAIL', 'support@coaching-saas.com'),
];
```

Why:
- Easy to test (override in tests)
- Documented (config file has comments)
- Type-safe
- Centralized

> **Config is code. Env vars are environment overrides.**

---

## Theory: APP_ENV and APP_DEBUG

```env
# Development
APP_ENV=local
APP_DEBUG=true

# Production
APP_ENV=production
APP_DEBUG=false
```

### What `APP_DEBUG=true` does

- Shows detailed error pages with stack traces
- Exposes `.env` variables in error pages
- Disables some caching
- **NEVER enable in production. Never. Ever.**

> **A junior once pushed APP_DEBUG=true to prod.** The error page exposed `DB_PASSWORD=xxx`. **Senior rule: false. Always false. In prod.**

Check on every deploy:

```bash
php artisan env
# Should output: Current application environment: production
```

---

## Theory: Database Per Environment

| Environment | Database |
|---|---|
| Local | `coaching_local` (or `:memory:` SQLite) |
| Staging | `coaching_staging` |
| Production | `coaching_prod` |

> **NEVER point your local `.env` at the production database.** A `migrate:fresh` will nuke it. **Senior's #1 rule.**

The database name is in `.env`. The user is in `.env`. The password is in `.env`. **None of this is in your code.**

---

## Theory: Seeding Per Environment

`database/seeders/DatabaseSeeder.php`:

```php
public function run(): void
{
    if (app()->environment('local')) {
        // Lots of fake data
        Tenant::factory()->count(10)->create()->each(function ($tenant) {
            User::factory()->for($tenant)->count(5)->create();
            Student::factory()->for($tenant)->count(50)->create();
        });
    }
    
    if (app()->environment('production')) {
        // Just one tenant (you, the owner)
        Tenant::factory()->create([
            'name' => 'Coaching SaaS HQ',
            'email' => 'admin@coaching-saas.com',
        ]);
    }
}
```

> **In production, seeding creates your admin tenant.** No fake students. No fake data. Real users create real data.

---

## Theory: The Branching Strategy

### Option 1: Trunk-based (senior's default for solo/small teams)

```
main ← always deployable
   │
feature branches ← short-lived (1-3 days)
```

- All work in feature branches
- Merge to main via PR
- Deploy from main

### Option 2: GitFlow (for larger teams)

```
main         ← production
develop      ← integration
   ├── feature/students
   ├── feature/fees
   └── hotfix/...
```

- `main` is production
- `develop` is integration
- Features branch from develop
- Hotfixes branch from main

> **Senior rule:** if you're a team of 1-5, use trunk. GitFlow is for teams of 10+.

---

## Theory: Staging Should Mirror Production

| | Staging | Production |
|---|---|---|
| OS | Ubuntu 22.04 | Ubuntu 22.04 |
| PHP | 8.3 | 8.3 |
| MySQL | 8.0 | 8.0 |
| Redis | 7 | 7 |
| App | Same code, soon-to-be-deployed | Latest deployed |
| Data | Anonymized copy | Real |
| Load | 0 | Real traffic |

> **If staging is on PHP 8.2 and production is 8.3, you haven't tested what you'll deploy.** Senior makes them identical except for data and traffic.

---

## Theory: Anonymized Production Data in Staging

```bash
# Dump production
mysqldump --single-transaction coaching_prod > dump.sql

# Anonymize (manual or with a tool)
php artisan anonymize:dump dump.sql > staging_dump.sql

# Load into staging
mysql coaching_staging < staging_dump.sql
```

`app/Console/Commands/AnonymizeDump.php`:

```php
public function handle()
{
    DB::table('users')->update([
        'name' => DB::raw("CONCAT('User ', id)"),
        'email' => DB::raw("CONCAT('user', id, '@example.com')"),
        'password' => bcrypt('password'),
    ]);
    
    DB::table('students')->update([
        'name' => DB::raw("CONCAT('Student ', id)"),
        'email' => DB::raw("CONCAT('student', id, '@example.com')"),
    ]);
    
    $this->info('Anonymized.');
}
```

> **Never copy real PII to staging.** GDPR, CCPA, HIPAA violations. Anonymize, always.

---

## Theory: Environment Variables in Production

In production, env vars come from your hosting platform:

- **Forge** — UI fields for each var
- **Vapor** — project config
- **Docker** — `docker run -e KEY=value` or compose file
- **Kubernetes** — secrets/configmaps
- **Raw VPS** — `.env` file (chmod 600)

> **Senior rule:** env vars are secrets. They live in the host, not the repo. They are rotated without code changes.**

---

## Theory: Secrets Management (Beyond `.env`)

For highly sensitive secrets (DB master password, encryption keys), use:

- **AWS Secrets Manager** — encrypted, audited, rotated
- **HashiCorp Vault** — overkill for most
- **Doppler** — simple SaaS
- **Infisical** — open source alternative

In Laravel, fetch secrets at boot:

```php
// config/database.php
'mysql' => [
    'host' => env('DB_HOST'),
    'password' => env('DB_PASSWORD'),
],
```

> **In a secret-managed setup, the env var resolves to a secret reference, not the actual value.** The platform fetches it at runtime.

---

## Theory: Feature Flags Per Environment

```php
// config/features.php
return [
    'new_dashboard' => [
        'local' => true,
        'staging' => true,
        'production' => env('FEATURE_NEW_DASHBOARD', false),
    ],
];
```

Use:

```php
if (config('features.new_dashboard.' . app()->environment())) {
    return view('dashboard.v2');
}

return view('dashboard.v1');
```

Or use a service like **LaunchDarkly** / **PostHog** for sophisticated flag management.

> **Senior rule:** feature flags let you deploy dark and turn on safely. Roll out to 10% of users, then 50%, then 100%. **No "all at once" deploys.**

---

## Theory: The `php artisan env:check` Pattern

Build a command that asserts the env is correct:

```php
public function handle()
{
    $errors = [];
    
    if (app()->environment('production')) {
        if (config('app.debug')) {
            $errors[] = 'APP_DEBUG is true in production!';
        }
        
        if (config('app.key') === 'SomeRandomString') {
            $errors[] = 'APP_KEY not set properly';
        }
        
        if (config('cache.default') === 'file') {
            $errors[] = 'Cache driver is file (slow)';
        }
    }
    
    if (empty(config('database.connections.mysql.password'))) {
        $errors[] = 'DB_PASSWORD is empty';
    }
    
    if ($errors) {
        $this->error('Environment check failed:');
        foreach ($errors as $error) {
            $this->line(' - ' . $error);
        }
        exit(1);
    }
    
    $this->info('Environment OK');
}
```

Run before every deploy:

```bash
php artisan env:check
```

---

## Why It Matters

- **Discipline:** same code, three configs, zero confusion
- **Safety:** a misconfig in prod gets caught in staging
- **Speed:** you deploy often because staging is reliable
- **Compliance:** GDPR, SOC 2, HIPAA all require env separation

> **A senior's environments are isolated, mirrored, and gated.** A junior's "I'll just push to prod" is how outages happen.

---

## Lab: Set Up Three Environments

### Step 1 — Create a clean `.env.example`

```bash
cd G:\php-laravel\coaching-saas
cp .env .env.example
# Edit .env.example to remove all real values
```

`G:\php-laravel\coaching-saas\.env.example`:

```env
APP_NAME="Coaching SaaS"
APP_ENV=production
APP_KEY=
APP_DEBUG=false
APP_URL=http://localhost

LOG_CHANNEL=stack
LOG_LEVEL=info

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=
DB_USERNAME=
DB_PASSWORD=

BROADCAST_DRIVER=log
CACHE_DRIVER=redis
FILESYSTEM_DISK=local
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

MAIL_MAILER=log
MAIL_FROM_ADDRESS="hello@example.com"
MAIL_FROM_NAME="${APP_NAME}"

POSTMARK_TOKEN=
STRIPE_KEY=
STRIPE_SECRET=
STRIPE_WEBHOOK_SECRET=

AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=

SCOUT_DRIVER=meilisearch
MEILISEARCH_HOST=
MEILISEARCH_KEY=

SENTRY_LARAVEL_DSN=
```

### Step 2 — Local `.env`

```env
APP_NAME="Coaching SaaS (Local)"
APP_ENV=local
APP_DEBUG=true
APP_URL=http://localhost:8000

DB_DATABASE=coaching_local

MAIL_MAILER=log  # or smtp to mailpit
CACHE_DRIVER=array  # or file
QUEUE_CONNECTION=sync
```

### Step 3 — Staging `.env`

```env
APP_NAME="Coaching SaaS (Staging)"
APP_ENV=staging
APP_DEBUG=true
APP_URL=https://staging.coaching-saas.com

DB_DATABASE=coaching_staging

MAIL_MAILER=postmark
POSTMARK_TOKEN=staging_token

CACHE_DRIVER=redis
QUEUE_CONNECTION=redis

SENTRY_LARAVEL_DSN=staging_dsn
```

### Step 4 — Production `.env`

```env
APP_NAME="Coaching SaaS"
APP_ENV=production
APP_DEBUG=false
APP_URL=https://coaching-saas.com

DB_DATABASE=coaching_prod

MAIL_MAILER=postmark
POSTMARK_TOKEN=production_token

CACHE_DRIVER=redis
QUEUE_CONNECTION=redis

SENTRY_LARAVEL_DSN=production_dsn
```

### Step 5 — Set up environment-specific seeders

```bash
php artisan make:seeder LocalSeeder
php artisan make:seeder ProductionSeeder
```

`database/seeders/LocalSeeder.php`:

```php
public function run(): void
{
    Tenant::factory()->count(5)->create()->each(function ($tenant) {
        $tenant->users()->createMany(
            User::factory()->count(3)->make()->toArray()
        );
        Student::factory()->for($tenant)->count(20)->create();
    });
}
```

In `DatabaseSeeder`:

```php
public function run(): void
{
    $this->call(match (app()->environment()) {
        'local' => LocalSeeder::class,
        'production' => ProductionSeeder::class,
        default => DatabaseSeeder::class,
    });
}
```

### Step 6 — Add the env check command

```bash
php artisan make:command EnvCheck
```

`app/Console/Commands/EnvCheck.php`:

```php
public function handle()
{
    $errors = $this->checkProduction();
    
    if ($errors) {
        foreach ($errors as $error) {
            $this->error($error);
        }
        return 1;
    }
    
    $this->info('Environment OK: ' . app()->environment());
    return 0;
}

private function checkProduction(): array
{
    $errors = [];
    
    if (app()->environment('production')) {
        if (config('app.debug')) {
            $errors[] = 'APP_DEBUG is true in production';
        }
        if (config('app.key') === '' || config('app.key') === null) {
            $errors[] = 'APP_KEY is not set';
        }
        if (config('cache.default') === 'database') {
            $errors[] = 'CACHE_DRIVER is database (slow)';
        }
    }
    
    return $errors;
}
```

### Step 7 — Set up branch protection on GitHub

In GitHub → Settings → Branches → Add rule:

```
Branch name pattern: main

☑ Require a pull request before merging
  ☑ Require approvals: 1
☑ Require status checks to pass before merging
  ☑ Require branches to be up to date
```

> **Nobody pushes directly to main.** Even you. **Senior discipline.**

### Step 8 — Document the deploy flow

`DEPLOY.md`:

```markdown
# Deploy Process

## To deploy to staging:
1. Push to `main`
2. CI runs tests + deploys to staging
3. Manually verify on https://staging.coaching-saas.com
4. Run smoke tests

## To deploy to production:
1. Click "Promote to production" in Forge (or run deploy script)
2. Monitor for 15 minutes
3. Roll back if any errors spike

## Rollback:
- Forge: one-click rollback to previous deploy
- Manual: `git checkout <previous-tag> && php artisan migrate:rollback`
```

---

## Quiz

**Q1.** Why is `APP_DEBUG=true` in production catastrophic? List 2 specific things it exposes.

**Q2.** Staging should mirror production. What's the ONE thing they should differ in?

**Q3.** You accidentally pushed your real Stripe secret to `.env.example` and committed. Walk through your remediation steps.

**Q4.** Why is putting `app.name = 'Coaching SaaS'` in `.env` wrong? Where should it go instead?

**Q5.** (Stretch) You want a feature to roll out to 10% of users, then 50%, then 100%. Walk through the implementation with feature flags.

---

## What You Now Have

✅ Three-environment discipline
✅ Clean `.env.example` template
✅ Env check command
✅ Branch protection
✅ Staging mirrors production
✅ Anonymized data pipeline

## What's Next

Lesson 44 covers **hosting choices** — Forge vs Vapor vs raw VPS, the trade-offs, and how to pick the right one for your coaching SaaS.

📖 **[Lesson 44 — Hosting Choices: Forge, Vapor, and Raw](44-hosting-choices-forge-vapor-and-raw.md)**
