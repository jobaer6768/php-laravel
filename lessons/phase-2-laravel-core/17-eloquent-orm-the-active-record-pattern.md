# Lesson 17 — Eloquent ORM: The Active Record Pattern

> **Phase 2: Laravel Core** | You'll be here: ~50 min

## The Senior's Question

You've been writing Mongoose (ODM for MongoDB). Now you're learning Eloquent (ORM for SQL). The biggest shift: in SQL, **relationships are first-class** — your data is spread across normalized tables and you JOIN them together. Get this wrong, and your app is slow, buggy, or both. Get it right, and the framework does the heavy lifting.

---

## Theory: ORM vs ODM

| | ODM (Mongoose) | ORM (Eloquent) |
|---|---|---|
| Data shape | Documents (nested objects) | Tables (rows + columns) |
| Relationships | Manual (ref + populate) | First-class (hasMany, belongsTo) |
| Schema | Flexible, schema-less | Strict, defined in migrations |
| Joins | `$lookup` (slow) | Eager loading (fast) |
| Consistency | Eventually consistent (per document) | ACID transactions |
| Best for | Variable shape, hierarchical | Structured, relational |

> **Eloquent is to SQL what Mongoose is to MongoDB, but with the relationships built in.** You write `Student::find(1)->class->teacher` and it just works.

---

## Theory: The Model — One Class Per Table

```php
// app/Models/Student.php
class Student extends Model
{
    // Eloquent assumes:
    // Table name: 'students' (snake_case plural of class name)
    // Primary key: 'id'
    // Timestamps: created_at, updated_at columns exist
    // Connection: default DB connection
}
```

That's the **minimum** model. With nothing in it, Eloquent gives you:

```php
$student = Student::find(1);
$student->name = 'Aisha';
$student->save();
Student::all();
Student::where('name', 'Aisha')->get();
Student::create(['name' => 'Aisha', 'email' => 'aisha@x.com']);
```

> **Convention is everything.** Table `students` ↔ class `Student`. Column `created_at` ↔ property `created_at`. If you follow conventions, you write almost no config.

### When you need to override conventions

```php
class Student extends Model
{
    protected $table = 'my_students';           // different table name
    protected $primaryKey = 'student_id';        // different PK column
    public $incrementing = false;                // PK is not auto-increment
    protected $keyType = 'string';               // PK is a string (e.g. UUID)
    public $timestamps = false;                  // no created_at/updated_at
    protected $dateFormat = 'U';                 // Unix timestamp
    protected $connection = 'mysql_legacy';     // use a different DB connection
}
```

> **You almost never need these.** If you find yourself overriding, ask "should this be a different schema?"

---

## Theory: Mass Assignment Protection

This is **critical** for security.

```php
$student = Student::create($request->all());
```

If a user submits a form with a hidden `is_admin` field, they could escalate their own privileges. **Never** trust mass assignment without `$fillable` or `$guarded`.

```php
class Student extends Model
{
    // Option 1: explicit allowlist (preferred)
    protected $fillable = ['name', 'email', 'class_id'];
    
    // Option 2: explicit denylist
    protected $guarded = ['id', 'is_admin', 'tenant_id'];
}
```

`$fillable` is an **allowlist:** "these columns may be set via mass assignment." Anything not in the list is ignored.

> **Senior rule:** always set `$fillable` on every model. It's the single most important security pattern in Eloquent.

---

## Theory: Casting — Type Safety at the ORM Level

```php
class Student extends Model
{
    protected $casts = [
        'is_active'      => 'boolean',
        'date_of_birth'  => 'date',
        'settings'       => 'array',
        'metadata'       => 'collection',  // special: returns a Collection
    ];
}
```

Now:
- `$student->is_active` is a real `bool`, not a string `'1'`
- `$student->date_of_birth` is a Carbon instance, not a string
- `$student->settings` is a PHP array, not a JSON string
- `$student->metadata` is a Laravel Collection

> **Senior rule:** cast every column that has a known type. It's the difference between `$student->is_active === true` (true) and `$student->is_active === '1'` (also true, but your IDE doesn't know).

### Custom casts (PHP 8.0+)

```php
protected $casts = [
    'email' => \App\Casts\Lowercase::class,
];

// app/Casts/Lowercase.php
class Lowercase implements \Illuminate\Contracts\Database\Eloquent\CastsAttributes
{
    public function get($model, string $key, $value, array $attributes): string
    {
        return strtolower($value);
    }
    
    public function set($model, string $key, $value, array $attributes): string
    {
        return strtolower($value);
    }
}
```

Now `$student->email` is always lowercase. Set or get.

---

## Theory: The Query Builder (Under the Hood)

Eloquent sits on top of Laravel's **query builder.** When you write:

```php
Student::where('is_active', true)->orderBy('name')->get();
```

It builds SQL:

```sql
SELECT * FROM students WHERE is_active = 1 ORDER BY name;
```

You can drop down to the raw query builder:

```php
DB::table('students')->where('is_active', true)->orderBy('name')->get();
```

This is faster (no model overhead) but you lose relationships, casts, events, etc.

> **Senior rule:** use Eloquent models unless you have a measured performance reason. Code clarity beats premature optimization.

---

## Theory: Common Query Methods

### Read

```php
Student::all();                              // everything
Student::find(1);                            // by PK
Student::find(1, 2, 3);                      // multiple
Student::findOrFail(1);                      // 404 if not found
Student::where('name', 'Aisha')->get();
Student::where('grade', '>=', 80)->get();
Student::whereIn('id', [1, 2, 3])->get();
Student::whereNull('deleted_at')->get();
Student::orderBy('name')->get();
Student::latest()->get();                    // ORDER BY created_at DESC
Student::oldest()->get();                    // ORDER BY created_at ASC
Student::limit(10)->get();
Student::skip(20)->take(10)->get();          // pagination
Student::count();
Student::exists();                           // boolean: any rows?
Student::pluck('name');                      // array of values
```

### Write

```php
$student = new Student;
$student->name = 'Aisha';
$student->email = 'aisha@x.com';
$student->save();

Student::create(['name' => 'Aisha', 'email' => 'aisha@x.com']);  // mass assign

$student = Student::find(1);
$student->update(['name' => 'New Name']);

$student->delete();
Student::destroy(1, 2, 3);
Student::where('is_active', false)->delete();
```

### Aggregates

```php
Student::count();
Student::where('class_id', 5)->count();
Student::max('grade');
Student::min('grade');
Student::avg('grade');
Student::sum('paid_amount');
```

---

## Theory: Relationships — The Big One

This is where Eloquent earns its name. Relationships are first-class:

### One-to-Many (the most common)

A `Class` has many `Student`s. A `Student` belongs to one `Class`.

```php
// app/Models/SchoolClass.php
class SchoolClass extends Model
{
    public function students()
    {
        return $this->hasMany(Student::class, 'class_id');
    }
}

// app/Models/Student.php
class Student extends Model
{
    public function schoolClass()
    {
        return $this->belongsTo(SchoolClass::class, 'class_id');
    }
}
```

> **Naming:** the method name on the parent (`students`) becomes the dynamic property. So `$class->students` returns a Collection of Student models.

### Using relationships

```php
$class = SchoolClass::find(1);

foreach ($class->students as $student) {
    echo $student->name;
}

$student = Student::find(1);
echo $student->schoolClass->name;
```

### One-to-One

A `User` has one `Profile`:

```php
// User model
public function profile()
{
    return $this->hasOne(Profile::class);
}

// Profile model
public function user()
{
    return $this->belongsTo(User::class);
}
```

### Many-to-Many

A `Student` can take many `Course`s. A `Course` can have many `Student`s.

You need a pivot table: `course_student` (singular, alphabetical, joined with `_`).

```php
// Student model
public function courses()
{
    return $this->belongsToMany(Course::class);
}

// Course model
public function students()
{
    return $this->belongsToMany(Student::class);
}
```

Use it:

```php
$student->courses()->attach($courseId);
$student->courses()->detach($courseId);
$student->courses()->sync([1, 2, 3]);  // set to exactly these
```

### Has-Many-Through

A `Country` has many `User`s. A `User` has many `Post`s. A `Country` has many `Post`s *through* `User`s.

```php
class Country extends Model
{
    public function posts()
    {
        return $this->hasManyThrough(Post::class, User::class);
    }
}
```

### Polymorphic — A Comment Can Be on a Post, Video, or Photo

```php
// Comment model
public function commentable()
{
    return $this->morphTo();
}

// Post model
public function comments()
{
    return $this->morphMany(Comment::class, 'commentable');
}
```

> **You'll use polymorphic for:** comments (on posts/lessons/events), attachments (on students/classes/assignments), activity logs.

---

## Theory: Eager Loading — The N+1 Problem

This is **the most important performance lesson in Eloquent.**

### The problem (N+1)

```php
$students = Student::all();  // 1 query

foreach ($students as $student) {
    echo $student->schoolClass->name;  // +1 query per student!
}
```

If you have 100 students, you run **101 queries.** This is the **N+1 query problem.**

### The fix (eager loading)

```php
$students = Student::with('schoolClass')->get();  // 2 queries total

foreach ($students as $student) {
    echo $student->schoolClass->name;  // already loaded
}
```

Now: 1 query for students + 1 query for all their classes = 2 queries total.

### Nested eager loading

```php
Student::with('schoolClass.teacher')->get();
// Loads students + their classes + the class's teacher
```

### Multiple relationships

```php
Student::with(['schoolClass', 'courses', 'guardian'])->get();
```

### Constrained eager loading

```php
Student::with([
    'schoolClass' => fn($q) => $q->where('is_active', true),
    'courses' => fn($q) => $q->orderBy('name'),
])->get();
```

> **Senior rule:** always think about N+1. When you loop over a collection and access a relationship, add `->with(...)`. Use Laravel Debugbar (Lesson 35) to see query counts in dev.

---

## Theory: Query Scopes — Reusable Query Logic

### Local scope

```php
// In Student model
public function scopeActive($query)
{
    return $query->where('is_active', true);
}

public function scopeInClass($query, int $classId)
{
    return $query->where('class_id', $classId);
}
```

Use:

```php
Student::active()->get();
Student::active()->inClass(5)->get();
```

> **The `scope` prefix is dropped when calling.** `scopeActive` becomes `active()`.

### Global scope (auto-applied)

```php
// app/Models/Scopes/TenantScope.php
class TenantScope implements Scope
{
    public function apply(Builder $builder, Model $model): void
    {
        $builder->where('tenant_id', app('current_tenant')->id);
    }
}

// In Student model
protected static function booted(): void
{
    static::addGlobalScope(new TenantScope);
}
```

Now **every** query on Student is automatically filtered by tenant. **This is how we build multi-tenant isolation in Lesson 24.**

---

## Why It Matters

Eloquent is your data layer. If you don't master it, your app will be:
- Slow (N+1 queries everywhere)
- Buggy (mass-assignment vulns)
- Verbose (rewriting the same queries)

A senior writes:
- One-liner queries that produce the right SQL
- Eager loads by default
- Casts for type safety
- Scopes to avoid duplication
- Relationships that reflect the real domain

---

## Lab: Build the Students ↔ Classes Relationship

### Step 1 — Generate the Class model + migration

```bash
cd G:\php-laravel\coaching-saas
php artisan make:model SchoolClass -m
```

> **Note the name `SchoolClass`, not `Class`.** `class` is a reserved keyword in PHP.

### Step 2 — Edit the migration

```php
public function up(): void
{
    Schema::create('school_classes', function (Blueprint $table) {
        $table->id();
        $table->foreignId('tenant_id')->constrained()->cascadeOnDelete();
        $table->string('name');                    // "Class 10A"
        $table->string('grade_level');             // "10"
        $table->integer('capacity')->default(40);
        $table->timestamps();
    });
}

public function down(): void
{
    Schema::dropIfExists('school_classes');
}
```

### Step 3 — Run the migration

```bash
php artisan migrate
```

### Step 4 — Add the relationship to Student

```php
// app/Models/Student.php
class Student extends Model
{
    protected $fillable = ['name', 'email', 'class_id', 'tenant_id'];
    
    protected $casts = [
        'is_active' => 'boolean',
    ];
    
    public function schoolClass()
    {
        return $this->belongsTo(SchoolClass::class, 'class_id');
    }
}
```

### Step 5 — Add the inverse relationship to SchoolClass

```php
// app/Models/SchoolClass.php
class SchoolClass extends Model
{
    protected $fillable = ['name', 'grade_level', 'capacity', 'tenant_id'];
    
    public function students()
    {
        return $this->hasMany(Student::class, 'class_id');
    }
}
```

### Step 6 — Add `class_id` to the students migration

The existing students table doesn't have `class_id`. We need a new migration:

```bash
php artisan make:migration add_class_id_to_students_table
```

```php
public function up(): void
{
    Schema::table('students', function (Blueprint $table) {
        $table->foreignId('class_id')
            ->nullable()
            ->after('email')
            ->constrained('school_classes')
            ->nullOnDelete();
    });
}

public function down(): void
{
    Schema::table('students', function (Blueprint $table) {
        $table->dropForeign(['class_id']);
        $table->dropColumn('class_id');
    });
}
```

Run:

```bash
php artisan migrate
```

### Step 7 — Test in tinker

```bash
php artisan tinker
```

```php
// Create a class
$class = \App\Models\SchoolClass::create([
    'name' => 'Class 10A',
    'grade_level' => '10',
    'tenant_id' => 1,
]);

// Create a student in that class
$student = \App\Models\Student::create([
    'name' => 'Aisha',
    'email' => 'aisha@example.com',
    'class_id' => $class->id,
    'tenant_id' => 1,
]);

// Access the relationship
$student->schoolClass->name;  // "Class 10A"
$class->students->count();    // 1

// Eager loading
$students = \App\Models\Student::with('schoolClass')->get();
$students->first()->schoolClass->name;  // no extra query

// Local scope
// (let's add one first)
```

### Step 8 — Add a local scope to Student

Edit `app/Models/Student.php`:

```php
public function scopeInClass($query, int $classId)
{
    return $query->where('class_id', $classId);
}

public function scopeWithEmail($query, string $email)
{
    return $query->where('email', $email);
}
```

In tinker:

```php
\App\Models\Student::inClass(1)->get();
\App\Models\Student::withEmail('aisha@example.com')->first();
```

### Step 9 — Verify eager loading with a query log

In tinker:

```php
\DB::enableQueryLog();

// This is N+1 (bad)
$students = \App\Models\Student::all();
$students->each(fn($s) => $s->schoolClass->name);

count(\DB::getQueryLog());  // 1 + N (where N = number of students)

// This is eager (good)
\DB::flushQueryLog();
$students = \App\Models\Student::with('schoolClass')->get();
$students->each(fn($s) => $s->schoolClass->name);

count(\DB::getQueryLog());  // 2 (1 for students, 1 for classes)
```

> **You just witnessed the N+1 problem and the eager-loading fix in production.** This is the #1 performance bug in Laravel apps.

---

## Quiz

**Q1.** Why is `$fillable` critical for security? What's the worst-case bug if you don't set it?

**Q2.** You have 50 orders and loop `foreach ($orders as $o) { echo $o->customer->name; }`. How many queries? How do you fix it?

**Q3.** A junior writes `Student::all()` to get students. You ask "but what about N+1?" They say "I'm only accessing the student fields." When is that answer correct, and when is it wrong?

**Q4.** What's the difference between a local scope and a global scope? When would you use each?

**Q5.** (Stretch) Design the database schema for a coaching SaaS: tenants, users (with roles), classes, students, attendance, exams, marks. What are the tables, foreign keys, and key indexes? (Sketch only — we'll build it in Lesson 27.)

---

## What You Now Have

✅ Eloquent model fluency
✅ Relationships: one-to-many, many-to-many, polymorphic
✅ Eager loading to prevent N+1
✅ Local and global scopes
✅ Casts and mass assignment protection

## What's Next

Lesson 18 covers **migrations and schema design** — the senior's approach to evolving your database over time without losing data or breaking production.

📖 **[Lesson 18 — Migrations, Schema, and the Why of Versioned DB](18-migrations-schema-and-the-why-of-versioned-db.md)**
