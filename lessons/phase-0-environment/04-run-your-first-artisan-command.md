# Lesson 04 — Run Your First Artisan Command

> **Phase 0: Environment** | You'll be here: ~30 min

## The Senior's Question

You have a Laravel project. Now you need to change the database schema. A junior opens phpMyAdmin and starts clicking. A senior types one command. What is Artisan, and why is it the most-used tool in a Laravel dev's day?

---

## Theory: What Is Artisan?

Artisan is Laravel's **command-line interface**. The word comes from "artisan" — a skilled craftsperson. Laravel chose it deliberately: the CLI is your workshop.

The `artisan` file at the project root is a tiny stub. When you run `php artisan something`, here's what happens:

```
$ php artisan make:model Student
      ↓
PHP loads bootstrap/app.php
      ↓
Laravel scans registered commands
      ↓
Finds MakeModelCommand
      ↓
Runs it: creates app/Models/Student.php
      ↓
Prints "Model created successfully."
```

### Why Artisan Exists (And npm Scripts Don't Cover This)

In MERN, you write your own scripts in `package.json`. There is no canonical way to "create a new model." You copy-paste from an old project. In Laravel, **every file you might create has a command**:

| You want to create... | Command |
|---|---|
| Eloquent model | `make:model` |
| Controller | `make:controller` |
| Migration | `make:migration` |
| Form Request | `make:request` |
| Middleware | `make:middleware` |
| Job (queueable) | `make:job` |
| Event & Listener | `make:event`, `make:listener` |
| Test | `make:test` |
| Seeder | `make:seeder` |
| Factory | `make:factory` |
| Policy (auth) | `make:policy` |
| Service | `make:provider` |
| Component (Blade) | `make:component` |

This is **massive productivity**. The command knows:
- The right namespace
- The right folder
- The right base class to extend
- The right boilerplate to fill in

You just write your code on top.

## Theory: The Two Kinds of Artisan Commands

### 1. **Generators** (`make:*`)
Create files. They don't touch the database. They just scaffold a starting point.

### 2. **Operational commands**
Do things: run migrations, clear cache, run queue worker, start dev server, run tests.

| Command | What it does |
|---|---|
| `php artisan serve` | Start dev server |
| `php artisan migrate` | Run pending migrations |
| `php artisan migrate:rollback` | Undo last migration batch |
| `php artisan migrate:fresh` | Drop everything, re-migrate (dev only!) |
| `php artisan db:seed` | Run seeders |
| `php artisan tinker` | Interactive PHP REPL with your app loaded |
| `php artisan test` | Run tests |
| `php artisan cache:clear` | Flush cache |
| `php artisan config:clear` | Clear compiled config |
| `php artisan route:list` | See all registered routes |
| `php artisan queue:work` | Run a queue worker |
| `php artisan storage:link` | Create symlink for uploaded files |

> **Senior rule:** when in doubt, run `php artisan list`. The full list is in your terminal. Always.

## Why It Matters

Artisan is the **difference between knowing Laravel and being fast at Laravel.** A senior runs:

```bash
php artisan make:model Student -a
```

The `-a` flag (all) generates:
- `app/Models/Student.php` (model)
- `database/migrations/...create_students_table.php` (migration)
- `database/factories/StudentFactory.php` (test factory)
- `database/seeders/StudentSeeder.php` (seeder)
- `app/Http/Controllers/StudentController.php` (controller)
- `app/Http/Requests/StoreStudentRequest.php` (form request)
- `app/Http/Requests/UpdateStudentRequest.php` (form request)
- `app/Policies/StudentPolicy.php` (auth policy)

Eight files, correctly named, correctly placed, with correct boilerplate. In one second. A junior would copy-paste and forget the policy.

> **This is the productivity curve we're climbing.** The tools do the boring work. You think about the business logic.

---

## Theory: Migrations — The Senior Way to Change Schema

A **migration** is a PHP file that describes a change to your database schema. It has two methods:

```php
public function up(): void {
    // Apply this change
}

public function down(): void {
    // Reverse this change
}
```

This is **version control for your database**. Git tracks your code. Migrations track your schema.

### Why This Matters For a Multi-Tenant SaaS

When you have 50 coaching centers on your platform, you cannot "edit the database." You run a migration. Every instance gets the change. The change is reviewable in a PR. If it breaks, you roll back.

### The Flow

```bash
# 1. Create the migration file
php artisan make:migration create_students_table

# 2. Edit the file (define the schema)

# 3. Apply the migration
php artisan migrate

# 4. Oops, wrong, undo it
php artisan migrate:rollback

# 5. Re-edit, re-apply
php artisan migrate
```

> **Senior rule:** you can also do `php artisan make:model Student -m` to create the model AND migration in one shot. The `-m` is a flag.

### What a Migration Looks Like

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('students', function (Blueprint $table) {
            $table->id();
            $table->foreignId('institution_id')->constrained()->cascadeOnDelete();
            $table->string('name');
            $table->string('email');
            $table->date('date_of_birth')->nullable();
            $table->timestamps();
            
            // An email should be unique within an institution, not globally
            $table->unique(['institution_id', 'email']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('students');
    }
};
```

Notice the `unique(['institution_id', 'email'])` — a **composite unique index**. We'll use this pattern heavily in Phase 3 for multi-tenancy.

---

## Lab: Your First Migration

### Step 1 — Explore default migrations

In your Laravel project, list what's already there:

```bash
ls database/migrations
```

You'll see Laravel's built-in tables:
- `users` — for the people who log in
- `password_reset_tokens` — for password recovery
- `sessions` — for session storage
- `cache` and `cache_locks` — for the cache driver
- `jobs`, `job_batches`, `failed_jobs` — for queues

These get created when you run `migrate`. We'll talk about each in its own lesson.

### Step 2 — Run `php artisan list`

```bash
php artisan list
```

This is the full command reference. Skim it. Notice sections like:
- `make:` — generators
- `migrate:` — migration operations
- `db:` — database operations
- `test:` — testing
- `queue:` — queue management
- `route:` — route introspection

### Step 3 — Create a model with migration

We're going to make a model for the `tenants` table. In Phase 3, this becomes the foundation of multi-tenancy. For now, just see how it's done.

```bash
php artisan make:model Tenant -m
```

Expected:
```
Model created successfully.
Created Migration: 2026_xx_xx_xxxxxx_create_tenants_table
```

> The migration filename has a timestamp prefix. This is how Laravel knows what order to run migrations in. **Never rename a migration** after it's been committed, even to fix the order — write a new one instead.

### Step 4 — Look at what got generated

```bash
# The model
cat app/Models/Tenant.php
```

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Tenant extends Model
{
    use HasFactory;
}
```

Empty class. That's normal. The senior adds fields and relationships.

```bash
# The migration
ls database/migrations/
cat database/migrations/2026_*_create_tenants_table.php
```

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('tenants', function (Blueprint $table) {
            $table->id();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('tenants');
    }
};
```

> Note the `return new class extends Migration` syntax — this is PHP 8's "anonymous class." Laravel uses it to avoid needing a named class for every migration. Cleaner.

### Step 5 — Edit the migration to add real fields

Open `database/migrations/2026_*_create_tenants_table.php` and replace its content with:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('tenants', function (Blueprint $table) {
            $table->id();
            $table->string('name');                 // "ABC Coaching Center"
            $table->string('slug')->unique();       // "abc-coaching" (for URLs)
            $table->string('email');
            $table->string('phone')->nullable();
            $table->enum('type', ['coaching', 'school', 'kindergarten'])->default('coaching');
            $table->json('settings')->nullable();   // free-form config
            $table->timestamp('trial_ends_at')->nullable();
            $table->timestamp('subscribed_at')->nullable();
            $table->boolean('is_active')->default(true);
            $table->timestamps();
            $table->softDeletes();                  // adds `deleted_at` column
            
            $table->index('type');
            $table->index('is_active');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('tenants');
    }
};
```

Read every line. Notice:

| Line | Why |
|---|---|
| `$table->string('slug')->unique();` | `string` is `VARCHAR(255)`. `unique()` adds a unique index. |
| `$table->enum('type', [...])` | Native DB enum, not a string with validation. |
| `$table->json('settings')` | Native JSON column in MySQL 5.7+. We use it for free-form config. |
| `$table->softDeletes()` | Adds `deleted_at`. Records aren't deleted, just hidden. **Critical** for SaaS — you don't actually delete tenant data. |
| `$table->index('type')` | Manually adds an index for query speed. |

### Step 6 — Add fields to the model

Open `app/Models/Tenant.php` and replace it with:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class Tenant extends Model
{
    use HasFactory, SoftDeletes;
    
    protected $fillable = [
        'name', 'slug', 'email', 'phone', 'type',
        'settings', 'trial_ends_at', 'subscribed_at', 'is_active',
    ];
    
    protected $casts = [
        'settings' => 'array',          // JSON column → PHP array automatically
        'trial_ends_at' => 'datetime',
        'subscribed_at' => 'datetime',
        'is_active' => 'boolean',
    ];
}
```

> **Senior reading:** the `$casts` array is *huge*. It's how Eloquent auto-converts DB types to PHP types. `datetime` columns become Carbon instances (PHP's date library). `json` columns become arrays. `boolean` becomes true/false. **Never** read a raw string and parse it yourself when you can cast.

### Step 7 — Try to run the migration (it will fail — that's the lesson)

```bash
php artisan migrate
```

You'll see an error like:

```
SQLSTATE[HY000] [1049] Unknown database 'coaching_saas'
```

**Read the error.** It tells you the database doesn't exist. This is the next lesson — we need to set up MySQL (or SQLite for now). We'll fix this in Lesson 05.

But — you've now:

✅ Used Artisan generators
✅ Edited a migration
✅ Edited a model
✅ Understood the `$casts` system
✅ Hit your first real error (and read it)

### Step 8 — Cleanup: rollback (still no DB, but see the command)

```bash
php artisan migrate:rollback
```

This will say "Nothing to rollback" because we never successfully migrated. That's fine. The command exists for when you do.

---

## Common Errors and Their Real Meanings

| Error | Real meaning |
|---|---|
| `Class "Tenant" not found` | Composer autoloader hasn't picked up the new file. Run `composer dump-autoload`. |
| `Base table or view not found: tenants` | Migration hasn't run. Run `php artisan migrate`. |
| `SQLSTATE[42S01]: Base table already exists` | Migration was already applied. Use `migrate:rollback` or `migrate:fresh` (dev only). |
| `Unknown database 'coaching_saas'` | DB itself doesn't exist. We'll fix in Lesson 05. |
| `Access denied for user 'root'@'localhost'` | DB credentials in `.env` are wrong. Fix `.env`. |

> **Senior rule:** **read the error message carefully.** It almost always tells you exactly what's wrong. Stack Overflow is for when you don't understand the error, not for when you haven't read it.

---

## Quiz

**Q1.** Why does Laravel ship `make:*` commands instead of expecting you to copy-paste files from old projects? (Hint: it's not just about speed.)

**Q2.** What's the difference between `$table->string('email')` and `$table->string('email')->unique()`? What does the second add at the database level, and what query performance impact does it have?

**Q3.** You accidentally run a migration that drops a column with important data. You want to undo it. You run `php artisan migrate:rollback`. What does this actually do, and when is it NOT enough to save you?

**Q4.** Why is `protected $casts = ['settings' => 'array']` so important? What goes wrong if you forget it and try to read `$tenant->settings`?

**Q5.** (Stretch) You `git pull` a teammate's branch. They added a new migration but you don't see the change in your local DB. Walk through exactly what commands you need to run, in order, and why each one is needed.

---

## What You Now Have

✅ Familiarity with Artisan
✅ A `Tenant` model and migration (not yet applied)
✅ Understanding of `$fillable`, `$casts`, `softDeletes`
✅ First real error, read carefully

## What's Next

In Lesson 05, we set up the database — SQLite for now (zero install), MySQL later — and finally run `migrate` successfully. You'll see your `tenants` table come to life and the `php artisan tinker` REPL for the first time.

📖 **[Lesson 05 — Install Database and Run Your First Migration](05-install-database-and-first-migration.md)**
