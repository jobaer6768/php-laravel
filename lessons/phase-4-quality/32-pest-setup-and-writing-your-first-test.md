# Lesson 32 — Pest Setup and Writing Your First Test

> **Phase 4: Quality** | You'll be here: ~40 min

## The Senior's Question

You've installed Pest. Now you need to write tests for your SaaS. The patterns that make tests **fast**, **readable**, and **maintainable** — what are they?

---

## Theory: Why Pest Over PHPUnit

| | PHPUnit | Pest |
|---|---|---|
| Syntax | Verbose class-based | Functional, terse |
| Readability | OK | Excellent |
| Laravel support | Native | First-class via plugin |
| Modern features | Limited | Datasets, higher-order tests, profiling |
| Adoption in 2026 | Standard | Growing fast |

> **Senior's choice:** Pest for new projects. PHPUnit is fine; Pest is just nicer to read and write. Both run the same assertions under the hood.

---

## Theory: Pest's Anatomy

```php
<?php

// test("description", function () { ... });
// it("description", function () { ... });

test('something happens', function () {
    expect($value)->toBe(...);
});

it('does something else', function () {
    expect($value)->toBe(...);
});
```

> **`test()` and `it()` are identical.** `it()` reads more like English.

### Expectations (Pest's assertions)

```php
expect($value)->toBe(5);
expect($value)->toEqual(5);                // loose
expect($value)->toBeTrue();
expect($value)->toBeFalse();
expect($value)->toBeNull();
expect($value)->toBeEmpty();
expect($value)->toContain('foo');
expect($value)->toHaveCount(3);
expect($value)->toHaveKey('name');
expect($array)->toMatchArray(['name' => 'John']);
expect($value)->toStartWith('hello');
expect($value)->toEndWith('world');
```

### Laravel-specific expectations

```php
expect($user)->toBeInstanceOf(User::class);
expect($user->name)->toBe('Aisha');

$this->assertDatabaseHas('students', ['name' => 'Aisha']);
$this->assertDatabaseMissing('students', ['name' => 'Nobody']);
$this->assertAuthenticatedAs($user);
$this->assertGuest();
```

---

## Theory: Factories — Generate Test Data

```bash
php artisan make:factory TenantFactory --model=Tenant
php artisan make:factory UserFactory --model=User
php artisan make:factory StudentFactory --model=Student
```

`database/factories/TenantFactory.php`:

```php
<?php

namespace Database\Factories;

use App\Enums\InstitutionType;
use App\Models\Tenant;
use Illuminate\Database\Eloquent\Factories\Factory;

class TenantFactory extends Factory
{
    protected $model = Tenant::class;

    public function definition(): array
    {
        $name = fake()->company();
        return [
            'name' => $name,
            'slug' => \Str::slug($name) . '-' . fake()->unique()->numberBetween(1000, 9999),
            'email' => fake()->unique()->companyEmail(),
            'phone' => fake()->phoneNumber(),
            'type' => fake()->randomElement(InstitutionType::cases())->value,
            'is_active' => true,
        ];
    }
}
```

`database/factories/UserFactory.php`:

```php
<?php

namespace Database\Factories;

use App\Enums\UserRole;
use App\Models\Tenant;
use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;

class UserFactory extends Factory
{
    protected $model = User::class;

    public function definition(): array
    {
        return [
            'tenant_id' => Tenant::factory(),
            'name' => fake()->name(),
            'email' => fake()->unique()->safeEmail(),
            'password' => 'password',
            'role' => UserRole::Student,
            'email_verified_at' => now(),
        ];
    }

    public function admin(): static
    {
        return $this->state(fn() => ['role' => UserRole::Admin]);
    }
    
    public function teacher(): static
    {
        return $this->state(fn() => ['role' => UserRole::Teacher]);
    }
}
```

### Use factories

```php
// Single
$tenant = Tenant::factory()->create();

// Multiple
$students = Student::factory()->count(10)->create();

// With relationship
$user = User::factory()->for($tenant)->create();

// With state
$admin = User::factory()->for($tenant)->admin()->create();

// With explicit values
$student = Student::factory()->create([
    'name' => 'Aisha',
    'email' => 'aisha@example.com',
]);
```

> **Senior rule:** factories describe a typical record. Tests override only the fields they care about. The factory does the boring part.

---

## Theory: Datasets (Test Many Inputs)

```php
test('email validation rejects invalid addresses', function (string $email) {
    $response = $this->post('/register', [
        'tenant_name' => 'Test',
        'name' => 'Test',
        'email' => $email,
        'password' => 'password',
        'password_confirmation' => 'password',
    ]);
    
    $response->assertSessionHasErrors('email');
})->with([
    'no @' => 'notanemail',
    'no domain' => 'foo@',
    'no local' => '@bar.com',
    'spaces' => 'foo @bar.com',
]);
```

> **One test, four assertions.** Datasets save code.

---

## Theory: Before/After Hooks

```php
beforeEach(function () {
    $this->tenant = Tenant::factory()->create();
});

afterEach(function () {
    // cleanup
});
```

These run for every test in the file. Use them to set up common state.

---

## Theory: Authentication in Tests

```php
test('admin can see students list', function () {
    $admin = User::factory()->for($this->tenant)->admin()->create();
    
    $this->actingAs($admin)
         ->get(route('students.index'))
         ->assertOk();
});
```

> **`actingAs($user)`** simulates a logged-in user. No need to go through the login flow.

---

## Theory: Multi-Tenancy in Tests

This is where it gets interesting. With shared schema, your tests need to be careful:

```php
beforeEach(function () {
    $this->tenant = Tenant::factory()->create();
    app()->instance('current_tenant', $this->tenant);
});

test('admin can create a student in their tenant', function () {
    $admin = User::factory()->for($this->tenant)->admin()->create();
    
    $this->actingAs($admin)
         ->post(route('students.store'), [
             'name' => 'Aisha',
             'email' => 'aisha@example.com',
         ]);
    
    expect(Student::count())->toBe(1);
    expect(Student::first()->tenant_id)->toBe($this->tenant->id);
});

test('students are scoped to current tenant', function () {
    $tenantB = Tenant::factory()->create();
    
    $studentA = Student::factory()->for($this->tenant)->create();
    
    app()->instance('current_tenant', $tenantB);
    
    expect(Student::count())->toBe(0);  // tenant A's student not visible
    expect(Student::withoutGlobalScopes()->count())->toBe(1);  // but it still exists
});
```

> **This is the senior's multi-tenant test.** It verifies the global scope is doing its job.

---

## Theory: Test Database

Pest with `RefreshDatabase` runs every test in a transaction. At the end, it rolls back. **No data leaks between tests.**

> **Senior rule:** every test starts with a clean DB. Use `RefreshDatabase` for unit/feature tests.

In `phpunit.xml` (or `tests/Pest.php`):

```xml
<env name="DB_CONNECTION" value="sqlite"/>
<env name="DB_DATABASE" value=":memory:"/>
```

> **`:memory:` SQLite is the fastest test DB.** In-memory, no disk I/O, dies when the process ends.

For MySQL, use a separate test database:

```xml
<env name="DB_CONNECTION" value="mysql"/>
<env name="DB_DATABASE" value="coaching_saas_test"/>
```

> **Never use your dev or prod database for tests.** A test that says `User::truncate()` will ruin your day.

---

## Lab: Build a Real Test Suite

### Step 1 — Create a StudentFactory

```bash
cd G:\php-laravel\coaching-saas
php artisan make:factory StudentFactory --model=Student
```

```php
<?php

namespace Database\Factories;

use App\Models\SchoolClass;
use App\Models\Student;
use App\Models\Tenant;
use Illuminate\Database\Eloquent\Factories\Factory;

class StudentFactory extends Factory
{
    protected $model = Student::class;

    public function definition(): array
    {
        return [
            'tenant_id' => Tenant::factory(),
            'school_class_id' => null,
            'student_id' => 'S-' . fake()->unique()->numberBetween(1000, 9999),
            'name' => fake()->name(),
            'email' => fake()->unique()->safeEmail(),
            'phone' => fake()->phoneNumber(),
            'admission_date' => fake()->dateTimeBetween('-2 years', 'now'),
            'is_active' => true,
        ];
    }
}
```

### Step 2 — Add the User factory state methods

Make sure `UserFactory` has:

```php
public function admin(): static
{
    return $this->state(fn() => ['role' => UserRole::Admin]);
}

public function teacher(): static
{
    return $this->state(fn() => ['role' => UserRole::Teacher]);
}
```

### Step 3 — Update Pest.php to bind the tenant

Edit `tests/Pest.php`:

```php
<?php

use Illuminate\Foundation\Testing\RefreshDatabase;

uses(Tests\TestCase::class, RefreshDatabase::class)->in('Feature');
uses(Tests\TestCase::class)->in('Unit');
```

### Step 4 — Write the Money test (unit)

`tests/Unit/MoneyTest.php`:

```php
<?php

use App\Domain\Money;

test('add returns new money with combined amount', function () {
    $a = new Money(100, 'USD');
    $b = new Money(50, 'USD');
    
    $c = $a->add($b);
    
    expect($c->amount)->toBe(150);
});

test('different currencies throw', function () {
    (new Money(100, 'USD'))->add(new Money(50, 'EUR'));
})->throws(CurrencyMismatchException::class);
```

### Step 5 — Write the Student test (feature)

`tests/Feature/StudentManagementTest.php`:

```php
<?php

use App\Models\SchoolClass;
use App\Models\Student;
use App\Models\Tenant;
use App\Models\User;
use App\Enums\UserRole;

beforeEach(function () {
    $this->tenant = Tenant::factory()->create();
    app()->instance('current_tenant', $this->tenant);
});

test('admin can create a student', function () {
    $admin = User::factory()->for($this->tenant)->admin()->create();
    
    $this->actingAs($admin)
         ->post(route('students.store'), [
             'name' => 'Aisha',
             'email' => 'aisha@example.com',
         ])
         ->assertRedirect(route('students.index'))
         ->assertSessionHas('success');
    
    expect(Student::where('email', 'aisha@example.com')->exists())->toBeTrue();
});

test('teacher cannot delete a student', function () {
    $teacher = User::factory()->for($this->tenant)->teacher()->create();
    $student = Student::factory()->for($this->tenant)->create();
    
    $this->actingAs($teacher)
         ->delete(route('students.destroy', $student))
         ->assertForbidden();
    
    expect(Student::count())->toBe(1);
});

test('students are scoped to current tenant', function () {
    $student = Student::factory()->for($this->tenant)->create();
    
    $otherTenant = Tenant::factory()->create();
    app()->instance('current_tenant', $otherTenant);
    
    expect(Student::count())->toBe(0);
});

test('cannot have duplicate email within tenant', function () {
    Student::factory()->for($this->tenant)->create(['email' => 'taken@example.com']);
    
    $admin = User::factory()->for($this->tenant)->admin()->create();
    
    $this->actingAs($admin)
         ->post(route('students.store'), [
             'name' => 'Other Aisha',
             'email' => 'taken@example.com',
         ])
         ->assertSessionHasErrors('email');
});
```

### Step 6 — Write a policy test

`tests/Feature/StudentPolicyTest.php`:

```php
<?php

use App\Models\Student;
use App\Models\Tenant;
use App\Models\User;
use App\Enums\UserRole;

beforeEach(function () {
    $this->tenant = Tenant::factory()->create();
    $this->student = Student::factory()->for($this->tenant)->create();
});

test('admin can view any student in their tenant', function () {
    $admin = User::factory()->for($this->tenant)->admin()->create();
    
    expect($admin->can('view', $this->student))->toBeTrue();
});

test('teacher can view students in their tenant', function () {
    $teacher = User::factory()->for($this->tenant)->teacher()->create();
    
    expect($teacher->can('view', $this->student))->toBeTrue();
});

test('admin from another tenant cannot view', function () {
    $otherTenant = Tenant::factory()->create();
    $admin = User::factory()->for($otherTenant)->admin()->create();
    
    expect($admin->can('view', $this->student))->toBeFalse();
});

test('admin can delete, teacher cannot', function () {
    $admin = User::factory()->for($this->tenant)->admin()->create();
    $teacher = User::factory()->for($this->tenant)->teacher()->create();
    
    expect($admin->can('delete', $this->student))->toBeTrue();
    expect($teacher->can('delete', $this->student))->toBeFalse();
});
```

### Step 7 — Run all tests

```bash
php artisan test
```

Expected:
```
PASS  Tests\Unit\MoneyTest
PASS  Tests\Feature\StudentManagementTest
PASS  Tests\Feature\StudentPolicyTest

Tests:  10 passed
```

### Step 8 — Use a dataset for validation tests

```php
test('email validation rejects bad addresses', function (string $badEmail) {
    $admin = User::factory()->for($this->tenant)->admin()->create();
    
    $this->actingAs($admin)
         ->post(route('students.store'), [
             'name' => 'Test',
             'email' => $badEmail,
         ])
         ->assertSessionHasErrors('email');
})->with([
    'no at sign' => 'notanemail',
    'no domain' => 'foo@',
    'spaces' => 'foo @bar.com',
]);
```

### Step 9 — Add a coverage report

```bash
composer require pestphp/pest-plugin-coverage --dev
php artisan test --coverage
```

> **Coverage isn't a goal.** It's a hint at what's untested. **Senior focus:** test the critical paths, not the percentage.

---

## Quiz

**Q1.** A junior writes `expect($user->name)->toBe('Aisha')` in 50 tests. A senior says "use a factory state." Why?

**Q2.** Why use `:memory:` SQLite for tests instead of MySQL? When would you switch to MySQL?

**Q3.** A test fails because of test data leaking from a previous test. What do you do?

**Q4.** Your coverage report says 85%. A senior says "100% is the wrong goal." Why? What should you focus on?

**Q5.** (Stretch) Pest's `dataset` feature can pull from external files. How would you use this for testing edge cases of fee calculations across many currencies?

---

## What You Now Have

✅ Pest fluency: tests, expectations, datasets
✅ Factories with state methods
✅ Multi-tenant test patterns
✅ Authentication in tests
✅ A real test suite for your SaaS

## What's Next

Lesson 33 covers **feature tests vs unit tests in depth** — when to use which, how to test services, how to test events, and the test patterns a senior reaches for first.

📖 **[Lesson 33 — Feature Tests vs Unit Tests in Depth](33-feature-tests-vs-unit-tests.md)**
