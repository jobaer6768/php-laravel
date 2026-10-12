# Lesson 29 — Events, Listeners, and Why Decoupling Matters

> **Phase 3: Architecture** | You'll be here: ~30 min

## The Senior's Question

When a student gets enrolled, you want to:
1. Save the enrollment
2. Send a welcome email
3. Log the action for audit
4. Update class capacity
5. Notify the teacher

The junior writes all 5 in the controller. The senior says "no — fire an event, let listeners handle the rest." Why?

---

## Theory: The Problem with Coupled Code

```php
// Junior code: do everything in the controller
public function enroll(Student $student, SchoolClass $class): RedirectResponse
{
    DB::transaction(function () use ($student, $class) {
        $student->update(['school_class_id' => $class->id]);
        
        AuditLog::create([...]);
        $student->user?->notify(new EnrolledInClass($class));
        $class->update(['enrolled_count' => $class->students()->count()]);
        $class->teacher?->notify(new StudentEnrolled($student));
    });
    
    return back()->with('success', 'Enrolled');
}
```

Problems:
- Controller does too much
- Can't reuse enrollment logic from a command or job
- Hard to test each side effect
- Adding a new side effect (e.g., "send SMS to parent") means editing the controller
- If the email send fails, the whole transaction rolls back (wrong — enrollment succeeded but no email)

---

## Theory: The Event-Based Solution

```php
// Controller: just does the work and fires an event
public function enroll(Student $student, SchoolClass $class): RedirectResponse
{
    DB::transaction(function () use ($student, $class) {
        $student->update(['school_class_id' => $class->id]);
    });
    
    event(new StudentEnrolled($student, $class));
    
    return back()->with('success', 'Enrolled');
}
```

The event is just data:

```php
// app/Events/StudentEnrolled.php
class StudentEnrolled
{
    public function __construct(
        public readonly Student $student,
        public readonly SchoolClass $class,
    ) {}
}
```

Listeners handle the side effects:

```php
// app/Listeners/SendEnrollmentNotification.php
class SendEnrollmentNotification
{
    public function handle(StudentEnrolled $event): void
    {
        $event->student->user?->notify(new EnrolledInClass($event->class));
    }
}

// app/Listeners/LogEnrollment.php
class LogEnrollment
{
    public function handle(StudentEnrolled $event): void
    {
        AuditLog::create([
            'action' => 'student.enrolled',
            'student_id' => $event->student->id,
            'class_id' => $event->class->id,
            'user_id' => auth()->id(),
        ]);
    }
}

// app/Listeners/UpdateClassCapacity.php
class UpdateClassCapacity
{
    public function handle(StudentEnrolled $event): void
    {
        $event->class->update([
            'enrolled_count' => $event->class->students()->count(),
        ]);
    }
}
```

> **Adding a new side effect is now a new listener class.** The controller doesn't change. **The enrollment logic is reusable** (in a controller, command, or job). **Each side effect can be tested in isolation.**

---

## Theory: The Eloquent Way — Model Events

Eloquent fires events automatically when models are saved, updated, deleted. You can hook into them.

```php
// app/Observers/StudentObserver.php
class StudentObserver
{
    public function created(Student $student): void
    {
        // Fired when Student::create() runs
        Log::info("Student created: {$student->name}");
    }
    
    public function updated(Student $student): void
    {
        // Fired when $student->save() updates
    }
    
    public function deleted(Student $student): void
    {
        // Fired on delete (soft or hard)
    }
    
    public function restored(Student $student): void
    {
        // Fired when restoring a soft-deleted record
    }
}
```

Register the observer:

```php
// app/Providers/AppServiceProvider.php
public function boot(): void
{
    Student::observe(StudentObserver::class);
}
```

> **Model events are great for cross-cutting concerns** that should always happen for a model. Examples: logging, cache invalidation, sending welcome email on user creation.

> **Domain events** (which we built above) are great for **business operations** that may or may not happen depending on context.

---

## Theory: When to Use What

| Need | Use |
|---|---|
| "Always log when a Student is created" | Eloquent observer |
| "Always invalidate cache when a Student is updated" | Eloquent observer |
| "When a student is enrolled, do these 3 things" | Custom event + listeners |
| "When payment succeeds, do these 4 things" | Custom event + listeners |
| "When a teacher records marks, also update gradebook" | Custom event + listeners |

> **Rule of thumb:** if the side effect is **intrinsic to the model** (logging, audit, cache), use an observer. If it's a **business workflow** (enrollment, payment, exam completion), use a custom event.

---

## Theory: Sync vs Queued Listeners

By default, listeners run **synchronously** — they execute in the same request that fired the event:

```php
class SendEnrollmentNotification
{
    public int $tries = 3;
    public int $backoff = 10;
    
    public function handle(StudentEnrolled $event): void
    {
        // This runs in the same request — slow if email sending is slow
        Mail::to($event->student->user)->send(new EnrolledEmail($event->class));
    }
}
```

Make it **queued** so the user doesn't wait:

```php
class SendEnrollmentNotification implements ShouldQueue
{
    use Queueable;
    
    public string $queue = 'notifications';
    public int $delay = 5;  // seconds
    
    public function handle(StudentEnrolled $event): void
    {
        Mail::to($event->student->user)->send(new EnrolledEmail($event->class));
    }
}
```

> **`ShouldQueue` is the magic interface.** Laravel automatically pushes the listener to the queue. The user gets a fast response, and the email sends in the background.

---

## Theory: Registering Events and Listeners

In `app/Providers/EventServiceProvider.php` (Laravel 11) or via auto-discovery:

```php
protected $listen = [
    StudentEnrolled::class => [
        SendEnrollmentNotification::class,
        LogEnrollment::class,
        UpdateClassCapacity::class,
        NotifyTeacher::class,
    ],
    
    PaymentReceived::class => [
        SendReceiptEmail::class,
        LogPayment::class,
        UpdateInvoiceStatus::class,
    ],
];
```

Or in Laravel 11+, just type-hint the listener with the event:

```php
class SendEnrollmentNotification
{
    public function handle(StudentEnrolled $event): void
    {
        // Laravel auto-discovers this listener based on the type-hint
    }
}
```

> **Auto-discovery is the modern way.** No registration needed.

---

## Theory: Practical Event List for a Coaching SaaS

| Event | Listeners |
|---|---|
| `TenantRegistered` | Create default settings, send welcome email, start trial |
| `UserInvited` | Send invite email, set expiry |
| `StudentEnrolled` | Send email, log audit, update class capacity, notify teacher |
| `AttendanceRecorded` | Update attendance %, notify if chronically absent |
| `MarksRecorded` | Calculate grade, update report card, send to parent (queued) |
| `PaymentReceived` | Send receipt, update invoice, log to accounting, queue tax report |
| `ExamPublished` | Notify students, generate report cards (queued) |
| `SubscriptionCancelled` | Suspend tenant, notify, schedule deletion |

> **Each event has 1-5 listeners.** Each listener is small and testable. Adding more is trivial.

---

## Why It Matters

A senior's controllers don't grow. They stay at 5-15 lines because business logic lives in:
- Models (for model logic)
- Scopes (for reusable queries)
- Actions (for multi-step operations)
- Listeners (for side effects)

Events decouple the "what happened" from the "what should happen next." **The user action (enroll) doesn't know about the side effects (email, log, update).** That's a powerful design.

---

## Lab: Refactor Enrollment with Events

### Step 1 — Create the event

```bash
cd G:\php-laravel\coaching-saas
php artisan make:event StudentEnrolled
```

Edit `app/Events/StudentEnrolled.php`:

```php
<?php
namespace App\Events;

use App\Models\SchoolClass;
use App\Models\Student;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class StudentEnrolled
{
    use Dispatchable, SerializesModels;
    
    public function __construct(
        public readonly Student $student,
        public readonly SchoolClass $schoolClass,
    ) {}
}
```

### Step 2 — Create the listeners

```bash
php artisan make:listener SendEnrollmentNotification --event=StudentEnrolled
php artisan make:listener LogEnrollment --event=StudentEnrolled
php artisan make:listener UpdateClassCapacity --event=StudentEnrolled
```

### Step 3 — Fill in each listener

`app/Listeners/SendEnrollmentNotification.php`:

```php
<?php
namespace App\Listeners;

use App\Events\StudentEnrolled;
use App\Notifications\StudentEnrolledNotification;
use Illuminate\Contracts\Queue\ShouldQueue;

class SendEnrollmentNotification implements ShouldQueue
{
    public string $queue = 'notifications';
    
    public function handle(StudentEnrolled $event): void
    {
        if ($event->student->user) {
            $event->student->user->notify(new StudentEnrolledNotification($event->schoolClass));
        }
    }
}
```

`app/Listeners/LogEnrollment.php`:

```php
<?php
namespace App\Listeners;

use App\Events\StudentEnrolled;
use App\Models\AuditLog;

class LogEnrollment
{
    public function handle(StudentEnrolled $event): void
    {
        AuditLog::create([
            'tenant_id' => $event->student->tenant_id,
            'action' => 'student.enrolled',
            'subject_type' => Student::class,
            'subject_id' => $event->student->id,
            'description' => "{$event->student->name} enrolled in {$event->schoolClass->name}",
            'user_id' => auth()->id(),
        ]);
    }
}
```

`app/Listeners/UpdateClassCapacity.php`:

```php
<?php
namespace App\Listeners;

use App\Events\StudentEnrolled;

class UpdateClassCapacity
{
    public function handle(StudentEnrolled $event): void
    {
        $count = $event->schoolClass->students()->count();
        $event->schoolClass->update(['enrolled_count' => $count]);
    }
}
```

### Step 4 — Create the notification

```bash
php artisan make:notification StudentEnrolledNotification
```

Edit `app/Notifications/StudentEnrolledNotification.php`:

```php
<?php
namespace App\Notifications;

use App\Models\SchoolClass;
use Illuminate\Bus\Queueable;
use Illuminate\Notifications\Messages\MailMessage;
use Illuminate\Notifications\Notification;

class StudentEnrolledNotification extends Notification
{
    use Queueable;
    
    public function __construct(public SchoolClass $schoolClass) {}
    
    public function via(object $notifiable): array
    {
        return ['mail', 'database'];
    }
    
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
            ->subject("Welcome to {$this->schoolClass->name}")
            ->line("You have been enrolled in {$this->schoolClass->name}.")
            ->action('View Class', url("/school-classes/{$this->schoolClass->id}"))
            ->line('Welcome aboard!');
    }
    
    public function toArray(object $notifiable): array
    {
        return [
            'class_id' => $this->schoolClass->id,
            'class_name' => $this->schoolClass->name,
        ];
    }
}
```

### Step 5 — Create the action class

```bash
mkdir app/Actions
```

Create `app/Actions/EnrollStudent.php`:

```php
<?php
namespace App\Actions;

use App\Events\StudentEnrolled;
use App\Models\SchoolClass;
use App\Models\Student;
use Illuminate\Support\Facades\DB;

class EnrollStudent
{
    public function execute(Student $student, SchoolClass $class): Student
    {
        return DB::transaction(function () use ($student, $class) {
            $student->update(['school_class_id' => $class->id]);
            
            event(new StudentEnrolled($student, $class));
            
            return $student;
        });
    }
}
```

### Step 6 — Create the controller method

```bash
php artisan make:controller EnrollmentController
```

```php
<?php
namespace App\Http\Controllers;

use App\Actions\EnrollStudent;
use App\Models\SchoolClass;
use App\Models\Student;
use Illuminate\Http\RedirectResponse;

class EnrollmentController extends Controller
{
    public function store(
        Student $student,
        SchoolClass $schoolClass,
        EnrollStudent $action,
    ): RedirectResponse {
        $this->authorize('update', $student);
        $this->authorize('update', $schoolClass);
        
        $action->execute($student, $schoolClass);
        
        return redirect()
            ->route('students.show', $student)
            ->with('success', 'Student enrolled.');
    }
}
```

### Step 7 — Add the route

```php
Route::post('/students/{student}/enroll/{schoolClass}', [EnrollmentController::class, 'store'])
    ->name('students.enroll');
```

### Step 8 — Test

```bash
php artisan tinker
```

```php
$student = \App\Models\Student::first();
$class = \App\Models\SchoolClass::first();
app(\App\Actions\EnrollStudent::class)->execute($student, $class);
```

You should see:
1. The student's class_id updated
2. (If user is set) An email queued
3. An audit log entry created
4. The class's enrolled_count incremented

> **Each side effect is in its own class.** Adding a 4th side effect = new listener, no changes to the action or controller.

---

## Quiz

**Q1.** Why is `event(new StudentEnrolled(...))` better than calling all the side effects inline in the controller? List two benefits.

**Q2.** When would you use an Eloquent observer vs a custom event? Give a concrete example of each.

**Q3.** A listener implements `ShouldQueue`. What does this do, and why would you do it for `SendEnrollmentNotification`?

**Q4.** Your `LogEnrollment` listener runs synchronously. Should it be queued? Why or why not?

**Q5.** (Stretch) Read Laravel's source for the `Dispatchable` trait (just `cat vendor/laravel/framework/src/Illuminate/Foundation/Events/Dispatchable.php`). What does it provide, and why is the `event()` helper useful?

---

## What You Now Have

✅ Events and listeners for decoupling
✅ Eloquent observers for model-level hooks
✅ Queued listeners for slow side effects
✅ A real-world event/listener flow for student enrollment
✅ A thin controller + action + events architecture

## What's Next

Lesson 30 covers **queues and jobs** — Laravel's way of running code asynchronously. Sending emails, generating reports, processing images — all of this should NOT block the user's request.

📖 **[Lesson 30 — Queues, Jobs, and Background Work](30-queues-jobs-and-background-work.md)**
