# Lesson 27 — Domain Modeling for a Coaching SaaS

> **Phase 3: Architecture** | You'll be here: ~50 min

## The Senior's Question

You've built Student and SchoolClass. But a coaching SaaS is **far more than CRUD on a few tables.** It has tenants, users, classes, students, teachers, attendance, exams, marks, fees, payments, schedules, notifications. How do you model this without creating a tangled mess?

---

## Theory: The Modeling Mindset

A senior doesn't start with tables. They start with **the business:**

1. **Who** are the actors? (tenants, admins, teachers, students, parents)
2. **What** do they own? (tenant owns users, teacher owns classes, student owns marks)
3. **What** are the things? (Student, Class, Exam, Mark, Fee, Payment)
4. **How** do they relate? (Student enrolled in Class, Exam has many Marks, Fee generates Payments)

The model emerges from the business, not the other way around.

---

## Theory: The Core Entities

```
Tenant (the institution: ABC Coaching)
├── Users (admin, teachers, students, parents)
├── SchoolClasses (Class 10A, Class 10B)
│   └── Students (enrolled in the class)
│   └── Subjects (Math, English, Physics)
│       └── Teacher (assigned to teach it)
├── Exams (Midterm, Final)
│   └── Marks (student's score)
├── Attendance (daily log)
├── Fees (monthly tuition)
│   └── Payments (transactions)
└── Settings (theme, language, currency)
```

Let me design each.

---

## Theory: Tenants

Already built in Lesson 05. But let me extend:

```php
Schema::create('tenants', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('slug')->unique();
    $table->string('email');
    $table->string('phone')->nullable();
    $table->enum('type', ['coaching', 'school', 'kindergarten'])->default('coaching');
    $table->json('settings')->nullable();           // free-form config
    $table->string('logo_path')->nullable();
    $table->string('address')->nullable();
    $table->string('city')->nullable();
    $table->string('country', 2)->default('BD');
    $table->timestamp('trial_ends_at')->nullable();
    $table->timestamp('subscribed_at')->nullable();
    $table->string('subscription_status')->default('trial'); // trial, active, cancelled
    $table->string('subscription_plan')->nullable();          // basic, pro, enterprise
    $table->timestamp('subscription_ends_at')->nullable();
    $table->boolean('is_active')->default(true);
    $table->timestamps();
    $table->softDeletes();
    
    $table->index('slug');
    $table->index('subscription_status');
    $table->index('is_active');
});
```

**Design notes:**
- `slug` for URLs (abc.coaching-saas.com)
- `settings` JSON for tenant-specific config (theme, language)
- `subscription_*` fields for billing state
- `softDeletes` so we never lose data

---

## Theory: Users

```php
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->foreignId('tenant_id')->nullable()->constrained()->nullOnDelete();
    $table->string('name');
    $table->string('email');                          // unique within tenant, not globally
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');
    $table->string('phone', 20)->nullable();
    $table->string('avatar_path')->nullable();
    $table->enum('role', ['admin', 'teacher', 'student', 'parent'])->default('student');
    $table->string('locale', 5)->default('en');
    $table->string('timezone', 50)->default('UTC');
    $table->rememberToken();
    $table->timestamps();
    $table->softDeletes();
    
    $table->unique(['tenant_id', 'email']);
    $table->index(['tenant_id', 'role']);
    $table->index('role');
});
```

**Design notes:**
- `tenant_id` is **nullable** because self-signup creates a user with no tenant initially (then they create one)
- `unique(['tenant_id', 'email'])` — same email can exist in different tenants
- `role` enum (or use the UserRole enum we built)

---

## Theory: SchoolClasses (and Subjects)

A class has students, subjects, and a schedule.

```php
Schema::create('school_classes', function (Blueprint $table) {
    $table->id();
    $table->foreignId('tenant_id')->constrained()->cascadeOnDelete();
    $table->string('name');                           // "Class 10A", "Grade 5 Blue"
    $table->string('grade_level');                    // "10", "5"
    $table->integer('capacity')->default(40);
    $table->foreignId('class_teacher_id')->nullable()->constrained('users')->nullOnDelete();
    $table->string('room_number')->nullable();
    $table->text('description')->nullable();
    $table->boolean('is_active')->default(true);
    $table->timestamps();
    $table->softDeletes();
    
    $table->index(['tenant_id', 'is_active']);
});
```

### Subjects

```php
Schema::create('subjects', function (Blueprint $table) {
    $table->id();
    $table->foreignId('tenant_id')->constrained()->cascadeOnDelete();
    $table->string('name');                            // "Mathematics"
    $table->string('code', 20)->nullable();            // "MATH101"
    $table->text('description')->nullable();
    $table->integer('default_total_marks')->default(100);
    $table->integer('default_pass_marks')->default(40);
    $table->boolean('is_active')->default(true);
    $table->timestamps();
    
    $table->unique(['tenant_id', 'code']);
});
```

### Class-Subject-Teacher (the join)

```php
Schema::create('class_subject_teacher', function (Blueprint $table) {
    $table->id();
    $table->foreignId('tenant_id')->constrained()->cascadeOnDelete();
    $table->foreignId('school_class_id')->constrained()->cascadeOnDelete();
    $table->foreignId('subject_id')->constrained()->cascadeOnDelete();
    $table->foreignId('teacher_id')->constrained('users')->cascadeOnDelete();
    $table->timestamps();
    
    // Same teacher can't be assigned same subject to same class twice
    $table->unique(['school_class_id', 'subject_id', 'teacher_id'], 'class_subject_teacher_unique');
    
    $table->index('teacher_id');
});
```

> **This is a many-to-many with a payload table.** A class can have many subjects, taught by different teachers. A teacher can teach multiple subjects across multiple classes. The join table is the answer.

---

## Theory: Students (Separate from Users)

A Student is **a person** in the system, but they may or may not have login access. Parents definitely don't. Young students might not.

```php
Schema::create('students', function (Blueprint $table) {
    $table->id();
    $table->foreignId('tenant_id')->constrained()->cascadeOnDelete();
    $table->foreignId('school_class_id')->nullable()->constrained()->nullOnDelete();
    $table->foreignId('user_id')->nullable()->constrained()->nullOnDelete();    // optional login
    $table->string('student_id', 30);                  // "S-2026-001" (display ID)
    $table->string('name');
    $table->string('email')->nullable();
    $table->string('phone', 20)->nullable();
    $table->date('date_of_birth')->nullable();
    $table->enum('gender', ['male', 'female', 'other'])->nullable();
    $table->string('address')->nullable();
    $table->string('blood_group', 5)->nullable();
    $table->string('photo_path')->nullable();
    $table->date('admission_date');
    $table->boolean('is_active')->default(true);
    $table->timestamps();
    $table->softDeletes();
    
    $table->unique(['tenant_id', 'student_id']);
    $table->index(['tenant_id', 'school_class_id']);
    $table->index('school_class_id');
    $table->index('is_active');
});
```

> **Why a separate `students` table and not just a `users` row?** Because:
> - A student is a record of someone who may or may not log in
> - Students have academic data (admission date, roll number) users don't
> - Parents see student records; they may not have a user account
> - The `user_id` link connects to a User if/when they get login access

---

## Theory: Attendance (Daily, per Student per Class)

```php
Schema::create('attendances', function (Blueprint $table) {
    $table->id();
    $table->foreignId('tenant_id')->constrained()->cascadeOnDelete();
    $table->foreignId('school_class_id')->constrained()->cascadeOnDelete();
    $table->foreignId('student_id')->constrained()->cascadeOnDelete();
    $table->date('date');
    $table->enum('status', ['present', 'absent', 'late', 'excused'])->default('present');
    $table->text('remarks')->nullable();
    $table->foreignId('marked_by')->constrained('users');   // teacher who marked
    $table->timestamp('marked_at');
    $table->timestamps();
    
    // A student has one attendance record per class per day
    $table->unique(['student_id', 'school_class_id', 'date']);
    
    $table->index(['school_class_id', 'date']);
    $table->index('date');
});
```

**Design notes:**
- `unique(['student_id', 'school_class_id', 'date'])` — no duplicate marks per day
- `marked_by` and `marked_at` for audit trail
- Status enum: present/absent/late/excused (senior's pick; could also be a `tinyint` 1/0/0.5)

---

## Theory: Exams and Marks

```php
Schema::create('exams', function (Blueprint $table) {
    $table->id();
    $table->foreignId('tenant_id')->constrained()->cascadeOnDelete();
    $table->foreignId('school_class_id')->constrained()->cascadeOnDelete();
    $table->foreignId('subject_id')->nullable()->constrained()->nullOnDelete();
    $table->string('name');                    // "Midterm 2026", "Final Math"
    $table->enum('type', ['class_test', 'midterm', 'final', 'assignment', 'quiz'])->default('class_test');
    $table->date('exam_date');
    $table->integer('total_marks')->default(100);
    $table->integer('pass_marks')->default(40);
    $table->text('description')->nullable();
    $table->boolean('is_published')->default(false);   // hidden from students until published
    $table->timestamps();
    
    $table->index(['school_class_id', 'exam_date']);
    $table->index(['tenant_id', 'is_published']);
});

Schema::create('marks', function (Blueprint $table) {
    $table->id();
    $table->foreignId('tenant_id')->constrained()->cascadeOnDelete();
    $table->foreignId('exam_id')->constrained()->cascadeOnDelete();
    $table->foreignId('student_id')->constrained()->cascadeOnDelete();
    $table->decimal('marks_obtained', 6, 2);
    $table->decimal('percentage', 5, 2)->nullable();
    $table->string('grade', 5)->nullable();        // "A+", "B", "F"
    $table->text('remarks')->nullable();
    $table->foreignId('entered_by')->constrained('users');
    $table->timestamps();
    
    $table->unique(['exam_id', 'student_id']);
    
    $table->index('student_id');
});
```

> **`is_published` is a flag pattern:** teachers enter marks, but students can't see them until the teacher marks the exam as published. **Senior pattern for any "draft" data.**

---

## Theory: Fees and Payments (The Money)

```php
Schema::create('fee_structures', function (Blueprint $table) {
    $table->id();
    $table->foreignId('tenant_id')->constrained()->cascadeOnDelete();
    $table->string('name');                       // "Monthly Tuition", "Lab Fee"
    $table->decimal('amount', 10, 2);
    $table->string('currency', 3)->default('BDT');
    $table->enum('frequency', ['one_time', 'monthly', 'quarterly', 'yearly'])->default('monthly');
    $table->foreignId('school_class_id')->nullable()->constrained()->nullOnDelete();
    $table->text('description')->nullable();
    $table->boolean('is_active')->default(true);
    $table->timestamps();
    
    $table->index(['tenant_id', 'is_active']);
});

Schema::create('fee_invoices', function (Blueprint $table) {
    $table->id();
    $table->foreignId('tenant_id')->constrained()->cascadeOnDelete();
    $table->string('invoice_number')->unique();
    $table->foreignId('student_id')->constrained()->cascadeOnDelete();
    $table->foreignId('fee_structure_id')->nullable()->constrained()->nullOnDelete();
    $table->decimal('subtotal', 12, 2);
    $table->decimal('discount', 12, 2)->default(0);
    $table->decimal('total', 12, 2);
    $table->decimal('paid', 12, 2)->default(0);
    $table->decimal('due', 12, 2);
    $table->date('issue_date');
    $table->date('due_date');
    $table->enum('status', ['unpaid', 'partial', 'paid', 'overdue', 'cancelled'])->default('unpaid');
    $table->text('notes')->nullable();
    $table->timestamps();
    
    $table->index(['student_id', 'status']);
    $table->index(['tenant_id', 'status']);
    $table->index('due_date');
});

Schema::create('payments', function (Blueprint $table) {
    $table->id();
    $table->foreignId('tenant_id')->constrained()->cascadeOnDelete();
    $table->string('payment_number')->unique();
    $table->foreignId('fee_invoice_id')->nullable()->constrained()->nullOnDelete();
    $table->foreignId('student_id')->constrained()->cascadeOnDelete();
    $table->decimal('amount', 12, 2);
    $table->string('currency', 3)->default('BDT');
    $table->enum('method', ['cash', 'bank_transfer', 'card', 'mobile_banking', 'other'])->default('cash');
    $table->string('transaction_id')->nullable();           // bank/Stripe ID
    $table->date('payment_date');
    $table->text('notes')->nullable();
    $table->foreignId('received_by')->constrained('users'); // staff who recorded
    $table->timestamps();
    
    $table->index(['student_id', 'payment_date']);
    $table->index('fee_invoice_id');
});
```

> **Money is `DECIMAL`, not `FLOAT`.** Always. This is the lesson from Lesson 01. Use `bcmath` in PHP, `DECIMAL` in MySQL.
> 
> **Notice the `paid` and `due` columns on invoices.** They could be calculated, but storing them is faster and lets you track history (e.g. "was paid, then refunded").

---

## Theory: Notifications (The "When" Stuff)

```php
Schema::create('notifications', function ($table) {
    $table->uuid('id')->primary();
    $table->string('type');
    $table->morphs('notifiable');     // can attach to any model
    $table->text('data');
    $table->timestamp('read_at')->nullable();
    $table->timestamps();
});
```

This is Laravel's built-in `notifications` table. Polymorphic, so it can be on users, students, or anything.

---

## Theory: The ER Diagram

```
              ┌──────────┐
              │  Tenant  │
              └─────┬────┘
                    │ 1:N
       ┌────────────┼────────────┬─────────────┐
       ↓            ↓            ↓             ↓
   ┌───────┐   ┌─────────┐  ┌─────────┐  ┌────────┐
   │ Users │   │ Classes │  │  Exams  │  │  Fees  │
   └───┬───┘   └────┬────┘  └────┬────┘  └───┬────┘
       │ 1:1        │ 1:N        │ 1:N       │ 1:N
       ↓            ↓            ↓            ↓
   ┌────────┐  ┌──────────┐  ┌────────┐  ┌──────────┐
   │Student │  │ Students│  │  Marks │  │ Invoices │
   │ (link) │  │  (list) │  │        │  │   (N:1)  │
   └────────┘  └──────────┘  └────────┘  └─────┬────┘
                                                │ 1:N
                                                ↓
                                          ┌──────────┐
                                          │ Payments │
                                          └──────────┘
```

---

## Theory: Soft Deletes Everywhere

For a SaaS, **never hard-delete important data.** Use `softDeletes`:

- `tenants` — keep records of churned customers
- `users` — keep records of past employees
- `students` — keep records of graduated students
- `classes` — keep records of past batches
- `exams` — keep records of past exams
- `payments` — financial data, **never delete**

> **Senior rule:** `softDeletes` on everything that has a history. Add the trait in every model.

---

## Why It Matters

A senior's data model is **the foundation of the entire app.** Get it right:

- Performance (proper indexes, no N+1)
- Security (FK constraints, tenant_id everywhere)
- Maintainability (clear relationships, named tables)
- Compliance (audit trails, soft deletes)

A junior designs tables as they go. A senior designs **the whole model up front**, even if they implement it incrementally.

---

## Lab: Build the Full Schema

### Step 1 — Create all migrations

```bash
cd G:\php-laravel\coaching-saas
php artisan make:migration create_subjects_table
php artisan make:migration create_class_subject_teacher_table
php artisan make:migration create_attendances_table
php artisan make:migration create_exams_table
php artisan make:migration create_marks_table
php artisan make:migration create_fee_structures_table
php artisan make:migration create_fee_invoices_table
php artisan make:migration create_payments_table
```

### Step 2 — Fill in each migration

> (Use the schemas from above. Each `Schema::create` block is the same pattern as we've used throughout.)

### Step 3 — Generate all models

```bash
php artisan make:model Subject
php artisan make:model ClassSubjectTeacher
php artisan make:model Attendance
php artisan make:model Exam
php artisan make:model Mark
php artisan make:model FeeStructure
php artisan make:model FeeInvoice
php artisan make:model Payment
```

### Step 4 — Add the `BelongsToTenant` trait to all

```php
// Each of these models
use App\Models\Concerns\BelongsToTenant;

class Subject extends Model {
    use BelongsToTenant;
    // ...
}
```

### Step 5 — Define relationships

```php
// Subject
public function teachers() {
    return $this->belongsToMany(User::class, 'class_subject_teacher', 'subject_id', 'teacher_id')
        ->withPivot('school_class_id');
}

// Attendance
public function student() { return $this->belongsTo(Student::class); }
public function schoolClass() { return $this->belongsTo(SchoolClass::class); }
public function markedBy() { return $this->belongsTo(User::class, 'marked_by'); }

// Exam
public function schoolClass() { return $this->belongsTo(SchoolClass::class); }
public function subject() { return $this->belongsTo(Subject::class); }
public function marks() { return $this->hasMany(Mark::class); }

// Mark
public function exam() { return $this->belongsTo(Exam::class); }
public function student() { return $this->belongsTo(Student::class); }

// FeeInvoice
public function student() { return $this->belongsTo(Student::class); }
public function feeStructure() { return $this->belongsTo(FeeStructure::class); }
public function payments() { return $this->hasMany(Payment::class); }

// Student
public function schoolClass() { return $this->belongsTo(SchoolClass::class); }
public function user() { return $this->belongsTo(User::class); }
public function attendances() { return $this->hasMany(Attendance::class); }
public function marks() { return $this->hasMany(Mark::class); }
public function feeInvoices() { return $this->hasMany(FeeInvoice::class); }
public function payments() { return $this->hasMany(Payment::class); }
```

### Step 6 — Run migrations

```bash
php artisan migrate
```

> **You now have the full schema for a coaching SaaS.** It supports: tenants, users (4 roles), classes, subjects, students, attendance, exams, marks, fees, invoices, payments.

### Step 7 — Seed some test data

In tinker:

```php
$tenant = \App\Models\Tenant::where('slug', 'default')->first();
$class = \App\Models\SchoolClass::first();
$subject = \App\Models\Subject::create([
    'name' => 'Mathematics',
    'code' => 'MATH101',
]);

// Create an exam
$exam = \App\Models\Exam::create([
    'school_class_id' => $class->id,
    'subject_id' => $subject->id,
    'name' => 'Midterm Math',
    'type' => 'midterm',
    'exam_date' => now()->addDays(7),
    'total_marks' => 100,
    'pass_marks' => 40,
]);

// Create students and marks
$student = \App\Models\Student::first();
\App\Models\Mark::create([
    'exam_id' => $exam->id,
    'student_id' => $student->id,
    'marks_obtained' => 85,
    'percentage' => 85.00,
    'grade' => 'A',
    'entered_by' => 1,
]);
```

### Step 8 — Test the relationships

```php
$exam->marks;          // Collection of Mark
$exam->schoolClass;    // SchoolClass
$student->marks;       // Collection of Mark
$student->schoolClass; // SchoolClass
$class->students;      // Collection of Student
```

> **Your SaaS now has a real data model.** In subsequent lessons we'll add services, controllers, and features.

---

## Quiz

**Q1.** Why is `Student` a separate table from `User`? List 3 concrete reasons.

**Q2.** Why use `DECIMAL(12,2)` for money instead of `FLOAT`? What's the bug it prevents?

**Q3.** You have `is_published` on the `exams` table. A junior wants to delete the exam after publishing. Why is this a smell? What's the senior's view?

**Q4.** Why use a `class_subject_teacher` join table instead of putting `subject_id` directly on the `school_classes` table?

**Q5.** (Stretch) Design the schema for a "parent can view their children's data" feature. What columns? What relationships? How do you link a parent user to their children (the students)?

---

## What You Now Have

✅ Full domain model for a coaching SaaS
✅ 8+ entity types with proper relationships
✅ Soft deletes for compliance
✅ Composite unique indexes
✅ Money as DECIMAL, not FLOAT
✅ Audit trail columns (entered_by, marked_by, etc.)

## What's Next

Lesson 28 covers **repository pattern, action classes, and service classes** — the architectural patterns a senior uses to keep controllers thin. We'll discuss when to use them and (importantly) when **not** to.

📖 **[Lesson 28 — Repository Pattern or No Repository Pattern](28-repository-pattern-or-no-repository-pattern.md)**
