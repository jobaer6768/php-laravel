# Lesson 16 — Blade Templating vs the React Mental Model

> **Phase 2: Laravel Core** | You'll be here: ~45 min

## The Senior's Question

You've been writing React. Now you need to write Blade templates. The mental shift: in React, the view is a function of state that runs in the browser. In Blade, the view is a function of data that runs on the server. Once you grok that, everything clicks.

---

## Theory: The Architectural Difference

### React (client-rendered)

```
Server:  sends JSON ───────────────► Browser
                                    ↓
                                  React renders HTML
                                  (in the user's browser)
```

### Blade (server-rendered)

```
Server:  Blade template runs in PHP → HTML string
         ────────────────────────────► Browser
                                      (just displays)
```

> **The same data, the same logic, but Blade runs the view before sending HTML.** The browser does no rendering work — it just displays what arrived.

This is **faster to first paint** (no JS to download before showing content) and **simpler** (no client-side state to manage). The trade-off: less interactivity. For a CRUD SaaS, that's almost always the right trade.

---

## Theory: A Blade File Is HTML + PHP Directives

A `.blade.php` file is mostly HTML, with `{{ }}` and `@directive` markers:

```blade
<!-- resources/views/students/index.blade.php -->
@extends('layouts.app')

@section('title', 'All Students')

@section('content')
    <h1>Students</h1>
    
    @if ($students->isEmpty())
        <p>No students yet.</p>
    @else
        <table>
            <thead>
                <tr>
                    <th>Name</th>
                    <th>Email</th>
                    <th>Actions</th>
                </tr>
            </thead>
            <tbody>
                @foreach ($students as $student)
                    <tr>
                        <td>{{ $student->name }}</td>
                        <td>{{ $student->email }}</td>
                        <td>
                            <a href="{{ route('students.show', $student) }}">View</a>
                        </td>
                    </tr>
                @endforeach
            </tbody>
        </table>
    @endif
@endsection
```

> **Notice the syntax:**
> - `{{ $value }}` — echoes a value, **escapes HTML** (security!)
> - `@if`, `@foreach` — control flow directives
> - `@extends('layouts.app')` — template inheritance
> - `@section('content')` — defines a section

---

## Theory: Echoing Data — `{{ }}` vs `{!! !!}`

```blade
{{ $student->name }}    <!-- escaped: < becomes &lt; -->
{!! $student->bio !!}   <!-- raw: rendered as-is -->
```

> **Senior rule:** use `{{ }}` always. Use `{!! !!}` only when you trust the data source 100% (your own admin-entered content). Never use it on user input.

For React developers: `{{ }}` is like `{value}` in JSX (auto-escaped). `{!! !!}` is like `dangerouslySetInnerHTML`.

---

## Theory: Control Flow

```blade
@if ($student->is_active)
    Active
@elseif ($student->is_pending)
    Pending
@else
    Inactive
@endif

@unless ($student->trashed())
    Currently enrolled
@endunless

@isset($records)
    {{ count($records) }} records
@endisset

@empty($students)
    No students
@endempty
```

> **These are equivalent to PHP `if/else/unless/isset/empty`.** Same logic, cleaner syntax.

---

## Theory: Loops

```blade
@foreach ($students as $student)
    <p>{{ $student->name }}</p>
@endforeach

@for ($i = 0; $i < 10; $i++)
    <p>Item {{ $i }}</p>
@endfor

@while ($condition)
    ...
@endwhile
```

### The `$loop` variable (inside `@foreach`)

```blade
@foreach ($students as $student)
    <p>
        {{ $loop->iteration }}. {{ $student->name }}
        @if ($loop->first) (first)
        @elseif ($loop->last) (last)
        @endif
    </p>
@endforeach
```

> **`$loop` is like React's `array.map((item, index) => ...)` index** but with more info: `first`, `last`, `index`, `iteration`, `count`, `remaining`, `depth` (for nested loops).

---

## Theory: Template Inheritance

This is Blade's killer feature. You define a **layout** and child views fill in slots:

### Layout

```blade
<!-- resources/views/layouts/app.blade.php -->
<!DOCTYPE html>
<html>
<head>
    <title>@yield('title', 'My App')</title>
    <link rel="stylesheet" href="{{ asset('css/app.css') }}">
</head>
<body>
    <nav>
        <a href="{{ route('home') }}">Home</a>
        <a href="{{ route('students.index') }}">Students</a>
    </nav>
    
    <main>
        @if (session('success'))
            <div class="alert alert-success">{{ session('success') }}</div>
        @endif
        
        @yield('content')
    </main>
    
    <footer>&copy; 2026 Coaching SaaS</footer>
</body>
</html>
```

### Child

```blade
<!-- resources/views/students/index.blade.php -->
@extends('layouts.app')

@section('title', 'All Students')

@section('content')
    <h1>Students</h1>
    
    <a href="{{ route('students.create') }}">+ New Student</a>
    
    @if ($students->isEmpty())
        <p>No students yet.</p>
    @else
        <ul>
            @foreach ($students as $student)
                <li>
                    <a href="{{ route('students.show', $student) }}">
                        {{ $student->name }}
                    </a>
                </li>
            @endforeach
        </ul>
    @endif
@endsection
```

> **The mental model:** the layout has named slots (`@yield('content')`). The child fills them in (`@section('content') ... @endsection`). In React, this is like a `<Layout>{children}</Layout>` component.

---

## Theory: Components (Modern Blade)

Since Laravel 9, the recommended pattern is **components** instead of `@extends/@section`. They're closer to React components:

```blade
<!-- resources/views/components/alert.blade.php -->
@props(['type' => 'info', 'message'])

<div class="alert alert-{{ $type }}">
    {{ $message }}
</div>
```

Use it:

```blade
<x-alert type="success" :message="$successMessage" />
<x-alert type="error" message="Something went wrong" />
```

> **In React terms:** `<x-alert />` is like importing and rendering `<Alert />`. The `:` prefix on `:message` passes a variable (vs. the string `"$successMessage"`).

### Anonymous components (single file, no class)

```blade
<!-- resources/views/components/student-card.blade.php -->
@props(['student'])

<div class="card">
    <h3>{{ $student->name }}</h3>
    <p>{{ $student->email }}</p>
    <a href="{{ route('students.show', $student) }}">View</a>
</div>
```

Use:

```blade
<x-student-card :student="$student" />
```

### Class-based components

```bash
php artisan make:component Alert
```

Generates `app/View/Components/Alert.php` and `resources/views/components/alert.blade.php`.

```php
// app/View/Components/Alert.php
class Alert extends Component
{
    public function __construct(
        public string $type = 'info',
        public string $message = '',
    ) {}
    
    public function render(): View
    {
        return view('components.alert');
    }
}
```

> **Senior rule:** for simple, presentational pieces, use anonymous components. For logic-heavy components (with methods, computed properties), use class-based.

---

## Theory: Includes and Slots

```blade
<!-- resources/views/partials/header.blade.php -->
<header>
    <h1>{{ $title ?? 'Welcome' }}</h1>
</header>

<!-- In another view -->
@include('partials.header', ['title' => 'Students'])
```

For more flexible composition, use slots:

```blade
<!-- components/card.blade.php -->
<div class="card">
    <div class="card-header">{{ $title }}</div>
    <div class="card-body">
        {{ $slot }}
    </div>
</div>

<!-- Use -->
<x-card title="Student Details">
    <p>Name: {{ $student->name }}</p>
    <p>Email: {{ $student->email }}</p>
</x-card>
```

> **The `$slot` is the children of the component, like React's `{children}`.**

### Named slots

```blade
<!-- components/modal.blade.php -->
<div class="modal">
    <div class="modal-header">{{ $title }}</div>
    <div class="modal-body">{{ $slot }}</div>
    <div class="modal-footer">
        {{ $footer }}
    </div>
</div>

<!-- Use -->
<x-modal>
    <x-slot:title>Confirm</x-slot:title>
    
    Are you sure you want to delete this?
    
    <x-slot:footer>
        <button>Cancel</button>
        <button>Delete</button>
    </x-slot:footer>
</x-modal>
```

---

## Theory: Common Directives Cheat-Sheet

| Directive | What it does |
|---|---|
| `@csrf` | Inserts CSRF token hidden field (required in forms) |
| `@method('PUT')` | Spoofs HTTP method (HTML forms only support GET/POST) |
| `@auth` ... `@endauth` | If user is logged in |
| `@guest` ... `@endguest` | If user is not logged in |
| `@can('edit', $post)` | If user has permission |
| `@production` ... @else | Only in production |
| `@env('local')` | Only in given environment |
| `@json($data)` | Output JSON for JS consumption |
| `@vite(['resources/css/app.css'])` | Include Vite-compiled assets |
| `@error('field')` | Show validation error for a field |
| `@dd($variable)` | Dump and die (debug) |

### The CSRF token (forms)

```blade
<form method="POST" action="{{ route('students.store') }}">
    @csrf
    <input name="name">
    <button type="submit">Save</button>
</form>
```

`@csrf` outputs:
```html
<input type="hidden" name="_token" value="...">
```

> **This is required for every POST/PUT/PATCH/DELETE form.** Skip it, you get 419 Page Expired. We saw this in Lesson 14.

### Spoofing PUT/DELETE in forms

HTML forms only support GET and POST. To send PUT/DELETE:

```blade
<form method="POST" action="{{ route('students.update', $student) }}">
    @csrf
    @method('PUT')
    <input name="name" value="{{ $student->name }}">
    <button>Update</button>
</form>
```

---

## Theory: Showing Validation Errors

After a form submission fails, the form is re-rendered with the user's input and errors. This is automatic:

```blade
<form method="POST" action="{{ route('students.store') }}">
    @csrf
    
    <div>
        <label>Name</label>
        <input name="name" value="{{ old('name') }}">
        @error('name')
            <p class="error">{{ $message }}</p>
        @enderror
    </div>
    
    <div>
        <label>Email</label>
        <input name="email" value="{{ old('email') }}">
        @error('email')
            <p class="error">{{ $message }}</p>
        @enderror
    </div>
    
    <button>Save</button>
</form>
```

> **`old('name')` retrieves the previous input.** `@error('name')` shows the error message for that field. Both work because of the `ShareErrorsFromSession` middleware (from Lesson 14).

---

## Why It Matters

Blade is **fast, simple, and secure by default.** Auto-escaping means you can't accidentally XSS yourself. The component model is intuitive for React developers. Template inheritance keeps your layout DRY.

For a CRUD SaaS, Blade is **the right tool.** You'll only reach for React (via Inertia or Livewire) when you need heavy interactivity — real-time updates, drag-and-drop, complex state.

---

## Lab: Build the Students Views

### Step 1 — Create the layout

Create `resources/views/layouts/app.blade.php`:

```blade
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>@yield('title', 'Coaching SaaS')</title>
    <style>
        body { font-family: system-ui, sans-serif; max-width: 900px; margin: 2rem auto; padding: 0 1rem; }
        nav { background: #f5f5f5; padding: 1rem; border-radius: 8px; margin-bottom: 2rem; }
        nav a { margin-right: 1rem; text-decoration: none; }
        .alert { padding: 1rem; border-radius: 4px; margin-bottom: 1rem; }
        .alert-success { background: #d4edda; color: #155724; }
        .alert-error { background: #f8d7da; color: #721c24; }
        table { width: 100%; border-collapse: collapse; }
        th, td { padding: 0.5rem; text-align: left; border-bottom: 1px solid #eee; }
        .error { color: #dc3545; font-size: 0.875rem; }
        form div { margin-bottom: 1rem; }
        label { display: block; margin-bottom: 0.25rem; font-weight: 500; }
        input { padding: 0.5rem; border: 1px solid #ccc; border-radius: 4px; width: 100%; max-width: 400px; }
        button { background: #007bff; color: white; padding: 0.5rem 1rem; border: none; border-radius: 4px; cursor: pointer; }
    </style>
</head>
<body>
    <nav>
        <a href="{{ route('students.index') }}">Students</a>
    </nav>
    
    @if (session('success'))
        <div class="alert alert-success">{{ session('success') }}</div>
    @endif
    
    @if (session('error'))
        <div class="alert alert-error">{{ session('error') }}</div>
    @endif
    
    <main>
        @yield('content')
    </main>
</body>
</html>
```

### Step 2 — Index view

Create `resources/views/students/index.blade.php`:

```blade
@extends('layouts.app')

@section('title', 'All Students')

@section('content')
    <h1>Students</h1>
    
    <p>
        <a href="{{ route('students.create') }}">+ New Student</a>
    </p>
    
    @if ($students->isEmpty())
        <p>No students yet. <a href="{{ route('students.create') }}">Add the first one</a>.</p>
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
                            <a href="{{ route('students.show', $student) }}">View</a>
                            <a href="{{ route('students.edit', $student) }}">Edit</a>
                        </td>
                    </tr>
                @endforeach
            </tbody>
        </table>
    @endif
@endsection
```

### Step 3 — Update the controller to return views

Edit `app/Http/Controllers/StudentController.php`:

```php
public function index(): View
{
    $students = Student::all();
    return view('students.index', compact('students'));
}

public function create(): View
{
    return view('students.create');
}

public function show(Student $student): View
{
    return view('students.show', compact('student'));
}

public function edit(Student $student): View
{
    return view('students.edit', compact('student'));
}
```

> **Add the `use Illuminate\View\View;` import at the top.**

### Step 4 — Create the form views

Create `resources/views/students/create.blade.php`:

```blade
@extends('layouts.app')

@section('title', 'New Student')

@section('content')
    <h1>New Student</h1>
    
    <form method="POST" action="{{ route('students.store') }}">
        @csrf
        
        <div>
            <label for="name">Name</label>
            <input id="name" name="name" value="{{ old('name') }}" required>
            @error('name')
                <p class="error">{{ $message }}</p>
            @enderror
        </div>
        
        <div>
            <label for="email">Email</label>
            <input id="email" name="email" type="email" value="{{ old('email') }}" required>
            @error('email')
                <p class="error">{{ $message }}</p>
            @enderror
        </div>
        
        <button type="submit">Save</button>
        <a href="{{ route('students.index') }}">Cancel</a>
    </form>
@endsection
```

Create `resources/views/students/edit.blade.php`:

```blade
@extends('layouts.app')

@section('title', 'Edit Student')

@section('content')
    <h1>Edit Student</h1>
    
    <form method="POST" action="{{ route('students.update', $student) }}">
        @csrf
        @method('PUT')
        
        <div>
            <label for="name">Name</label>
            <input id="name" name="name" value="{{ old('name', $student->name) }}" required>
            @error('name')
                <p class="error">{{ $message }}</p>
            @enderror
        </div>
        
        <div>
            <label for="email">Email</label>
            <input id="email" name="email" type="email" value="{{ old('email', $student->email) }}" required>
            @error('email')
                <p class="error">{{ $message }}</p>
            @enderror
        </div>
        
        <button type="submit">Update</button>
        <a href="{{ route('students.show', $student) }}">Cancel</a>
    </form>
@endsection
```

Create `resources/views/students/show.blade.php`:

```blade
@extends('layouts.app')

@section('title', $student->name)

@section('content')
    <h1>{{ $student->name }}</h1>
    
    <p><strong>Email:</strong> {{ $student->email }}</p>
    <p><strong>Created:</strong> {{ $student->created_at->diffForHumans() }}</p>
    
    <a href="{{ route('students.edit', $student) }}">Edit</a>
    
    <form method="POST" action="{{ route('students.destroy', $student) }}" style="display: inline;">
        @csrf
        @method('DELETE')
        <button type="submit" onclick="return confirm('Delete this student?')">Delete</button>
    </form>
    
    <p><a href="{{ route('students.index') }}">Back to list</a></p>
@endsection
```

### Step 5 — Add the store and update methods

Edit `app/Http/Controllers/StudentController.php`:

```php
public function store(Request $request): RedirectResponse
{
    $data = $request->validate([
        'name'  => 'required|string|max:255',
        'email' => 'required|email|max:255',
    ]);
    
    Student::create($data);
    
    return redirect()->route('students.index')
        ->with('success', 'Student added successfully.');
}

public function update(Request $request, Student $student): RedirectResponse
{
    $data = $request->validate([
        'name'  => 'required|string|max:255',
        'email' => 'required|email|max:255',
    ]);
    
    $student->update($data);
    
    return redirect()->route('students.show', $student)
        ->with('success', 'Student updated.');
}

public function destroy(Student $student): RedirectResponse
{
    $student->delete();
    
    return redirect()->route('students.index')
        ->with('success', 'Student deleted.');
}
```

> **Add the `use Illuminate\Http\RedirectResponse;` import.**

### Step 6 — Test in the browser

Start the dev server (if not running):

```bash
php artisan serve
```

Visit:
- `http://127.0.0.1:8000/students` — see the list
- `http://127.0.0.1:8000/students/create` — new form
- Submit, edit, delete — the full CRUD flow

> **Note:** the `email` field will fail validation because we set `unique` in the `tenant_id` scope. We'll fix this with Form Requests in Lesson 19. For now, use unique emails each time.

### Step 7 — Try breaking it

1. Submit the form with an empty name → see validation error
2. Submit the form with an invalid email → see validation error
3. Try a non-existent URL → 404
4. Try POSTing to `/students/1` without CSRF token → 419

> **Each "broken" state teaches you a Laravel invariant.** The framework is opinionated, and the opinions are protecting you.

---

## Quiz

**Q1.** In React, you write `<div>{user.name}</div>`. What's the closest Blade equivalent, and what's the security difference between `{{ }}` and `{!! !!}`?

**Q2.** Why does every POST form in Laravel need `@csrf`? What would happen without it?

**Q3.** A junior writes `Route::post('/students', ...)` and a Blade form with `action="/students"`. They submit and get 419. Why?

**Q4.** You're building a students list page. In React, you'd use a `<table>` component. In Blade, you write `@foreach` directly. What's the philosophical difference, and which approach is better for a CRUD SaaS?

**Q5.** (Stretch) Look at the `@error('name') ... @enderror` block. How does it know there's an error to show? Trace the path from a failed form submission to the error appearing on screen — through which middleware and which session keys.

---

## What You Now Have

✅ Blade templating fluency
✅ Understanding of layout inheritance and components
✅ A working CRUD UI: list, create, show, edit, delete
✅ CSRF protection built in
✅ Validation errors rendered automatically

## What's Next

Lesson 17 covers **Eloquent** — Laravel's ORM. This is where you'll spend most of your time. Models, relationships, queries, eager loading, and the patterns that make Laravel data access elegant.

📖 **[Lesson 17 — Eloquent ORM: The Active Record Pattern](17-eloquent-orm-the-active-record-pattern.md)**
