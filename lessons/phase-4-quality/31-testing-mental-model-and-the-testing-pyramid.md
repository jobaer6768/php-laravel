# Lesson 31 — Testing Mental Model and the Testing Pyramid

> **Phase 4: Quality** | You'll be here: ~30 min

## The Senior's Question

You have a working SaaS. Now what? **How do you know it still works after you change something?** The junior manually clicks through every page. The senior writes tests. What kind of tests, how many, and how do they fit together?

---

## Theory: The Testing Pyramid

```
        ╱╲
       ╱  ╲         E2E (very few)
      ╱ UI ╲        - Selenium, Dusk
     ╱______╲       - Slow, brittle, but tests the full stack
    ╱        ╲
   ╱ Feature  ╲     Feature/Integration (moderate)
  ╱   Tests    ╲    - Hit URLs, check responses
 ╱______________╲   - Fast, real DB, real code
╱                ╲
╱    Unit Tests    ╲  Unit (many)
╱____________________╲ - Single functions/classes
                      - Fast, no DB, no framework
```

> **More at the bottom, fewer at the top.** Unit tests are fast and easy. E2E tests are slow and brittle. A senior writes 100s of unit tests, dozens of feature tests, and a handful of E2E tests.

---

## Theory: The Three Layers in Practice

### Unit Tests — test single functions/classes

```php
test('money add returns a new money with summed amount', function () {
    $a = new Money(100, 'USD');
    $b = new Money(50, 'USD');
    
    $c = $a->add($b);
    
    expect($c->amount)->toBe(150);
    expect($c->currency)->toBe('USD');
});
```

> **No DB, no framework, no Laravel boot.** Just pure PHP. **Runs in milliseconds.**

### Feature Tests — test full request → response

```php
test('admin can create a student', function () {
    $tenant = Tenant::factory()->create();
    $admin = User::factory()->for($tenant)->admin()->create();
    
    $this->actingAs($admin)
         ->post(route('students.store'), [
             'name' => 'Aisha',
             'email' => 'aisha@example.com',
         ])
         ->assertRedirect(route('students.index'))
         ->assertSessionHas('success');
    
    $this->assertDatabaseHas('students', [
        'name' => 'Aisha',
        'email' => 'aisha@example.com',
        'tenant_id' => $tenant->id,
    ]);
});
```

> **Hits the actual route, runs middleware, hits the controller, hits the DB, returns a response.** Real Laravel boot, real DB. **Runs in 100ms-1s.**

### E2E / Browser Tests — test in a real browser

```php
test('user can register and see dashboard', function () {
    $this->browse(function (Browser $browser) {
        $browser->visit('/register')
                ->type('name', 'Aisha')
                ->type('email', 'aisha@example.com')
                ->type('password', 'password123')
                ->type('password_confirmation', 'password123')
                ->press('Register')
                ->assertPathIs('/dashboard')
                ->assertSee('Welcome, Aisha');
    });
});
```

> **Drives a real browser** (Chrome via Dusk). Tests the entire UI. **Slow (seconds), brittle, but catches issues nothing else does.**

---

## Theory: The Senior's Test Mix for a SaaS

For your coaching SaaS:

| Layer | Count | Examples |
|---|---|---|
| Unit | ~50-200 | `Money::add()`, `Student::scopeActive()`, `RecordPayment` action |
| Feature | ~30-100 | "admin can create student", "user can pay fee" |
| E2E | ~5-20 | "user can register", "user can navigate dashboard" |

> **The ratio matters.** If you have 5 unit tests and 50 E2E tests, you'll be slow and frustrated. **If you have 200 unit tests and 10 E2E tests, you'll ship fast.**

---

## Theory: What to Test (The Senior's Rule)

> **Test behavior, not implementation.**

- ✅ "When I submit a valid form, a student is created and I see a success message"
- ❌ "The `store()` method calls `Student::create()` exactly once"
- ❌ "The view contains `<input name='name'>`"

The first test verifies what the user sees. The others test how you implemented it. **Refactoring the second test breaks it for the wrong reason.**

> **Senior rule:** tests should pass even when you rewrite the internals. They should only fail when the behavior changes.

---

## Theory: What NOT to Test

- Framework code (Laravel's `redirect()` works, you don't need to test it)
- Trivial getters/setters
- Configuration loading
- Third-party packages (Stripe, Mailgun) — trust their tests

> **Senior rule:** don't test code you don't own. Test YOUR code that USES that code.

---

## Theory: Test Naming

A senior names tests as sentences:

```php
// Bad
test('test1');
test('studentCreate');

// Good
test('admin can create a student');
test('student creation validates email is unique within tenant');
test('non-admin cannot delete a student');
test('soft deleted students do not appear in default queries');
```

> **When a test fails, the name should tell you exactly what broke.**

---

## Theory: Arrange-Act-Assert (AAA)

Every test follows this structure:

```php
test('admin can create a student', function () {
    // Arrange — set up the world
    $tenant = Tenant::factory()->create();
    $admin = User::factory()->for($tenant)->admin()->create();
    
    // Act — do the thing
    $this->actingAs($admin)
         ->post(route('students.store'), [...]);
    
    // Assert — check the result
    $this->assertDatabaseHas('students', [...]);
});
```

> **AAA makes tests readable.** Future-you reading tests in 6 months will thank present-you.

---

## Theory: Test Doubles (Mocks, Fakes, Stubs)

Sometimes you don't want to call the real thing:

### Fake — a working alternative

```php
Mail::fake();
Notification::fake();

// Run the code
$this->post(...);

// Assert an email would have been sent
Mail::assertSent(WelcomeEmail::class);
Mail::assertSentTo($user->email, WelcomeEmail::class);
Mail::assertNothingSent();
```

> **Fakes don't actually send.** They record what *would* have been sent. **Senior's default.**

### Mock — a strict expectation

```php
$mock = Mockery::mock(BillingService::class);
$mock->shouldReceive('charge')
     ->once()
     ->with(100.0, 'USD')
     ->andReturn(true);

app()->instance(BillingService::class, $mock);
```

> **Mocks are stricter.** They check the call happened exactly as expected. Use when the interaction is the point.

### Stub — a value-returning fake

```php
$stub = new class {
    public function charge($amount, $currency) { return true; }
};
app()->instance(PaymentGateway::class, $stub);
```

> **Stubs are simple.** They return values. Use when the test doesn't care about interactions.

> **Senior rule:** prefer fakes > stubs > mocks. Mocks are brittle.

---

## Why It Matters

A senior ships fast **because** they have tests. They refactor without fear. They add features without breaking old ones. They deploy on Friday afternoon without anxiety.

> "I have 100% test coverage" is not the goal. **"I have tests that catch real bugs"** is the goal.

---

## Lab: Set Up Pest (Senior's Test Framework)

### Step 1 — Install Pest

```bash
cd G:\php-laravel\coaching-saas
composer require pestphp/pest --dev --with-all-dependencies
```

### Step 2 — Install Pest's plugin for Laravel

```bash
composer require pestphp/pest-plugin-laravel --dev
php artisan pest:install
```

> This creates `tests/Pest.php` and updates `phpunit.xml`.

### Step 3 — Configure Pest

Edit `tests/Pest.php`:

```php
<?php

use Illuminate\Foundation\Testing\RefreshDatabase;

uses(Tests\TestCase::class, RefreshDatabase::class)->in('Feature');
uses(Tests\TestCase::class)->in('Unit');

expect()->extend('toBeOne', function () {
    return $this->toBe(1);
});
```

### Step 4 — Look at the test directory

```
tests/
├── Pest.php              ← Pest config
├── TestCase.php          ← base class
├── Feature/              ← feature tests
│   └── ExampleTest.php
└── Unit/                 ← unit tests
    └── ExampleTest.php
```

### Step 5 — Run the existing tests

```bash
php artisan test
```

Or with Pest directly:

```bash
./vendor/bin/pest
```

You should see "Tests: 2 passed" or similar.

### Step 6 — Write your first unit test

Create `tests/Unit/MoneyTest.php`:

```php
<?php

use App\Domain\Money;

test('can add two money objects of the same currency', function () {
    $a = new Money(100, 'USD');
    $b = new Money(50, 'USD');
    
    $c = $a->add($b);
    
    expect($c->amount)->toBe(150);
    expect($c->currency)->toBe('USD');
});

test('cannot add money of different currencies', function () {
    $a = new Money(100, 'USD');
    $b = new Money(50, 'EUR');
    
    $a->add($b);
})->throws(\App\Exceptions\CurrencyMismatchException::class);

test('cannot have negative amount', function () {
    new Money(-1, 'USD');
})->throws(\InvalidArgumentException::class);
```

Run:

```bash
php artisan test --filter=MoneyTest
```

### Step 7 — Write a feature test

Create `tests/Feature/StudentRegistrationTest.php`:

```php
<?php

use App\Models\Student;
use App\Models\Tenant;
use App\Models\User;
use App\Enums\UserRole;

beforeEach(function () {
    $this->tenant = Tenant::factory()->create();
});

test('admin can create a student', function () {
    $admin = User::factory()->for($this->tenant)->create([
        'role' => UserRole::Admin,
    ]);
    
    $this->actingAs($admin)
         ->post(route('students.store'), [
             'name' => 'Aisha',
             'email' => 'aisha@example.com',
         ])
         ->assertRedirect(route('students.index'))
         ->assertSessionHas('success');
    
    expect(Student::where('email', 'aisha@example.com')->exists())->toBeTrue();
});
```

### Step 8 — Run the test

```bash
php artisan test --filter=StudentRegistrationTest
```

> **You have a passing test that verifies the user can create a student end-to-end.** The database is refreshed automatically.

---

## Quiz

**Q1.** Why does the testing pyramid have more unit tests than E2E tests? What's the trade-off?

**Q2.** A junior tests `expect($controller->method)->toHaveBeenCalledOnce()`. Why is this a smell?

**Q3.** When would you use a `Fake` vs a `Mock`? Give a concrete example of each.

**Q4.** Your test name is `test1`. A senior says "rename it." What's the senior's principle?

**Q5.** (Stretch) Look at the AAA pattern. Why does it matter? What happens in tests that mix arrange/act/assert in a non-linear way?

---

## What You Now Have

✅ Mental model: unit, feature, E2E
✅ The testing pyramid and your test mix
✅ AAA pattern for test structure
✅ When to use fakes, mocks, stubs
✅ Pest installed with Laravel

## What's Next

Lesson 32 covers **Pest in depth** — the modern testing framework's syntax, factories, datasets, and patterns that make tests expressive and fast.

📖 **[Lesson 32 — Pest Setup and Writing Your First Test](32-pest-setup-and-writing-your-first-test.md)**
