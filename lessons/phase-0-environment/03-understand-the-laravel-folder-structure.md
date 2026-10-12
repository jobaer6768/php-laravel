# Lesson 03 — Understand the Laravel Folder Structure (In Detail)

> **Phase 0: Environment** | You'll be here: ~25 min

## The Senior's Question

You have a Laravel project. Before you write a line of code, you need to know *exactly* where each kind of code goes. A senior's speed comes from never hesitating on this — they know the convention by heart. Let's build that muscle.

---

## Theory: The "Convention Over Configuration" Idea

Laravel is opinionated. **There is a correct place for every kind of code, and deviating from it is a smell.** This is the opposite of Express.js, which is a thin HTTP layer and lets you organize your code however you want.

| Framework | Organization style | Cost of doing it wrong |
|---|---|---|
| Express (MERN) | You decide | "Where do controllers go?" — debate forever |
| Laravel | Framework decides | You can't even do it wrong if you follow conventions |

The senior's takeaway: **don't fight the framework.** When in doubt, look at where Laravel's own code lives and put yours in the same place.

---

## Deep Dive: The Folders

### `app/` — Your Code

This is the *only* folder where you write application logic. Subdirectories:

#### `app/Http/Controllers/`

Controllers handle HTTP requests. They take input, do work, return a response.

```php
// app/Http/Controllers/StudentController.php
namespace App\Http\Controllers;

use App\Models\Student;
use Illuminate\Http\Request;

class StudentController extends Controller
{
    public function index() {
        $students = Student::all();
        return view('students.index', compact('students'));
    }

    public function show(Student $student) {
        return view('students.show', compact('student'));
    }
}
```

> **Senior rule:** controllers should be *thin*. They orchestrate, they don't do business logic. Logic goes in models, services, or actions.

#### `app/Http/Middleware/`

Middleware filters requests before they hit a controller. Think of them as a pipeline:

```
Request → Middleware A → Middleware B → Controller → Response
            ↓                              ↓
         (auth check)                  (return data)
         (log request)
         (set locale)
```

You'll use middleware for:
- Authentication
- Tenant scoping (more on this in Phase 3)
- Logging
- Setting the current language for the institution

#### `app/Http/Requests/`

Form Request classes are how Laravel does **server-side validation**. They are a powerful pattern:

```php
// app/Http/Requests/StoreStudentRequest.php
class StoreStudentRequest extends FormRequest {
    public function authorize(): bool { return true; }
    public function rules(): array {
        return [
            'name' => 'required|string|max:255',
            'email' => 'required|email|unique:students,email',
            'class_id' => 'required|exists:classes,id',
        ];
    }
}
```

Then in the controller:
```php
public function store(StoreStudentRequest $request) {
    // If we reach here, validation already passed.
    Student::create($request->validated());
}
```

> **Why this is genius:** validation rules, error messages, and authorization are *in one place*, separated from the controller. You can test them in isolation. You can reuse them.

#### `app/Models/`

Eloquent models represent database tables. This is the heart of your app's data layer.

```php
// app/Models/Student.php
class Student extends Model {
    protected $fillable = ['name', 'email', 'class_id'];
    
    public function class() {
        return $this->belongsTo(SchoolClass::class, 'class_id');
    }
}
```

We'll spend an entire lesson (Lesson 17) on Eloquent. For now, just know: **one model per database table, by convention**.

#### `app/Providers/`

Service providers wire up the framework at boot. In Laravel 11+, most providers are auto-discovered. You'll rarely write one from scratch, but you'll modify `AppServiceProvider` to register your own services.

### `bootstrap/app.php`

This is the **new Laravel 11+ bootstrap**. It used to be split across multiple files (`Http/Kernel.php`, `Console/Kernel.php`, `Exceptions/Handler.php`). Now it's one file with a fluent API:

```php
return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {
        // Add global middleware
    })
    ->withExceptions(function (Exceptions $exceptions) {
        // Customize error handling
    })
    ->create();
```

Cleaner. One place to see how the app is wired.

### `config/` — The Knobs

Every configurable aspect of Laravel has a file here. ~30 files. Some you'll touch often:

| File | What it controls |
|---|---|
| `app.php` | App name, env, debug, timezone, providers |
| `database.php` | DB connections, default driver |
| `cache.php` | Cache driver, prefix, ttl |
| `queue.php` | Queue driver, connection |
| `mail.php` | Mail driver (SMTP, log, etc.) |
| `logging.php` | Log channels |

Rule: **read these files.** They are the most well-commented config files in any framework. The comments tell you what each setting does.

### `database/`

#### `database/migrations/`

PHP files that define your database schema, version-controlled. **This is your database's Git history.**

```php
// database/migrations/2026_01_15_120000_create_students_table.php
public function up(): void {
    Schema::create('students', function (Blueprint $table) {
        $table->id();
        $table->foreignId('institution_id')->constrained();
        $table->string('name');
        $table->string('email')->unique();
        $table->timestamps();
    });
}
```

> **Senior rule:** never edit the database with a GUI tool. Always write a migration. **Always.**

#### `database/seeders/`

Insert initial/test data. Used for:
- Default data (countries list, plans list)
- Local development
- Demos

#### `database/factories/`

Generate fake data for tests. Used heavily in testing.

### `public/`

**The only folder exposed to the web.** Nginx points here. The single entry point is `public/index.php`. Everything else is one folder up and unreachable from the browser.

This is **security through structure**. A misconfigured server can't accidentally expose your `.env` file because it's not under `public/`.

### `resources/`

- `views/` — Blade templates (server-rendered HTML)
- `css/` and `js/` — Source assets, compiled by Vite into `public/`

### `routes/`

- `web.php` — Browser routes. Have session, CSRF, cookies.
- `api.php` — Stateless API routes. Prefixed with `/api/`. No sessions.
- `console.php` — Artisan commands defined as closures.

> **Senior rule:** every URL a user can hit must be defined here, in one of these files. If a URL works but isn't in `routes/`, that's a security bug.

### `storage/`

Writable runtime data:
- `app/` — Files your app generates (uploaded student photos go here by default)
- `framework/cache/` — Cached data
- `framework/sessions/` — Session data
- `framework/views/` — Compiled Blade
- `logs/` — Laravel + PHP error logs

> **Senior rule:** never `git commit` this folder. It's in `.gitignore` by default. Lose it, your app loses state, but no code is lost.

### `tests/`

- `Feature/` — Test full request flows (hit URL, check response)
- `Unit/` — Test single functions/classes in isolation

We'll spend four lessons on this in Phase 4.

### `vendor/`

Composer dependencies. **Never edit, never commit, never wonder what's in here.** It's regenerated from `composer.lock`.

---

## Why It Matters

Knowing where things go is the difference between:

- "I added a new endpoint and now I have no idea where its logic is" (junior)
- "I added a controller, a route, a request validator, a view, and a test" (senior)

**A senior's speed is pattern recognition.** When they hear "I need a new form to create a student", they immediately know:
- New `StudentController@store` method
- New `StoreStudentRequest` validator
- New `students/create.blade.php` view
- New migration if the schema changed
- New test in `tests/Feature/StudentTest.php`

Five file types, five locations, muscle memory. That's what we're building.

---

## Lab: Map the Conventions

### Step 1 — Open the project in your editor

```bash
cd G:\php-laravel\coaching-saas
code .
```

(Or open it in your editor of choice.)

### Step 2 — Read the comments in three config files

Open and skim:
- `config/app.php` — read the comments at the top of each section
- `config/database.php` — see the connection array structure
- `config/cache.php` — see the stores array

Goal: see how well-commented Laravel configs are. They're documentation disguised as config.

### Step 3 — Find the welcome page route

Open `routes/web.php`. You'll see:

```php
Route::get('/', function () {
    return view('welcome');
});
```

The `view('welcome')` resolves to `resources/views/welcome.blade.php`. Open that file. **This is the HTML you saw in your browser.**

### Step 4 — Trace the request

Use the diagram below to physically trace what happened when you hit `http://127.0.0.1:8000`:

```
Nginx/PHP-server
   ↓
public/index.php                  ← entry point
   ↓
bootstrap/app.php                 ← loads framework
   ↓
routes/web.php                    ← matches GET /
   ↓
Closure: view('welcome')          ← returns Blade view
   ↓
resources/views/welcome.blade.php ← rendered to HTML
   ↓
HTTP response back to browser
```

Write this down on paper. Yes, on paper. Muscle memory requires handwriting for some people.

### Step 5 — Note what's NOT here yet

You have:
- ✅ Routes
- ✅ A view
- ❌ Controllers
- ❌ Models
- ❌ Migrations (besides default ones)
- ❌ Tests

These are all empty. That's normal for a fresh project. You'll add them as you build features.

---

## Quiz

**Q1.** A teammate says "I put my database queries inside my Blade template because the controller was getting long." Why does this break Laravel's design? List two concrete problems.

**Q2.** You need to validate that a student's email is unique within their *institution*, not globally. Where do you put this logic — in the controller, in a Form Request, or in the model? Why?

**Q3.** A junior `git push`es their project and the team lead is furious. They committed the `vendor/` folder. What's the senior's reasoning for why `vendor/` must never be committed?

**Q4.** You upload a student photo from a form. Where does the file land by default? What's the security reason the destination is where it is?

**Q5.** (Stretch) A senior says "if it's not in `routes/web.php`, it doesn't exist as a URL." A junior says "but I could still expose it via a controller method, right?" Who's right, and what's the security implication?

---

## What You Now Have

✅ Mental model of every folder's purpose
✅ Understanding of the request lifecycle trace
✅ Pattern recognition for where each kind of code goes

## What's Next

In Lesson 04, we'll run your first **Artisan commands** and create your first real **migration** — you'll modify the database, hands-on, and see how `php artisan` is the dev's superpower.

📖 **[Lesson 04 — Run Your First Artisan Command](04-run-your-first-artisan-command.md)**
