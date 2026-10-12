# Lesson 26 — Authorization, Roles, Policies, and Gates

> **Phase 3: Architecture** | You'll be here: ~40 min

## The Senior's Question

You have 4 user types in your SaaS: institution admin, teacher, student, parent. Each can do different things. The junior writes `if ($user->is_admin) ...` everywhere. The senior uses **Laravel's authorization layer** — declarative, testable, and centralized. How?

---

## Theory: The Three Layers (Recap)

```
Route Middleware        — "This URL requires admin role"
        ↓
Form Request authorize() — "Is this user allowed to submit?"
        ↓
Policy                  — "Can THIS user do THIS action on THIS model?"
        ↓
Controller method runs
```

You've seen all three. Now let's build them for a real SaaS.

---

## Theory: Roles vs Permissions — The Distinction

| Concept | Example | Use when |
|---|---|---|
| **Role** | "admin", "teacher", "student" | A user has one role (mostly) |
| **Permission** | "edit-students", "view-reports" | A user has many permissions |

For a coaching SaaS, **4 roles** is enough:

| Role | What they can do |
|---|---|
| `admin` (institution owner) | Everything: manage users, billing, settings |
| `teacher` | View/edit students in their classes, take attendance, record marks |
| `student` | View their own marks, attendance, fees |
| `parent` | View their child's data |

You can implement this with **Spatie's Laravel Permission** package (heavy but proven) or **simple role column + Policies** (light, no extra dependencies).

> **Senior's choice:** start with a `role` column on users. Add Spatie only when you need fine-grained permissions beyond what roles give you.

---

## Theory: Adding a `role` Column (Lightweight)

```php
// migration
Schema::table('users', function (Blueprint $table) {
    $table->string('role')->default('student')->after('tenant_id');
    $table->index(['tenant_id', 'role']);
});
```

```php
// app/Models/User.php
class User extends Authenticatable
{
    use BelongsToTenant;
    
    public function isAdmin(): bool
    {
        return $this->role === 'admin';
    }
    
    public function isTeacher(): bool
    {
        return $this->role === 'teacher';
    }
    
    public function isStudent(): bool
    {
        return $this->role === 'student';
    }
    
    public function isParent(): bool
    {
        return $this->role === 'parent';
    }
}
```

> **An enum is even better for roles:**

```php
// app/Enums/UserRole.php
enum UserRole: string
{
    case Admin = 'admin';
    case Teacher = 'teacher';
    case Student = 'student';
    case Parent = 'parent';
    
    public function label(): string
    {
        return match($this) {
            self::Admin => 'Institution Admin',
            self::Teacher => 'Teacher',
            self::Student => 'Student',
            self::Parent => 'Parent',
        };
    }
}
```

Update the migration:

```php
$table->enum('role', array_column(UserRole::cases(), 'value'))->default('student');
```

Update the model:

```php
protected $casts = [
    'role' => UserRole::class,
];

public function isAdmin(): bool
{
    return $this->role === UserRole::Admin;
}
// etc.
```

---

## Theory: Middleware for Role-Based Routes

```bash
php artisan make:middleware EnsureUserHasRole
```

```php
<?php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class EnsureUserHasRole
{
    public function handle(Request $request, Closure $next, string ...$roles): Response
    {
        $user = $request->user();
        
        if (!$user) {
            return redirect()->route('login');
        }
        
        if (!in_array($user->role->value, $roles, true)) {
            abort(403, 'You do not have permission to access this page.');
        }
        
        return $next($request);
    }
}
```

Register in `bootstrap/app.php`:

```php
$middleware->alias([
    'role' => \App\Http\Middleware\EnsureUserHasRole::class,
    'tenant' => \App\Http\Middleware\ResolveTenant::class,
]);
```

Use in routes:

```php
Route::middleware(['auth', 'tenant'])->group(function () {
    
    // Admin only
    Route::middleware('role:admin')->group(function () {
        Route::resource('users', UserController::class);
        Route::get('/billing', [BillingController::class, 'index']);
    });
    
    // Teacher + Admin
    Route::middleware('role:admin,teacher')->group(function () {
        Route::resource('students', StudentController::class);
        Route::resource('exams', ExamController::class);
        Route::resource('marks', MarkController::class);
    });
    
    // All authenticated users
    Route::get('/dashboard', [DashboardController::class, 'index']);
    Route::get('/profile', [ProfileController::class, 'edit']);
});
```

> **Clean separation:** admins manage users and billing. Teachers manage academic data. Students/parents can only see their own data.

---

## Theory: Policies for Fine-Grained Authorization

A Policy answers: "Can THIS user do THIS action on THIS model?"

```bash
php artisan make:policy StudentPolicy --model=Student
```

Edit `app/Policies/StudentPolicy.php`:

```php
<?php
namespace App\Policies;

use App\Models\Student;
use App\Models\User;
use App\Enums\UserRole;

class StudentPolicy
{
    public function viewAny(User $user): bool
    {
        // Admin and teachers can list students
        return in_array($user->role, [UserRole::Admin, UserRole::Teacher], true);
    }

    public function view(User $user, Student $student): bool
    {
        // Must be in same tenant
        if ($user->tenant_id !== $student->tenant_id) {
            return false;
        }
        
        // Admin and teachers can view any student in their tenant
        if (in_array($user->role, [UserRole::Admin, UserRole::Teacher], true)) {
            return true;
        }
        
        // Students can view their own record
        if ($user->role === UserRole::Student && $user->student_id === $student->id) {
            return true;
        }
        
        // Parents can view their children
        if ($user->role === UserRole::Parent) {
            return $user->children->contains($student->id);
        }
        
        return false;
    }

    public function create(User $user): bool
    {
        return $user->role === UserRole::Admin;
    }

    public function update(User $user, Student $student): bool
    {
        if ($user->tenant_id !== $student->tenant_id) {
            return false;
        }
        
        return in_array($user->role, [UserRole::Admin, UserRole::Teacher], true);
    }

    public function delete(User $user, Student $student): bool
    {
        if ($user->tenant_id !== $student->tenant_id) {
            return false;
        }
        
        return $user->role === UserRole::Admin;
    }
}
```

> **Notice the multi-tenant check first:** `tenant_id` must match. **Always** check this before role-based logic. Otherwise, an admin from Tenant A could modify students in Tenant B (unlikely but possible if policies are inconsistent).

### Use the policy in a controller

```php
public function show(Student $student): View
{
    $this->authorize('view', $student);
    return view('students.show', compact('student'));
}

public function update(UpdateStudentRequest $request, Student $student): RedirectResponse
{
    $this->authorize('update', $student);
    $student->update($request->validated());
    return redirect()->route('students.show', $student);
}
```

### Use the policy in Blade

```blade
@can('view', $student)
    <a href="{{ route('students.show', $student) }}">View</a>
@endcan

@can('update', $student)
    <a href="{{ route('students.edit', $student) }}">Edit</a>
@endcan

@cannot('delete', $student)
    <p>You don't have permission to delete.</p>
@endcannot
```

### Use the policy in a Form Request

```php
public function authorize(): bool
{
    $student = $this->route('student');
    
    if (!$student) {
        return $this->user()->can('create', Student::class);
    }
    
    return $this->user()->can('update', $student);
}
```

---

## Theory: Gates for Non-Model Actions

Sometimes authorization is about an action, not a model:

```php
// app/Providers/AppServiceProvider.php
use Illuminate\Support\Facades\Gate;

public function boot(): void
{
    Gate::define('manage-billing', function (User $user) {
        return $user->role === UserRole::Admin;
    });
    
    Gate::define('view-tenant-settings', function (User $user) {
        return $user->role === UserRole::Admin;
    });
    
    Gate::define('export-data', function (User $user) {
        return in_array($user->role, [UserRole::Admin, UserRole::Teacher], true);
    });
}
```

Use it:

```php
// In controller
if (Gate::denies('manage-billing')) {
    abort(403);
}

// Or shorter
Gate::authorize('manage-billing');

// In route
Route::middleware('can:manage-billing')->get('/billing', ...);

// In Blade
@can('export-data')
    <button>Export</button>
@endcan
```

---

## Theory: When to Use What

| Concern | Tool |
|---|---|
| "Must be logged in" | `auth` middleware |
| "Must be admin" | `role:admin` middleware |
| "Can this user view THIS student?" | Policy |
| "Can this user manage billing (any tenant)?" | Gate |
| "Can this user submit THIS form?" | Form Request `authorize()` |
| "Can this user access THIS URL?" | Route middleware |

---

## Theory: The Authorization Flow for a Request

A student wants to view their own marks:

```
Request: GET /students/42
   ↓
[auth middleware]      — User logged in? Yes.
   ↓
[tenant middleware]    — Tenant resolved? Yes.
   ↓
[role middleware]      — Allowed role? Yes (teacher).
   ↓
Router matches → StudentController@show
   ↓
$this->authorize('view', $student)
   ↓
StudentPolicy@view($user, $student)
   ↓
Returns true/false
   ↓
If true: returns view
If false: throws 403
```

---

## Why It Matters

Authorization done right is **declarative, centralized, testable:**

- Routes declare what roles are required
- Policies declare who can do what to which models
- Gates declare non-model permissions
- Form Requests declare submission authorization
- Controllers just trust the framework

A senior writes `Route::middleware('role:admin,teacher')` and `Gate::authorize('view-billing')` instead of `if (auth()->user()->is_admin) ...`.

---

## Lab: Build Full Authorization

### Step 1 — Add the role enum

Create `app/Enums/UserRole.php`:

```php
<?php
namespace App\Enums;

enum UserRole: string
{
    case Admin = 'admin';
    case Teacher = 'teacher';
    case Student = 'student';
    case Parent = 'parent';
    
    public function label(): string
    {
        return match($this) {
            self::Admin => 'Institution Admin',
            self::Teacher => 'Teacher',
            self::Student => 'Student',
            self::Parent => 'Parent',
        };
    }
}
```

### Step 2 — Update the migration to use the enum

```bash
php artisan make:migration add_role_to_users --table=users
```

```php
public function up(): void
{
    Schema::table('users', function (Blueprint $table) {
        $table->enum('role', array_column(\App\Enums\UserRole::cases(), 'value'))
            ->default('student')
            ->after('tenant_id');
        $table->index(['tenant_id', 'role']);
    });
}
```

### Step 3 — Update the User model

```php
class User extends Authenticatable
{
    use BelongsToTenant;
    
    protected $fillable = ['name', 'email', 'password', 'tenant_id', 'role'];
    
    protected $casts = [
        'email_verified_at' => 'datetime',
        'password' => 'hashed',
        'role' => UserRole::class,
    ];
    
    public function tenant()
    {
        return $this->belongsTo(Tenant::class);
    }
    
    public function isAdmin(): bool { return $this->role === UserRole::Admin; }
    public function isTeacher(): bool { return $this->role === UserRole::Teacher; }
    public function isStudent(): bool { return $this->role === UserRole::Student; }
    public function isParent(): bool { return $this->role === UserRole::Parent; }
}
```

### Step 4 — Run migration

```bash
php artisan migrate
```

### Step 5 — Create the role middleware

```bash
php artisan make:middleware EnsureUserHasRole
```

Fill in:

```php
<?php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class EnsureUserHasRole
{
    public function handle(Request $request, Closure $next, string ...$roles): Response
    {
        $user = $request->user();
        
        if (!$user) {
            return redirect()->route('login');
        }
        
        if (!in_array($user->role->value, $roles, true)) {
            abort(403, "This area is restricted to: " . implode(', ', $roles));
        }
        
        return $next($request);
    }
}
```

Register in `bootstrap/app.php`:

```php
$middleware->alias([
    'role' => \App\Http\Middleware\EnsureUserHasRole::class,
    'tenant' => \App\Http\Middleware\ResolveTenant::class,
]);
```

### Step 6 — Use role middleware in routes

```php
Route::middleware(['auth', 'tenant', 'verified'])->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index'])->name('dashboard');
    
    Route::middleware('role:admin,teacher')->group(function () {
        Route::resource('students', StudentController::class);
        Route::resource('school-classes', SchoolClassController::class);
    });
    
    Route::middleware('role:admin')->group(function () {
        Route::resource('users', UserController::class);
    });
});
```

### Step 7 — Generate the Student Policy

```bash
php artisan make:policy StudentPolicy --model=Student
```

Fill in:

```php
<?php
namespace App\Policies;

use App\Models\Student;
use App\Models\User;
use App\Enums\UserRole;

class StudentPolicy
{
    public function viewAny(User $user): bool
    {
        return in_array($user->role, [UserRole::Admin, UserRole::Teacher], true);
    }

    public function view(User $user, Student $student): bool
    {
        if ($user->tenant_id !== $student->tenant_id) return false;
        return in_array($user->role, [UserRole::Admin, UserRole::Teacher], true);
    }

    public function create(User $user): bool
    {
        return $user->role === UserRole::Admin;
    }

    public function update(User $user, Student $student): bool
    {
        if ($user->tenant_id !== $student->tenant_id) return false;
        return in_array($user->role, [UserRole::Admin, UserRole::Teacher], true);
    }

    public function delete(User $user, Student $student): bool
    {
        if ($user->tenant_id !== $student->tenant_id) return false;
        return $user->role === UserRole::Admin;
    }
}
```

### Step 8 — Update the controller to use the policy

Edit `app/Http/Controllers/StudentController.php`:

```php
public function index(): View
{
    $this->authorize('viewAny', Student::class);
    $students = Student::all();
    return view('students.index', compact('students'));
}

public function show(Student $student): View
{
    $this->authorize('view', $student);
    return view('students.show', compact('student'));
}

public function create(): View
{
    $this->authorize('create', Student::class);
    return view('students.create');
}

public function edit(Student $student): View
{
    $this->authorize('update', $student);
    return view('students.edit', compact('student'));
}

public function update(UpdateStudentRequest $request, Student $student): RedirectResponse
{
    $student->update($request->validated());
    return redirect()->route('students.show', $student)->with('success', 'Updated.');
}

public function destroy(Student $student): RedirectResponse
{
    $this->authorize('delete', $student);
    $student->delete();
    return redirect()->route('students.index')->with('success', 'Deleted.');
}
```

### Step 9 — Update Blade views with @can directives

Edit `resources/views/students/index.blade.php`:

```blade
@extends('layouts.app')

@section('title', 'All Students')

@section('content')
    <h1>Students</h1>
    
    @can('create', App\Models\Student::class)
        <a href="{{ route('students.create') }}">+ New Student</a>
    @endcan
    
    @if ($students->isEmpty())
        <p>No students yet.</p>
    @else
        <table>
            <thead>
                <tr>
                    <th>ID</th>
                    <th>Name</th>
                    <th>Email</th>
                    <th>Actions</th>
                </tr>
            </thead>
            <tbody>
                @foreach ($students as $student)
                    <tr>
                        <td>{{ $student->id }}</td>
                        <td>{{ $student->name }}</td>
                        <td>{{ $student->email }}</td>
                        <td>
                            @can('view', $student)
                                <a href="{{ route('students.show', $student) }}">View</a>
                            @endcan
                            @can('update', $student)
                                <a href="{{ route('students.edit', $student) }}">Edit</a>
                            @endcan
                        </td>
                    </tr>
                @endforeach
            </tbody>
        </table>
    @endif
@endsection
```

### Step 10 — Test

1. Register a new tenant. You become the admin.
2. Log in as admin. Visit `/students`. You can view, create, edit, delete.
3. Manually create a teacher user in tinker:
   ```php
   \App\Models\User::create([
       'name' => 'Mr. Karim',
       'email' => 'karim@abc.com',
       'password' => 'password',
       'tenant_id' => 1,
       'role' => 'teacher',
   ]);
   ```
4. Log in as the teacher. Visit `/students`. You can view and edit but not create or delete.
5. Try to access `/users` (admin-only route). You get 403.

> **The authorization layer is working end-to-end:** routes, controllers, views, all enforce consistently.

---

## Quiz

**Q1.** Why use a Policy instead of `if (auth()->user()->is_admin)` in the controller? Give two concrete benefits.

**Q2.** A junior writes `if ($student->tenant_id !== auth()->user()->tenant_id) abort(403);` in the controller. A senior says "put it in the policy." Why is the senior right?

**Q3.** When would you use a Gate vs a Policy? Give a concrete example of each from a coaching SaaS.

**Q4.** You have `Route::middleware('role:admin,teacher')` on `/students`. Inside the controller, you also call `$this->authorize('view', $student)`. Is one of these redundant? Why do senior devs include both?

**Q5.** (Stretch) A student user registers themselves (without an institution). They need to be linked to a student record in some institution. Walk through the data model and the linking flow.

---

## What You Now Have

✅ Role enum and role-based middleware
✅ Policies for fine-grained authorization
✅ Gates for non-model permissions
✅ Tenant-aware authorization (always checks tenant_id first)
✅ Multi-layer defense: middleware + Form Request + Policy

## What's Next

Lesson 27 covers **domain modeling** — designing the data model for the full coaching SaaS. Tenants, users, classes, students, attendance, exams, marks, fees, payments. The full picture.

📖 **[Lesson 27 — Domain Modeling for a Coaching SaaS](27-domain-modeling-for-coaching-saas.md)**
