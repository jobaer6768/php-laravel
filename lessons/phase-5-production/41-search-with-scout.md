# Lesson 41 — Search with Scout

> **Phase 5: Production** | You'll be here: ~30 min

## The Senior's Question

A tenant searches for "Aisha" in the students list. With 5,000 students, `WHERE name LIKE '%aisha%'` takes 800ms. The senior uses **Laravel Scout** — full-text search engine that returns results in 20ms with typo tolerance, ranking, and faceting. Let me show you.

---

## Theory: Why `LIKE` Queries Fail

```sql
SELECT * FROM students WHERE name LIKE '%aisha%';
```

This:
- Scans every row (no index can help with leading `%`)
- Doesn't rank by relevance
- Doesn't tolerate typos ("aish" should match "Aisha")
- Doesn't handle multi-word search well
- Doesn't consider word stems ("running" should match "run")

> **Senior rule:** SQL is for exact matches. Search is for relevance. **Use a dedicated search engine for "fuzzy" search.**

---

## Theory: Laravel Scout

Scout is Laravel's official search abstraction. It supports:

- **Algolia** — paid SaaS, best DX
- **Meilisearch** — open source, fast, lightweight
- **Typesense** — open source, similar to Algolia
- **Database** — basic, no extra service, but limited

> **Senior choice:** Meilisearch. Open source, fast, easy to run, generous free tier in cloud.

---

## Theory: Install Meilisearch

### Local (Docker)

```bash
docker run -d --name meilisearch \
  -p 7700:7700 \
  -e MEILI_MASTER_KEY=masterKey \
  -v $(pwd)/meili_data:/meili_data \
  getmeili/meilisearch:v1.6
```

### Production

Use Meilisearch Cloud (free tier: 100k docs) or self-host on a $5 VPS.

Verify it's running:

```bash
curl http://localhost:7700/health
# {"status":"available"}
```

---

## Theory: Install Scout

```bash
composer require laravel/scout
php artisan vendor:publish --provider="Laravel\Scout\ScoutServiceProvider"
```

In `.env`:

```env
SCOUT_DRIVER=meilisearch
SCOUT_QUEUE=true
MEILISEARCH_HOST=http://127.0.0.1:7700
MEILISEARCH_KEY=masterKey
```

> **`SCOUT_QUEUE=true`** makes indexing async. Indexing 10,000 records doesn't block the request.

---

## Theory: Make a Model Searchable

```php
use Laravel\Scout\Searchable;

class Student extends Model
{
    use Searchable;
    
    public function toSearchableArray(): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            'student_id' => $this->student_id,
            'class_name' => $this->class?->name,
            'guardian_name' => $this->guardian?->name,
            'is_active' => $this->is_active,
            'tenant_id' => $this->tenant_id,
        ];
    }
}
```

> **`toSearchableArray()`** is what gets indexed. Include the fields you want to search, plus any fields you'll filter on (`tenant_id` for multi-tenancy!).

---

## Theory: Indexing

### One-time import

```bash
php artisan scout:import "App\Models\Student"
```

Indexes all existing records. Run this once after install.

### On-save (automatic)

With `Searchable` trait, every save/soft-delete triggers indexing (queued).

```php
$student = Student::find(1);
$student->update(['name' => 'Aisha Khan']);
// → Queued: update in Meilisearch
```

> **If `SCOUT_QUEUE=true`**, the indexing job is queued. Without it, every save waits for Meilisearch.

---

## Theory: Searching

```php
$results = Student::search('Aisha')->get();
```

Returns a Collection. To use Eloquent features (relationships, scopes):

```php
$results = Student::search('Aisha')->paginate(20);
```

Or with raw results (without hydration):

```php
$results = Student::search('Aisha')->raw();
```

---

## Theory: Tenant Isolation in Search

This is **critical** for multi-tenant SaaS:

```php
public function toSearchableArray(): array
{
    return [
        // ... other fields
        'tenant_id' => $this->tenant_id,  // ← MUST include
    ];
}

// When searching
$results = Student::search('Aisha')
    ->where('tenant_id', app('current_tenant')->id)
    ->get();
```

> **Without `where('tenant_id')`, tenant A's search returns tenant B's students.** The biggest multi-tenant search bug.

Wrap in a query scope for safety:

```php
public function scopeForTenant($query, Tenant $tenant)
{
    return $query->where('tenant_id', $tenant->id);
}

// Usage
Student::search('Aisha')->forTenant($tenant)->get();
```

Or better, an observer that auto-filters:

```php
// In a service provider
Event::listen(function (\Laravel\Scout\Events\SearchPerformed $event) {
    $builder = $event->builder;
    if (app()->bound('current_tenant')) {
        $builder->where('tenant_id', app('current_tenant')->id);
    }
});
```

> **Senior rule:** make tenant isolation the default, not the developer's responsibility. Use a global filter or trait.

---

## Theory: Advanced Search Features

### Filters

```php
$results = Student::search('aisha')
    ->where('is_active', true)
    ->where('tenant_id', $tenantId)
    ->where('class_id', 5)
    ->get();
```

### Pagination

```php
$results = Student::search('aisha')->paginate(20);
```

### Raw queries

```php
$results = Student::search('')->raw([
    'filter' => 'tenant_id = 1 AND class_id = 5',
    'limit' => 50,
]);
```

---

## Theory: Where to Render Search

### Server-side (classic Laravel)

```php
public function search(Request $request)
{
    $results = Student::search($request->q)
        ->where('tenant_id', app('current_tenant')->id)
        ->paginate(20);
    
    return view('students.search', compact('results'));
}
```

### Live search (AJAX)

```php
public function apiSearch(Request $request)
{
    $results = Student::search($request->q)
        ->where('tenant_id', app('current_tenant')->id)
        ->take(10)
        ->get()
        ->map(fn($s) => [
            'id' => $s->id,
            'name' => $s->name,
            'class' => $s->class?->name,
            'url' => route('students.show', $s),
        ]);
    
    return response()->json($results);
}
```

Frontend (with Alpine.js / Vue):

```javascript
fetch(`/api/search?q=${query}`)
  .then(r => r.json())
  .then(showResults);
```

### InstantSearch.js (Algolia-style)

For a "Google-like" experience:

```html
<div id="searchbox"></div>
<div id="hits"></div>
<div id="pagination"></div>

<script src="https://cdn.jsdelivr.net/npm/instantsearch.js@4"></script>
<script>
const search = instantsearch({
  indexName: 'students',
  searchClient: algoliasearch(/* ... */),
});

search.addWidget(instantsearch.widgets.searchBox({ container: '#searchbox' }));
search.addWidget(instantsearch.widgets.hits({ container: '#hits' }));
search.start();
</script>
```

> **InstantSearch** is overkill for most apps. Start with server-side, add live AJAX if needed.

---

## Theory: Index Settings (Per Index)

Each index has settings: searchable attributes, filterable attributes, ranking, etc.

In `config/scout.php`:

```php
'meilisearch' => [
    'host' => env('MEILISEARCH_HOST'),
    'key' => env('MEILISEARCH_KEY'),
    'index-settings' => [
        App\Models\Student::class => [
            'filterableAttributes' => ['tenant_id', 'class_id', 'is_active'],
            'sortableAttributes' => ['name', 'created_at'],
            'searchableAttributes' => ['name', 'email', 'student_id', 'class_name'],
        ],
    ],
],
```

Or in code:

```php
Student::makeAllSearchable();

\Meilisearch\Client::make()->index('students')->updateSettings([
    'filterableAttributes' => ['tenant_id', 'class_id', 'is_active'],
    'sortableAttributes' => ['name', 'created_at'],
    'searchableAttributes' => ['name', 'email', 'student_id'],
]);
```

> **The senior tunes search settings based on real queries.** "Users search by name most" → make name top priority.

---

## Theory: Multi-Search (Different Models)

```php
$students = Student::search($query)->get();
$classes = SchoolClass::search($query)->get();
$payments = Payment::search($query)->get();

return view('search.global', compact('students', 'classes', 'payments'));
```

> **Each model has its own index.** Scout doesn't combine. You orchestrate in the controller.

---

## Theory: Searchable Data Hygiene

Don't index:
- Sensitive data (passwords, tokens)
- Large text fields (descriptions, comments — use the DB)
- Computed values (compute on the fly instead)
- Relationships to N+1 (denormalize the data you need)

```php
public function toSearchableArray(): array
{
    return [
        'id' => $this->id,
        'name' => $this->name,
        'class_name' => $this->class->name,  // ← denormalized
        // DON'T: 'class' => $this->class  (would store entire model)
    ];
}
```

---

## Theory: When NOT to Use Scout

- **Tiny tables** (< 1000 rows) — `LIKE` is fine
- **Exact-match searches** — use SQL `WHERE`
- **Filtering only** — no full-text? SQL is enough
- **Reporting/aggregations** — use SQL

> **Senior rule:** Scout is for relevance-ranked text search. For everything else, use SQL.

---

## Theory: Performance Tuning

### Search latency

```php
$start = microtime(true);
$results = Student::search('Aisha')->get();
$duration = (microtime(true) - $start) * 1000;
Log::info('search.duration', ['ms' => $duration, 'q' => 'Aisha']);
```

### Indexing latency

Watch the queue:
- If jobs.backlog > 100, indexing can't keep up
- Common cause: importing 100k records at once
- Fix: chunked imports

```php
Student::makeAllSearchable();  // chunks automatically
```

---

## Why It Matters

Search is the user's primary navigation. Get it wrong:
- "Search returns nothing" → "Where's the search?" → users churn
- "Search returns everyone across all tenants" → security breach
- "Search takes 3 seconds" → users give up

Get it right:
- "I typed 2 letters and the right student appeared"
- Tenant isolation enforced
- Sub-100ms results
- Typo-tolerant

> **A senior's search is fast, accurate, and safe across tenants.** A junior's `LIKE` query is slow, incomplete, and buggy.

---

## Lab: Add Search to Your SaaS

### Step 1 — Start Meilisearch

```bash
docker run -d --name meilisearch \
  -p 7700:7700 \
  -e MEILI_MASTER_KEY=masterKey \
  getmeili/meilisearch:v1.6
```

### Step 2 — Install Scout

```bash
composer require laravel/scout
php artisan vendor:publish --provider="Laravel\Scout\ScoutServiceProvider"
```

`.env`:

```env
SCOUT_DRIVER=meilisearch
SCOUT_QUEUE=true
MEILISEARCH_HOST=http://127.0.0.1:7700
MEILISEARCH_KEY=masterKey
```

### Step 3 — Make Student searchable

`app/Models/Student.php`:

```php
use Laravel\Scout\Searchable;

class Student extends Model
{
    use BelongsToTenant, Searchable;
    
    public function toSearchableArray(): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            'student_id' => $this->student_id,
            'class_name' => $this->class?->name,
            'guardian_name' => $this->guardian?->name,
            'is_active' => $this->is_active,
            'tenant_id' => $this->tenant_id,
        ];
    }
}
```

### Step 4 — Configure index settings

`config/scout.php`:

```php
'meilisearch' => [
    'host' => env('MEILISEARCH_HOST'),
    'key' => env('MEILISEARCH_KEY'),
    'index-settings' => [
        App\Models\Student::class => [
            'filterableAttributes' => ['tenant_id', 'class_id', 'is_active'],
            'sortableAttributes' => ['name', 'created_at'],
            'searchableAttributes' => ['name', 'email', 'student_id', 'class_name'],
            'displayedAttributes' => ['id', 'name', 'email', 'class_name'],
        ],
    ],
],
```

### Step 5 — Import existing students

```bash
php artisan scout:import "App\Models\Student"
```

You should see "All records imported."

### Step 6 — Add the search route

`routes/web.php`:

```php
Route::middleware(['auth', 'tenant', 'subscribed'])->group(function () {
    Route::get('/students/search', [StudentController::class, 'search'])->name('students.search');
});
```

### Step 7 — Add the search controller method

`app/Http/Controllers/StudentController.php`:

```php
public function search(Request $request)
{
    $request->validate(['q' => 'required|string|min:2']);
    
    $tenant = app('current_tenant');
    
    $students = Student::search($request->q)
        ->where('tenant_id', $tenant->id)
        ->paginate(20);
    
    return view('students.search', compact('students'));
}
```

### Step 8 — Add the search view

`resources/views/students/search.blade.php`:

```blade
@extends('layouts.app')

@section('content')
<div class="container">
    <h1>Search Results</h1>
    
    <form method="GET" action="{{ route('students.search') }}">
        <input type="text" name="q" value="{{ request('q') }}" 
               placeholder="Search students..." autofocus>
        <button type="submit">Search</button>
    </form>
    
    @if($students->count() > 0)
        <table>
            <thead>
                <tr>
                    <th>Name</th>
                    <th>Class</th>
                    <th>Email</th>
                </tr>
            </thead>
            <tbody>
                @foreach($students as $student)
                <tr>
                    <td><a href="{{ route('students.show', $student) }}">{{ $student->name }}</a></td>
                    <td>{{ $student->class?->name }}</td>
                    <td>{{ $student->email }}</td>
                </tr>
                @endforeach
            </tbody>
        </table>
        
        {{ $students->links() }}
    @else
        <p>No results for "{{ request('q') }}"</p>
    @endif
</div>
@endsection
```

### Step 9 — Add live search (AJAX)

`routes/web.php`:

```php
Route::get('/api/students/search', [StudentController::class, 'apiSearch']);
```

```php
public function apiSearch(Request $request)
{
    $request->validate(['q' => 'required|string|min:1']);
    
    $tenant = app('current_tenant');
    
    $students = Student::search($request->q)
        ->where('tenant_id', $tenant->id)
        ->take(10)
        ->get();
    
    return response()->json(
        $students->map(fn($s) => [
            'id' => $s->id,
            'name' => $s->name,
            'class' => $s->class?->name,
            'url' => route('students.show', $s),
        ])
    );
}
```

Add to the search input:

```html
<input type="text" id="search-input" name="q" autocomplete="off">
<div id="search-results"></div>

<script>
let timeout;
document.getElementById('search-input').addEventListener('input', (e) => {
    clearTimeout(timeout);
    const q = e.target.value;
    if (q.length < 2) return;
    
    timeout = setTimeout(() => {
        fetch(`/api/students/search?q=${q}`)
            .then(r => r.json())
            .then(results => {
                const html = results.map(r => 
                    `<a href="${r.url}">${r.name} (${r.class})</a>`
                ).join('');
                document.getElementById('search-results').innerHTML = html;
            });
    }, 200);
});
</script>
```

### Step 10 — Add the global tenant filter

Create a service provider:

```php
// In AppServiceProvider boot()
\Event::listen(function (\Laravel\Scout\Events\SearchPerformed $event) {
    if (app()->bound('current_tenant')) {
        $event->builder->where('tenant_id', app('current_tenant')->id);
    }
});
```

> **Now any search automatically includes the tenant filter.** Defense in depth.

### Step 11 — Index when guardian changes

`app/Models/Guardian.php`:

```php
protected static function booted()
{
    static::saved(function ($guardian) {
        // Re-index students belonging to this guardian
        $guardian->students->each->searchable();
    });
}
```

### Step 12 — Performance check

```bash
php artisan tinker
>>> measure(fn() => Student::search('aisha')->get());
# ~30ms with Meilisearch
# vs ~800ms with LIKE
```

---

## Quiz

**Q1.** Why is `WHERE name LIKE '%aisha%'` slow? Why can't a regular index help?

**Q2.** Your tenant searches "Aisha" and gets students from another tenant. Walk through what went wrong and how to prevent it.

**Q3.** You have 100,000 students. `SCOUT_QUEUE=false` makes the page slow. What does `SCOUT_QUEUE=true` do? What's the trade-off?

**Q4.** A user types "aisah" (typo). Scout returns no results. What setting would help?

**Q5.** (Stretch) You want to search across students, classes, AND payments in one search box. How do you structure this?

---

## What You Now Have

✅ Scout + Meilisearch setup
✅ Searchable models with proper data
✅ Multi-tenant safe search
✅ Server-side and live AJAX search
✅ Tuned index settings
✅ Performance awareness

## What's Next

Lesson 42 covers **API development vs web routes** — when to expose a JSON API, when to stay web-only, how to design REST endpoints, and the patterns for building public APIs.

📖 **[Lesson 42 — API Development vs Web Routes](42-api-development-vs-web-routes.md)**
