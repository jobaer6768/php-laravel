# Lesson 15 — Routing, Controllers, and Responses

> **Phase 2: Laravel Core** | You'll be here: ~45 min

## The Senior's Question

A senior opens a Laravel project and the first thing they do is `php artisan route:list`. Why? Because every URL the user can hit lives in the routing layer. If you don't understand routing, you can't control what your app does.

---

## Theory: Routes Define Your App

A route is **a URL + HTTP method + handler.** The mapping is explicit. No auto-discovery. No file-based magic. You write:

```php
// routes/web.php
Route::get('/students', [StudentController::class, 'index']);
```

…and now `/students` returns the `index` method of `StudentController`. Period.

This is **security through explicitness.** Every endpoint is a documented, intentional choice. If a URL works, it's because someone wrote a route for it.

---

## Theory: HTTP Verbs

Laravel's router knows all HTTP methods:

```php
Route::get($uri, $handler);       // GET — read
Route::post($uri, $handler);      // POST — create
Route::put($uri, $handler);       // PUT — full update
Route::patch($uri, $handler);     // PATCH — partial update
Route::delete($uri, $handler);    // DELETE — remove
Route::options($uri, $handler);   // OPTIONS — preflight
Route::match(['get', 'post'], $uri, $handler);  // multiple
Route::any($uri, $handler);       // any method
```

> **Senior rule:** never use `Route::any()`. Every endpoint should have a specific verb. It's part of your API contract.

---

## Theory: Route Parameters

```php
Route::get('/students/{id}', [StudentController::class, 'show']);
Route::get('/classes/{class}/students/{student}', [StudentController::class, 'showInClass']);
```

The controller receives the parameters:

```php
public function show(string $id): View {
    $student = Student::find($id);
    return view('students.show', compact('student'));
}

public function showInClass(string $class, string $student): View {
    // ...
}
```

### Optional parameters

```php
Route::get('/students/{id?}', [StudentController::class, 'show']);
```

### Parameter constraints (regex)

```php
Route::get('/students/{id}', [...])->where('id', '[0-9]+');
Route::get('/students/{name}', [...])->where('name', '[A-Za-z]+');
Route::get('/students/{id}/{name}', [...])->where(['id' => '[0-9]+', 'name' => '[A-Za-z]+']);
```

> **Senior rule:** always constrain URL parameters. It prevents `/students/abc` from hitting a numeric-ID handler and producing a confusing error.

---

## Theory: Named Routes

```php
Route::get('/students/{id}', [StudentController::class, 'show'])->name('students.show');
```

Now you can reference this route by name anywhere:

```php
// In a controller
return redirect()->route('students.show', ['id' => 42]);
// Generates: /students/42

// In a Blade view
<a href="{{ route('students.show', $student->id) }}">View</a>

// In tests
$this->get(route('students.show', $student));
```

> **Senior rule:** always name your routes. Always. Never hardcode URLs. If you rename a route, only one place to change.

---

## Theory: Route Groups

When many routes share attributes (prefix, middleware, namespace), group them:

```php
Route::middleware(['auth', 'tenant'])->prefix('admin')->name('admin.')->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index'])->name('dashboard');
    Route::resource('students', StudentController::class);
    Route::resource('teachers', TeacherController::class);
});
```

This creates routes like:
- `GET /admin/dashboard` → name `admin.dashboard`, requires auth + tenant
- `GET /admin/students` → name `admin.students.index`
- `POST /admin/students` → name `admin.students.store`
- etc.

> **Senior rule:** if you find yourself repeating middleware/prefix/name on 5+ routes, group them.

---

## Theory: Resource Routes

The single most useful route helper. Defines all 7 RESTful routes for a resource in one line:

```php
Route::resource('students', StudentController::class);
```

This creates:

| Verb | URI | Action | Route name |
|---|---|---|---|
| GET | `/students` | index | students.index |
| GET | `/students/create` | create | students.create |
| POST | `/students` | store | students.store |
| GET | `/students/{student}` | show | students.show |
| GET | `/students/{student}/edit` | edit | students.edit |
| PUT/PATCH | `/students/{student}` | update | students.update |
| DELETE | `/students/{student}` | destroy | students.destroy |

> **Senior rule:** always use `Route::resource()` for CRUD. It enforces REST conventions and makes your routes self-documenting.

### Limiting the resource

```php
// Only these routes
Route::resource('students', StudentController::class)->only(['index', 'show', 'store']);

// All except these
Route::resource('students', StudentController::class)->except(['destroy']);
```

### API resources

```php
Route::apiResource('students', StudentController::class);
```

This is the same as `resource()` but excludes `create` and `edit` (which are HTML form pages, not API endpoints).

---

## Theory: Controllers

A controller is **a class that groups related request handlers.** Each method is an action.

### Generate one

```bash
php artisan make:controller StudentController --resource
```

The `--resource` flag generates all 7 methods. Without it, you get an empty class.

### A resource controller

```php
<?php
namespace App\Http\Controllers;

use App\Models\Student;
use Illuminate\Http\Request;
use Illuminate\Http\RedirectResponse;
use Illuminate\View\View;

class StudentController extends Controller
{
    public function index(): View
    {
        $students = Student::latest()->paginate(20);
        return view('students.index', compact('students'));
    }

    public function create(): View
    {
        return view('students.create');
    }

    public function store(StoreStudentRequest $request): RedirectResponse
    {
        Student::create($request->validated());
        return redirect()->route('students.index')->with('success', 'Student added');
    }

    public function show(Student $student): View
    {
        return view('students.show', compact('student'));
    }

    public function edit(Student $student): View
    {
        return view('students.edit', compact('student'));
    }

    public function update(UpdateStudentRequest $request, Student $student): RedirectResponse
    {
        $student->update($request->validated());
        return redirect()->route('students.show', $student)->with('success', 'Updated');
    }

    public function destroy(Student $student): RedirectResponse
    {
        $student->delete();
        return redirect()->route('students.index')->with('success', 'Deleted');
    }
}
```

> **Notice the type-hinted parameters:**
> - `StoreStudentRequest` — auto-validates before the method runs
> - `UpdateStudentRequest` — same
> - `Student $student` — **route model binding.** Laravel finds the student by ID and injects the model.

---

## Theory: Route Model Binding

This is Laravel's signature feature for clean controllers.

```php
// Route
Route::get('/students/{student}', [StudentController::class, 'show']);

// Controller
public function show(Student $student): View {
    return view('students.show', compact('student'));
}
```

When the URL is `/students/42`:
1. Laravel sees `{student}` and the `Student` type-hint
2. It runs `Student::find(42)` automatically
3. If found, it injects the model
4. If not found, it returns a 404

You don't write `Student::find()` yourself. The framework does it.

> **Senior rule:** use route model binding. It removes 90% of "find or fail" boilerplate from your controllers.

### Customizing the key

```php
// Route
Route::get('/students/{student:slug}', [StudentController::class, 'show']);
```

Now `{student:slug}` means "find the Student where `slug` column matches the URL value."

### Implicit vs explicit binding

- **Implicit:** the type-hint matches the parameter name (the example above)
- **Explicit:** you call `Route::bind('student`, function ($value) { ... })` for custom logic

---

## Theory: Responses

Controllers return **Response objects** (or things that become Response objects).

| Return type | What it produces |
|---|---|
| `view('name', $data)` | HTML (Blade template) |
| `redirect()->route('x')` | 302 redirect |
| `back()` | Redirect to previous page |
| `response()->json($data)` | JSON response |
| `response()->download($path)` | File download |
| `response()->file($path)` | File inline |
| `response()->make($string, $code)` | Raw string response |
| `response()->noContent()` | 204 No Content |
| `abort(404)` | Throw HttpException |
| A string or array | Auto-converted to response |
| A Model | Auto-converted to JSON |
| A view | HTML response |

### HTTP status codes

```php
return response()->json($data, 201);  // Created
return response()->json($data, 422);  // Validation error
return response()->noContent();        // 204
abort(403, 'Forbidden');              // 403
abort_if($student->is_archived, 410, 'Archived');  // 410
abort_unless($user->can('view', $student), 403);   // 403
```

> **Senior rule:** always set the right status code. A 200 for a created resource is a junior mistake. A 422 for validation errors is the right move.

---

## Theory: Single Action Controllers

Sometimes a controller has just one action. Make the method `__invoke` and Laravel will call it:

```php
class ShowDashboardController extends Controller
{
    public function __invoke(): View
    {
        return view('dashboard');
    }
}

// Route
Route::get('/dashboard', ShowDashboardController::class);
```

> **Use this for:** profile pages, reports, dashboard, settings — pages with one canonical action.

---

## Why It Matters

Every Laravel feature flows through a route. Mastering routing means mastering **how URLs become actions.** A senior:

- Names every route
- Uses resource routes for CRUD
- Uses route model binding to avoid `find()` calls
- Groups routes that share concerns
- Returns the right status code always

This is the difference between 200 lines of clean code and 800 lines of repetitive code.

---

## Lab: Build a Real Resource

### Step 1 — Generate the controller

```bash
cd G:\php-laravel\coaching-saas
php artisan make:controller StudentController --resource --model=Student
```

> The `--model=Student` flag tells Artisan which model the controller will use. It adds the type-hints correctly.

### Step 2 — Define the route

Open `routes/web.php` and add:

```php
use App\Http\Controllers\StudentController;

Route::resource('students', StudentController::class);
```

### Step 3 — See what got generated

```bash
php artisan route:list
```

You should see 7 new routes. Their handler is `App\Http\Controllers\StudentController@<method>`.

### Step 4 — Look at the generated controller

```bash
cat app/Http/Controllers/StudentController.php
```

Each method has the right signature. The `show`, `edit`, `update`, `destroy` methods all type-hint `Student $student` — that's route model binding.

### Step 5 — Add some test data (so we can see things work)

Open `database/seeders/DatabaseSeeder.php` and replace it with:

```php
<?php
namespace Database\Seeders;

use App\Models\Tenant;
use App\Models\Student;
use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        $tenant = Tenant::create([
            'name' => 'ABC Coaching',
            'slug' => 'abc',
            'email' => 'admin@abc.com',
            'type' => 'coaching',
        ]);

        Student::create([
            'tenant_id' => $tenant->id,
            'name' => 'Aisha',
            'email' => 'aisha@abc.com',
        ]);
    }
}
```

Wait — the `students` table doesn't exist yet. We need a migration. Let's add it (the senior's way: with a real migration):

```bash
php artisan make:migration create_students_table
```

Edit the migration file:

```php
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
```

Run migrations and seed:

```bash
php artisan migrate:fresh --seed
```

### Step 6 — Hit a real route

Start the dev server (if not running):

```bash
php artisan serve
```

In browser or via curl:

```bash
curl -i http://127.0.0.1:8000/students
```

You'll get a 500 error — the `index` method tries to return `view('students.index')` but the view doesn't exist. **That's the next lesson (Blade).** For now, replace the controller's `index` method to debug:

```php
public function index()
{
    $students = Student::all();
    return response()->json($students);
}
```

Hit it again:

```bash
curl -i http://127.0.0.1:8000/students
```

Now you see JSON with the student data. The route works.

### Step 7 — Test route model binding

```bash
curl -i http://127.0.0.1:8000/students/1
```

The `show` method type-hints `Student $student`. Laravel finds Student ID 1 and injects it.

Try a non-existent ID:

```bash
curl -i http://127.0.0.1:8000/students/999
```

You get a 404 — Laravel's automatic "not found" for missing model bindings. **This is route model binding doing its job.**

### Step 8 — Add a named route and use it

Open `routes/web.php` and add a name to the resource:

```php
Route::resource('students', StudentController::class)->names('students');
```

(Actually, `Route::resource` already names them by default, but this shows the syntax.)

Use the named route anywhere:

```bash
php artisan tinker
>>> route('students.show', 1);
=> "http://127.0.0.1:8000/students/1"
```

> **Senior insight:** Laravel can generate URLs from route names. This is how you decouple your templates and code from the actual URL structure.

### Step 9 — Clean up (revert the JSON hack)

In Lesson 16 (Blade), we'll add real views. For now, leave the JSON-returning `index` so you can see the route works.

---

## Quiz

**Q1.** A junior writes `Route::get('/students/all', [StudentController::class, 'all']);`. A senior says "use a resource." What's the senior's reasoning?

**Q2.** Why is `Route::resource('students', StudentController::class)` better than seven separate `Route::get/post/put/delete` calls?

**Q3.** What's the difference between these two route definitions?

```php
Route::get('/students/{id}', [StudentController::class, 'show']);
Route::get('/students/{student}', [StudentController::class, 'show']);
```

In the controller, the first takes `string $id`, the second takes `Student $student`. Why does the second one require no `find()` call?

**Q4.** A controller returns `response()->json($user, 200)` after creating a user. What status code should it return instead, and why?

**Q5.** (Stretch) You have a route `/admin/students`. A junior says "I can use the same `StudentController`." A senior says "create `Admin\StudentController`." Why?

---

## What You Now Have

✅ Mastery of routes: verbs, parameters, naming, grouping, resources
✅ Resource controllers and all 7 RESTful methods
✅ Route model binding — Laravel's cleanest feature
✅ Response types and status codes
✅ A working route that returns real data from the DB

## What's Next

Lesson 16 covers **Blade** — the server-rendered templating engine. You'll build the HTML for your students index, edit, and show pages. We'll also compare to your React/JSX mental model so the analogies are clear.

📖 **[Lesson 16 — Blade Templating vs the React Mental Model](16-blade-templating-vs-react-mental-model.md)**
