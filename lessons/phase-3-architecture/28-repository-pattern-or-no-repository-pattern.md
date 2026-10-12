# Lesson 28 — Repository Pattern or No Repository Pattern

> **Phase 3: Architecture** | You'll be here: ~25 min

## The Senior's Question

The "repository pattern" is taught in many tutorials. Should you use it in Laravel? **Most senior Laravel devs say no** — Laravel's Eloquent is already a repository. But there are nuances. Let me explain.

---

## Theory: What the Repository Pattern Is

In traditional layered architecture:

```
Controller → Service → Repository → Database
```

The repository abstracts the database. The service contains business logic. The controller handles HTTP.

```php
// Repository interface
interface StudentRepository {
    public function find(int $id): ?Student;
    public function findByEmail(string $email): ?Student;
    public function all(): Collection;
    public function create(array $data): Student;
}

// Concrete implementation
class EloquentStudentRepository implements StudentRepository {
    public function find(int $id): ?Student {
        return Student::find($id);
    }
    // ...
}
```

---

## Theory: The Senior's View in Laravel

**Don't use the repository pattern over Eloquent.** Reasons:

1. **Eloquent IS the repository.** It already abstracts the database behind a clean API (`Student::find()`, `Student::where(...)`).

2. **You double the code.** Every repository has an interface + implementation + DI binding. For what value?

3. **You lose Eloquent's power.** Relationships, scopes, eager loading — all of it is in the model. A repository hides it.

4. **It fights Laravel's conventions.** Laravel assumes you use Eloquent directly.

5. **Testing is not improved.** You can mock Eloquent with `Mockery` or use `RefreshDatabase`. No need for a repository abstraction.

> **The senior's rule:** Eloquent IS your data layer. Use it directly. Don't wrap it.

---

## Theory: The Cases Where You DO Want a Layer

There ARE cases where you want a layer above Eloquent:

### Case 1: Complex queries that need to be reused

```php
// In a Student model — perfectly fine
public function scopeActiveInClass($query, int $classId) {
    return $query->where('is_active', true)->where('class_id', $classId);
}

// In a controller
$students = Student::activeInClass(5)->get();
```

> **This is the senior's preferred pattern: scopes on the model.** Not a separate repository.

### Case 2: Domain logic that involves multiple models

```php
// Service class — the right place for cross-model logic
class EnrollStudentAction {
    public function execute(Student $student, SchoolClass $class): void {
        $student->school_class_id = $class->id;
        $student->save();
        
        // Audit log
        AuditLog::create([...]);
        
        // Notification
        $student->user?->notify(new EnrolledInClass($class));
    }
}
```

> **Use service/action classes for multi-step business operations.** Not repositories.

### Case 3: External data sources

```php
// Pulling student data from an external API
class ExternalStudentRepository {
    public function findById(string $id): ?ExternalStudent {
        return Http::get("https://api.example.com/students/{$id}")->json();
    }
}
```

> **Use a class for external integrations.** This is the legitimate use of "repository."

---

## Theory: The Senior's Pattern: Action Classes

For business operations, the senior uses **Action classes** (also called "Command" or "Use Case"):

```php
// app/Actions/EnrollStudent.php
namespace App\Actions;

use App\Models\Student;
use App\Models\SchoolClass;
use App\Models\AuditLog;
use Illuminate\Support\Facades\DB;

class EnrollStudent
{
    public function execute(Student $student, SchoolClass $class): void
    {
        DB::transaction(function () use ($student, $class) {
            $student->update(['school_class_id' => $class->id]);
            
            AuditLog::create([
                'action' => 'student.enrolled',
                'student_id' => $student->id,
                'class_id' => $class->id,
                'user_id' => auth()->id(),
            ]);
        });
    }
}
```

In a controller:

```php
public function enroll(Student $student, SchoolClass $class, EnrollStudent $action): RedirectResponse
{
    $this->authorize('update', $student);
    
    $action->execute($student, $class);
    
    return back()->with('success', 'Student enrolled.');
}
```

> **Action classes:**
> - Have a single `execute()` method
> - Take inputs as parameters
> - Encapsulate a business operation
> - Are testable in isolation
> - Can be queued (e.g., for slow operations)

---

## Theory: When to Use What — The Decision Tree

```
Single CRUD operation?
   ↓
   Yes → Controller + Eloquent (no service needed)
   ↓
   No, but simple business logic?
   ↓
   Yes → Add a method to the model OR a scope
   ↓
   No, multi-model operation?
   ↓
   Yes → Action class
   ↓
   No, but you need to call an external service (Stripe, SMS, etc.)?
   ↓
   Yes → Service class with injected external client
```

> **Default to Eloquent. Add abstraction only when there's a real reason.**

---

## Theory: The Real Anti-Patterns

### Bad: A repository that just wraps Eloquent

```php
// BAD: adds nothing
class StudentRepository {
    public function find($id) { return Student::find($id); }
    public function all() { return Student::all(); }
    public function create($data) { return Student::create($data); }
}
```

This is **noise.** A senior sees this and asks "what value does this add?"

### Good: A scope that does something useful

```php
// GOOD: encapsulates a query the app needs often
class Student extends Model {
    public function scopeWithUnpaidFees($query) {
        return $query->whereHas('feeInvoices', fn($q) => $q->where('status', 'unpaid'));
    }
}
```

### Bad: A "service" that just calls Eloquent

```php
// BAD: thin wrapper
class StudentService {
    public function getStudent($id) { return Student::find($id); }
    public function listStudents() { return Student::all(); }
}
```

### Good: A service that does something Eloquent can't

```php
// GOOD: business logic that Eloquent shouldn't know
class ReportService {
    public function monthlyReport(int $tenantId, Carbon $month): array {
        // Aggregates, calculations, format conversions
        // This is logic, not data access
    }
}
```

---

## Theory: Eloquent + Form Request + Policy is Enough

For 80% of your SaaS, **this stack is enough:**

```
Form Request  → validates and authorizes input
   ↓
Controller    → thin, calls Eloquent and returns response
   ↓
Eloquent      → data access
   ↓
Policy        → authorization
```

Only add services/actions when:
- Multiple models are involved
- External APIs are called
- Complex business rules need their own home
- The same operation runs from multiple places (controller, command, job)

---

## Why It Matters

The repository pattern is cargo-culted from Java/C# code into Laravel. **It doesn't fit.** Laravel's Eloquent is already a repository. Adding a layer on top is duplication.

A senior:
- Uses Eloquent directly in controllers
- Reaches for scopes on models for reusable queries
- Uses action classes for multi-step business operations
- Uses service classes for external integrations
- **Never** wraps Eloquent in a "repository" just for the pattern's sake

---

## Lab: Refactor a Controller Into the Right Pieces

### Step 1 — Look at a "fat" controller

```php
// Before: fat controller
class FeeController extends Controller
{
    public function pay(Request $request, Student $student)
    {
        $amount = $request->amount;
        $method = $request->method;
        
        $invoice = FeeInvoice::where('student_id', $student->id)
            ->where('status', 'unpaid')
            ->first();
        
        if (!$invoice) {
            return back()->with('error', 'No unpaid invoice');
        }
        
        if ($amount > $invoice->due) {
            return back()->with('error', 'Amount exceeds due');
        }
        
        // Create payment
        $payment = Payment::create([
            'student_id' => $student->id,
            'fee_invoice_id' => $invoice->id,
            'amount' => $amount,
            'method' => $method,
            'payment_date' => now(),
            'received_by' => auth()->id(),
        ]);
        
        // Update invoice
        $invoice->paid += $amount;
        $invoice->due -= $amount;
        $invoice->status = $invoice->due <= 0 ? 'paid' : 'partial';
        $invoice->save();
        
        // Send notification
        $student->user?->notify(new PaymentReceived($payment));
        
        return back()->with('success', 'Payment recorded');
    }
}
```

### Step 2 — Refactor: scope on the model

Add to `Student` model:

```php
public function scopeUnpaidInvoices($query)
{
    return $query->whereHas('feeInvoices', fn($q) => $q->where('status', 'unpaid'));
}
```

### Step 3 — Refactor: action class for the business operation

Create `app/Actions/RecordPayment.php`:

```php
<?php
namespace App\Actions;

use App\Models\FeeInvoice;
use App\Models\Payment;
use App\Models\Student;
use App\Notifications\PaymentReceived;
use Illuminate\Support\Facades\DB;
use InvalidArgumentException;

class RecordPayment
{
    public function execute(Student $student, float $amount, string $method): Payment
    {
        if ($amount <= 0) {
            throw new InvalidArgumentException('Payment amount must be positive');
        }
        
        return DB::transaction(function () use ($student, $amount, $method) {
            $invoice = FeeInvoice::where('student_id', $student->id)
                ->whereIn('status', ['unpaid', 'partial', 'overdue'])
                ->orderBy('due_date')
                ->first();
            
            if (!$invoice) {
                throw new \DomainException('No unpaid invoice');
            }
            
            if ($amount > $invoice->due) {
                throw new \DomainException("Amount exceeds due ({$invoice->due})");
            }
            
            $payment = Payment::create([
                'tenant_id' => $student->tenant_id,
                'student_id' => $student->id,
                'fee_invoice_id' => $invoice->id,
                'amount' => $amount,
                'method' => $method,
                'payment_date' => now(),
                'payment_number' => 'PAY-' . strtoupper(uniqid()),
                'received_by' => auth()->id(),
            ]);
            
            $invoice->paid += $amount;
            $invoice->due -= $amount;
            $invoice->status = $invoice->due <= 0 ? 'paid' : 'partial';
            $invoice->save();
            
            $student->user?->notify(new PaymentReceived($payment));
            
            return $payment;
        });
    }
}
```

### Step 4 — Refactor: thin controller

```php
class FeeController extends Controller
{
    public function pay(
        PayFeeRequest $request,
        Student $student,
        RecordPayment $action,
    ): RedirectResponse {
        try {
            $action->execute(
                student: $student,
                amount: (float) $request->amount,
                method: $request->method,
            );
        } catch (\DomainException $e) {
            return back()->with('error', $e->getMessage());
        }
        
        return back()->with('success', 'Payment recorded');
    }
}
```

### Step 5 — Compare

| | Before | After |
|---|---|---|
| Lines in controller | 40+ | 12 |
| Testable in isolation | No | Yes (the action) |
| Reusable | Only in this controller | Anywhere (command, job, API) |
| Transactions | Implicit (none) | Explicit `DB::transaction` |
| Domain rules | Scattered | In one place |

> **The action class is the right level of abstraction.** The controller is just a transport.

---

## Quiz

**Q1.** A junior wraps every model in a repository "for testability." Why does a senior push back?

**Q2.** When IS a separate data-access class warranted? Give a concrete example that's not just Eloquent wrapping.

**Q3.** What's the difference between a scope on a model and an action class? When do you use each?

**Q4.** Your `RecordPayment` action uses `DB::transaction()`. Why? What bug does it prevent?

**Q5.** (Stretch) A senior says "an action class should be a single `execute()` method with explicit inputs, no hidden state." Why? What does this design decision buy you?

---

## What You Now Have

✅ A senior's view of repositories (don't use them over Eloquent)
✅ Action classes for business operations
✅ The decision tree for when to use what
✅ Refactored a fat controller into a thin one
✅ Understanding of `DB::transaction()`

## What's Next

Lesson 29 covers **events and listeners** — Laravel's way of decoupling your code. When something happens (a student is enrolled), other things can react (send notification, log to audit, update reports) without the main code knowing about them.

📖 **[Lesson 29 — Events, Listeners, and Why Decoupling Matters](29-events-listeners-and-why-decoupling-matters.md)**
