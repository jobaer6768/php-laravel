# Lesson 18 — Migrations, Schema, and the Why of Versioned DB

> **Phase 2: Laravel Core** | You'll be here: ~40 min

## The Senior's Question

Your database schema will change 200 times before your SaaS hits 1,000 users. Each change must be safe, reversible, and reviewable. How do senior engineers evolve databases without breaking production?

---

## Theory: Why Migrations Exist

A migration is **a file that describes a change to your database schema**, written in PHP, version-controlled with Git, and runnable from the command line.

Compare to the alternatives:

| Approach | Problem |
|---|---|
| Edit the database by hand (phpMyAdmin) | "It works on my machine." Teammates don't have your changes. Production is different from local. |
| Share SQL files in Slack | No rollback, no review, no automation, easy to miss. |
| Use a database diff tool | You still need to apply it everywhere consistently. |
| **Migrations** | Single source of truth. Reviewable in PRs. Reversible. Automatic. |

> **Senior rule:** the database schema is code. Treat it like code. Migrations are how.

---

## Theory: The Migration File

```php
<?php
// database/migrations/2026_01_15_120000_create_students_table.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('students', function (Blueprint $table) {
            $table->id();
            $table->foreignId('tenant_id')->constrained()->cascadeOnDelete();
            $table->string('name');
            $table->string('email');
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('students');
    }
};
```

The two methods:

- `up()` — apply this change
- `down()` — reverse this change

> **Senior rule:** always write both. The `down()` lets you roll back when you need to.

---

## Theory: Running Migrations

```bash
# Run all pending migrations
php artisan migrate

# See what migrations have run
php artisan migrate:status

# Roll back the last batch
php artisan migrate:rollback

# Roll back the last 3 batches
php artisan migrate:rollback --step=3

# Drop everything and re-run from scratch (DEV ONLY)
php artisan migrate:fresh

# Drop, re-run, then seed (DEV ONLY)
php artisan migrate:fresh --seed
```

> **Never use `migrate:fresh` in production.** It drops all data.

---

## Theory: Schema Builder Methods (The Common Ones)

| Method | SQL type | Example |
|---|---|---|
| `$table->id()` | BIGINT UNSIGNED AUTO_INCREMENT PK | `$table->id();` |
| `$table->uuid('id')` | CHAR(36) | `$table->uuid('id');` |
| `$table->string('name')` | VARCHAR(255) | `$table->string('name', 100);` |
| `$table->text('description')` | TEXT | `$table->text('bio');` |
| `$table->integer('count')` | INT | `$table->integer('hits');` |
| `$table->bigInteger('count')` | BIGINT | `$table->bigInteger('views');` |
| `$table->boolean('is_active')` | TINYINT(1) | `$table->boolean('is_active');` |
| `$table->date('birthdate')` | DATE | `$table->date('dob');` |
| `$table->dateTime('published_at')` | DATETIME | `$table->dateTime('published_at');` |
| `$table->timestamp('verified_at')` | TIMESTAMP | `$table->timestamp('email_verified_at');` |
| `$table->decimal('amount', 8, 2)` | DECIMAL(8,2) | `$table->decimal('price', 8, 2);` |
| `$table->float('rating')` | FLOAT | rare, prefer decimal |
| `$table->json('settings')` | JSON | `$table->json('preferences');` |
| `$table->enum('status', ['a', 'b'])` | ENUM | `$table->enum('role', ['admin', 'user']);` |

### Modifiers

```php
$table->string('email')->unique();
$table->string('email')->nullable();
$table->string('email')->default('guest@x.com');
$table->integer('count')->default(0);
$table->string('name')->comment('Full name of the student');
$table->timestamp('verified_at')->useCurrent();
$table->string('slug')->index();
```

### Foreign keys

```php
$table->foreignId('tenant_id')
    ->constrained()                      // assumes 'tenants' table
    ->cascadeOnDelete();                 // if tenant deleted, delete this too

$table->foreignId('class_id')
    ->nullable()
    ->constrained('school_classes')      // explicit table
    ->nullOnDelete();                    // if class deleted, set class_id to null

$table->foreignId('user_id')
    ->constrained()
    ->restrictOnDelete();                // can't delete user if referenced
```

> **Senior rule:** always define foreign keys. They prevent orphaned data and let the DB enforce relationships.

### Indexes (the speed layer)

```php
$table->string('email')->unique();        // creates UNIQUE index
$table->string('slug')->index();          // creates INDEX

// Composite index
$table->index(['tenant_id', 'email']);

// Or as a unique composite
$table->unique(['tenant_id', 'email']);
```

> **When to add an index:** for any column you `WHERE` on frequently, `ORDER BY`, or `JOIN` on. Don't index every column — indexes have a write cost.

---

## Theory: Altering Existing Tables (The Senior's Real Workflow)

You don't drop and recreate. You **add** a migration:

```bash
php artisan make:migration add_phone_to_students_table
```

```php
public function up(): void
{
    Schema::table('students', function (Blueprint $table) {
        $table->string('phone')->nullable()->after('email');
    });
}

public function down(): void
{
    Schema::table('students', function (Blueprint $table) {
        $table->dropColumn('phone');
    });
}
```

Common alterations:

```php
// Add a column
$table->string('phone')->nullable();

// Drop a column
$table->dropColumn('phone');

// Rename a column
$table->renameColumn('name', 'full_name');

// Change column type (DBAL)
\DB::statement('ALTER TABLE students MODIFY name TEXT');

// Add an index
$table->index('email');

// Drop an index
$table->dropIndex(['email']);

// Add a foreign key
$table->foreign('class_id')->references('id')->on('school_classes');
```

> **Senior warning:** renaming or dropping a column with data is dangerous. **Always back up first.** Always test on staging.

---

## Theory: Data Migrations (Transforming Existing Data)

Sometimes you need to **change data, not structure.** E.g. split `name` into `first_name` and `last_name`.

```bash
php artisan make:migration split_name_into_first_and_last
```

```php
public function up(): void
{
    // Step 1: add the new columns
    Schema::table('students', function (Blueprint $table) {
        $table->string('first_name')->nullable()->after('id');
        $table->string('last_name')->nullable()->after('first_name');
    });

    // Step 2: backfill from the old column
    \DB::statement('
        UPDATE students
        SET
            first_name = SUBSTRING_INDEX(name, " ", 1),
            last_name = SUBSTRING_INDEX(name, " ", -1)
    ');

    // Step 3: drop the old column
    Schema::table('students', function (Blueprint $table) {
        $table->dropColumn('name');
    });
}

public function down(): void
{
    // Reverse the whole thing
    Schema::table('students', function (Blueprint $table) {
        $table->string('name')->nullable();
    });

    \DB::statement('
        UPDATE students
        SET name = CONCAT(first_name, " ", last_name)
    ');

    Schema::table('students', function (Blueprint $table) {
        $table->dropColumn(['first_name', 'last_name']);
    });
}
```

> **Senior rule:** never deploy a code change that depends on a column that hasn't been backfilled.** If your code reads `first_name` but the data migration hasn't run, the app breaks. Order matters.

---

## Theory: Safe Migrations in Production

This is the part that separates a junior from a senior.

### The Problem

You have 100,000 students. You need to add a `middle_name` column. You run the migration. It locks the table for 30 seconds. **The app is down for 30 seconds.**

### The Solution (Laravel's approach)

Laravel is smart about this. By default, it uses **online DDL** for MySQL 5.6+, which doesn't lock for most alterations. But:

- **Adding nullable columns** — instant
- **Adding columns with defaults** — instant in MySQL 8
- **Renaming columns** — can lock
- **Adding indexes** — can lock (but Laravel uses `ALGORITHM=INPLACE` when possible)

### Senior checklist for production migrations

1. **Test on staging first.** Always.
2. **Run during low traffic.** Late night, early morning.
3. **Have a rollback plan.** Know your `down()` works.
4. **Back up the database first.** Just in case.
5. **Monitor as you run.** Have the error log open.
6. **Don't change a column that's being read by running code.** Coordinate with code deploys.

### The "expand and contract" pattern

When you need to change a column, **don't change it in place.** Instead:

1. **Expand:** add a new column with the new shape
2. **Migrate:** backfill data from old to new
3. **Switch code:** read from new column
4. **Contract:** drop the old column (later, when safe)

This is how companies like GitHub evolve their schemas without downtime. **Master this pattern.**

---

## Theory: Foreign Key Cascades — The Critical Decision

```php
$table->foreignId('tenant_id')->constrained()->cascadeOnDelete();
$table->foreignId('class_id')->constrained()->nullOnDelete();
$table->foreignId('user_id')->constrained()->restrictOnDelete();
```

What do these mean?

| Action | Effect |
|---|---|
| `cascadeOnDelete()` | If parent is deleted, delete this row too |
| `cascadeOnUpdate()` | If parent's PK changes, update this FK |
| `nullOnDelete()` | If parent is deleted, set this FK to NULL |
| `restrictOnDelete()` | If this row exists, parent CANNOT be deleted |
| `noActionOnDelete()` | Same as restrict in MySQL |

> **Senior rule:** think hard about cascade vs restrict. Cascade deletes can wipe out tons of data accidentally. Restrict forces you to clean up first. For a SaaS, **default to restrict** for important business data. Use cascade only when the relationship is truly owned (e.g. `tenant → users`).

---

## Why It Matters

A senior:

- Writes migrations for every schema change
- Always implements `down()` 
- Tests on staging
- Uses foreign keys
- Adds indexes for queried columns
- Uses expand-and-contract for risky changes
- Has a backup before running

A junior:

- Edits the DB by hand
- Skips `down()`
- "It works on my machine"
- Discovers the missing index when production is slow
- Loses data because they didn't have a backup

---

## Lab: Add Real Schema

### Step 1 — Add `phone` to students

```bash
cd G:\php-laravel\coaching-saas
php artisan make:migration add_phone_to_students_table --table=students
```

Edit:

```php
public function up(): void
{
    Schema::table('students', function (Blueprint $table) {
        $table->string('phone', 20)->nullable()->after('email');
        $table->index('phone');  // we'll search by phone often
    });
}

public function down(): void
{
    Schema::table('students', function (Blueprint $table) {
        $table->dropIndex(['phone']);
        $table->dropColumn('phone');
    });
}
```

Run:

```bash
php artisan migrate
php artisan migrate:status
```

You should see the new migration listed as "Ran".

Roll back to test:

```bash
php artisan migrate:rollback
```

Re-run:

```bash
php artisan migrate
```

### Step 2 — Add an index to existing column

```bash
php artisan make:migration add_index_to_students_email --table=students
```

```php
public function up(): void
{
    Schema::table('students', function (Blueprint $table) {
        $table->index('email');
    });
}

public function down(): void
{
    Schema::table('students', function (Blueprint $table) {
        $table->dropIndex(['email']);
    });
}
```

### Step 3 — Build the exams table (a real new table)

```bash
php artisan make:model Exam -m
```

Migration:

```php
public function up(): void
{
    Schema::create('exams', function (Blueprint $table) {
        $table->id();
        $table->foreignId('tenant_id')->constrained()->cascadeOnDelete();
        $table->foreignId('class_id')->constrained('school_classes')->cascadeOnDelete();
        $table->string('name');              // "Midterm", "Final"
        $table->date('exam_date');
        $table->integer('total_marks')->default(100);
        $table->integer('pass_marks')->default(40);
        $table->timestamps();

        $table->index(['class_id', 'exam_date']);
    });
}
```

```bash
php artisan migrate
```

### Step 4 — Build the marks table

```bash
php artisan make:model Mark -m
```

```php
public function up(): void
{
    Schema::create('marks', function (Blueprint $table) {
        $table->id();
        $table->foreignId('tenant_id')->constrained()->cascadeOnDelete();
        $table->foreignId('exam_id')->constrained()->cascadeOnDelete();
        $table->foreignId('student_id')->constrained()->cascadeOnDelete();
        $table->decimal('marks_obtained', 6, 2);
        $table->decimal('percentage', 5, 2)->virtualAs('(marks_obtained / (SELECT total_marks FROM exams WHERE exams.id = marks.exam_id)) * 100');
        $table->string('grade', 2)->nullable();
        $table->text('remarks')->nullable();
        $table->timestamps();

        // A student can have only one row per exam
        $table->unique(['exam_id', 'student_id']);

        $table->index('student_id');
    });
}
```

> **Notice the virtual generated column `percentage`.** It calculates the percentage at the DB level — no need to compute in PHP. **Senior move.**

### Step 5 — Examine the actual SQL Laravel runs

For any migration, you can see the SQL:

```bash
php artisan migrate:install --pretend
php artisan migrate --pretend
```

`--pretend` doesn't actually run; it shows the SQL that would run. **Use this to verify before running in production.**

### Step 6 — Look at the database

```bash
php artisan db:show
php artisan db:table marks
```

You see the columns, types, and indexes.

---

## Quiz

**Q1.** Why is `php artisan migrate:fresh` dangerous? When is it safe?

**Q2.** You need to add a `middle_name` column to a `students` table with 100k rows. Walk through the expand-and-contract pattern step by step.

**Q3.** When would you choose `restrictOnDelete()` over `cascadeOnDelete()` for a foreign key? Give a concrete example from your SaaS.

**Q4.** You add an index to a column but queries are still slow. What's the next thing to check? (Hint: it's not always the missing index.)

**Q5.** (Stretch) Your migration has `public function up()` and `public function down()`. A teammate pushes a migration with an empty `down()`. What's the senior's view on this? What's the team rule you would propose?

---

## What You Now Have

✅ Migration fluency: create, alter, rollback, fresh, pretend
✅ Schema builder for all common types
✅ Index and foreign key strategy
✅ The expand-and-contract pattern for safe schema changes
✅ Production migration discipline

## What's Next

Lesson 19 covers **Form Requests and validation** — the senior's way to handle user input. No more inline validation. No more untrusted data reaching your models.

📖 **[Lesson 19 — Form Requests, Validation, and Authorization](19-form-requests-validation-and-authorization.md)**
