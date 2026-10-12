# Lesson 34 — Test-Driven Development for Real

> **Phase 4: Quality** | You'll be here: ~30 min

## The Senior's Question

TDD is taught as "write tests first, then code." Most developers try it, give up, and say "it slows me down." The senior knows **TDD isn't dogma — it's a tool.** Used right, it shapes better designs. Used wrong, it's friction. Let me show you the senior's view.

---

## Theory: The Red-Green-Refactor Cycle

```
1. RED    — Write a test that fails
2. GREEN  — Make it pass with the simplest code
3. REFACTOR — Clean up while tests are green
```

```
    Test fails
        ↓
    Write minimum code to pass
        ↓
    Tests pass
        ↓
    Refactor
        ↓
    Tests still pass
        ↓
    (Repeat)
```

---

## Theory: When TDD Shines

### Use TDD for:

- **Business rules** (fee calculation, grading, discount logic)
- **Edge cases** (what if amount is negative? what if class is full?)
- **Money/financial logic** (the bugs are expensive)
- **Authorization** (the bugs are security issues)
- **Multi-tenant scoping** (the bugs are catastrophic)
- **Pure functions** (no DB, no framework, easy to test)

### Don't TDD for:

- **UI/HTML** (refactor-driven, hard to test first)
- **Trivial getters/setters** (no logic to test)
- **Configuration** (just config, no logic)
- **Spike/exploration** (you're learning, not building)
- **Throwaway prototypes** (you'll throw it away)

> **Senior rule:** TDD where the logic matters. Skip TDD where the code is throwaway.

---

## Theory: A TDD Walkthrough — Building `RecordPayment`

Let me build the action **using TDD** so you can see the cycle.

### Iteration 1: Test the happy path

```php
test('records a payment and updates invoice', function () {
    $student = Student::factory()->for($tenant)->create();
    $invoice = FeeInvoice::factory()->for($student)->create([
        'total' => 1000, 'paid' => 0, 'due' => 1000, 'status' => 'unpaid',
    ]);
    
    app(RecordPayment::class)->execute($student, 500, 'cash');
    
    $invoice->refresh();
    expect($invoice->paid)->toBe(500.0);
    expect($invoice->status)->toBe('partial');
});
```

Run it: **RED** (class doesn't exist).

Create the action:

```php
class RecordPayment
{
    public function execute(Student $student, float $amount, string $method): void
    {
        $invoice = FeeInvoice::where('student_id', $student->id)
            ->where('status', 'unpaid')
            ->first();
        
        $invoice->paid += $amount;
        $invoice->due -= $amount;
        $invoice->status = $invoice->due <= 0 ? 'paid' : 'partial';
        $invoice->save();
    }
}
```

Run: **GREEN**. Refactor: not much to do here yet.

### Iteration 2: Test the "no invoice" case

```php
test('throws when no unpaid invoice', function () {
    $student = Student::factory()->for($tenant)->create();
    
    app(RecordPayment::class)->execute($student, 100, 'cash');
})->throws(DomainException::class);
```

Run: **RED** (no exception thrown).

Add the check:

```php
public function execute(Student $student, float $amount, string $method): void
{
    $invoice = FeeInvoice::where('student_id', $student->id)
        ->where('status', 'unpaid')
        ->firstOrFail();  // throws ModelNotFoundException
    // ...
}
```

Run: **RED** (wrong exception type).

Improve:

```php
$invoice = FeeInvoice::where('student_id', $student->id)
    ->where('status', 'unpaid')
    ->first();

if (!$invoice) {
    throw new \DomainException('No unpaid invoice');
}
```

Run: **GREEN**. Refactor: maybe extract the query.

### Iteration 3: Test the "exceeds due" case

```php
test('throws when amount exceeds due', function () {
    $student = Student::factory()->for($tenant)->create();
    FeeInvoice::factory()->for($student)->create(['due' => 100, 'status' => 'unpaid']);
    
    app(RecordPayment::class)->execute($student, 200, 'cash');
})->throws(DomainException::class, 'exceeds due');
```

Run: **RED**.

Add the check:

```php
if ($amount > $invoice->due) {
    throw new \DomainException("Amount exceeds due ({$invoice->due})");
}
```

Run: **GREEN**. Refactor.

### Iteration 4: Test the "negative amount" case

```php
test('throws when amount is zero or negative', function () {
    $student = Student::factory()->for($tenant)->create();
    FeeInvoice::factory()->for($student)->create(['due' => 100]);
    
    app(RecordPayment::class)->execute($student, 0, 'cash');
})->throws(\InvalidArgumentException::class);
```

Run: **RED**.

Add:

```php
if ($amount <= 0) {
    throw new \InvalidArgumentException('Amount must be positive');
}
```

### Iteration 5: Test the "fully paid" case

```php
test('marks invoice as paid when fully paid', function () {
    $student = Student::factory()->for($tenant)->create();
    $invoice = FeeInvoice::factory()->for($student)->create([
        'total' => 1000, 'paid' => 0, 'due' => 1000,
    ]);
    
    app(RecordPayment::class)->execute($student, 1000, 'cash');
    
    expect($invoice->fresh()->status)->toBe('paid');
});
```

Run: **GREEN** (already handled).

### Iteration 6: Test the side effect (notification)

```php
test('sends receipt notification after payment', function () {
    Notification::fake();
    
    $student = Student::factory()->for($tenant)->create();
    FeeInvoice::factory()->for($student)->create(['due' => 100]);
    
    app(RecordPayment::class)->execute($student, 50, 'cash');
    
    Notification::assertSentTo($student->user, PaymentReceivedNotification::class);
});
```

Run: **RED**.

Add:

```php
event(new PaymentReceived($payment));
```

Run: **GREEN**.

> **Each test forced a small, focused change.** The code evolved naturally. **The tests are the spec.**

---

## Theory: TDD's Hidden Benefit — Design

TDD forces you to **design for testability.** When you write the test first, you discover:

- Hard-to-construct dependencies
- Tightly coupled code
- Hidden global state
- Unclear responsibilities

```php
// Hard to test
class PaymentProcessor {
    public function process() {
        $config = config('payments.stripe');  // global
        $stripe = new StripeClient($config['key']);  // constructed
        $logger = Log::channel('payments');  // global
        $stripe->charge(100, 'USD');
        $logger->info('charged');
    }
}

// Easy to test (because we designed it testable)
class RecordPayment {
    public function __construct(
        private PaymentGateway $gateway,
        private LoggerInterface $logger,
    ) {}
    
    public function execute(Student $student, float $amount, string $method) {
        // Pure logic, dependencies injected
    }
}
```

> **Senior insight:** "if it's hard to test, the design is bad." TDD reveals design problems before they become refactoring nightmares.

---

## Theory: When TDD Slows You Down

Sometimes TDD is the wrong tool:

### Case 1: You're exploring an API

```php
// You don't know what the Stripe API returns yet
// Writing tests first is premature
// Better: spike, throw away, then TDD the real implementation
```

> **Use "spike then TDD":** build a quick prototype, learn the API, throw it away, then TDD the production version.

### Case 2: You're writing UI

```html
<!-- What does the form look like? -->
<!-- You don't know until you sketch it -->
<!-- TDD the BACKEND, but the UI is design-first -->
```

### Case 3: The test is harder than the code

```php
test('renders the dashboard with proper styles', function () {
    // How do you test CSS? Snapshot tests? Brittle.
    // Better: test the data, not the rendering
});
```

> **Senior rule:** TDD the logic, not the presentation. Test that the data is right; trust the visual.

---

## Theory: The Pragmatic Test-First Approach

A senior doesn't religiously TDD everything. They TDD:

1. **First, write the test for the new behavior** (RED)
2. **Implement just enough to pass** (GREEN)
3. **Refactor** (still GREEN)
4. **Move to the next test**

But also:

- **Add a test when you find a bug** (regression test)
- **Add a test when you refactor** (verify behavior preserved)
- **Add a test when you integrate a new system** (boundary test)

> **Tests grow with the codebase, not all at once.**

---

## Theory: Outside-In vs Inside-Out TDD

### Outside-in (London school)

```
Start with a feature test (the user flow)
    ↓
Write a controller test
    ↓
Write an action test
    ↓
Write a unit test for a value object
    ↓
Build back up
```

> **Good for:** greenfield projects, when you know the user flow.

### Inside-out (Chicago school)

```
Start with the lowest-level piece (e.g., Money)
    ↓
Then a higher-level operation
    ↓
Then the controller
    ↓
Then a feature test
```

> **Good for:** new features in existing codebases.

For a senior, both work. The senior picks based on context.

---

## Why It Matters

TDD isn't about test coverage. It's about:

- **Forcing you to think before coding** — what should this do?
- **Catching design problems early** — bad designs are hard to test
- **Keeping tests in sync with code** — written together, they don't drift
- **Building confidence to refactor** — tests say "you didn't break anything"

> **A senior doesn't religiously TDD. They use TDD as a design tool where it helps, and skip it where it doesn't.**

---

## Lab: TDD a New Feature

### Step 1 — Pick a feature

Let's build "Send attendance digest to parents weekly."

### Step 2 — Start with the test (RED)

`tests/Feature/AttendanceDigestTest.php`:

```php
<?php

use App\Jobs\SendAttendanceDigest;
use Illuminate\Support\Facades\Queue;

beforeEach(function () {
    $this->tenant = \App\Models\Tenant::factory()->create();
    app()->instance('current_tenant', $this->tenant);
});

test('weekly digest job is dispatched', function () {
    Queue::fake();
    
    SendAttendanceDigest::dispatch($this->tenant);
    
    Queue::assertPushed(SendAttendanceDigest::class);
});
```

Run: **RED** (class doesn't exist).

### Step 3 — Make it pass (GREEN)

```bash
php artisan make:job SendAttendanceDigest
```

```php
class SendAttendanceDigest implements ShouldQueue
{
    use Dispatchable, Queueable;
    
    public function __construct(public Tenant $tenant) {}
    
    public function handle(): void
    {
        // TODO
    }
}
```

Run: **GREEN**. Now expand the test.

### Step 4 — Test the actual work (RED)

```php
test('digest job sends email to parents of students with absences', function () {
    Mail::fake();
    
    $student = Student::factory()->for($this->tenant)->create();
    $parent = User::factory()->for($this->tenant)->create(['role' => UserRole::Parent]);
    
    // 3 absences this week
    Attendance::factory()->for($student)->count(3)->create(['status' => 'absent']);
    
    (new SendAttendanceDigest($this->tenant))->handle();
    
    Mail::assertSent(AttendanceDigestMail::class, fn($mail) => $mail->hasTo($parent->email));
});
```

Run: **RED**.

### Step 5 — Implement (GREEN)

```php
public function handle(): void
{
    $absentStudents = Student::whereHas('attendances', function ($q) {
        $q->where('status', 'absent')
          ->whereBetween('date', [now()->startOfWeek(), now()->endOfWeek()]);
    })->get();
    
    foreach ($absentStudents as $student) {
        $parents = User::where('role', UserRole::Parent)
            ->whereHas('children', fn($q) => $q->where('id', $student->id))
            ->get();
        
        foreach ($parents as $parent) {
            Mail::to($parent->email)->send(new AttendanceDigestMail($student, $student->attendances));
        }
    }
}
```

Run: **GREEN**.

### Step 6 — Test edge case: no absences (RED)

```php
test('digest does nothing when no absences', function () {
    Mail::fake();
    
    (new SendAttendanceDigest($this->tenant))->handle();
    
    Mail::assertNothingSent();
});
```

Run: **GREEN** (already works).

### Step 7 — Test edge case: parents with multiple children (RED)

```php
test('digest sends one email per parent, not per child', function () {
    Mail::fake();
    
    $parent = User::factory()->for($this->tenant)->create(['role' => UserRole::Parent]);
    $studentA = Student::factory()->for($this->tenant)->create();
    $studentB = Student::factory()->for($this->tenant)->create();
    
    // Both children absent
    Attendance::factory()->for($studentA)->create(['status' => 'absent']);
    Attendance::factory()->for($studentB)->create(['status' => 'absent']);
    
    // Link both to parent (this is the children relationship)
    // ... (depends on your parent-child relationship)
    
    (new SendAttendanceDigest($this->tenant))->handle();
    
    Mail::assertSentCount(1);  // one email to the parent
});
```

Continue iterating. Each test forces a small change.

### Step 8 — Refactor

Once tests are green, refactor the job. Extract methods. Clean up.

> **Tests still pass? You didn't break anything. Confidence to refactor.**

---

## Quiz

**Q1.** TDD has a reputation for slowing development. When is this a fair criticism?

**Q2.** Why does TDD improve design? What's the connection between testability and good design?

**Q3.** A junior says "I TDD everything, including CSS classes." A senior says "no." Why?

**Q4.** You have a bug report: "Sometimes payments go to the wrong student." The bug is in `RecordPayment`. Walk through how TDD would help fix it.

**Q5.** (Stretch) "Outside-in TDD" starts with a feature test. But feature tests can be slow and brittle. How do you balance speed and coverage in practice?

---

## What You Now Have

✅ TDD fluency: red-green-refactor
✅ When to TDD and when not to
✅ How TDD improves design
✅ A real TDD walkthrough
✅ Pragmatic test-first approach

## What's Next

Lesson 35 covers **code quality, static analysis, and CI** — running automated tools to catch issues before code review. PHPStan, Pint, GitHub Actions.

📖 **[Lesson 35 — Code Quality, Static Analysis, and CI](35-code-quality-static-analysis-and-ci.md)**
