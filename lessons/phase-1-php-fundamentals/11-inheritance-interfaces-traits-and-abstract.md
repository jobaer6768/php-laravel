# Lesson 11 — Inheritance, Interfaces, Traits, and Abstract

> **Phase 1: PHP Fundamentals** | You'll be here: ~40 min

## The Senior's Question

PHP gives you four ways to share behavior between classes. Each has a different purpose, and choosing the wrong one creates spaghetti. How do you decide?

---

## Theory: The Four Mechanisms, In One Glance

| Mechanism | What it shares | When to use |
|---|---|---|
| **Inheritance** (`extends`) | Implementation + interface contract | "is-a" relationship: `Dog extends Animal` |
| **Interface** (`implements`) | Contract only — what methods must exist | "can-do": `Payable`, `Notifiable`, `Cacheable` |
| **Abstract class** | Partial implementation + contract | "is-a, but base class shouldn't be instantiated" |
| **Trait** | Reusable code chunks | Mix in behavior across unrelated classes |

A senior's golden rule:

> **"Favor composition over inheritance."** Use inheritance only when the `is-a` relationship is real and stable. Use interfaces for capabilities. Use traits to share code without forcing inheritance.

---

## Theory: Inheritance (`extends`)

```php
<?php
declare(strict_types=1);

class Person {
    public function __construct(
        protected string $name,
        protected int $age,
    ) {}
    
    public function introduce(): string {
        return "I'm {$this->name}, {$this->age} years old.";
    }
}

class Student extends Person {
    public function __construct(
        string $name,
        int $age,
        protected string $studentId,
    ) {
        parent::__construct($name, $age);
    }
    
    public function introduce(): string {
        return parent::introduce() . " My ID is {$this->studentId}.";
    }
    
    public function study(): string {
        return "{$this->name} is studying.";
    }
}

$student = new Student('Aisha', 16, 'S-2026-001');
echo $student->introduce();
// I'm Aisha, 16 years old. My ID is S-2026-001.
echo $student->study();
```

### The Liskov Substitution Principle

If `Student extends Person`, then everywhere you use a `Person`, you should be able to use a `Student` without breaking anything. If you have a `Person::vote()` method, can `Student` do it? Yes (well, in our domain they can). But if you have `Person::retire()`, can a 16-year-old `Student` do it? No — that breaks LSP.

> **Senior rule:** before extending, ask: "Is the subclass truly a more specific version of the parent, in a way that doesn't break the parent's promises?"

### When inheritance is right

- `Teacher extends Person` — yes, a teacher **is a** person
- `AdminUser extends User` — yes
- `JsonResponse extends Response` — yes
- `StudentRepository extends BaseRepository` — **probably no** (composition is better)

### When inheritance is wrong

- `User extends Model extends BaseModel extends ActiveRecord...` — too deep, the base class becomes a god class
- `Course extends Student` — no, a course is not a student
- Multiple inheritance in disguise via traits pretending to be classes

---

## Theory: Interfaces (`implements`)

An interface is a **contract.** It says what methods a class must have, but not how they work.

```php
<?php
declare(strict_types=1);

interface Notifiable {
    public function notificationEmail(): string;
    public function sendNotification(string $subject, string $body): void;
}

interface Billable {
    public function monthlyFee(): float;
    public function charge(float $amount): bool;
}

class Student implements Notifiable, Billable {
    public function __construct(
        public readonly string $name,
        public readonly string $email,
        public readonly float $tuition,
    ) {}
    
    public function notificationEmail(): string {
        return $this->email;
    }
    
    public function sendNotification(string $subject, string $body): void {
        // Pretend this calls mail()
        echo "To: {$this->email} | Subject: $subject | $body" . PHP_EOL;
    }
    
    public function monthlyFee(): float {
        return $this->tuition;
    }
    
    public function charge(float $amount): bool {
        // Pretend this calls Stripe
        echo "Charged \$$amount to {$this->name}" . PHP_EOL;
        return true;
    }
}
```

Now any code that needs "something billable" can type-hint `Billable`:

```php
function processMonthlyBilling(Billable $customer): void {
    $amount = $customer->monthlyFee();
    $success = $customer->charge($amount);
    if ($success) {
        $customer->sendNotification(
            'Payment received',
            "Thank you for paying \$$amount.",
        );
    }
}

$student = new Student('Aisha', 'aisha@example.com', 500.00);
processMonthlyBilling($student);
```

> **The magic:** `processMonthlyBilling` doesn't know or care if it's processing a `Student`, a `Teacher`, or a `Tenant`. As long as it implements `Billable`, it works. **This is polymorphism.**

### Interface inheritance (extending interfaces)

```php
interface AdminBillable extends Billable {
    public function applyDiscount(float $percent): void;
}
```

### Constants in interfaces

```php
interface Status {
    public const ACTIVE = 'active';
    public const INACTIVE = 'inactive';
}
```

---

## Theory: Abstract Classes

An abstract class is **a partial implementation** that can't be instantiated. It's a base class meant to be extended.

```php
<?php
declare(strict_types=1);

abstract class Person {
    public function __construct(
        protected string $name,
        protected int $age,
    ) {}
    
    public function introduce(): string {
        return "I'm {$this->name}, {$this->age} years old.";
    }
    
    // Abstract method — no body, subclasses MUST implement
    abstract public function dailyActivity(): string;
}

class Student extends Person {
    public function dailyActivity(): string {
        return "{$this->name} attends classes and studies.";
    }
}

class Teacher extends Person {
    public function dailyActivity(): string {
        return "{$this->name} teaches classes and grades papers.";
    }
}

// $p = new Person('X', 30);  // ERROR — can't instantiate abstract
```

### When to use abstract vs interface

| Need | Use |
|---|---|
| Some shared code, but no full implementation | Abstract class |
| Pure contract, no shared code | Interface |
| Multiple capabilities (e.g. Notifiable + Billable) | Multiple interfaces |
| Shared code across unrelated classes | Trait |

---

## Theory: Traits — Reusable Code Chunks

A trait is **a chunk of code you can mix into any class.** It's like "horizontal inheritance."

```php
<?php
declare(strict_types=1);

trait HasTimestamps {
    public ?string $createdAt = null;
    public ?string $updatedAt = null;
    
    public function touch(): void {
        $this->updatedAt = date('c');
    }
}

trait HasUuid {
    public string $id;
    
    public function generateUuid(): void {
        $this->id = bin2hex(random_bytes(4)) . '-' . bin2hex(random_bytes(2)) . '-' . bin2hex(random_bytes(2)) . '-' . bin2hex(random_bytes(6));
    }
}

class Student {
    use HasTimestamps, HasUuid;
}

class Course {
    use HasTimestamps;
}

$s = new Student();
$s->generateUuid();
$s->touch();
echo "{$s->id} created at {$s->createdAt}" . PHP_EOL;
```

> **You can't have multiple inheritance in PHP** — a class can only `extend` one parent. But you can `use` multiple traits. This solves some of the same problems.

### When to use traits

- Sharing utility code across many classes (timestamps, UUIDs, slug generation)
- Soft delete functionality (`SoftDeletes` trait in Laravel)
- Notification sending (`Notifiable` trait in Laravel)

### When NOT to use traits

- When you really need an interface contract
- When the trait depends on specific class properties that may not exist
- When you have more than 3-4 traits on one class (complexity smell)

> **Senior rule:** traits are a tool, not a default. Use them when the same exact code applies to multiple unrelated classes. Don't use them as a workaround for bad class design.

---

## Theory: Composition (The Senior's Default)

> "Favor composition over inheritance" — *The Gang of Four, 1994*

Instead of inheriting behavior, **inject it.**

```php
// Inheritance (avoid)
class Student extends Person { ... }
class Teacher extends Person { ... }

// Composition (prefer)
class Student {
    public function __construct(
        public readonly Person $person,  // has a person
        public readonly string $studentId,
    ) {}
}

$student = new Student(
    new Person('Aisha', 16),
    'S-2026-001',
);
```

The student **has-a** person, rather than **is-a** person. This is more flexible:

- A student can later have multiple people (parent, guardian)
- The Person class doesn't need to know about students
- Testing is easier — you can mock the Person

### Real Laravel example

In Laravel, Models don't extend a base "Tenant" or "User" — they use traits:

```php
class Student extends Model {
    use HasFactory, SoftDeletes, Notifiable;
    // Each trait adds specific behavior
}
```

Not a god class. Small, composable, mix-and-match.

---

## Why It Matters

Every time you see:

- `extends` — ask "is this really is-a?"
- `implements` — ask "what contract am I enforcing?"
- `abstract` — ask "do I need shared code AND a contract?"
- `trait` — ask "do multiple unrelated classes need this code?"

A senior makes these decisions on autopilot. A junior reaches for `extends` first because it's familiar. **You'll be the senior.**

---

## Lab: Apply the Four Mechanisms

### Step 1 — Create a domain hierarchy

```bash
cd G:\php-laravel
mkdir -p practice/Domain
```

### Step 2 — Define the interfaces

Create `practice/Domain/Contracts.php`:

```php
<?php
declare(strict_types=1);

namespace Practice\Domain;

interface Identifiable {
    public function id(): string;
}

interface Person {
    public function name(): string;
    public function age(): int;
}

interface Enrollable {
    public function enrolledClasses(): array;
    public function enroll(string $classId): void;
}

interface Teach {
    public function teachingClasses(): array;
    public function teach(string $classId): void;
}
```

### Step 3 — Create a base Person

Create `practice/Domain/BasePerson.php`:

```php
<?php
declare(strict_types=1);

namespace Practice\Domain;

abstract class BasePerson implements Person, Identifiable {
    public function __construct(
        protected string $name,
        protected int $age,
        protected string $id,
    ) {}
    
    public function name(): string { return $this->name; }
    public function age(): int { return $this->age; }
    public function id(): string { return $this->id; }
    
    abstract public function role(): string;
}
```

### Step 4 — A `Student` class

Create `practice/Domain/Student.php`:

```php
<?php
declare(strict_types=1);

namespace Practice\Domain;

class Student extends BasePerson implements Enrollable {
    private array $classes = [];
    
    public function role(): string {
        return 'student';
    }
    
    public function enrolledClasses(): array {
        return $this->classes;
    }
    
    public function enroll(string $classId): void {
        if (!in_array($classId, $this->classes, true)) {
            $this->classes[] = $classId;
        }
    }
}
```

### Step 5 — A `Teacher` class

Create `practice/Domain/Teacher.php`:

```php
<?php
declare(strict_types=1);

namespace Practice\Domain;

class Teacher extends BasePerson implements Teach {
    private array $classes = [];
    
    public function role(): string {
        return 'teacher';
    }
    
    public function teachingClasses(): array {
        return $this->classes;
    }
    
    public function teach(string $classId): void {
        $this->classes[] = $classId;
    }
}
```

### Step 6 — A reusable trait

Create `practice/Domain/HasContactInfo.php`:

```php
<?php
declare(strict_types=1);

namespace Practice\Domain;

trait HasContactInfo {
    protected ?string $email = null;
    protected ?string $phone = null;
    
    public function setEmail(string $email): void {
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new \InvalidArgumentException("Invalid email: $email");
        }
        $this->email = $email;
    }
    
    public function email(): ?string { return $this->email; }
    public function phone(): ?string { return $this->phone; }
    public function setPhone(string $phone): void { $this->phone = $phone; }
}
```

### Step 7 — Add the trait to Student and Teacher

Edit `practice/Domain/Student.php`:

```php
class Student extends BasePerson implements Enrollable {
    use HasContactInfo;
    
    // ... rest unchanged
}
```

Edit `practice/Domain/Teacher.php`:

```php
class Teacher extends BasePerson implements Teach {
    use HasContactInfo;
    
    // ... rest unchanged
}
```

### Step 8 — Update autoload and demo

Edit `coaching-saas/composer.json` to make sure the new namespace maps correctly (it should already via the `Practice\\` mapping, but let's verify):

```json
"autoload": {
    "psr-4": {
        "App\\": "app/",
        "Practice\\": "../practice/Classes/",
        "Practice\\Domain\\": "../practice/Domain/"  ← add if not present
    }
},
```

Run:

```bash
cd G:\php-laravel\coaching-saas
composer dump-autoload
```

Create `practice/06-polymorphism-demo.php`:

```php
<?php
declare(strict_types=1);

require __DIR__ . '/../coaching-saas/vendor/autoload.php';

use Practice\Domain\Student;
use Practice\Domain\Teacher;

$student = new Student('Aisha', 16, 'S-001');
$student->setEmail('aisha@example.com');
$student->enroll('MATH-101');
$student->enroll('PHY-201');

$teacher = new Teacher('Mr. Karim', 45, 'T-001');
$teacher->setEmail('karim@example.com');
$teacher->teach('MATH-101');
$teacher->teach('CHEM-301');

// Use the Identifiable interface — they share it
function describe(Identifiable $entity): string {
    return "{$entity->id()}: {$entity->name()}";
}

echo describe($student) . PHP_EOL;
echo describe($teacher) . PHP_EOL;

// Use the Person interface — also shared
function greet(Person $person): string {
    return "Hello, {$person->name()}. You are {$person->age()} years old.";
}

echo greet($student) . PHP_EOL;
echo greet($teacher) . PHP_EOL;

// Specific to Student
$student->enroll('BIO-101');
echo "Aisha's classes: " . implode(', ', $student->enrolledClasses()) . PHP_EOL;

// Specific to Teacher
echo "Mr. Karim's classes: " . implode(', ', $teacher->teachingClasses()) . PHP_EOL;
```

Run:

```bash
php practice/06-polymorphism-demo.php
```

> **The `describe()` and `greet()` functions don't know if they got a Student or a Teacher.** They only know the interface. This is polymorphism. **This is what makes OOP powerful.**

---

## Quiz

**Q1.** You have a `Bird` class with a `fly()` method. You create `Penguin extends Bird`. Now your code tries to make penguins fly. What's wrong with this design? How would you fix it?

**Q2.** When would you use an abstract class instead of an interface? Give a concrete example from a coaching SaaS.

**Q3.** Why is "composition over inheritance" the senior's default? Give a code example showing the cost of deep inheritance.

**Q4.** You have a `Loggable` trait with `protected array $logs = []` and `public function log(string $msg): void`. You use it on two unrelated classes. A junior asks "why not just make `Loggable` an abstract class and extend it?" Give the senior's answer.

**Q5.** (Stretch) The `Student` and `Teacher` both implement different interfaces (`Enrollable` vs `Teach`). What if you wanted a `TeacherAssistant` who is both? Show the class definition and explain why interfaces make this trivial.

---

## What You Now Have

✅ Mastery of all four polymorphism mechanisms
✅ Judgment for when to use each
✅ Composition over inheritance mindset
✅ Real OOP code you wrote, tested, and understood

## What's Next

Lesson 12 covers **modern PHP 8 features** — the syntax tricks that make PHP elegant. Attributes, named arguments, match expressions, enums (covered), fibers, and patterns you'll see in Laravel.

📖 **[Lesson 12 — Modern PHP 8 Features (In Depth)](12-modern-php-8-features.md)**
