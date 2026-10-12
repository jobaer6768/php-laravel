# Lesson 19 — Form Requests, Validation, and Authorization

> **Phase 2: Laravel Core** | You'll be here: ~40 min

## The Senior's Question

A user submits a form. What should your controller do? The senior's answer: **as little as possible.** Validation, authorization, sanitization — all of it happens *before* the controller method runs, in a dedicated class. The controller is just "do the work, return the response."

---

## Theory: The Three Layers of Request Processing

```
Request comes in
   ↓
1. MIDDLEWARE (auth, throttle, tenant)        — Is the request allowed?
   ↓
2. FORM REQUEST (validation, authorization)   — Is the input valid? Is this user allowed?
   ↓
3. CONTROLLER (business logic)                — Do the work.
   ↓
Response
```

A senior **separates these concerns.** Middleware knows about infrastructure. Form requests know about input and rules. Controllers know about business logic.

---

## Theory: Form Request Classes

```bash
php artisan make:request StoreStudentRequest
```

```php
<?php
// app/Http/Requests/StoreStudentRequest.php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class StoreStudentRequest extends FormRequest
{
    /**
     * Is the user authorized to make this request?
     */
    public function authorize(): bool
    {
        return $this->user()->can('create', \App\Models\Student::class);
    }

    /**
     * What are the validation rules?
     */
    public function rules(): array
    {
        return [
            'name'     => ['required', 'string', 'max:255'],
            'email'    => [
                'required',
                'email',
                'max:255',
                Rule::unique('students', 'email')
                    ->where('tenant_id', $this->user()->tenant_id),
            ],
            'class_id' => ['nullable', 'exists:school_classes,id'],
            'phone'    => ['nullable', 'string', 'max:20'],
        ];
    }

    /**
     * Custom error messages (optional)
     */
    public function messages(): array
    {
        return [
            'email.unique' => 'A student with this email already exists in your institution.',
        ];
    }
}
```

In the controller:

```php
public function store(StoreStudentRequest $request): RedirectResponse
{
    // If we reach here, the request is:
    // 1. Authorized (authorize() returned true)
    // 2. Validated (all rules passed)
    // 3. The user is redirected back with errors if not

    Student::create($request->validated());
    return redirect()->route('students.index');
}
```

> **Notice:** the controller method does not call `validate()`. The Form Request does it automatically because Laravel sees the type-hint and **injects the request after running it through validation.**

If validation fails:
- Web request: redirects back with errors in the session
- JSON request: returns 422 with the error array

> **Senior rule:** every form has a Form Request. Every Form Request has `authorize()` and `rules()`. No exceptions.

---

## Theory: Validation Rules — The Common Ones

### Type checks

```php
'string'              // is a string
'integer'             // is an int
'boolean'             // is a bool
'array'               // is an array
'nullable'            // null allowed
'required'            // must be present and not empty
'sometimes'           // only validate if present
```

### String constraints

```php
'email'               // valid email
'url'                 // valid URL
'min:5'               // min length 5
'max:255'             // max length 255
'size:10'             // exactly 10
'between:5,10'        // length 5-10
'regex:/^[A-Z]+$/'    // matches regex
'in:foo,bar,baz'      // in this list
'not_in:foo,bar'      // not in this list
```

### Number constraints

```php
'numeric'             // is numeric
'integer'             // is int
'min:0'               // min value 0
'max:100'             // max value 100
'gt:0'                // greater than 0
'gte:0'               // greater than or equal
'lt:100'              // less than
'lte:100'             // less than or equal
```

### Database constraints

```php
'unique:students,email'                              // unique in table
'unique:students,email,NULL,id,tenant_id,1'         // unique except for row id=1, where tenant_id=1
'exists:school_classes,id'                          // must exist in school_classes.id
```

The `Rule::` builder makes complex rules readable:

```php
use Illuminate\Validation\Rule;

Rule::unique('students', 'email')
    ->where('tenant_id', $tenantId)
    ->ignore($studentId);  // exclude self when updating

Rule::exists('school_classes', 'id')
    ->where('tenant_id', $tenantId);
```

### File constraints

```php
'file'                // is an uploaded file
'image'               // is an image (jpeg, png, bmp, gif, svg, webp)
'mimes:jpg,png'       // specific MIME types
'max:2048'            // max 2MB (in kilobytes)
```

### Date constraints

```php
'date'                // is a date
'date_format:Y-m-d'   // matches format
'after:2026-01-01'    // after this date
'before:2026-12-31'   // before this date
```

---

## Theory: Custom Validation Rules

```bash
php artisan make:rule ValidBangladeshiPhone
```

```php
// app/Rules/ValidBangladeshiPhone.php
namespace App\Rules;

use Closure;
use Illuminate\Contracts\Validation\ValidationRule;

class ValidBangladeshiPhone implements ValidationRule
{
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        // Bangladeshi phone: +8801XXXXXXXXX or 01XXXXXXXXX (11 digits after country code)
        if (!preg_match('/^(\+880|0)1[3-9]\d{8}$/', $value)) {
            $fail('The :attribute is not a valid Bangladeshi phone number.');
        }
    }
}
```

Use it:

```php
use App\Rules\ValidBangladeshiPhone;

public function rules(): array
{
    return [
        'phone' => ['required', new ValidBangladeshiPhone],
    ];
}
```

> **Senior rule:** if you have complex validation that doesn't fit a one-liner, make a custom rule class. Keeps your Form Request clean.

---

## Theory: Conditional Validation

```php
public function rules(): array
{
    return [
        'name'     => ['required', 'string'],
        'email'    => ['required', 'email'],
        'guardian' => ['required_if:is_minor,true'],
        'discount' => ['required_if:has_coupon,true', 'numeric', 'min:0', 'max:50'],
    ];
}
```

Or programmatically:

```php
public function rules(): array
{
    $rules = ['name' => 'required', 'email' => 'required|email'];
    
    if ($this->has_coupon) {
        $rules['coupon_code'] = ['required', 'string', 'exists:coupons,code'];
    }
    
    return $rules;
}
```

> **Use `withValidator()` for cross-field validation:**

```php
public function withValidator($validator): void
{
    $validator->after(function ($validator) {
        if ($this->start_date && $this->end_date && $this->start_date > $this->end_date) {
            $validator->errors()->add('end_date', 'End date must be after start date.');
        }
    });
}
```

---

## Theory: Authorization — Three Layers

### Layer 1: Middleware (route-level)

```php
Route::middleware(['auth', 'can:viewAny,App\Models\Student'])->group(function () {
    Route::resource('students', StudentController::class);
});
```

The `can` middleware checks a Gate or Policy before the request even hits the controller.

### Layer 2: Form Request `authorize()` (request-level)

We saw this above. It's a method that returns bool: "is the current user allowed to make this request?"

### Layer 3: Policy (model-level)

```bash
php artisan make:policy StudentPolicy --model=Student
```

```php
// app/Policies/StudentPolicy.php
namespace App\Policies;

use App\Models\Student;
use App\Models\User;

class StudentPolicy
{
    public function viewAny(User $user): bool
    {
        return $user->isInstitutionAdmin();
    }

    public function view(User $user, Student $student): bool
    {
        return $user->tenant_id === $student->tenant_id;
    }

    public function create(User $user): bool
    {
        return $user->canManageStudents();
    }

    public function update(User $user, Student $student): bool
    {
        return $user->tenant_id === $student->tenant_id
            && ($user->isTeacher() || $user->isAdmin());
    }

    public function delete(User $user, Student $student): bool
    {
        return $user->tenant_id === $student->tenant_id
            && $user->isAdmin();
    }
}
```

Laravel **auto-discovers** this policy for the `Student` model. Use it in your controller:

```php
public function update(UpdateStudentRequest $request, Student $student): RedirectResponse
{
    $this->authorize('update', $student);  // throws 403 if not allowed

    $student->update($request->validated());
    return redirect()->route('students.show', $student);
}
```

Or in a Blade view:

```blade
@can('update', $student)
    <a href="{{ route('students.edit', $student) }}">Edit</a>
@endcan

@cannot('delete', $student)
    <p>You don't have permission to delete this student.</p>
@endcannot
```

> **Senior rule:** authorization is **always at the boundary, never inside the controller.** Don't sprinkle `if` checks. Use Policies.

---

## Theory: Gates (for actions that aren't tied to a model)

```php
// app/Providers/AuthServiceProvider.php or via the boot() method of AppServiceProvider
use Illuminate\Support\Facades\Gate;

Gate::define('access-admin-panel', function (User $user) {
    return $user->is_admin && $user->tenant_id !== null;
});
```

Use it:

```php
if (Gate::allows('access-admin-panel')) {
    // ...
}

if (Gate::denies('access-admin-panel')) {
    abort(403);
}
```

Or via the `can` middleware:

```php
Route::middleware('can:access-admin-panel')->group(function () { ... });
```

> **Use Policies for model-related actions. Use Gates for everything else.**

---

## Theory: Combining Form Request + Policy

The cleanest pattern:

```php
class StoreStudentRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('create', Student::class);
    }

    public function rules(): array
    {
        return [
            'name'  => 'required|string|max:255',
            'email' => [
                'required', 'email',
                Rule::unique('students')->where('tenant_id', $this->user()->tenant_id),
            ],
        ];
    }
}
```

> **Now:** the Form Request is the single source of truth for "is this input OK and is this user allowed to submit it." The controller just trusts it.

---

## Why It Matters

Without Form Requests, controllers look like this:

```php
// Junior code
public function store(Request $request) {
    $validated = $request->validate([
        'name' => 'required',
        'email' => 'required|email',
    ]);
    
    if (!auth()->user()->can('create', Student::class)) {
        abort(403);
    }
    
    Student::create($validated);
    return redirect()->route('students.index');
}
```

With Form Requests:

```php
// Senior code
public function store(StoreStudentRequest $request) {
    Student::create($request->validated());
    return redirect()->route('students.index');
}
```

> **The senior code is 3 lines and self-evident.** The junior code is 9 lines and has the validation, auth, and creation all jumbled. **Same behavior, very different quality.**

---

## Lab: Replace Inline Validation

### Step 1 — Create the Form Request classes

```bash
cd G:\php-laravel\coaching-saas
php artisan make:request StoreStudentRequest
php artisan make:request UpdateStudentRequest
```

### Step 2 — Fill in StoreStudentRequest

Edit `app/Http/Requests/StoreStudentRequest.php`:

```php
<?php
namespace App\Http\Requests;

use App\Models\Student;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class StoreStudentRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;  // we'll wire up real auth in Lesson 25
    }

    public function rules(): array
    {
        return [
            'name'     => ['required', 'string', 'max:255'],
            'email'    => [
                'required', 'email', 'max:255',
                Rule::unique('students', 'email')->where('tenant_id', 1),  // hardcoded for now
            ],
            'class_id' => ['nullable', 'exists:school_classes,id'],
            'phone'    => ['nullable', 'string', 'max:20'],
        ];
    }

    public function messages(): array
    {
        return [
            'email.unique' => 'A student with this email already exists.',
        ];
    }
}
```

### Step 3 — Fill in UpdateStudentRequest

```php
<?php
namespace App\Http\Requests;

use App\Models\Student;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class UpdateStudentRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        $studentId = $this->route('student')->id;  // current student being updated

        return [
            'name'     => ['required', 'string', 'max:255'],
            'email'    => [
                'required', 'email', 'max:255',
                Rule::unique('students', 'email')
                    ->ignore($studentId)
                    ->where('tenant_id', 1),
            ],
            'class_id' => ['nullable', 'exists:school_classes,id'],
            'phone'    => ['nullable', 'string', 'max:20'],
        ];
    }
}
```

### Step 4 — Update the controller

Edit `app/Http/Controllers/StudentController.php`:

```php
public function store(StoreStudentRequest $request): RedirectResponse
{
    $data = $request->validated();
    $data['tenant_id'] = 1;  // hardcoded for now

    Student::create($data);

    return redirect()->route('students.index')
        ->with('success', 'Student added.');
}

public function update(UpdateStudentRequest $request, Student $student): RedirectResponse
{
    $student->update($request->validated());

    return redirect()->route('students.show', $student)
        ->with('success', 'Student updated.');
}
```

> **Compare to before:** the controller went from doing validation to just trusting `$request->validated()`. The Form Request is the gatekeeper.

### Step 5 — Add a custom rule

```bash
php artisan make:rule ValidBangladeshiPhone
```

Edit `app/Rules/ValidBangladeshiPhone.php`:

```php
<?php
namespace App\Rules;

use Closure;
use Illuminate\Contracts\Validation\ValidationRule;

class ValidBangladeshiPhone implements ValidationRule
{
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        if (!preg_match('/^(\+880|0)1[3-9]\d{8}$/', $value)) {
            $fail("The :attribute must be a valid Bangladeshi phone number (e.g. 01712345678).");
        }
    }
}
```

Add it to the Form Request:

```php
use App\Rules\ValidBangladeshiPhone;

public function rules(): array
{
    return [
        // ...
        'phone' => ['nullable', 'string', 'max:20', new ValidBangladeshiPhone],
    ];
}
```

### Step 6 — Test the validation

Try submitting:
- Empty name → see "The name field is required"
- Invalid email → see "The email must be a valid email address"
- Duplicate email → see the custom message
- Invalid phone → see the custom rule's message
- Bad class_id → see "The selected class id is invalid"

All errors display in the form via `@error('name') ... @enderror`. **The form is now production-quality.**

### Step 7 — Generate a Policy

```bash
php artisan make:policy StudentPolicy --model=Student
```

Edit `app/Policies/StudentPolicy.php`:

```php
<?php
namespace App\Policies;

use App\Models\Student;
use App\Models\User;

class StudentPolicy
{
    public function viewAny(?User $user): bool
    {
        return $user !== null;
    }

    public function view(?User $user, Student $student): bool
    {
        return $user !== null && $user->tenant_id === $student->tenant_id;
    }

    public function create(?User $user): bool
    {
        return $user !== null;
    }

    public function update(?User $user, Student $student): bool
    {
        return $user !== null && $user->tenant_id === $student->tenant_id;
    }

    public function delete(?User $user, Student $student): bool
    {
        return $user !== null && $user->tenant_id === $student->tenant_id;
    }
}
```

> **Note: we'll wire `User` to have a real `tenant_id` in Phase 3 (multi-tenancy). For now, the structure is correct.**

### Step 8 — Use the policy in the controller

Add to the controller methods:

```php
public function show(Student $student): View
{
    $this->authorize('view', $student);
    return view('students.show', compact('student'));
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

> **The `authorize()` method throws `AuthorizationException` (becomes 403) if not allowed.** This is the cleanest way to enforce permissions.

### Step 9 — Use the policy in the Blade views

Edit `resources/views/students/index.blade.php`:

```blade
@can('create', App\Models\Student::class)
    <a href="{{ route('students.create') }}">+ New Student</a>
@endcan
```

Edit `resources/views/students/show.blade.php`:

```blade
@can('update', $student)
    <a href="{{ route('students.edit', $student) }}">Edit</a>
@endcan

@can('delete', $student)
    <form method="POST" action="{{ route('students.destroy', $student) }}" style="display: inline;">
        @csrf
        @method('DELETE')
        <button type="submit" onclick="return confirm('Delete?')">Delete</button>
    </form>
@endcan
```

> **The `@can` directives check the policy automatically.** Buttons hide if the user can't do the action.

---

## Quiz

**Q1.** What's the difference between `$request->validate([...])` in a controller and a Form Request class? When would you use each?

**Q2.** Why is the `Rule::unique('students')->where('tenant_id', ...)` pattern critical for a multi-tenant SaaS? What bug does it prevent?

**Q3.** When would you use a Gate vs a Policy? Give a concrete example of each.

**Q4.** You have a Form Request. A user submits invalid data. What HTTP response do they get on a web request vs a JSON request? Where is this configured?

**Q5.** (Stretch) A junior puts authorization logic in the Blade template (`@if (auth()->user()->is_admin) ... @endif`). A senior uses a Policy. Why is the senior's approach better, and what bug does the junior's approach risk?

---

## What You Now Have

✅ Form Request classes for validation
✅ Authorization: middleware, Form Request, Policy, Gate
✅ Custom validation rules
✅ Tenant-aware uniqueness
✅ `@can` directives in Blade

## What's Next

Lesson 20 covers **middleware** — the pipeline that filters every request. You'll learn to write your own middleware for tenant scoping, logging, and request signing.

📖 **[Lesson 20 — Middleware: The Pipeline Pattern](20-middleware-the-pipeline-pattern.md)**
