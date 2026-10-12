# Lesson 42 — API Development vs Web Routes

> **Phase 5: Production** | You'll be here: ~30 min

## The Senior's Question

You have a working web app. A customer says "I want a mobile app." Or "I want Zapier to integrate." Or "I want our internal system to push data." Now what? The senior knows **when to expose a JSON API, when to stay web-only, and how to design REST endpoints that age well.**

---

## Theory: The Three Surfaces

A Laravel app can have multiple "surfaces" that hit the same business logic:

```
                    ┌─────────────┐
                    │ Domain Code │
                    │  (Actions,  │
                    │   Models)   │
                    └──────┬──────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
   ┌────▼─────┐      ┌─────▼─────┐      ┌─────▼─────┐
   │   Web    │      │    API    │      │   CLI     │
   │ (Blade)  │      │  (JSON)   │      │ (Artisan) │
   └──────────┘      └───────────┘      └───────────┘
   HTML pages         REST endpoints      Commands
   Session auth       Token auth          No auth needed
   CSRF protection    CORS                Programmatic
```

> **Senior rule:** the domain (models, actions) is the source of truth. The web/API/CLI are just different ways to expose it.

---

## Theory: When to Add an API

### Add an API when:

- **Mobile app** (iOS, Android, React Native) — needs HTTP/JSON
- **Third-party integration** (Zapier, partners) — programmatic access
- **SPA frontend** (React, Vue) on a different domain — same-origin vs cross-origin
- **Webhooks** to other services (Stripe, Slack) — they call you
- **Public data** (a directory of coaching centers, public stats)

### Skip the API when:

- **Server-rendered only** (Blade) — no need
- **No external consumers** — over-engineering
- **Just one internal dashboard** — web routes are enough

> **Senior rule:** "we might need an API someday" is not a reason to build one. Build when there's a real consumer.**

---

## Theory: Laravel's API Routes

Laravel separates web and API routes:

```php
// routes/web.php — session auth, CSRF
Route::middleware('web')->group(function () {
    Route::resource('students', StudentController::class);
});

// routes/api.php — token auth, JSON
Route::middleware('api')->group(function () {
    Route::resource('students', Api\StudentController::class);
});
```

`routes/api.php` is automatically prefixed with `/api`. So `/api/students`.

Install API support in Laravel 11+:

```bash
php artisan install:api
```

This creates `routes/api.php` and Sanctum for token auth.

---

## Theory: API Resources (The Senior's Output Format)

Don't return models directly. Use API Resources:

```bash
php artisan make:resource StudentResource
```

`app/Http/Resources/StudentResource.php`:

```php
class StudentResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            'student_id' => $this->student_id,
            'class' => $this->whenLoaded('class', fn() => [
                'id' => $this->class->id,
                'name' => $this->class->name,
            ]),
            'is_active' => $this->is_active,
            'created_at' => $this->created_at->toIso8601String(),
            'guardian' => $this->whenLoaded('guardian', fn() => new GuardianResource($this->guardian)),
        ];
    }
}
```

> **API Resources are your contract.** They define what the API returns. Hide sensitive fields, format dates, transform data — here, not in the model.

Use it:

```php
public function show(Student $student)
{
    return new StudentResource($student->load(['class', 'guardian']));
}

public function index()
{
    return StudentResource::collection(
        Student::with(['class', 'guardian'])->paginate(20)
    );
}
```

---

## Theory: API Authentication (Sanctum)

Laravel Sanctum is the senior's choice for API auth. Three modes:

1. **SPA cookie auth** — same-domain frontend, no token
2. **API tokens** — first-party mobile apps, integrations
3. **Personal access tokens** — for users, manually issued

### Install

```bash
composer require laravel/sanctum
php artisan install:api  # or php artisan vendor:publish and migrate
```

### Add to User model

```php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens;
}
```

### Issue a token

```php
$token = $user->createToken('mobile-app')->plainTextToken;
```

The user gets back a string like `1|abc123...`. They send it as `Authorization: Bearer 1|abc123...`.

### Protect routes

```php
Route::middleware('auth:sanctum')->group(function () {
    Route::resource('students', Api\StudentController::class);
});
```

> **Sanctum is the senior's default.** JWT adds complexity. OAuth2 is for "log in with Google." Sanctum is for "I trust this user."

---

## Theory: RESTful Design

The 7 REST principles a senior follows:

### 1. Resources, not RPC

```
# Bad — RPC style
POST /api/createStudent
POST /api/updateStudent/5

# Good — REST style
POST   /api/students
PUT    /api/students/5
DELETE /api/students/5
```

### 2. Plural nouns

```
GET    /api/students       — list
GET    /api/students/5     — show
POST   /api/students       — create
PUT    /api/students/5     — update
DELETE /api/students/5     — delete
```

### 3. Use HTTP status codes

```
200 OK                — success, returning data
201 Created           — resource created
204 No Content        — success, no body
400 Bad Request       — validation error
401 Unauthorized      — not authenticated
403 Forbidden         — not allowed
404 Not Found         — resource doesn't exist
422 Unprocessable     — semantic error (validation)
500 Server Error      — bug
```

### 4. Use HTTP verbs correctly

- `GET` — safe, idempotent, no side effects
- `POST` — create
- `PUT`/`PATCH` — update
- `DELETE` — delete

### 5. Use query params for filtering

```
GET /api/students?is_active=true&class_id=5
GET /api/students?sort=name&order=asc
GET /api/students?per_page=50
```

### 6. Return pagination metadata

```json
{
  "data": [...],
  "meta": {
    "current_page": 1,
    "per_page": 20,
    "total": 145
  },
  "links": {
    "next": "...",
    "prev": "..."
  }
}
```

Laravel paginators do this automatically.

### 7. Version your API

```
/api/v1/students
/api/v2/students
```

> **Don't break v1 once shipped.** Add v2, deprecate v1, give users 6 months.

---

## Theory: Form Requests (Validation)

Use Form Requests for API too:

```bash
php artisan make:request StoreStudentRequest
```

```php
public function rules(): array
{
    return [
        'name' => 'required|string|max:255',
        'email' => ['required', 'email', Rule::unique('students')->where('tenant_id', app('current_tenant')->id)],
        'class_id' => 'nullable|exists:school_classes,id',
    ];
}
```

> **Form Requests** handle validation AND authorization (`authorize()` method).

Laravel auto-returns 422 with errors on validation failure:

```json
{
  "message": "The given data was invalid.",
  "errors": {
    "email": ["This email is already taken."]
  }
}
```

---

## Theory: API Controllers (Slim, Action-Oriented)

`app/Http/Controllers/Api/StudentController.php`:

```php
class StudentController extends Controller
{
    public function index(Request $request)
    {
        $tenant = app('current_tenant');
        
        $students = Student::where('tenant_id', $tenant->id)
            ->with('class')
            ->when($request->search, fn($q) => $q->where('name', 'like', "%{$request->search}%"))
            ->when($request->class_id, fn($q) => $q->where('class_id', $request->class_id))
            ->paginate($request->per_page ?? 20);
        
        return StudentResource::collection($students);
    }
    
    public function store(StoreStudentRequest $request)
    {
        $student = app(CreateStudent::class)->execute($request->validated());
        
        return (new StudentResource($student))
            ->response()
            ->setStatusCode(201);
    }
    
    public function show(Student $student)
    {
        $this->authorize('view', $student);
        
        return new StudentResource($student->load('class', 'guardian'));
    }
    
    public function update(UpdateStudentRequest $request, Student $student)
    {
        $this->authorize('update', $student);
        
        app(UpdateStudent::class)->execute($student, $request->validated());
        
        return new StudentResource($student->fresh());
    }
    
    public function destroy(Student $student)
    {
        $this->authorize('delete', $student);
        
        $student->delete();
        
        return response()->noContent();
    }
}
```

> **API controllers are slim.** They validate, call actions, return resources. The action does the work.

---

## Theory: Error Handling (Consistent Errors)

In `bootstrap/app.php` (Laravel 11+):

```php
->withExceptions(function (Exceptions $exceptions) {
    $exceptions->render(function (ModelNotFoundException $e, $request) {
        if ($request->is('api/*')) {
            return response()->json(['message' => 'Not found'], 404);
        }
    });
    
    $exceptions->render(function (ValidationException $e, $request) {
        if ($request->is('api/*')) {
            return response()->json([
                'message' => $e->getMessage(),
                'errors' => $e->errors(),
            ], 422);
        }
    });
    
    $exceptions->render(function (AuthenticationException $e, $request) {
        if ($request->is('api/*')) {
            return response()->json(['message' => 'Unauthenticated'], 401);
        }
    });
})
```

> **Consistent error format.** Every error returns JSON for API clients.

---

## Theory: CORS (Cross-Origin)

If your API is called from a different domain (frontend on app.com, API on api.com), you need CORS:

```bash
php artisan install:api  # already installs Sanctum
```

Configure in `config/cors.php`:

```php
'paths' => ['api/*'],
'allowed_origins' => ['https://app.coaching-saas.com'],
'allowed_methods' => ['*'],
'allowed_headers' => ['*'],
```

> **Never set `allowed_origins: ['*']` in production with credentials.** That's a security hole.

---

## Theory: Rate Limiting

Protect your API from abuse:

`bootstrap/app.php`:

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->throttleApi('60,1');  // 60 requests per minute
})
```

Or per-user:

```php
RateLimiter::for('api', function (Request $request) {
    return $request->user()
        ? Limit::perMinute(1000)->by($request->user()->id)
        : Limit::perMinute(60)->by($request->ip());
});
```

> **Senior rule:** rate-limit everything. Default: 60/min for anonymous, 1000/min for authenticated.

---

## Theory: API Documentation (OpenAPI/Swagger)

Document your API so consumers can use it:

```bash
composer require darkaonline/l5-swagger
```

Or use **Scribe** (more popular for Laravel):

```bash
composer require knuckleswtf/scribe
php artisan vendor:publish --tag=scribe-config
```

Annotate controllers:

```php
/**
 * Get all students
 *
 * @group Student management
 *
 * @queryParam is_active bool Filter by active status. Example: true
 * @queryParam per_page int Results per page. Example: 50
 *
 * @authenticated
 */
public function index(Request $request) { ... }
```

Generate:

```bash
php artisan scribe:generate
```

Visit `/docs` for interactive API documentation.

> **Senior rule:** documentation is a feature. Every endpoint documented. Updated on every change.

---

## Theory: When to Use Webhooks Instead of API

If a third party needs to be **notified** of events, use webhooks. They don't need to call you.

```
Event happens in your app
  ↓
Your app → POSTs to their URL
  ↓
Their system processes it
```

```php
class WebhookDispatcher
{
    public function send(Tenant $tenant, string $event, array $payload): void
    {
        $url = $tenant->webhook_url;
        if (!$url) return;
        
        Http::post($url, [
            'event' => $event,
            'data' => $payload,
            'timestamp' => now()->toIso8601String(),
        ])->throw();
    }
}
```

> **Senior rule:** always offer webhooks. They're push instead of poll. Easier for the consumer.

---

## Theory: Idempotency

Mobile networks are unreliable. The same request might fire twice. **Idempotency keys** prevent double-processing:

```php
public function store(StoreStudentRequest $request)
{
    $key = $request->header('Idempotency-Key');
    
    if ($key && Cache::has("idempotency:{$key}")) {
        return response()->json(Cache::get("idempotency:{$key}"));
    }
    
    $student = app(CreateStudent::class)->execute($request->validated());
    $response = (new StudentResource($student))->response()->setStatusCode(201);
    
    if ($key) {
        Cache::put("idempotency:{$key}", $response->getData(true), 3600);
    }
    
    return $response;
}
```

> **Idempotency keys** are how Stripe handles the "user double-clicks Pay" problem.

---

## Theory: API Versioning Strategy

### URL versioning (most common, recommended)

```
/api/v1/students
/api/v2/students
```

```php
// routes/api.php
Route::prefix('v1')->group(function () {
    require base_path('routes/api_v1.php');
});

Route::prefix('v2')->group(function () {
    require base_path('routes/api_v2.php');
});
```

### Header versioning (more "pure" REST)

```
Accept: application/vnd.coaching.v2+json
```

> **URL versioning is simpler.** Header versioning is more "correct" but harder to debug.

---

## Why It Matters

API decisions affect:
- Mobile apps (1-year release cycle, can't break)
- Third-party integrations (1-day release cycle, can break)
- Future you (will this scale?)

> **A senior designs APIs as contracts.** Once shipped, they're hard to change. Internal code can refactor; public APIs need deprecation cycles.

---

## Lab: Add a JSON API to Your SaaS

### Step 1 — Install API support

```bash
cd G:\php-laravel\coaching-saas
php artisan install:api
```

This installs Sanctum and creates `routes/api.php`.

### Step 2 — Add API resources

```bash
php artisan make:resource StudentResource
php artisan make:resource GuardianResource
```

### Step 3 — Configure the resource

`app/Http/Resources/StudentResource.php`:

```php
class StudentResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id' => $this->id,
            'student_id' => $this->student_id,
            'name' => $this->name,
            'email' => $this->email,
            'phone' => $this->phone,
            'is_active' => $this->is_active,
            'class' => new \App\Http\Resources\SchoolClassResource($this->whenLoaded('class')),
            'guardian' => new \App\Http\Resources\GuardianResource($this->whenLoaded('guardian')),
            'created_at' => $this->created_at?->toIso8601String(),
        ];
    }
}
```

### Step 4 — Create the API controller

```bash
php artisan make:controller Api/StudentController
```

```php
namespace App\Http\Controllers\Api;

use App\Actions\CreateStudent;
use App\Actions\UpdateStudent;
use App\Http\Requests\StoreStudentRequest;
use App\Http\Requests\UpdateStudentRequest;
use App\Http\Resources\StudentResource;
use App\Models\Student;
use Illuminate\Http\Request;

class StudentController extends \App\Http\Controllers\Controller
{
    public function index(Request $request)
    {
        $tenant = app('current_tenant');
        
        $students = Student::where('tenant_id', $tenant->id)
            ->with(['class', 'guardian'])
            ->when($request->search, fn($q) => $q->where('name', 'like', "%{$request->search}%"))
            ->when($request->class_id, fn($q) => $q->where('class_id', $request->class_id))
            ->when($request->is_active !== null, fn($q) => $q->where('is_active', $request->boolean('is_active')))
            ->paginate($request->integer('per_page') ?: 20);
        
        return StudentResource::collection($students);
    }
    
    public function store(StoreStudentRequest $request)
    {
        $student = app(CreateStudent::class)->execute(
            $request->validated() + ['tenant_id' => app('current_tenant')->id]
        );
        
        return (new StudentResource($student->load(['class', 'guardian'])))
            ->response()
            ->setStatusCode(201);
    }
    
    public function show(Student $student)
    {
        $this->authorize('view', $student);
        
        return new StudentResource($student->load(['class', 'guardian']));
    }
    
    public function update(UpdateStudentRequest $request, Student $student)
    {
        $this->authorize('update', $student);
        
        app(UpdateStudent::class)->execute($student, $request->validated());
        
        return new StudentResource($student->fresh()->load(['class', 'guardian']));
    }
    
    public function destroy(Student $student)
    {
        $this->authorize('delete', $student);
        
        $student->delete();
        
        return response()->noContent();
    }
}
```

### Step 5 — Define API routes

`routes/api.php`:

```php
use App\Http\Controllers\Api\StudentController;

Route::middleware('auth:sanctum')->group(function () {
    Route::apiResource('students', StudentController::class);
});
```

### Step 6 — Test the API

```bash
# Get a token
php artisan tinker
>>> $user = User::first();
>>> $token = $user->createToken('test')->plainTextToken;
=> "1|abc123..."

# Test
curl -H "Authorization: Bearer 1|abc123..." \
     -H "Accept: application/json" \
     http://localhost:8000/api/students
```

### Step 7 — Configure CORS

`config/cors.php`:

```php
'paths' => ['api/*'],
'allowed_origins' => explode(',', env('API_ALLOWED_ORIGINS', 'http://localhost:3000')),
'allowed_methods' => ['*'],
'allowed_headers' => ['*'],
'supports_credentials' => true,
```

`.env`:

```env
API_ALLOWED_ORIGINS=http://localhost:3000,https://app.coaching-saas.com
```

### Step 8 — Add rate limiting

`bootstrap/app.php`:

```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;

RateLimiter::for('api', function (Request $request) {
    return $request->user()
        ? Limit::perMinute(1000)->by($request->user()->id)
        : Limit::perMinute(60)->by($request->ip());
});
```

Apply:

```php
Route::middleware(['auth:sanctum', 'throttle:api'])->group(function () {
    Route::apiResource('students', StudentController::class);
});
```

### Step 9 — Document with Scribe

```bash
composer require knuckleswtf/scribe
php artisan vendor:publish --tag=scribe-config
```

Annotate the controller methods. Run `php artisan scribe:generate`. Visit `/docs`.

### Step 10 — Add API tests

`tests/Feature/Api/StudentApiTest.php`:

```php
test('authenticated user can list students', function () {
    $user = User::factory()->for($this->tenant)->create();
    
    Student::factory()->for($this->tenant)->count(3)->create();
    
    $response = $this->actingAs($user, 'sanctum')
        ->getJson('/api/students');
    
    $response->assertOk()
        ->assertJsonCount(3, 'data');
});

test('unauthenticated request returns 401', function () {
    $this->getJson('/api/students')->assertUnauthorized();
});

test('store validates required fields', function () {
    $user = User::factory()->for($this->tenant)->create();
    
    $this->actingAs($user, 'sanctum')
        ->postJson('/api/students', [])
        ->assertStatus(422)
        ->assertJsonValidationErrors(['name', 'email']);
});
```

---

## Quiz

**Q1.** You have a server-rendered web app. A customer asks for a mobile app. What's the minimum API surface you'd build first?

**Q2.** Why use API Resources instead of `return $student`? List 2 specific benefits.

**Q3.** A client says "the API returned 500." You check your code — there's no try/catch. What should happen globally for API exceptions?

**Q4.** You want to add a new field to the Student API. Some clients are still on v1. How do you handle this without breaking them?

**Q5.** (Stretch) Your API is hit 10,000 times per second. The DB is the bottleneck. Walk through your scaling plan (caching, queues, replicas).

---

## What You Now Have

✅ When to add an API (and when not to)
✅ Sanctum for token auth
✅ API Resources for output
✅ RESTful design
✅ Form Requests for validation
✅ CORS, rate limiting, error handling
✅ API documentation with Scribe
✅ Idempotency for safety

## What's Next

**Phase 5 complete!** You now have a production-grade SaaS with caching, billing, email, storage, monitoring, search, and APIs.

**Phase 6: Deploy & Scale** begins with **environments** — dev, staging, production, the configuration discipline that separates "works on my machine" from "works in prod."

📖 **[Phase 6 starts: Lesson 43 — Environments: Dev, Staging, Production](phase-6-deploy-scale/43-environments-dev-staging-production.md)**
