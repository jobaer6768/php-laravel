# Lesson 10 — OOP: Classes, Objects, Namespaces

> **Phase 1: PHP Fundamentals** | You'll be here: ~45 min

## The Senior's Question

In MERN, you write classes sometimes but mostly use functions. In Laravel, you write **classes for almost everything.** Models are classes. Controllers are classes. Even middleware is a class. Why is Laravel so class-heavy, and what do you need to know about PHP's OOP model to thrive in it?

---

## Theory: PHP's OOP DNA

PHP was originally a procedural templating language. Then in PHP 5, OOP was added. In PHP 8, it's a fully modern, type-safe OOP system. **The entire Laravel framework is built on OOP.** You cannot write Laravel without classes.

If you've used TypeScript classes, you're 70% there. The differences are mostly syntax and the absence of some JS-specific features (no decorators in PHP — we use **Attributes** instead).

## Theory: The Class — The Basic Unit

```php
<?php
declare(strict_types=1);

namespace App\Models;

class Student
{
    // Properties
    public string $name;
    public int $age;
    private ?string $email = null;
    
    // Constructor (PHP 8 constructor promotion)
    public function __construct(
        string $name,
        int $age,
    ) {
        $this->name = $name;
        $this->age = $age;
    }
    
    // Method
    public function introduce(): string
    {
        return "Hi, I'm {$this->name}, {$this->age} years old.";
    }
    
    // Getter
    public function email(): ?string
    {
        return $this->email;
    }
    
    // Setter with validation
    public function setEmail(string $email): void
    {
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new \InvalidArgumentException("Invalid email");
        }
        $this->email = $email;
    }
}

$student = new Student('Aisha', 16);
echo $student->introduce();
$student->setEmail('aisha@example.com');
echo $student->email();
```

> **Three things to notice:**

1. **Visibility modifiers** (`public`, `private`, `protected`) — like TS
2. **Typed properties** — `public string $name` — like TS
3. **Constructor property promotion** — `public function __construct(public string $name, ...)` — short form

### Constructor Property Promotion (PHP 8.0+)

Long form:
```php
class Student {
    public string $name;
    public function __construct(string $name) {
        $this->name = $name;
    }
}
```

Short form (same thing):
```php
class Student {
    public function __construct(public string $name) {}
}
```

> **Senior rule:** use the short form. Less boilerplate, same result.

---

## Theory: Visibility

| Modifier | Accessible from |
|---|---|
| `public` | Anywhere |
| `protected` | This class + subclasses |
| `private` | Only this class |

```php
class BankAccount {
    private float $balance = 0;  // can't be touched directly outside
    
    public function deposit(float $amount): void {
        if ($amount <= 0) {
            throw new \InvalidArgumentException("Amount must be positive");
        }
        $this->balance += $amount;
    }
    
    public function balance(): float {
        return $this->balance;
    }
}

$account = new BankAccount();
$account->deposit(100);
echo $account->balance();  // 100
// $account->balance = 999999;  // ERROR — private
```

> **Senior rule:** default to `private`. Make it `protected` only if you need a subclass to access. Make it `public` only if it's part of the public API.

---

## Theory: Namespaces — The File-System-to-Class Map

This is **the** critical concept for understanding Laravel's structure.

```php
namespace App\Models;

class Student { ... }
```

This file lives at `app/Models/Student.php`. The namespace `App\Models` maps to that path. PSR-4 (the autoloading standard Composer uses) says: namespace `X\Y\Z` → folder `X/Y/Z/`.

So:

| Namespace | Folder |
|---|---|
| `App\Models\Student` | `app/Models/Student.php` |
| `App\Http\Controllers\UserController` | `app/Http/Controllers/UserController.php` |
| `Illuminate\Support\Collection` | `vendor/illuminate/support/Collection.php` |

When you write:

```php
use App\Models\Student;
use Illuminate\Support\Collection;
```

…the autoloader (PSR-4) finds the file. **No manual imports. No bundler config.** This is how Laravel "just finds" your code.

### Why `\` in front of global classes?

When you write:
```php
namespace App\Models;

$invalid = new DateTime();  // ❌ looks for App\Models\DateTime
$valid = new \DateTime();   // ✓ uses global DateTime
```

Inside a namespace, **unqualified class names resolve to the current namespace first.** To get the global one, prefix with `\`.

> **Senior rule:** in Laravel, most of your code lives in a namespace. Always use the leading `\` for global classes (like `\Exception`, `\RuntimeException`, `\DateTime`).

---

## Theory: `use` Statements

```php
<?php
namespace App\Http\Controllers;

use App\Models\Student;
use Illuminate\Http\Request;
use Illuminate\Http\RedirectResponse;

class StudentController extends Controller
{
    public function store(Request $request): RedirectResponse
    {
        $student = Student::create($request->validated());
        return redirect()->route('students.show', $student);
    }
}
```

`use` is **imports**, not aliases (in the simple case). `use App\Models\Student;` says "when I write `Student`, I mean `App\Models\Student`."

You can also alias:
```php
use App\Models\Student as StudentModel;

$student = new StudentModel('Aisha', 16);
```

> **You can do this** but a senior rarely needs to. The PSR-4 convention is so strict that names don't clash.

### Group imports (PHP 7+)

```php
use App\Models\{Student, Teacher, Classroom};
use Illuminate\Support\Facades\{Route, View, Cache};
```

---

## Theory: Constants, Static Methods, and Properties

```php
class Student {
    public const MAX_AGE = 25;          // class constant
    public static int $count = 0;       // static property (shared across instances)
    
    public function __construct(public string $name) {
        self::$count++;
        // or: static::$count++;
    }
    
    public static function total(): int {
        return self::$count;
    }
}

echo Student::MAX_AGE;          // 25
$a = new Student('Aisha');
$b = new Student('Bilal');
echo Student::total();          // 2
```

> **Senior rule:** static state is **shared across all instances.** This is the same landmine as in JS. Use carefully. Prefer instance state when possible.

---

## Theory: The `::class` Constant

```php
use App\Models\Student;

$className = Student::class;  // "App\Models\Student"
```

This is a **class name as string.** Used everywhere in Laravel:

```php
Route::resource('students', StudentController::class);
$student instanceof Student::class;
```

It's safer than typing the string — your IDE will refactor it.

---

## Theory: Enums (PHP 8.1+)

This replaces a common anti-pattern in old PHP code:

```php
// Old way — strings everywhere
$type = 'coaching';
if ($type === 'coaching') { ... }  // typo = silent bug
```

The new way:
```php
namespace App\Enums;

enum InstitutionType: string
{
    case Coaching = 'coaching';
    case School = 'school';
    case Kindergarten = 'kindergarten';
    
    public function label(): string
    {
        return match($this) {
            self::Coaching => 'Coaching Center',
            self::School => 'School',
            self::Kindergarten => 'Kindergarten',
        };
    }
}
```

Use it:

```php
use App\Enums\InstitutionType;

$type = InstitutionType::Coaching;
echo $type->value;          // "coaching"
echo $type->label();        // "Coaching Center"

// In a migration:
$table->enum('type', ['coaching', 'school', 'kindergarten']);
// Or better, use the enum's cases:
$table->enum('type', array_column(InstitutionType::cases(), 'value'));
```

> **Senior rule:** if you have a fixed set of values (status, role, type, plan tier), use an enum. Always. Strings are a code smell.

---

## Theory: Readonly Classes and Properties (PHP 8.1+ / 8.2+)

```php
class Money {
    public function __construct(
        public readonly int $amount,
        public readonly string $currency,
    ) {}
}

$price = new Money(99, 'USD');
$price->amount = 50;  // ERROR — readonly
```

`readonly` properties can only be set once (in the constructor). This is the senior's answer to "I want immutable data structures."

> **PHP 8.2 added `readonly class`:** every property is readonly by default.

---

## Why It Matters

Laravel is class-heavy. You'll write:
- Models (one per DB table)
- Controllers (one per resource)
- Form Requests (one per form)
- Middleware (one per concern)
- Jobs (one per background task)
- Events (one per thing that happens)
- Listeners (one per event handler)
- Policies (one per model for authorization)
- Service classes (one per business operation)

**Each is a class with a single responsibility.** This is the **Single Responsibility Principle** in action.

A senior doesn't fight this. They lean into it. They know that 20 small, focused classes beat 3 large, multi-purpose ones every time.

---

## Lab: Build Real Classes

### Step 1 — Create a folder for practice classes

```bash
cd G:\php-laravel
mkdir practice/Classes
```

### Step 2 — A `Money` value object

Create `practice/Classes/Money.php`:

```php
<?php
declare(strict_types=1);

namespace Practice;

use InvalidArgumentException;

class Money
{
    public function __construct(
        public readonly int $amount,        // in minor units (cents/paisa)
        public readonly string $currency,
    ) {
        if ($amount < 0) {
            throw new InvalidArgumentException("Amount cannot be negative");
        }
        if (strlen($currency) !== 3) {
            throw new InvalidArgumentException("Currency must be 3-letter ISO code");
        }
    }
    
    public function add(self $other): self
    {
        $this->assertSameCurrency($other);
        return new self($this->amount + $other->amount, $this->currency);
    }
    
    public function format(): string
    {
        $symbol = match($this->currency) {
            'USD' => '$',
            'EUR' => '€',
            'BDT' => '৳',
            'INR' => '₹',
            default => $this->currency . ' ',
        };
        
        $whole = intdiv($this->amount, 100);
        $fraction = str_pad((string)($this->amount % 100), 2, '0', STR_PAD_LEFT);
        
        return "{$symbol}{$whole}.{$fraction}";
    }
    
    private function assertSameCurrency(self $other): void
    {
        if ($this->currency !== $other->currency) {
            throw new InvalidArgumentException(
                "Cannot add {$other->currency} to {$this->currency}"
            );
        }
    }
}
```

> **Senior reading:** this is a **value object.** It represents a value, not an identity. Two `Money(100, 'USD')` instances are equal. You can't have "different versions" of $1 USD. The `readonly` properties enforce immutability. The methods return **new** instances instead of mutating. This is functional-style OOP.

### Step 3 — A `Student` class

Create `practice/Classes/Student.php`:

```php
<?php
declare(strict_types=1);

namespace Practice;

use Practice\Money;

class Student
{
    private array $fees = [];
    
    public function __construct(
        public readonly string $name,
        public readonly string $email,
        public readonly int $gradeLevel,
    ) {}
    
    public function charge(Money $amount, string $description): void
    {
        $this->fees[] = ['amount' => $amount, 'description' => $description];
    }
    
    public function totalDue(string $currency = 'USD'): Money
    {
        $total = array_reduce(
            $this->fees,
            fn(Money $carry, array $fee) => $carry->add($fee['amount']),
            new Money(0, $currency),
        );
        return $total;
    }
    
    public function feeReport(): array
    {
        return array_map(
            fn(array $fee) => [
                'description' => $fee['description'],
                'amount'      => $fee['amount']->format(),
            ],
            $this->fees,
        );
    }
}
```

### Step 4 — Bootstrap them with Composer's autoloader

Add the `Practice` namespace to Composer's autoloader. Edit `coaching-saas/composer.json`:

```json
"autoload": {
    "psr-4": {
        "App\\": "app/",
        "Practice\\": "../practice/Classes/"  ← add this line
    }
},
```

Then run:

```bash
cd G:\php-laravel\coaching-saas
composer dump-autoload
```

### Step 5 — Use the classes

Create `practice/05-classes-demo.php`:

```php
<?php
declare(strict_types=1);

require __DIR__ . '/../coaching-saas/vendor/autoload.php';

use Practice\Student;
use Practice\Money;

$student = new Student('Aisha', 'aisha@example.com', 10);

$student->charge(new Money(5000, 'USD'), 'Tuition - January');
$student->charge(new Money(5000, 'USD'), 'Tuition - February');
$student->charge(new Money(1500, 'USD'), 'Lab fee');
$student->charge(new Money(50, 'USD'), 'Late fee');

echo "Student: {$student->name}" . PHP_EOL;
echo "Total due: " . $student->totalDue()->format() . PHP_EOL;
echo PHP_EOL . "Fee report:" . PHP_EOL;
foreach ($student->feeReport() as $fee) {
    echo "  - {$fee['description']}: {$fee['amount']}" . PHP_EOL;
}
```

Run:

```bash
php practice/05-classes-demo.php
```

Expected:
```
Student: Aisha
Total due: $115.50

Fee report:
  - Tuition - January: $50.00
  - Tuition - February: $50.00
  - Lab fee: $15.00
  - Late fee: $0.50
```

### Step 6 — Try the validation

Add to the bottom of the demo file:

```php
try {
    $bad = new Money(-100, 'USD');
} catch (\InvalidArgumentException $e) {
    echo "Caught: " . $e->getMessage() . PHP_EOL;
}

try {
    $wrong = new Money(100, 'BANANA');
} catch (\InvalidArgumentException $e) {
    echo "Caught: " . $e->getMessage() . PHP_EOL;
}
```

> **The exceptions fire** because the constructor validates.

### Step 7 — Try an enum

Create `practice/Classes/InstitutionType.php`:

```php
<?php
declare(strict_types=1);

namespace Practice;

enum InstitutionType: string
{
    case Coaching = 'coaching';
    case School = 'school';
    case Kindergarten = 'kindergarten';
    
    public function label(): string
    {
        return match($this) {
            self::Coaching => 'Coaching Center',
            self::School => 'School',
            self::Kindergarten => 'Kindergarten',
        };
    }
    
    public function defaultCurrency(): string
    {
        return match($this) {
            self::Coaching => 'BDT',
            self::School => 'BDT',
            self::Kindergarten => 'BDT',
        };
    }
}
```

Add `'Practice\\InstituitionType' => '../practice/Classes/'` to autoload (we already have `Practice\\` mapped, so any class in that folder will autoload). Re-run `composer dump-autoload`.

Test:

```php
use Practice\InstitutionType;

$type = InstitutionType::Coaching;
echo $type->value . PHP_EOL;       // coaching
echo $type->label() . PHP_EOL;     // Coaching Center
echo $type->defaultCurrency() . PHP_EOL;  // BDT

// Loop over all
foreach (InstitutionType::cases() as $case) {
    echo $case->name . ' → ' . $case->value . PHP_EOL;
}
```

---

## Quiz

**Q1.** Translate this TS class to PHP 8.3. Use constructor property promotion and readonly where appropriate.

```typescript
class Course {
  constructor(
    public readonly id: string,
    public title: string,
    public credits: number = 3
  ) {}
  
  describe(): string {
    return `${this.title} (${this.credits} credits)`;
  }
}
```

**Q2.** A junior writes `new DateTime()` inside a namespaced class and it silently fails. Why? What's the fix?

**Q3.** Why is `public readonly` better than just `public` for a value object? Give a concrete bug it prevents.

**Q4.** A senior reaches for an enum when they see a string field with fixed possible values. Why? What's the bug it prevents?

**Q5.** (Stretch) Look at the `Money` class. Why does `add()` return a new instance instead of mutating `$this`? What's the senior's reasoning?

---

## What You Now Have

✅ Class fundamentals: properties, methods, constructors
✅ Visibility rules
✅ Namespaces and PSR-4 autoloading
✅ Modern PHP 8.0+ features: enums, readonly, property promotion
✅ Value object pattern in practice

## What's Next

Lesson 11 covers **inheritance, interfaces, traits, abstract classes** — the polymorphism toolkit. This is where OOP gets philosophical, and where seniors start making judgment calls about composition vs inheritance.

📖 **[Lesson 11 — Inheritance, Interfaces, Traits, Abstract](11-inheritance-interfaces-traits-and-abstract.md)**
