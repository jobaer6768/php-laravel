# Lesson 05 — Install Database and Run Your First Migration

> **Phase 0: Environment** | You'll be here: ~45 min

## The Senior's Question

You have a Laravel project with a migration waiting. You need a database. There are three real choices: MySQL, PostgreSQL, or SQLite. A senior doesn't pick randomly. They pick based on production needs, dev speed, and team familiarity. Let's pick correctly, and ship.

---

## Theory: The Database Decision

Your SaaS will store:
- Tenant data (institutions, plans, billing)
- Students, parents, teachers
- Classes, batches, attendance, marks
- Invoices, payments, refunds
- Audit logs

This is **relational data with strong consistency needs.** You need:
- ACID transactions (a payment that deducts from one account and credits another must both succeed or both fail)
- Foreign keys (a `student.class_id` must reference a real class)
- Indexes (searching 100k students must be fast)
- Mature tooling (backups, replication)

### The Three Real Choices

| DB | Best for | Pros | Cons |
|---|---|---|---|
| **MySQL** | Most web apps, what most hosting supports | Universal, fast, well-known, Laravel's default | Slightly weaker JSON support than Postgres |
| **PostgreSQL** | Apps needing advanced SQL (CTEs, window functions, JSONB) | Most powerful SQL, true JSONB, full-text search, better data integrity | Slightly slower for simple reads, less universal on cheap hosting |
| **SQLite** | Local dev, small apps, prototypes | Zero-install, single file, fast | No concurrent writes, no real network access, not for multi-server prod |

### What We'll Choose (and Why)

**For your SaaS: MySQL 8 in production. SQLite in early local dev.**

Reasoning:

1. **Most Laravel tutorials and packages assume MySQL.** This reduces friction.
2. **Cheap shared hosting (your first 100 customers) supports MySQL.** Postgres requires more expensive hosts.
3. **Laravel's first-party packages (Cashier, Scout, etc.) are tested against MySQL first.**
4. **SQLite in local dev** is a senior trick. Zero install, zero config, file-based, perfect for development. We'll use it until we need MySQL-specific features (real JSON column operations, full-text search).

> **When to switch to Postgres:** if you find yourself wanting JSONB instead of JSON, full-text search without an external service, or window functions for reports. That's Lesson 49+ material.

---

## Theory: Connection Strings and `.env`

Laravel reads DB credentials from `.env`:

```bash
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=coaching_saas
DB_USERNAME=root
DB_PASSWORD=
```

For SQLite, it becomes:

```bash
DB_CONNECTION=sqlite
```

That's it. No host, no port, no credentials. The file lives at `database/database.sqlite`.

---

## Lab Part A: Start with SQLite (Fastest Path to "It Works")

### Step 1 — Switch to SQLite in `.env`

Open `G:\php-laravel\coaching-saas\.env` in your editor.

Find the DB section and change it to:

```bash
DB_CONNECTION=sqlite
```

(Comment out the other DB lines with `#`, or just leave them — they won't be used.)

### Step 2 — Create the SQLite file

```bash
cd G:\php-laravel\coaching-saas
touch database/database.sqlite
```

> On Windows PowerShell, if `touch` doesn't work, use:
> ```powershell
> New-Item -Path database/database.sqlite -ItemType File
> ```

This creates an empty file. SQLite treats any file as a database. Empty file = empty database.

### Step 3 — Run the migration

```bash
php artisan migrate
```

Expected output:

```
INFO  Preparing database.

Creating migration table ...................................... DONE
0001_01_01_000000_create_users_table ........................... DONE
0001_01_01_000001_create_cache_table .......................... DONE
0001_01_01_000002_create_jobs_table ........................... DONE
2026_xx_xx_xxxxxx_create_tenants_table ........................ DONE
```

> **Notice the order.** Laravel's built-in migrations run first (because their timestamps are earlier), then your `tenants` migration runs last. The timestamp prefix is the sort order.

### Step 4 — Verify the table exists

```bash
php artisan db:show
```

You'll see a list of all tables in the database. You should see:

- `cache`, `cache_locks`
- `failed_jobs`, `job_batches`, `jobs`
- `migrations` (this tracks what has been applied)
- `password_reset_tokens`
- `sessions`
- `tenants` ← yours
- `users`

### Step 5 — Inspect the `tenants` table

```bash
php artisan db:table tenants
```

This prints the columns of the `tenants` table — names, types, defaults, indexes. Senior engineers use this all the time to confirm schema.

### Step 6 — Meet Tinker

```bash
php artisan tinker
```

You should see:

```
Psy Shell v0.x.x (PHP 8.3.x — cli) by Justin Hileman
>>
```

This is a **PHP REPL with your entire app loaded.** You can interact with your database, models, and code live.

Try this:

```php
>>> Tenant::count();
=> 0

>>> $t = new Tenant;
=> App\Models\Tenant {#5000}

>>> $t->name = 'ABC Coaching';
=> "ABC Coaching"

>>> $t->slug = 'abc-coaching';
=> "abc-coaching"

>>> $t->email = 'admin@abc.com';
=> "admin@abc.com"

>>> $t->type = 'coaching';
=> "coaching"

>>> $t->save();
=> true

>>> Tenant::count();
=> 1

>>> Tenant::first();
=> App\Models\Tenant {
     id: 1,
     name: "ABC Coaching",
     slug: "abc-coaching",
     ...
   }

>>> $t->settings = ['timezone' => 'Asia/Dhaka', 'currency' => 'BDT'];
=> [...]

>>> $t->save();
=> true

>>> $t->fresh()->settings;
=> [...]

>>> exit
```

> **Senior note:** `Tenant::first()` returns a *cached* model — `$t->settings` would still be null because the in-memory object hasn't been refreshed. `$t->fresh()` reloads from the DB. **Always** remember this distinction.

### Step 7 — Confirm with SQL

Still in tinker:

```php
>>> \DB::select('SELECT * FROM tenants');
=> [
     {#5075
       +"id": 1,
       +"name": "ABC Coaching",
       +"slug": "abc-coaching",
       +"settings": "{\"timezone\":\"Asia\/Dhaka\",\"currency\":\"BDT\"}",
       +"created_at": "2026-...",
       +"updated_at": "2026-...",
       +"deleted_at": null,
     },
   ]
```

Notice the `settings` field is stored as a JSON **string** in the database. But when accessed via the model (`$t->settings`), the cast converts it to an array. **This is the magic of `$casts`.**

Exit tinker:

```php
>>> exit
```

### Step 8 — Try a rollback

```bash
php artisan migrate:rollback
```

This will undo the last batch. Your `tenants` table will be dropped, but the Laravel defaults stay. To roll back everything:

```bash
php artisan migrate:rollback --step=10
```

> **Senior warning:** `php artisan migrate:fresh` drops ALL tables and re-runs everything. **Only use in dev.** Never on a database with real data.

Now re-apply:

```bash
php artisan migrate
```

---

## Lab Part B: Switch to MySQL (For Production Parity)

SQLite is great for dev, but for production realism, let's also set up MySQL. This way you can switch with one env line.

### Step 1 — Install MySQL

Pick one:

**Option A: Herd (already installed)** — Herd includes MySQL option. Check:
```bash
herd services
```

If MySQL is listed but not running:
```bash
herd services start mysql
```

**Option B: Standalone MySQL install**
```powershell
winget install Oracle.MySQL
```

Or download from [dev.mysql.com/downloads/installer](https://dev.mysql.com/downloads/installer/).

**Option C: Docker** (if you have Docker Desktop)
```bash
docker run --name coaching-mysql -e MYSQL_ROOT_PASSWORD=secret -p 3306:3306 -d mysql:8
```

For now, **Option A (Herd) is easiest** if available.

### Step 2 — Create the database

Connect to MySQL:
```bash
mysql -u root -p
```

In the MySQL prompt:
```sql
CREATE DATABASE coaching_saas CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
SHOW DATABASES;
EXIT;
```

> **Why `utf8mb4` and not `utf8`?** MySQL's `utf8` is a 3-byte encoding that *can't store emoji or some Asian characters*. `utf8mb4` is true 4-byte UTF-8. **Always use `utf8mb4`** for a SaaS with user-generated content.

### Step 3 — Update `.env` to use MySQL

```bash
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=coaching_saas
DB_USERNAME=root
DB_PASSWORD=your_password_here
```

### Step 4 — Test the connection

```bash
php artisan db:show
```

If it lists tables, you're connected. If it errors, read the error — usually wrong credentials or wrong port.

### Step 5 — Re-run migrations on MySQL

```bash
php artisan migrate:fresh
```

The `fresh` command drops everything and re-applies. **Dev only.**

### Step 6 — Confirm with tinker

```bash
php artisan tinker
>>> Tenant::count();
=> 0
>>> exit
```

---

## Theory: How the Connection Switching Works

Laravel reads `config/database.php`, which reads `.env`. The default connection is determined by `DB_CONNECTION`. You can define **multiple connections** and switch per query, but for now one connection is fine.

```php
// config/database.php (excerpt)
'connections' => [
    'sqlite' => [
        'driver' => 'sqlite',
        'database' => env('DB_DATABASE', database_path('database.sqlite')),
    ],
    'mysql' => [
        'driver' => 'mysql',
        'host' => env('DB_HOST', '127.0.0.1'),
        'port' => env('DB_PORT', '3306'),
        'database' => env('DB_DATABASE', 'forge'),
        'username' => env('DB_USERNAME', 'forge'),
        'password' => env('DB_PASSWORD', ''),
        'charset' => 'utf8mb4',
        'collation' => 'utf8mb4_unicode_ci',
    ],
],
```

> **Senior reading:** `env('DB_DATABASE', database_path('database.sqlite'))` — the second argument is a default value. If `.env` doesn't have `DB_DATABASE`, it uses the SQLite path. This is why SQLite "just works" even if you forget to set it.

---

## Theory: A Note About Tinker

`php artisan tinker` is powered by **PsySH**, an independent PHP REPL. You'll use it constantly for:

- Debugging ("does this query return what I think?")
- Quick data fixes ("update this one row's wrong value")
- Trying out new code before writing a test
- Exploring your models

**Make tinker a habit.** A senior reaches for tinker before adding `dd()` (dump-and-die) to their code.

---

## Common Errors

| Error | Meaning | Fix |
|---|---|---|
| `SQLSTATE[HY000] [2002] No connection could be made` | MySQL not running | Start MySQL service or Docker container |
| `Access denied for user 'root'@'localhost'` | Wrong password | Check `.env` `DB_PASSWORD` |
| `Unknown database 'coaching_saas'` | DB not created | `CREATE DATABASE coaching_saas;` in MySQL |
| `Class 'PDO' not found` | PHP `pdo_mysql` extension missing | Reinstall PHP with the extension (Lesson 01) |
| `SQLSTATE[HY000] [1118] Row size too large` | Schema has too many columns or wrong row format | Add `ROW_FORMAT=DYNAMIC` or split the table |

---

## Quiz

**Q1.** Why is `utf8mb4` mandatory for a coaching SaaS, but you might not have thought about it in your MERN apps? (Hint: think about student names, parent messages, fee receipts.)

**Q2.** A junior runs `php artisan migrate:fresh` on the production server during a bug. What did they just do? Why is this catastrophic?

**Q3.** In tinker, you do `$t = Tenant::first();` then `Tenant::where('id', 1)->update(['name' => 'New']);` then `echo $t->name;`. What does it print, and why? (Hint: this is the difference between a model instance and a database row.)

**Q4.** Why would you choose SQLite for local dev but MySQL for production? List two specific reasons.

**Q5.** (Stretch) You commit your `.env` file by accident. The team lead wants you to fix it. Walk through the recovery: what files do you edit, what do you tell the team, and how do you prevent this in the future?

---

## What You Now Have

✅ A working database connection (SQLite for dev, MySQL ready)
✅ A real `tenants` table with proper columns, types, indexes
✅ A `Tenant` model with `$fillable` and `$casts`
✅ First data created via Tinker
✅ Understanding of the migration system

## What's Next

You have a working Laravel app with a database. **Phase 0 is done.**

In **Phase 1**, we switch from environment setup to **PHP itself**. You've been writing JavaScript/TypeScript. PHP is different in ways that matter. Eight lessons will bring you from "I can read PHP" to "I think in PHP."

📖 **[Phase 1 starts: Lesson 06 — PHP Syntax for MERN Developers](phase-1-php-fundamentals/06-php-syntax-for-mern-developers.md)**
