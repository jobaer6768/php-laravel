# Lesson 22 — Artisan: The Swiss Army Knife of Laravel

> **Phase 2: Laravel Core** | You'll be here: ~30 min

## The Senior's Question

Why do senior Laravel devs reach for `php artisan` constantly, even for things a junior would write a script for? Because Artisan isn't just for framework commands — it's **your** command-line toolkit. Custom commands, scheduled tasks, and deployment scripts all live here.

---

## Theory: Two Kinds of Artisan Commands

Laravel ships with ~100 commands. You'll use them constantly:

| Category | Common commands |
|---|---|
| **Generators** | `make:controller`, `make:model`, `make:migration`, `make:request`, `make:job` |
| **Migrations** | `migrate`, `migrate:rollback`, `migrate:fresh`, `migrate:status` |
| **Database** | `db:seed`, `db:show`, `db:table` |
| **Routing** | `route:list` |
| **Cache** | `cache:clear`, `config:clear`, `view:clear`, `route:clear` |
| **Queue** | `queue:work`, `queue:listen`, `queue:failed` |
| **Testing** | `test` (runs PHPUnit/Pest) |
| **Tinker** | `tinker` (interactive shell) |
| **Maintenance** | `down` (enable maintenance mode), `up` (disable) |

But the **most powerful** thing Artisan does is **let you write your own commands.**

---

## Theory: Writing a Custom Command

```bash
php artisan make:command SendTenantReminders
```

```php
<?php
// app/Console/Commands/SendTenantReminders.php
namespace App\Console\Commands;

use App\Models\Tenant;
use App\Models\Subscription;
use Illuminate\Console\Command;
use Illuminate\Support\Facades\Mail;

class SendTenantReminders extends Command
{
    protected $signature = 'reminders:send
                            {--days=7 : Days before subscription expires to send reminder}
                            {--dry-run : Show what would be sent without sending}';
    
    protected $description = 'Send subscription renewal reminders to tenants';

    public function handle(): int
    {
        $days = (int) $this->option('days');
        $dryRun = $this->option('dry-run');
        
        $expiryDate = now()->addDays($days);
        
        $tenants = Tenant::query()
            ->where('is_active', true)
            ->whereNotNull('subscribed_at')
            ->where('subscribed_at', '<=', $expiryDate)
            ->get();
        
        $this->info("Found {$tenants->count()} tenants with subscriptions expiring in $days days");
        
        if ($dryRun) {
            foreach ($tenants as $tenant) {
                $this->line("  - {$tenant->name} ({$tenant->email}) expires {$tenant->subscribed_at}");
            }
            return self::SUCCESS;
        }
        
        $bar = $this->output->createProgressBar($tenants->count());
        $bar->start();
        
        $sent = 0;
        foreach ($tenants as $tenant) {
            Mail::to($tenant->email)->send(new \App\Mail\RenewalReminder($tenant, $days));
            $sent++;
            $bar->advance();
        }
        
        $bar->finish();
        $this->newLine();
        $this->info("Sent $sent reminder emails");
        
        return self::SUCCESS;
    }
}
```

Run it:

```bash
php artisan reminders:send --days=7 --dry-run
php artisan reminders:send --days=14
```

> **Notice:**
> - The signature defines the command + options + arguments
> - `handle()` does the work
> - `$this->info()`, `$this->error()`, `$this->warn()`, `$this->line()` for colored output
> - Return `self::SUCCESS` (0) or `self::FAILURE` (1)
> - Progress bars for long operations

---

## Theory: Arguments vs Options

```php
protected $signature = 'students:promote
                        {student : The student ID or email}    // argument (required)
                        {--to= : The class to promote to}      // option with value
                        {--force}                              // boolean flag
                        {--dry-run}';
```

```bash
php artisan students:promote aisha@x.com --to=11A
php artisan students:promote 42 --to=11A --force
php artisan students:promote 42 --dry-run
```

| Type | Syntax | Required? | Has value? |
|---|---|---|---|
| Argument | `{name}` | Yes, unless `{name?}` | Yes |
| Argument with default | `{name=default}` | No | Yes |
| Option | `{--name}` | No | No (flag) |
| Option with value | `{--name=}` or `{--name=default}` | No | Yes |
| Option short alias | `{--N|name=}` | No | Yes |

---

## Theory: Interactive Prompts

```php
public function handle(): int
{
    $name = $this->ask('What is the tenant name?');
    $type = $this->choice('What type of institution?', ['coaching', 'school', 'kindergarten'], 0);
    $confirmed = $this->confirm('Create this tenant?', true);
    
    if (!$confirmed) {
        $this->warn('Cancelled.');
        return self::SUCCESS;
    }
    
    Tenant::create([
        'name' => $name,
        'slug' => \Str::slug($name),
        'email' => 'admin@' . \Str::slug($name) . '.com',
        'type' => $type,
    ]);
    
    $this->info("Created tenant: $name");
    
    return self::SUCCESS;
}
```

> **Use these for:** one-off scripts where you don't want to build a CLI parser. **Senior move:** keep prompts minimal; pass via options instead so the command is scriptable.

---

## Theory: Scheduled Commands (The Senior's Most Powerful Use)

You can schedule commands to run automatically:

Edit `routes/console.php` (Laravel 11+):

```php
use Illuminate\Support\Facades\Schedule;

// Run daily at midnight
Schedule::command('reminders:send --days=7')
    ->dailyAt('00:00')
    ->withoutOverlapping()
    ->onOneServer();

// Run every 15 minutes
Schedule::command('queue:prune-failed --hours=48')
    ->everyFifteenMinutes();

// Weekly on Monday at 9am
Schedule::command('reports:weekly')
    ->weeklyOn(1, '09:00')
    ->emailOutputOnFailure('admin@example.com');
```

In production, you need ONE cron job to trigger Laravel's scheduler:

```
* * * * * cd /path/to/project && php artisan schedule:run >> /dev/null 2>&1
```

> **Why this matters:** instead of cron + bash scripts + maintenance nightmares, you define all recurring tasks in PHP. They're version-controlled, testable, and discoverable in one place.

---

## Theory: Useful Real Commands You'll Write

### Prune soft-deleted records older than 90 days

```php
// app/Console/Commands/PruneOldRecords.php
protected $signature = 'records:prune {--days=90}';

public function handle(): int
{
    $cutoff = now()->subDays($this->option('days'));
    
    $deleted = Student::onlyTrashed()
        ->where('deleted_at', '<', $cutoff)
        ->forceDelete();
    
    $this->info("Pruned $deleted old soft-deleted records");
    return self::SUCCESS;
}
```

### Import students from CSV

```php
// app/Console/Commands/ImportStudents.php
protected $signature = 'students:import {file : Path to CSV file}';

public function handle(): int
{
    $path = $this->argument('file');
    
    if (!file_exists($path)) {
        $this->error("File not found: $path");
        return self::FAILURE;
    }
    
    $handle = fopen($path, 'r');
    $header = fgetcsv($handle);  // first row: column names
    
    $count = 0;
    while (($row = fgetcsv($handle)) !== false) {
        $data = array_combine($header, $row);
        Student::create([
            'tenant_id' => $data['tenant_id'],
            'name'      => $data['name'],
            'email'     => $data['email'],
        ]);
        $count++;
    }
    fclose($handle);
    
    $this->info("Imported $count students");
    return self::SUCCESS;
}
```

```bash
php artisan students:import /path/to/file.csv
```

---

## Theory: Maintenance Mode

```bash
# Lock the app (returns 503 for non-whitelisted IPs)
php artisan down

# Lock with a secret bypass URL
php artisan down --secret=my-secret-123

# Custom render view
php artisan down --render="errors.maintenance"

# Bring back up
php artisan up
```

> **`--secret=my-secret-123` creates a cookie that bypasses maintenance mode.** You can access the live site at `https://example.com/my-secret-123` even when it's "down." Useful for testing a deployment before going public.

### Maintenance mode file

`php artisan down` creates `storage/framework/maintenance.php`. `php artisan up` deletes it. **Never commit this file.** It's already in `.gitignore`.

---

## Why It Matters

Artisan commands are **where you put business operations that don't fit in HTTP requests:**

- Daily/weekly/monthly jobs (reports, cleanup, notifications)
- One-off data migrations
- CLI tools for support staff
- Deployment scripts (cache clear, optimize, etc.)

A senior:
- Writes a command for every scheduled task
- Uses Artisan for one-off scripts instead of bash
- Makes commands idempotent (running twice = running once)
- Logs everything
- Uses the scheduler instead of crontab for recurring tasks

---

## Lab: Write a Subscription Reminder Command

### Step 1 — Create the command

```bash
cd G:\php-laravel\coaching-saas
php artisan make:command SendSubscriptionReminders
```

### Step 2 — Implement it

Edit `app/Console/Commands/SendSubscriptionReminders.php`:

```php
<?php
namespace App\Console\Commands;

use App\Models\Tenant;
use Illuminate\Console\Command;
use Illuminate\Support\Facades\Mail;
use Illuminate\Support\Facades\Log;

class SendSubscriptionReminders extends Command
{
    protected $signature = 'reminders:subscription
                            {--days=7 : Days before expiry to remind}
                            {--dry-run : Don\'t actually send emails}';

    protected $description = 'Send subscription renewal reminders to tenants';

    public function handle(): int
    {
        $days = (int) $this->option('days');
        $dryRun = $this->option('dry-run');
        
        $cutoff = now()->addDays($days);
        
        $tenants = Tenant::query()
            ->where('is_active', true)
            ->whereNotNull('subscribed_at')
            ->where('subscribed_at', '<=', $cutoff)
            ->orderBy('subscribed_at')
            ->get();
        
        if ($tenants->isEmpty()) {
            $this->info("No tenants need reminders.");
            return self::SUCCESS;
        }
        
        $this->info("Found {$tenants->count()} tenants to remind:");
        
        $sent = 0;
        $skipped = 0;
        
        $bar = $this->output->createProgressBar($tenants->count());
        $bar->start();
        
        foreach ($tenants as $tenant) {
            $daysLeft = (int) now()->diffInDays($tenant->subscribed_at);
            
            if ($dryRun) {
                $this->newLine();
                $this->line("  Would remind: {$tenant->name} ({$tenant->email}) - $daysLeft days left");
                $skipped++;
            } else {
                try {
                    Mail::raw(
                        "Hi {$tenant->name}, your subscription expires in $daysLeft days. Please renew.",
                        function ($message) use ($tenant) {
                            $message->to($tenant->email)
                                    ->subject('Subscription renewal reminder');
                        }
                    );
                    $sent++;
                    Log::info("Sent subscription reminder", ['tenant_id' => $tenant->id]);
                } catch (\Throwable $e) {
                    Log::error("Failed to send reminder", [
                        'tenant_id' => $tenant->id,
                        'error' => $e->getMessage(),
                    ]);
                    $this->newLine();
                    $this->error("Failed: {$tenant->name} - {$e->getMessage()}");
                }
            }
            
            $bar->advance();
        }
        
        $bar->finish();
        $this->newLine(2);
        
        if ($dryRun) {
            $this->info("DRY RUN: would have sent {$tenants->count()} reminders");
        } else {
            $this->info("Sent $sent reminders, 0 failed");
        }
        
        return self::SUCCESS;
    }
}
```

### Step 3 — Run it in dry-run mode

```bash
php artisan reminders:subscription --dry-run
```

Expected output:
```
Found 0 tenants to remind.
No tenants need reminders.
```

> **No tenants match** because no tenants have `subscribed_at` set. Let's set one.

### Step 4 — Set up a tenant that's "expiring" soon

In tinker:

```php
$tenant = \App\Models\Tenant::find(1);
$tenant->subscribed_at = now()->addDays(5);  // expires in 5 days
$tenant->save();
```

### Step 5 — Run it again

```bash
php artisan reminders:subscription --days=7 --dry-run
```

Expected:
```
Found 1 tenants to remind:
DRY RUN: would have sent 1 reminders
```

> **Without `--dry-run`, it sends the email.** Configure your mail driver first (use `MAIL_MAILER=log` in `.env` to write to log instead of actually sending).

### Step 6 — Schedule it

Edit `routes/console.php`:

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('reminders:subscription --days=7')
    ->dailyAt('09:00')
    ->withoutOverlapping()
    ->onOneServer();
```

> **In production, add the cron entry. In dev, you can run it manually with `php artisan schedule:run` to trigger all due scheduled commands.**

### Step 7 — See all scheduled tasks

```bash
php artisan schedule:list
```

You'll see your command, scheduled for daily at 9 AM.

### Step 8 — Test by manually running scheduled tasks

```bash
php artisan schedule:run
```

> **You just built and scheduled a production-ready business command.** Imagine how much harder this would be in MERN — a cron entry, a separate script file, no version control, no logs.

---

## Quiz

**Q1.** A junior writes a bash script to clear the cache nightly. A senior uses Laravel's scheduler. Why is the senior's approach better?

**Q2.** What's the difference between an argument and an option in a command signature? Give an example of each.

**Q3.** Why does `--dry-run` matter for a command like `reminders:subscription`? What's the senior's principle behind it?

**Q4.** You have a command that should run every 5 minutes. Walk through the production setup: what file do you edit, what cron entry do you add, and why is the cron entry only one line?

**Q5.** (Stretch) A senior says "every command should be idempotent." What does that mean, and how would you make `reminders:subscription` idempotent (so running it twice doesn't send the email twice)?

---

## What You Now Have

✅ Custom Artisan commands: signature, options, arguments, prompts
✅ Progress bars, colored output, dry-run support
✅ Scheduled commands via `routes/console.php`
✅ Real-world command patterns: imports, pruning, reminders
✅ Maintenance mode for deployments

## What's Next

Lesson 23 covers **configuration and environments** — the senior's discipline for managing settings across dev, staging, and production. Includes env-specific logic, feature flags, and secrets management.

📖 **[Lesson 23 — Configuration and Environments](23-configuration-and-environments.md)**
