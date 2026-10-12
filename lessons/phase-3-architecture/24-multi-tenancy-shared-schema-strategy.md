# Lesson 24 — Multi-Tenancy: Shared Schema Strategy

> **Phase 3: Architecture** | You'll be here: ~45 min

## The Senior's Question

You sell your SaaS to 50 coaching centers. Each has students, teachers, fees, marks. **Tenant A must never see Tenant B's data.** This isn't a feature — it's a security boundary. How do you architect it correctly from day one?

---

## Theory: The Three Strategies

There are three ways to isolate tenants in a SaaS:

### 1. Separate Database Per Tenant

```
[App] → [DB Router] → [coaching_abc]   (Tenant ABC's data)
                      [coaching_xyz]   (Tenant XYZ's data)
                      [coaching_pqr]   (Tenant PQR's data)
```

| Pros | Cons |
|---|---|
| Maximum isolation (one DB per customer) | Expensive at scale (50 customers = 50 DBs) |
| Easy to back up per tenant | Migrations need to run on every DB |
| Easy to comply with data residency laws | Hard to do cross-tenant analytics |
| Easy to migrate a tenant off | Connection pooling is complex |

### 2. Schema Per Tenant (Postgres-specific)

```
[App] → [DB] → public.coaching (Tenant ABC)
                tenant_xyz.coaching
                tenant_pqr.coaching
```

| Pros | Cons |
|---|---|
| Good isolation | Postgres-only (no MySQL) |
| Cheaper than separate DBs | Still hard at scale |
| Cross-tenant analytics possible | Connection limits |

### 3. Shared Schema with `tenant_id` Column (what we'll use)

```
[App] → [DB] → students (all tenants share the table)
                - id, tenant_id, name, email
```

| Pros | Cons |
|---|---|
| Cheapest to operate (one DB) | Risk of data leaks if you forget to filter |
| Easiest to scale horizontally | One bad query can return cross-tenant data |
| Cross-tenant analytics trivial | Need rigorous scoping discipline |
| Single migration, single connection | Tenant data is colocated |

### What we choose and why

For your coaching SaaS, **shared schema with `tenant_id` column** is the right call:

- ✅ Coaching centers have similar data shapes (students, classes, marks)
- ✅ Up to ~5,000 tenants, you can run on one server
- ✅ Cost matters (you're pre-funding)
- ✅ Cross-tenant analytics is useful (e.g., "how many students across all tenants?")

> **The senior's caveat:** this strategy requires **discipline.** Every query must filter by `tenant_id`. We use **global scopes** to enforce this automatically. We also use **middleware** to set the current tenant. We never trust a request without verifying the tenant.

---

## Theory: The Mental Model

```
┌──────────────────────────────────────────────────────┐
│ Request: GET /students                                │
│ Host: abc.coaching-saas.com                          │
└──────────────────────┬───────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────┐
│ Middleware: ResolveTenant                            │
│   - Extract "abc" from subdomain                     │
│   - Find Tenant with slug="abc"                      │
│   - Store in app container as 'current_tenant'       │
└──────────────────────┬───────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────┐
│ Route: GET /students → StudentController@index       │
└──────────────────────┬───────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────┐
│ Controller:                                          │
│   $students = Student::all();   ← This is the magic  │
│   The global scope adds: WHERE tenant_id = 1         │
└──────────────────────┬───────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────┐
│ Database:                                            │
│   SELECT * FROM students WHERE tenant_id = 1;        │
└──────────────────────────────────────────────────────┘
```

**The global scope is the security boundary.** It runs on every query, automatically. You cannot accidentally forget to filter.

---

## Theory: The Schema

Every "tenant-owned" table has a `tenant_id` foreign key:

```php
// migration
Schema::create('students', function (Blueprint $table) {
    $table->id();
    $table->foreignId('tenant_id')->constrained()->cascadeOnDelete();
    $table->string('name');
    $table->string('email');
    $table->timestamps();

    // Composite unique: email is unique within a tenant, not globally
    $table->unique(['tenant_id', 'email']);
});
```

> **The composite unique is critical.** Aisha in ABC Coaching and Aisha in XYZ Coaching can both exist — they're in different tenants. But within one tenant, the email is unique.

---

## Theory: The Global Scope

```php
// app/Models/Scopes/TenantScope.php
namespace App\Models\Scopes;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Scope;

class TenantScope implements Scope
{
    public function apply(Builder $builder, Model $model): void
    {
        $tenant = app('current_tenant');
        
        if ($tenant) {
            $builder->where($model->getTable() . '.tenant_id', $tenant->id);
        }
    }
}
```

> **The scope is applied to every query on the model.** When you call `Student::all()`, the SQL becomes `SELECT * FROM students WHERE tenant_id = ?`.

---

## Theory: The Trait (Apply to All Tenant Models)

```php
// app/Models/Concerns/BelongsToTenant.php
namespace App\Models\Concerns;

use App\Models\Scopes\TenantScope;
use Illuminate\Database\Eloquent\Model;

trait BelongsToTenant
{
    public static function bootBelongsToTenant(): void
    {
        static::addGlobalScope(new TenantScope);
        static::creating(function (Model $model) {
            if (!$model->tenant_id && app()->bound('current_tenant')) {
                $model->tenant_id = app('current_tenant')->id;
            }
        });
    }
}
```

> **The trait does two things:**
> 1. Adds the global scope (filters queries)
> 2. Auto-fills `tenant_id` on create (so devs can't forget)
>
> **This is "make the right thing easy" engineering.** A developer who uses the trait can't accidentally create an orphan record.

---

## Theory: Apply the Trait to Models

```php
// app/Models/Student.php
class Student extends Model
{
    use BelongsToTenant;
    
    protected $fillable = ['name', 'email', 'class_id'];
    
    public function schoolClass()
    {
        return $this->belongsTo(SchoolClass::class);
    }
}
```

Same for `SchoolClass`, `Exam`, `Mark`, `User`, etc. — anything that belongs to a tenant.

---

## Theory: Bypass the Scope (When You Need To)

Sometimes you **want** to query across tenants (e.g., admin tasks, analytics, cron jobs). Laravel provides `withoutGlobalScope`:

```php
// Get ALL students across all tenants
$allStudents = Student::withoutGlobalScope(TenantScope::class)->get();
```

Or all scopes at once:

```php
$allStudents = Student::withoutGlobalScopes()->get();
```

> **Senior rule:** use this in admin/cron contexts only. In a request-driven flow, never bypass the scope.

---

## Theory: The Resolver (Set the Current Tenant)

The middleware we wrote in Lesson 20:

```php
class ResolveTenant
{
    public function handle(Request $request, Closure $next): Response
    {
        $host = $request->getHost();
        $parts = explode('.', $host);
        
        $slug = match(true) {
            $host === '127.0.0.1' || $host === 'localhost' => $request->header('X-Tenant', 'default'),
            default => $parts[0],
        };
        
        $tenant = Tenant::where('slug', $slug)->firstOrFail();
        
        if (!$tenant->is_active) {
            abort(403, "Institution is not active");
        }
        
        app()->instance('current_tenant', $tenant);
        
        return $next($request);
    }
}
```

This sets the tenant once per request. The global scope reads it.

---

## Theory: Subdomain-Based vs Path-Based vs Header-Based

Three ways to identify the tenant:

### Subdomain (production)

```
abc.coaching-saas.com  → Tenant "abc"
xyz.coaching-saas.com  → Tenant "xyz"
```

✅ Clean URLs. ❌ Requires DNS setup. ❌ Localhost is hard.

### Path-based (alternative)

```
coaching-saas.com/abc/students  → Tenant "abc"
coaching-saas.com/xyz/students  → Tenant "xyz"
```

✅ No DNS needed. ❌ URLs are uglier. ❌ Harder with SSL certs (SNI).

### Header-based (dev, API)

```
GET /students
X-Tenant: abc
```

✅ Works in localhost. ✅ Easy to test. ❌ Not user-friendly.

### The senior's choice

**Subdomain in production, header in dev.** The middleware we wrote handles both.

---

## Theory: Cross-Tenant Data Leaks — The Critical Bugs

Even with all this, you can leak data if you:

1. **Use raw queries:** `DB::table('students')->get()` — does NOT apply global scopes
2. **Forget the trait:** a new model without `BelongsToTenant` is unscoped
3. **Use eager loading wrong:** `$student->load('class')` — if Class doesn't have the trait, the join leaks
4. **Use polymorphic relations:** comments on a post in tenant A but tied to a post in tenant B

> **Senior rule:** every model that "belongs to a tenant" must have the `BelongsToTenant` trait. **Every.** **Single.** **One.**

---

## Theory: Composite Unique Indexes (Already Covered, Reinforced)

```php
// Email is unique within a tenant
$table->unique(['tenant_id', 'email']);

// Roll number is unique per class
$table->unique(['class_id', 'roll_number']);

// User is unique per tenant
$table->unique(['tenant_id', 'email']);
```

> **This is the senior's pattern.** The DB enforces business rules. The application doesn't have to.

---

## Theory: Migrations Across the Tenant Boundary

When you add a new tenant-owned table, every tenant gets the same schema. **No special handling needed** — just run the migration.

When you change a column, same thing.

> **If you had separate DBs per tenant, you'd run migrations on all 50. With shared schema, you run once.**

---

## Why It Matters

Multi-tenancy done wrong = data leaks. Data leaks = lawsuits, lost customers, company-ending events.

Multi-tenancy done right = automatic, enforced at the framework level, and you can't accidentally break it because the global scope catches you.

A senior designs multi-tenancy **on day one**, not "later." Retrofitting is a nightmare.

---

## Lab: Wire Up Multi-Tenancy

### Step 1 — Add `BelongsToTenant` trait

Create `app/Models/Concerns/BelongsToTenant.php`:

```php
<?php
namespace App\Models\Concerns;

use App\Models\Scopes\TenantScope;
use Illuminate\Database\Eloquent\Model;

trait BelongsToTenant
{
    public static function bootBelongsToTenant(): void
    {
        static::addGlobalScope(new TenantScope);
        
        static::creating(function (Model $model) {
            if (empty($model->tenant_id) && app()->bound('current_tenant')) {
                $model->tenant_id = app('current_tenant')->id;
            }
        });
    }
}
```

Create `app/Models/Scopes/TenantScope.php`:

```php
<?php
namespace App\Models\Scopes;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Scope;

class TenantScope implements Scope
{
    public function apply(Builder $builder, Model $model): void
    {
        if (!app()->bound('current_tenant')) {
            return;  // No tenant resolved yet (e.g. in console)
        }
        
        $tenant = app('current_tenant');
        $builder->where($model->getTable() . '.tenant_id', $tenant->id);
    }
}
```

### Step 2 — Apply to all tenant-owned models

Update `app/Models/Student.php`:

```php
namespace App\Models;

use App\Models\Concerns\BelongsToTenant;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class Student extends Model
{
    use BelongsToTenant, SoftDeletes;
    
    protected $fillable = ['name', 'email', 'class_id', 'phone'];
    protected $casts = ['is_active' => 'boolean'];
    
    public function schoolClass()
    {
        return $this->belongsTo(SchoolClass::class, 'class_id');
    }
}
```

Update `app/Models/SchoolClass.php`:

```php
namespace App\Models;

use App\Models\Concerns\BelongsToTenant;
use Illuminate\Database\Eloquent\Model;

class SchoolClass extends Model
{
    use BelongsToTenant;
    
    protected $fillable = ['name', 'grade_level', 'capacity'];
    
    public function students()
    {
        return $this->hasMany(Student::class, 'class_id');
    }
}
```

> **Exam, Mark, User** — apply the same pattern.

### Step 3 — Update the ResolveTenant middleware

Make sure it's registered in `bootstrap/app.php`:

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->alias([
        'tenant' => \App\Http\Middleware\ResolveTenant::class,
    ]);
})
```

Apply to your routes:

```php
Route::middleware(['tenant'])->group(function () {
    Route::resource('students', StudentController::class);
    Route::resource('school-classes', SchoolClassController::class);
});
```

### Step 4 — Create another tenant for testing

In tinker:

```php
\App\Models\Tenant::create([
    'name' => 'XYZ Coaching',
    'slug' => 'xyz',
    'email' => 'admin@xyz.com',
    'type' => 'coaching',
]);
```

### Step 5 — Test cross-tenant isolation

Make sure both tenants have data. In tinker:

```php
$abc = \App\Models\Tenant::where('slug', 'default')->first();
$xyz = \App\Models\Tenant::where('slug', 'xyz')->first();

// Bypass scope to insert raw
\App\Models\Student::withoutGlobalScopes()->create([
    'tenant_id' => $abc->id,
    'name' => 'Aisha',
    'email' => 'aisha@abc.com',
]);
\App\Models\Student::withoutGlobalScopes()->create([
    'tenant_id' => $xyz->id,
    'name' => 'Bilal',
    'email' => 'bilal@xyz.com',
]);
```

> **We're using `withoutGlobalScopes()` to seed test data across tenants.** In production code, you'd seed separately per tenant.

### Step 6 — Test the scope is working

In tinker:

```php
// Pretend we're tenant "default"
app()->instance('current_tenant', \App\Models\Tenant::where('slug', 'default')->first());

// Now query
\App\Models\Student::all();
// Returns only Aisha (the one for default tenant)

// Switch to xyz
app()->instance('current_tenant', \App\Models\Tenant::where('slug', 'xyz')->first());

\App\Models\Student::all();
// Returns only Bilal (the one for xyz tenant)
```

> **The scope reads the current tenant from the container and filters automatically.** This is multi-tenant security at the framework level.

### Step 7 — Test in the browser

Start the server:

```bash
php artisan serve
```

With `X-Tenant: default` header (since we're on localhost):

```bash
curl -H "X-Tenant: default" http://127.0.0.1:8000/students
```

Returns Aisha.

```bash
curl -H "X-Tenant: xyz" http://127.0.0.1:8000/students
```

Returns Bilal.

> **Try it without the header** — you should get a 404 because the middleware can't resolve a tenant.

### Step 8 — Verify the SQL Laravel generates

In tinker (with tenant set):

```php
\DB::enableQueryLog();
\App\Models\Student::all();
dd(\DB::getQueryLog());
```

You should see:

```sql
SELECT * FROM `students` WHERE `students`.`tenant_id` = 1
```

> **The scope is invisible in your code but it's there in the SQL.** This is the magic.

### Step 9 — Test that creating auto-fills tenant_id

In tinker (with tenant set):

```php
$student = \App\Models\Student::create([
    'name' => 'Chen',
    'email' => 'chen@abc.com',
]);
$student->tenant_id;  // 1 (the current tenant's id) — auto-filled!
```

> **The trait's `creating` hook filled it.** You didn't write it. You can't forget it.

---

## Quiz

**Q1.** A junior creates a new `Grade` model and forgets to add the `BelongsToTenant` trait. What's the security implication? How would you catch this in code review?

**Q2.** Your SaaS grows to 5,000 tenants on shared schema. Performance starts to degrade on the `students` table (5M rows). What indexes would you add, and what other strategies help?

**Q3.** A client needs their data on a separate server for regulatory reasons. How would you migrate a single tenant from shared schema to dedicated DB without major code changes? Sketch the approach.

**Q4.** Why is the composite unique `['tenant_id', 'email']` better than a global unique on `email`? What's the bug it prevents?

**Q5.** (Stretch) Your `BelongsToTenant` trait auto-fills `tenant_id` on `creating`. A junior writes a test that creates a student without setting a tenant. The test passes. What does this tell you about the trait's behavior in test environments, and how would you fix it?

---

## What You Now Have

✅ Multi-tenant architecture: shared schema with `tenant_id`
✅ Global scopes that auto-filter queries
✅ Trait that auto-fills `tenant_id` on create
✅ Middleware that resolves the current tenant
✅ Composite unique indexes
✅ Cross-tenant data isolation verified

## What's Next

Lesson 25 covers **authentication** — using Laravel Breeze or Fortify to add login, registration, and password reset flows. The foundation of every user-facing SaaS.

📖 **[Lesson 25 — Authentication with Laravel Breeze](25-authentication-with-laravel-breeze.md)**
