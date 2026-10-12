# Lesson 25 — Authentication with Laravel Breeze

> **Phase 3: Architecture** | You'll be here: ~30 min

## The Senior's Question

Authentication is **table stakes** for a SaaS. Users log in, register, reset passwords, verify emails. Laravel has multiple ways to do this — which one is the senior's choice for a coaching SaaS, and why?

---

## Theory: The Options

| Package | What it gives you | Verdict |
|---|---|---|
| **Laravel Breeze** | Login, register, password reset, email verify. Blade views. | **Senior's choice** for a Blade SaaS. Simple, complete, opinionated. |
| **Laravel Fortify** | Backend-only auth (no UI). You build the views. | Use when you have a custom UI or SPA. |
| **Laravel Jetstream** | Breeze + profile, 2FA, teams, API tokens. Livewire or Inertia. | Heavy. We don't need all that. |
| **Laravel Sanctum** | API token authentication (for SPAs and mobile). | Add later for API access. |
| **Roll your own** | Custom login flow. | **Never.** Insecure, time-consuming, brittle. |

> **For your SaaS, the answer is Breeze + Sanctum.** Breeze gives you the auth UI (Blade). Sanctum gives you API tokens later for mobile or SPA. Both are official Laravel packages.

---

## Theory: What Breeze Installs

- Login form
- Registration form
- Forgot password
- Reset password
- Email verification
- Logout
- Routes, controllers, views for all of the above
- Tailwind CSS for styling (you can switch to plain CSS)

---

## Lab: Install Laravel Breeze

### Step 1 — Install Breeze via Composer

```bash
cd G:\php-laravel\coaching-saas
composer require laravel/breeze --dev
```

> **`--dev` because it's only needed during development.** Production doesn't need Breeze; it just needs the routes it generated.

### Step 2 — Run the Breeze installer

```bash
php artisan breeze:install blade
```

> **`blade` for server-rendered views.** Other options: `react` (Inertia + React), `vue` (Inertia + Vue), `livewire` (Livewire components).

The installer will:
- Publish auth controllers to `app/Http/Controllers/Auth/`
- Publish auth views to `resources/views/auth/`
- Add auth routes to `routes/auth.php`
- Add Tailwind CSS (or you can opt out)
- Run npm install for frontend assets

### Step 3 — Run migrations

```bash
php artisan migrate
```

> Breeze doesn't add new tables, but it does add `email_verified_at` to users.

### Step 4 — Compile assets

```bash
npm install
npm run dev
```

> **Breeze uses Vite + Tailwind.** `npm run dev` starts the asset compiler.

### Step 5 — Try the auth pages

Start the server (in two terminals if you haven't):

```bash
# Terminal 1
php artisan serve

# Terminal 2
npm run dev
```

Visit:
- `http://127.0.0.1:8000/register` — register a new user
- `http://127.0.0.1:8000/login` — log in
- After login: redirected to `/dashboard`

> **You now have working authentication.** Breeze handled all the boilerplate.

---

## Theory: How Auth Works (Under the Hood)

### The User model

`app/Models/User.php` is updated by Breeze to use the `Authenticatable` trait:

```php
class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;  // added by Breeze
    
    protected $fillable = ['name', 'email', 'password'];
    protected $hidden = ['password', 'remember_token'];
    protected $casts = [
        'email_verified_at' => 'datetime',
        'password' => 'hashed',  // Breeze added this!
    ];
}
```

> **`'password' => 'hashed'` cast:** when you `User::create([... 'password' => 'secret'])`, it's automatically hashed with bcrypt. You never store plain-text passwords.

### The auth controllers

Breeze generated 5 controllers in `app/Http/Controllers/Auth/`:

| Controller | Purpose |
|---|---|
| `RegisteredUserController` | Registration |
| `AuthenticatedSessionController` | Login + logout |
| `PasswordResetLinkController` | Forgot password (sends email) |
| `NewPasswordController` | Reset password (with token) |
| `EmailVerificationPromptController` | "Verify your email" page |
| `VerifyEmailController` | Handles the verification link |

### The routes

`routes/auth.php` has all auth routes:

```php
Route::middleware('guest')->group(function () {
    Route::get('register', [RegisteredUserController::class, 'create']);
    Route::post('register', [RegisteredUserController::class, 'store']);
    Route::get('login', [AuthenticatedSessionController::class, 'create']);
    Route::post('login', [AuthenticatedSessionController::class, 'store']);
    // ... etc
});

Route::middleware('auth')->group(function () {
    Route::post('logout', [AuthenticatedSessionController::class, 'destroy']);
    // ... etc
});
```

> **Note the `guest` and `auth` middleware.** `guest` redirects logged-in users away from login/register. `auth` redirects anonymous users to login.

---

## Theory: The Auth Facade (How Auth Actually Works)

Laravel's auth is centered around "guards" and "providers."

### Guards — how the user is authenticated

- `web` (session-based) — for normal browser requests
- `sanctum` (token-based) — for API requests

```php
// Log in a user (session guard)
auth()->login($user);
auth()->attempt(['email' => $email, 'password' => $password]);
auth()->logout();

// Get the current user
auth()->user();
auth()->id();
auth()->check();
```

### Providers — where users come from

Default: Eloquent provider, uses the `users` table. Configured in `config/auth.php`.

> **For 99% of apps, you never change this.** Default works.

---

## Theory: Customizing Auth for Your SaaS

For a multi-tenant SaaS, you need to extend the default:

### Add `tenant_id` to users

```bash
php artisan make:migration add_tenant_id_to_users_table --table=users
```

```php
public function up(): void
{
    Schema::table('users', function (Blueprint $table) {
        $table->foreignId('tenant_id')
            ->nullable()
            ->after('email')
            ->constrained()
            ->nullOnDelete();
        $table->index('tenant_id');
    });
}
```

> **`nullable` because the user registers first, picks a tenant later** (or the tenant invites them).

### Add `role` to users

```bash
php artisan make:migration add_role_to_users_table --table=users
```

```php
public function up(): void
{
    Schema::table('users', function (Blueprint $table) {
        $table->string('role')->default('student')->after('tenant_id');
        // roles: 'admin' (institution owner), 'teacher', 'student', 'parent'
        $table->index('role');
    });
}
```

### Update the User model

```php
class User extends Authenticatable
{
    use BelongsToTenant, HasFactory, Notifiable, HasRoles;  // Spatie roles
    
    protected $fillable = ['name', 'email', 'password', 'tenant_id', 'role'];
    
    public function tenant()
    {
        return $this->belongsTo(Tenant::class);
    }
    
    public function isAdmin(): bool
    {
        return $this->role === 'admin';
    }
    
    public function isTeacher(): bool
    {
        return $this->role === 'teacher';
    }
}
```

---

## Theory: Registration Flow (For a Multi-Tenant SaaS)

When a new user registers, you need to decide:

### Option A: Self-signup creates a new tenant

```
User signs up → creates Tenant + User with role='admin'
                (the signup form asks for institution name)
```

Use this for: open SaaS where anyone can sign up.

### Option B: Admin invites users

```
Tenant admin invites teacher/student via email
User clicks link → sets password → joins tenant
```

Use this for: closed SaaS where customers are pre-sold.

### Senior's choice for coaching SaaS

**Both.** Self-signup creates the tenant + admin. Once the admin is in, they invite teachers and students.

---

## Theory: The Flow for Self-Signup

```php
// app/Http/Controllers/Auth/RegisteredUserController.php
public function store(RegistrationRequest $request): RedirectResponse
{
    $tenant = Tenant::create([
        'name' => $request->tenant_name,
        'slug' => \Str::slug($request->tenant_name) . '-' . \Str::random(4),
        'email' => $request->email,
        'type' => $request->tenant_type ?? 'coaching',
        'trial_ends_at' => now()->addDays(config('coaching.billing.trial_days')),
    ]);
    
    $user = User::create([
        'name' => $request->name,
        'email' => $request->email,
        'password' => $request->password,
        'tenant_id' => $tenant->id,
        'role' => 'admin',
    ]);
    
    event(new Registered($user));
    
    Auth::login($user);
    
    return redirect()->route('dashboard');
}
```

The registration form asks for:
- Institution name
- Your name (admin)
- Email
- Password

---

## Theory: Password Security

Laravel uses **bcrypt by default** for password hashing. It:

- Slows brute force attacks
- Auto-generates salt per password
- One-way (cannot be reversed)

```php
// 'password' => 'hashed' cast handles this automatically:
User::create(['password' => 'secret']);  // stored as '$2y$10$...'
```

For higher security, use **Argon2id** (PHP 7.2+):

```php
// config/hashing.php
'driver' => 'argon2id',
```

> **Argon2id is the modern recommendation.** Bcrypt is fine; argon2id is better for new projects.

### Password validation rules

```php
'password' => [
    'required',
    'confirmed',           // needs `password_confirmation` field
    Password::min(8)
        ->letters()
        ->mixedCase()
        ->numbers()
        ->symbols(),
],
```

> **This is the modern, strong password rule.** 8+ chars, mixed case, number, symbol. We cover this in detail in Lesson 31.

---

## Theory: Email Verification

Breeze includes email verification out of the box. To enforce it on a route:

```php
Route::middleware(['auth', 'verified'])->group(function () {
    // Only verified users can access these
});
```

In a controller:

```php
public function __construct()
{
    $this->middleware('verified');
}
```

> **For your SaaS, decide:** do you require email verification before they can use the product? Probably yes — otherwise people can sign up with fake emails.

---

## Theory: Rate Limiting (Built-In)

Laravel rate-limits login attempts automatically. Configured in `RouteServiceProvider` (or `bootstrap/app.php` in Laravel 11+).

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->append(\Illuminate\Routing\Middleware\ThrottleRequests::class);
})
```

In `routes/auth.php`:

```php
Route::post('login', [AuthenticatedSessionController::class, 'store'])
    ->middleware('throttle:5,1');  // 5 attempts per minute
```

> **Senior rule:** rate-limit all auth endpoints. Brute force is real.

---

## Why It Matters

Authentication is the front door. **Done wrong, your entire SaaS is compromised.** Breeze is the senior's choice because:

- It follows Laravel's conventions
- It's maintained by Laravel's core team
- It uses bcrypt/argon2 by default
- It includes email verification
- It includes rate limiting
- It includes password reset

**Don't roll your own auth. Ever.** Use Breeze, extend as needed.

---

## Lab: Customize Breeze for Multi-Tenancy

### Step 1 — Modify the registration form

Edit `resources/views/auth/register.blade.php` to ask for institution name and type:

```blade
<form method="POST" action="{{ route('register') }}">
    @csrf
    
    <!-- Institution fields -->
    <div>
        <label>Institution Name</label>
        <input name="tenant_name" value="{{ old('tenant_name') }}" required>
        @error('tenant_name')
            <p class="error">{{ $message }}</p>
        @enderror
    </div>
    
    <div>
        <label>Type</label>
        <select name="tenant_type" required>
            <option value="coaching">Coaching Center</option>
            <option value="school">School</option>
            <option value="kindergarten">Kindergarten</option>
        </select>
    </div>
    
    <!-- Admin user fields -->
    <div>
        <label>Your Name</label>
        <input name="name" value="{{ old('name') }}" required>
    </div>
    
    <div>
        <label>Email</label>
        <input name="email" type="email" value="{{ old('email') }}" required>
    </div>
    
    <div>
        <label>Password</label>
        <input name="password" type="password" required>
    </div>
    
    <div>
        <label>Confirm Password</label>
        <input name="password_confirmation" type="password" required>
    </div>
    
    <button type="submit">Create Account</button>
</form>
```

### Step 2 — Create a RegistrationRequest

```bash
php artisan make:request Auth/RegistrationRequest
```

Edit `app/Http/Requests/Auth/RegistrationRequest.php`:

```php
<?php
namespace App\Http\Requests\Auth;

use App\Enums\InstitutionType;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rules\Password;

class RegistrationRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'tenant_name' => ['required', 'string', 'max:255', 'unique:tenants,name'],
            'tenant_type' => ['required', 'in:coaching,school,kindergarten'],
            'name'        => ['required', 'string', 'max:255'],
            'email'       => ['required', 'email', 'unique:users,email'],
            'password'    => ['required', 'confirmed', Password::min(8)->mixedCase()->numbers()],
        ];
    }
}
```

### Step 3 — Update the registration controller

Edit `app/Http/Controllers/Auth/RegisteredUserController.php`:

```php
use App\Http\Requests\Auth\RegistrationRequest;
use App\Models\Tenant;
use App\Models\User;
use Illuminate\Support\Str;

public function store(RegistrationRequest $request): RedirectResponse
{
    $tenant = Tenant::create([
        'name' => $request->tenant_name,
        'slug' => Str::slug($request->tenant_name) . '-' . Str::random(4),
        'email' => $request->email,
        'type' => $request->tenant_type,
        'trial_ends_at' => now()->addDays(config('coaching.billing.trial_days')),
    ]);
    
    $user = User::create([
        'name' => $request->name,
        'email' => $request->email,
        'password' => $request->password,
        'tenant_id' => $tenant->id,
        'role' => 'admin',
    ]);
    
    event(new Registered($user));
    
    Auth::login($user);
    
    return redirect()->route('dashboard');
}
```

### Step 4 — Add a dashboard route and view

```bash
php artisan make:controller DashboardController
```

```php
class DashboardController extends Controller
{
    public function index(): View
    {
        $tenant = app('current_tenant');
        $stats = [
            'students' => Student::count(),
            'classes' => SchoolClass::count(),
            'exams' => Exam::count(),
        ];
        
        return view('dashboard', compact('tenant', 'stats'));
    }
}
```

Add the route:

```php
Route::middleware(['auth', 'tenant', 'verified'])->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index'])->name('dashboard');
    Route::resource('students', StudentController::class);
    Route::resource('school-classes', SchoolClassController::class);
});
```

Create `resources/views/dashboard.blade.php`:

```blade
@extends('layouts.app')

@section('content')
    <h1>{{ $tenant->name }}</h1>
    <p>Welcome back, {{ auth()->user()->name }}!</p>
    
    <div class="stats">
        <div>
            <h3>{{ $stats['students'] }}</h3>
            <p>Students</p>
        </div>
        <div>
            <h3>{{ $stats['classes'] }}</h3>
            <p>Classes</p>
        </div>
        <div>
            <h3>{{ $stats['exams'] }}</h3>
            <p>Exams</p>
        </div>
    </div>
@endsection
```

### Step 5 — Test the flow

1. Visit `http://127.0.0.1:8000/register`
2. Fill in the form
3. You're redirected to `/dashboard`
4. You see your tenant name, the admin user, and stats

> **You now have a multi-tenant SaaS with authentication.** The admin logs in, sees their institution's data, and can manage students and classes.

### Step 6 — Verify tenant isolation

Log in as one admin. They see only their tenant's data.

Log out, register a different institution. They see only their data.

> **The global scope, the middleware, and the auth work together to enforce multi-tenancy.** This is the foundation.

---

## Quiz

**Q1.** Why does Breeze use `'password' => 'hashed'` cast on the User model? What attack does it prevent, and what would happen without it?

**Q2.** A junior uses `User::create(['password' => $request->password])` and stores plain text. What's the consequence? How would you detect this in code review?

**Q3.** Why is rate limiting on the login endpoint important? What's the attack it prevents?

**Q4.** Your SaaS has self-signup that creates a tenant. A junior wants to allow users to register without creating a tenant (then pick a tenant later). Walk through the changes needed: schema, model, registration flow.

**Q5.** (Stretch) Email verification prevents spam signups. But the user can't access the app until verified. A senior says "use a magic link instead of email/password for first login." Why is this better? What does the flow look like?

---

## What You Now Have

✅ Laravel Breeze installed and customized
✅ Multi-tenant registration (tenant + admin user)
✅ Email verification, password reset, rate limiting built-in
✅ Dashboard with tenant-specific stats
✅ Authentication foundation ready for production

## What's Next

Lesson 26 covers **authorization and roles** — distinguishing admin, teacher, student, parent. We use Policies for fine-grained control and middleware for route-level access.

📖 **[Lesson 26 — Authorization, Roles, Policies, and Gates](26-authorization-roles-policies-and-gates.md)**
