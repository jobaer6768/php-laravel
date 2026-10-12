# Lesson 45 — Deploying on Laravel Forge

> **Phase 6: Deploy & Scale** | You'll be here: ~40 min

## The Senior's Question

You picked Forge. Now you need to actually deploy. The junior SSHes in, runs `git pull`, crosses fingers. The senior has a **deploy script, a queue worker supervisor, zero-downtime deploys, and a rollback plan.** Let me show you the workflow.

---

## Theory: What Forge Does for You

When you provision a server through Forge, you get:

- ✅ Ubuntu 22.04 LTS
- ✅ PHP 8.3 + extensions (mbstring, pdo_mysql, etc.)
- ✅ Nginx (web server)
- ✅ MySQL 8 or PostgreSQL
- ✅ Redis 7
- ✅ Composer
- ✅ Node.js
- ✅ Let's Encrypt SSL (free, auto-renewing)
- ✅ UFW firewall (only 22, 80, 443 open)
- ✅ Fail2ban (SSH brute force protection)
- ✅ Daily security updates

> **All in 5-10 minutes.** The senior uses Forge because the alternative is spending 4 hours setting this up manually.

---

## Theory: Provisioning a Server

### Step 1 — Sign up for Forge

https://forge.laravel.com — $12/mo Basic plan.

### Step 2 — Connect a server provider

Click "Create Server" → choose provider:

- DigitalOcean (easiest for beginners)
- AWS
- Hetzner (cheapest)
- Linode
- Vultr

Authorize Forge to create servers on your behalf.

### Step 3 — Create the server

- **Name:** `coaching-saas-prod`
- **Size:** 4GB / 2 vCPU / 80GB SSD ($24/mo on DO)
- **Region:** closest to your users
- **PHP version:** 8.3
- **Database:** MySQL 8

Click "Create Server". Wait 5-10 minutes. Forge does the rest.

> **Senior tip:** name servers clearly. `prod`, `staging`. **Don't** name them `myserver1`, `test123`. Future-you will be confused.

---

## Theory: Adding Your Site

In Forge → Servers → your server → Sites → Create Site:

- **Domain:** `coaching-saas.com` (and `www.coaching-saas.com`)
- **Project type:** Laravel
- **Web directory:** `/public` (Laravel default)

Forge creates the Nginx config, sets up SSL.

> **The web directory must be `/public`.** This is the only public-facing folder. Config, app code, and storage are NOT web-accessible.

---

## Theory: Connecting to GitHub

Forge → Site → Git Repository → Connect:

- **Repository:** `yourname/coaching-saas`
- **Branch:** `main`
- **Composer:** install (production)
- **Node:** build (production)

> **Forge pulls your repo on deploy.** Set the branch to `main` so only tested code is deployed.

---

## Theory: Environment Variables in Forge

Forge → Site → Environment:

```
APP_NAME=Coaching SaaS
APP_ENV=production
APP_DEBUG=false
APP_KEY=base64:xxx
APP_URL=https://coaching-saas.com

DB_DATABASE=coaching_prod
DB_USERNAME=forge
DB_PASSWORD=xxx

REDIS_PASSWORD=xxx

POSTMARK_TOKEN=xxx
STRIPE_KEY=pk_live_xxx
STRIPE_SECRET=sk_live_xxx
...
```

> **Click "Generate APP_KEY".** Forge generates a secure 32-char key. **Don't skip this.**

> **Senior rule:** click "Hide on dashboard" for secrets.** Anyone with Forge access shouldn't see them. Stays in your password manager.

---

## Theory: The Deploy Script

This is the heart of it. Forge → Site → Deploy Script:

```bash
cd /home/forge/coaching-saas.com

git pull origin main

composer install --no-dev --no-interaction --prefer-dist --optimize-autoloader

# NPM (if you have frontend assets)
npm ci --no-audit
npm run build

# Laravel optimizations
php artisan migrate --force
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache

# Restart queue workers
php artisan queue:restart

# Done
echo "Deploy complete"
```

> **Each line matters.** Without `migrate`, your schema is stale. Without `config:cache`, requests run slowly. Without `queue:restart`, workers run old code.

### Why these specific commands?

| Command | Why |
|---|---|
| `composer install --no-dev` | Skip dev packages (PHPUnit, debugbar) |
| `--optimize-autoloader` | Build classmap, faster boot |
| `migrate --force` | Apply DB changes (in prod) |
| `config:cache` | Combine all configs into one cached file |
| `route:cache` | Combine all routes into one cached file |
| `view:cache` | Compile Blade templates |
| `event:cache` | Cache event-listener mappings |
| `queue:restart` | Tell workers to restart after current job |

> **Cache files = faster requests.** Without them, Laravel reads every config file on every request.

---

## Theory: Zero-Downtime Deploys

Forge does zero-downtime by default:

1. New code is deployed to a **new release folder** (`releases/20231012_143022/`)
2. The `current` symlink is updated to point to the new folder
3. Old folder is kept for rollback
4. **Requests in flight on the old version complete normally**
5. New requests hit the new version

> **Senior rule:** zero-downtime means deploys don't disrupt users. Deploy at 5 PM Friday. They won't even notice.

To rollback:

```bash
cd /home/forge/coaching-saas.com
./deploy.sh rollback
# Or click "Rollback to previous release" in Forge UI
```

---

## Theory: Queue Workers (Supervisor)

Forge → Server → Queue → Create Worker:

- **Connection:** `redis`
- **Queue:** `default`
- **Processes:** 2 (more if you have heavy jobs)
- **Timeout:** 60 seconds
- **Sleep:** 5 seconds (when no jobs)
- **Max tries:** 3

> **Supervisor** keeps the worker process alive. If it dies, Supervisor restarts it.

> **Senior rule:** always have at least 1 queue worker. Email sending, payment processing, anything queued MUST have a worker. Without it, jobs pile up forever.

### Multiple queues

If you have high-priority and low-priority jobs:

```php
SendReceiptEmail::dispatch($payment)->onQueue('high');
GenerateReport::dispatch($tenant)->onQueue('low');
```

Forge workers:

- Worker 1: queue `high`, 4 processes
- Worker 2: queue `low`, 1 process

> **High-priority jobs always have workers.** Low-priority waits.

---

## Theory: Scheduled Tasks (Cron)

Forge → Server → Scheduled Jobs:

```bash
* * * * * cd /home/forge/coaching-saas.com && php artisan schedule:run >> /dev/null 2>&1
```

This runs Laravel's scheduler every minute. Define schedules in `routes/console.php` (Laravel 11+) or `app/Console/Kernel.php`:

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('invoices:generate-monthly')->monthlyOn(1, '00:00');
Schedule::command('attendance:digest')->weeklyOn(1, '08:00');
Schedule::command('subscriptions:check-trials')->daily();
```

> **Senior rule:** scheduled tasks run via cron, not in the request cycle.** A scheduled task triggered by a user request is a recipe for "it works for me but not in prod."

---

## Theory: SSL with Let's Encrypt

Forge → Site → SSL → Let's Encrypt:

- **Domain:** `coaching-saas.com`, `www.coaching-saas.com`
- Click "Obtain Certificate"

Forge runs `certbot`, gets a cert, configures Nginx. **Auto-renews every 60 days.**

> **Always use HTTPS in production.** Mixed content (HTTP images on HTTPS page) breaks features and looks unprofessional.

Force HTTPS in `AppServiceProvider`:

```php
public function boot()
{
    if (app()->environment('production')) {
        \URL::forceScheme('https');
    }
}
```

---

## Theory: Database Backups

Forge → Server → Backups → Create:

- **Database:** `coaching_prod`
- **Schedule:** Daily at 3 AM
- **Storage:** Forge's own, or copy to S3

> **Daily backups are not enough.** Also do:
> - 7-day retention on the server
> - 30-day retention in S3
> - Test restore quarterly

Test restore (in staging, not prod!):

```bash
# In staging
mysql coaching_staging < backup-2023-10-12.sql
```

---

## Theory: Monitoring and Alerts

Forge → Server → Monitoring:

- **CPU > 80% for 5 min** → email alert
- **Memory > 90%** → email alert
- **Disk > 90%** → email alert
- **MySQL down** → email alert

> **Senior rule:** monitor the four golden signals — CPU, memory, disk, network.** And database specifically. If MySQL dies, your app dies.

For application-level monitoring, use Sentry (covered earlier).

---

## Theory: SSH Access (For Real Debugging)

Forge gives you SSH access:

```bash
ssh forge@coaching-saas.com
cd /home/forge/coaching-saas.com
```

Useful commands:

```bash
# View logs
tail -f storage/logs/laravel.log

# Check queue size
php artisan queue:size

# Check failed jobs
php artisan queue:failed

# Restart queue workers
php artisan queue:restart

# Tinker
php artisan tinker

# Check app status
php artisan about
```

> **Senior tip:** install `pail` for real-time log tailing:** `php artisan pail --filter=error`.

---

## Theory: Common Forge Pitfalls

### 1. `.env` not set

Deploy fails: "No application encryption key has been specified."

Fix: Set `APP_KEY` in Forge → Environment.

### 2. Storage not writable

Logs fail: "Permission denied on storage/logs/"

Fix:

```bash
chmod -R 775 storage bootstrap/cache
chown -R forge:forge storage bootstrap/cache
```

### 3. Storage symlink broken

User uploads return 404.

Fix:

```bash
php artisan storage:link
```

> **In Forge, this happens automatically on first deploy if configured.**

### 4. Queue workers using old code

Jobs fail mysteriously after deploy.

Fix: ensure deploy script has `php artisan queue:restart`.

### 5. Migrations fail

Fix: check database connection. `DB_USERNAME=forge` (Forge's default user).

---

## Theory: The Deploy Workflow

```
1. Develop on feature branch
   ↓ git push origin feature/x
2. Open PR to main
   ↓ CI runs tests
3. Merge to main
   ↓
4. Auto-deploy to staging (via webhook)
   ↓ Manual verification
5. Click "Deploy" in Forge for production
   ↓
6. Monitor for 15 minutes
   ↓
7. Done. Roll back if needed.
```

> **Senior rule:** the deploy button is sacred.** Click it intentionally. Watch the output. **Then monitor for a while.**

---

## Theory: Deployment Notifications

Slack notification on deploy:

`config/logging.php`:

```php
'slack' => [
    'driver' => 'slack',
    'url' => env('LOG_SLACK_WEBHOOK_URL'),
    'level' => 'info',
],
```

`.env`:

```env
LOG_STACK=single,slack
LOG_SLACK_WEBHOOK_URL=https://hooks.slack.com/services/xxx
```

Now every `Log::info('deploy.complete', ...)` goes to Slack.

> **Senior rule:** deploy to Slack, not just a single developer.** The whole team knows production changed.

---

## Theory: First Deploy Checklist

Before clicking "Deploy" for the first time:

- [ ] `.env` set in Forge (especially `APP_KEY`)
- [ ] `APP_DEBUG=false` in production
- [ ] Database created and credentials correct
- [ ] `php artisan migrate` runs without errors
- [ ] SSL certificate obtained
- [ ] Queue worker running
- [ ] Scheduled tasks configured
- [ ] Backups configured
- [ ] Monitoring alerts configured
- [ ] Sentry DSN set
- [ ] Email sending tested
- [ ] File uploads tested
- [ ] Stripe webhook URL updated to production
- [ ] Domain DNS pointed at server

> **The senior checks each item.** A junior skips and finds out at 3 AM.

---

## Why It Matters

Deploying is the **moment of truth.** Everything before is theory. After deploy, you find out what works in production.

Get deploys right:
- They're fast (under 2 minutes)
- They're zero-downtime
- They're observable (logs, alerts)
- They're reversible (rollback in 30 seconds)
- They happen often (multiple per day)

Get them wrong:
- Downtime (customers angry)
- Bugs (fired emails, lost payments)
- Stress (5 PM Friday deploys, no one wants to)

> **A senior makes deploys boring. Boring = reliable = ship often.**

---

## Lab: Walk Through a Production Deploy

> **You don't need to actually deploy to follow this.** The lab is the senior's mental checklist.

### Step 1 — Server requirements

- 4GB RAM, 2 vCPU, 80GB SSD minimum
- Ubuntu 22.04 LTS
- SSH key from your machine

### Step 2 — Forge setup

```
Server: coaching-saas-prod (DO 4GB $24/mo)
Site: coaching-saas.com + www.coaching-saas.com
PHP: 8.3
Database: coaching_prod (MySQL 8)
```

### Step 3 — Environment variables

Critical vars (minimum):

```env
APP_NAME="Coaching SaaS"
APP_ENV=production
APP_DEBUG=false
APP_KEY=base64:GENERATED_BY_FORGE
APP_URL=https://coaching-saas.com

LOG_CHANNEL=stack
LOG_STACK=single
LOG_LEVEL=info

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=coaching_prod
DB_USERNAME=forge
DB_PASSWORD=GENERATED_BY_FORGE

CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis
FILESYSTEM_DISK=s3

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=GENERATED_BY_FORGE
REDIS_PORT=6379

MAIL_MAILER=postmark
POSTMARK_TOKEN=YOUR_PRODUCTION_TOKEN

STRIPE_KEY=pk_live_xxx
STRIPE_SECRET=sk_live_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx

AWS_ACCESS_KEY_ID=xxx
AWS_SECRET_ACCESS_KEY=xxx
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=coaching-saas-prod

SCOUT_DRIVER=meilisearch
MEILISEARCH_HOST=https://xxx.meilisearch.io
MEILISEARCH_KEY=xxx

SENTRY_LARAVEL_DSN=https://xxx@sentry.io/123
```

### Step 4 — Deploy script

```bash
cd /home/forge/coaching-saas.com

git pull origin main

composer install --no-dev --no-interaction --prefer-dist --optimize-autoloader

npm ci --no-audit
npm run build

php artisan migrate --force --no-interaction
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache

php artisan queue:restart
php artisan storage:link

echo "Deploy complete at $(date)"
```

### Step 5 — Queue workers

Forge → Server → Queue Workers:

- Connection: redis
- Queue: default,high,emails
- Processes: 2
- Timeout: 90
- Max Tries: 3

### Step 6 — Scheduled tasks

```bash
* * * * * cd /home/forge/coaching-saas.com && php artisan schedule:run >> /dev/null 2>&1
```

### Step 7 — Backups

- Database: daily 3 AM
- Retention: 7 days on server, 30 days in S3
- Test restore quarterly

### Step 8 — Monitoring

- CPU > 80% → email
- Memory > 90% → email
- Disk > 90% → email

### Step 9 — SSL

- Let's Encrypt auto-issue
- Auto-renew (Forge handles)
- Force HTTPS via `URL::forceScheme('https')`

### Step 10 — First deploy click

1. Push code to `main`
2. Forge auto-deploys (or click "Deploy Now")
3. Watch the log output
4. Check the site: `https://coaching-saas.com`
5. Check the queue: `https://coaching-saas.com/horizon` (if using Horizon)
6. Check Sentry for errors
7. Test a critical flow (sign up, payment, etc.)
8. Done

### Step 11 — Rollback plan

If deploy fails:

1. Forge UI: "Rollback to previous release"
2. SSH: `./deploy.sh rollback`
3. Manual: edit the `current` symlink

```bash
cd /home/forge/coaching-saas.com
ln -nfs releases/20231011_120000 current
```

### Step 12 — Post-deploy monitoring

For 15-30 minutes after deploy:

```bash
# SSH and watch
tail -f storage/logs/laravel.log
php artisan queue:failed  # should be 0
php artisan tinker
>>> User::latest()->first()  # did anyone sign up?
```

---

## Quiz

**Q1.** Why run `php artisan config:cache route:cache view:cache event:cache` on every deploy? What does it do?

**Q2.** Your deploy script doesn't have `php artisan queue:restart`. After deploy, jobs start failing. Walk through the cause.

**Q3.** Forge does zero-downtime deploys. How? What does "release folder" mean?

**Q4.** You click "Deploy" and the page shows a 500 error. Walk through your debugging steps (in order).

**Q5.** (Stretch) You have a critical security fix. You can't wait for the normal PR + test + deploy cycle. Walk through an emergency hotfix deploy.

---

## What You Now Have

✅ Forge server provisioning
✅ Site setup with SSL
✅ GitHub integration
✅ Production `.env` discipline
✅ Zero-downtime deploy script
✅ Queue worker setup
✅ Backups and monitoring
✅ Rollback plan

## What's Next

Lesson 46 covers **Docker for PHP applications** — containerize your app, the standard way to ship consistent environments from dev to prod.

📖 **[Lesson 46 — Docker for PHP Applications](46-docker-for-php-applications.md)**
