# Lesson 33 — Feature Tests vs Unit Tests in Depth

> **Phase 4: Quality** | You'll be here: ~35 min

## The Senior's Question

You have a `RecordPayment` action. You have a `StudentController`. You have a `SendPaymentReceipt` listener. **How do you decide what to test, and at what level?** The senior's pattern: **the test level matches the abstraction.**

---

## Theory: The Test Level Decision

```
Is the logic the user's behavior?
   Yes → Feature test (hit the URL)
   No ↓

Is the logic a domain operation?
   Yes → Unit test on the action/service
   No ↓

Is the logic a side effect (email, log, queue)?
   Yes → Test the listener
   No ↓

Is the logic in a value object (Money, Email)?
   Yes → Unit test on the value object
```

---

## Theory: Feature Tests — The User's Perspective

A feature test asks: "When the user does X, does Y happen?"

```php
test('admin records a payment for a student', function () {
    // Arrange
    $admin = User::factory()->for($this->tenant)->admin()->create();
    $student = Student::factory()->for($this->tenant)->create();
    $invoice = FeeInvoice::factory()->for($student)->create([
        'total' => 1000,
        'paid' => 0,
        'due' => 1000,
        'status' => 'unpaid',
    ]);
    
    // Act
    $this->actingAs($admin)
         ->post(route('fees.pay', $student), [
             'amount' => 500,
             'method' => 'cash',
         ])
         ->assertRedirect()
         ->assertSessionHas('success');
    
    // Assert
    $invoice->refresh();
    expect($invoice->paid)->toBe(500.0);
    expect($invoice->due)->toBe(500.0);
    expect($invoice->status)->toBe('partial');
    
    $this->assertDatabaseHas('payments', [
        'student_id' => $student->id,
        'amount' => 500,
    ]);
});
```

> **This test covers:** route, middleware, authorization, validation, action, transaction, side effects. **One test, many layers.**

---

## Theory: Unit Tests — The Implementation's Perspective

A unit test asks: "Does this code do what I think it does?"

```php
test('RecordPayment throws when no unpaid invoice', function () {
    $student = Student::factory()->for($this->tenant)->create();
    
    $action = app(RecordPayment::class);
    
    expect(fn() => $action->execute($student, 100, 'cash'))
        ->toThrow(DomainException::class, 'No unpaid invoice');
});

test('RecordPayment throws when amount exceeds due', function () {
    $student = Student::factory()->for($this->tenant)->create();
    FeeInvoice::factory()->for($student)->create([
        'due' => 50,
        'status' => 'unpaid',
    ]);
    
    $action = app(RecordPayment::class);
    
    expect(fn() => $action->execute($student, 100, 'cash'))
        ->toThrow(DomainException::class, 'exceeds due');
});

test('RecordPayment throws when amount is zero or negative', function () {
    $student = Student::factory()->for($this->tenant)->create();
    
    $action = app(RecordPayment::class);
    
    expect(fn() => $action->execute($student, 0, 'cash'))
        ->toThrow(InvalidArgumentException::class);
});
```

> **These tests are fast** (no HTTP, no auth, no URL resolution). They test the **action's business rules** directly.

---

## Theory: When to Use Which

### Feature test

When you want to verify:
- The user can reach the page
- Authorization works
- Validation works
- The controller wires up correctly
- Side effects (DB, queue, mail) happen

### Unit test

When you want to verify:
- A business rule
- A calculation
- A value object's behavior
- An edge case that's hard to set up via HTTP

> **Senior rule:** write a feature test for the "happy path" through every endpoint. Write unit tests for tricky business logic.

---

## Theory: Testing Listeners (Side Effects)

```php
test('payment received triggers email and audit log', function () {
    Notification::fake();
    
    $student = Student::factory()->for($this->tenant)->create();
    $payment = Payment::factory()->for($student)->create();
    
    event(new PaymentReceived($payment));
    
    Notification::assertSentTo(
        $student->user,
        PaymentReceivedNotification::class
    );
    
    $this->assertDatabaseHas('audit_logs', [
        'action' => 'payment.received',
        'subject_id' => $payment->id,
    ]);
});
```

> **Notice:** the test doesn't care HOW the event is dispatched. It tests the listener's behavior.

---

## Theory: Testing Queued Jobs

```php
use Illuminate\Support\Facades\Queue;

test('payment triggers queued receipt email', function () {
    Queue::fake();
    
    $student = Student::factory()->for($this->tenant)->create();
    
    event(new PaymentReceived(Payment::factory()->for($student)->create()));
    
    Queue::assertPushed(SendReceiptEmail::class);
});
```

> **`Queue::fake()`** swaps the real queue with a fake one. The job is "pushed" but never runs. **You assert it was pushed, not that it ran.**

If you want to test the job itself runs:

```php
test('SendReceiptEmail actually sends the email', function () {
    Mail::fake();
    
    $student = Student::factory()->for($this->tenant)->create();
    $payment = Payment::factory()->for($student)->create();
    
    (new SendReceiptEmail($payment))->handle();
    
    Mail::assertSent(ReceiptEmail::class);
});
```

> **You call `handle()` directly.** No queue involved. Just the job's logic.

---

## Theory: Testing Authorization

```php
test('teacher cannot delete student', function () {
    $teacher = User::factory()->for($this->tenant)->teacher()->create();
    $student = Student::factory()->for($this->tenant)->create();
    
    expect($teacher->can('delete', $student))->toBeFalse();
});
```

> **Direct policy test.** Doesn't go through the controller. Tests the policy in isolation.

Or via the controller:

```php
test('teacher gets 403 when deleting student', function () {
    $teacher = User::factory()->for($this->tenant)->teacher()->create();
    $student = Student::factory()->for($this->tenant)->create();
    
    $this->actingAs($teacher)
         ->delete(route('students.destroy', $student))
         ->assertForbidden();
});
```

> **Both are valid.** The direct policy test is faster. The controller test verifies the wiring.

---

## Theory: Testing Database State

```php
// Assert row exists
$this->assertDatabaseHas('students', ['email' => 'aisha@x.com']);

// Assert row missing
$this->assertDatabaseMissing('students', ['name' => 'Nobody']);

// Assert row count
$this->assertDatabaseCount('students', 5);

// Assert soft-deleted
$this->assertSoftDeleted('students', ['id' => 1]);
```

Or with Pest expectations:

```php
expect(Student::where('email', 'aisha@x.com')->exists())->toBeTrue();
expect(Student::count())->toBe(0);
```

---

## Theory: Testing Validation

```php
test('student creation validates required fields', function () {
    $admin = User::factory()->for($this->tenant)->admin()->create();
    
    $this->actingAs($admin)
         ->post(route('students.store'), [])
         ->assertSessionHasErrors(['name', 'email']);
})->with([
    'name' => 'name',
    'email' => 'email',
]);
```

Or with dataset:

```php
test('student creation validates field types', function (string $field, mixed $value) {
    $admin = User::factory()->for($this->tenant)->admin()->create();
    
    $this->actingAs($admin)
         ->post(route('students.store'), [
             'name' => 'Aisha',
             'email' => 'aisha@x.com',
             $field => $value,
         ])
         ->assertSessionHasErrors($field);
})->with([
    'class_id must exist' => ['class_id', 99999],
    'email must be valid' => ['email', 'not-an-email'],
]);
```

---

## Theory: Pest Arch Testing — Architecture Tests

This is a **senior's superpower.** You can test that your code follows architectural rules:

```php
// tests/Feature/ArchTest.php

test('controllers are in the Http namespace')
    ->expect('App\Http\Controllers')
    ->toBeClasses()
    ->toExtend('App\Http\Controllers\Controller');

test('models extend Eloquent')
    ->expect('App\Models')
    ->toExtend('Illuminate\Database\Eloquent\Model');

test('policies are in the Policies namespace')
    ->expect('App\Policies')
    ->toBeClasses();

test('controllers do not have direct DB queries outside actions')
    ->expect('App\Http\Controllers')
    ->not->toUse('Illuminate\Support\Facades\DB');

test('no dd() or dump() in production code')
    ->expect(['dd', 'dump', 'var_dump', 'print_r'])
    ->not->toBeUsedIn('App');
```

> **This is the senior's safety net.** It catches architectural mistakes automatically. A new dev can't accidentally bypass the patterns.

---

## Theory: Coverage Strategy for Your SaaS

For a coaching SaaS, here's what to test:

| Path | Coverage target |
|---|---|
| Auth (login, register, password reset) | 100% |
| Multi-tenant isolation | 100% (critical security) |
| Money calculations (RecordPayment, fees) | 100% (financial accuracy) |
| Authorization (policies) | 100% (security) |
| CRUD (students, classes, etc.) | Happy path + key errors |
| Reports | A few representative scenarios |
| Background jobs | At least one per job |
| Listeners | At least one per listener |
| Edge cases | Where business rules are tricky |

> **Senior rule:** you don't need 100% coverage. You need 100% coverage of **what matters.** Money, security, multi-tenancy. Skip the trivial getters.

---

## Why It Matters

Tests aren't about catching bugs (though they do). They're about:

- **Confidence to refactor** — change the code, run the tests, know it still works
- **Documentation** — a test shows how the code is supposed to be used
- **Faster debugging** — when a test fails, you know exactly what broke
- **Living spec** — the test suite is the executable specification of behavior

A senior ships **fast because** they have tests.

---

## Lab: Build Out the Test Suite

### Step 1 — Test the RecordPayment action (unit)

`tests/Unit/RecordPaymentTest.php`:

```php
<?php

use App\Actions\RecordPayment;
use App\Models\FeeInvoice;
use App\Models\Payment;
use App\Models\Student;

beforeEach(function () {
    $this->tenant = \App\Models\Tenant::factory()->create();
    app()->instance('current_tenant', $this->tenant);
});

test('records a partial payment', function () {
    $student = Student::factory()->for($this->tenant)->create();
    $invoice = FeeInvoice::factory()->for($student)->create([
        'total' => 1000,
        'paid' => 0,
        'due' => 1000,
        'status' => 'unpaid',
    ]);
    
    $payment = app(RecordPayment::class)->execute($student, 400, 'cash');
    
    expect($payment->amount)->toBe(400.0);
    
    $invoice->refresh();
    expect($invoice->paid)->toBe(400.0);
    expect($invoice->due)->toBe(600.0);
    expect($invoice->status)->toBe('partial');
});

test('marks invoice as paid when fully paid', function () {
    $student = Student::factory()->for($this->tenant)->create();
    $invoice = FeeInvoice::factory()->for($student)->create([
        'total' => 1000,
        'paid' => 0,
        'due' => 1000,
        'status' => 'unpaid',
    ]);
    
    app(RecordPayment::class)->execute($student, 1000, 'cash');
    
    $invoice->refresh();
    expect($invoice->status)->toBe('paid');
});

test('throws when no unpaid invoice', function () {
    $student = Student::factory()->for($this->tenant)->create();
    
    app(RecordPayment::class)->execute($student, 100, 'cash');
})->throws(\DomainException::class);

test('throws when amount exceeds due', function () {
    $student = Student::factory()->for($this->tenant)->create();
    FeeInvoice::factory()->for($student)->create([
        'due' => 100,
        'status' => 'unpaid',
    ]);
    
    app(RecordPayment::class)->execute($student, 200, 'cash');
})->throws(\DomainException::class, 'exceeds due');
```

### Step 2 — Test the listener (side effects)

`tests/Feature/Listeners/SendReceiptOnPaymentTest.php`:

```php
<?php

use App\Events\PaymentReceived;
use App\Jobs\SendReceiptEmail;
use App\Listeners\SendPaymentReceipt;
use App\Models\Payment;
use App\Models\Student;
use Illuminate\Support\Facades\Queue;

beforeEach(function () {
    $this->tenant = \App\Models\Tenant::factory()->create();
    app()->instance('current_tenant', $this->tenant);
});

test('payment event triggers queued receipt email', function () {
    Queue::fake();
    
    $student = Student::factory()->for($this->tenant)->create();
    $payment = Payment::factory()->for($student)->create();
    
    event(new PaymentReceived($payment));
    
    Queue::assertPushed(SendReceiptEmail::class, fn($job) => $job->payment->id === $payment->id);
});
```

### Step 3 — Test the controller (feature)

`tests/Feature/FeeControllerTest.php`:

```php
<?php

use App\Models\FeeInvoice;
use App\Models\Student;
use App\Models\User;

beforeEach(function () {
    $this->tenant = \App\Models\Tenant::factory()->create();
    app()->instance('current_tenant', $this->tenant);
});

test('admin can record a payment', function () {
    $admin = User::factory()->for($this->tenant)->admin()->create();
    $student = Student::factory()->for($this->tenant)->create();
    FeeInvoice::factory()->for($student)->create(['due' => 500, 'status' => 'unpaid']);
    
    $this->actingAs($admin)
         ->post(route('fees.pay', $student), [
             'amount' => 500,
             'method' => 'cash',
         ])
         ->assertRedirect()
         ->assertSessionHas('success');
    
    $this->assertDatabaseHas('payments', [
        'student_id' => $student->id,
        'amount' => 500,
    ]);
});

test('cannot record payment exceeding due', function () {
    $admin = User::factory()->for($this->tenant)->admin()->create();
    $student = Student::factory()->for($this->tenant)->create();
    FeeInvoice::factory()->for($student)->create(['due' => 100, 'status' => 'unpaid']);
    
    $this->actingAs($admin)
         ->post(route('fees.pay', $student), [
             'amount' => 200,
             'method' => 'cash',
         ])
         ->assertSessionHasErrors();
});
```

### Step 4 — Add an Arch test

`tests/Feature/ArchTest.php`:

```php
<?php

test('models extend Eloquent')
    ->expect('App\Models')
    ->toExtend('Illuminate\Database\Eloquent\Model');

test('policies are in the Policies namespace')
    ->expect('App\Policies')
    ->toBeClasses();

test('controllers are in the Http Controllers namespace')
    ->expect('App\Http\Controllers')
    ->toBeClasses()
    ->toExtend('App\Http\Controllers\Controller');

test('no debugging functions in production code')
    ->expect(['dd', 'dump', 'var_dump', 'print_r'])
    ->not->toBeUsedIn('App');

test('all models have BelongsToTenant trait or are explicitly global')
    ->expect('App\Models')
    ->toUse(\App\Models\Concerns\BelongsToTenant::class);
```

### Step 5 — Run all tests

```bash
php artisan test
```

Expected output:
```
PASS  Tests\Unit\MoneyTest
PASS  Tests\Unit\RecordPaymentTest
PASS  Tests\Feature\StudentManagementTest
PASS  Tests\Feature\StudentPolicyTest
PASS  Tests\Feature\FeeControllerTest
PASS  Tests\Feature\Listeners\SendReceiptOnPaymentTest
PASS  Tests\Feature\ArchTest

Tests:  20 passed
```

> **You have a meaningful test suite.** The critical paths (payment, multi-tenancy, auth) are covered. The architecture is enforced.

---

## Quiz

**Q1.** You have a `RecordPayment` action and a `FeeController`. You write 1 feature test for the controller. What else do you need?

**Q2.** Why is `Queue::fake()` better than letting the queue run in tests?

**Q3.** A junior writes `test('it works')`. A senior says "rename it to 'admin can record a partial payment for an invoice with due 500'." Why does naming matter?

**Q4.** Pest's arch testing can enforce that all controllers extend a base class. Why would you want this?

**Q5.** (Stretch) You have a feature test that uses 5 factories, a few events, and an action. It takes 800ms. A senior says "make it faster." What do you do?

---

## What You Now Have

✅ Clear decision tree: feature vs unit
✅ Patterns for testing actions, listeners, jobs, controllers
✅ Pest arch tests for architectural safety
✅ Coverage strategy (focus on what matters)
✅ A meaningful test suite for your SaaS

## What's Next

Lesson 34 covers **TDD for real** — test-driven development in practice. How to write tests first, when to break the rule, and how TDD changes your design.

📖 **[Lesson 34 — Test-Driven Development for Real](34-test-driven-development-for-real.md)**
