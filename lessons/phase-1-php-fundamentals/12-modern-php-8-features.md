# Lesson 12 — Modern PHP 8 Features (In Depth)

> **Phase 1: PHP Fundamentals** | You'll be here: ~30 min

## The Senior's Question

You will read and write PHP that uses features that didn't exist 5 years ago. Which features matter, which are trivia, and which ones should change how you design your code?

---

## Theory: A Senior's Triage of PHP 8 Features

| Feature | Status | Senior's verdict |
|---|---|---|
| Constructor property promotion | Essential | Use it. Always. |
| Named arguments | Essential | Use it for clarity, not laziness. |
| Match expression | Essential | Replaces 90% of `switch` uses. |
| Enums | Essential | Use for fixed sets of values. |
| Readonly properties / classes | Essential | Use for value objects. |
| Nullsafe operator (`?->`) | Essential | Cleaner than nested null checks. |
| Union types | Essential | When a parameter/return can be multiple types. |
| Intersection types | Useful | Rare, but powerful. |
| First-class callables | Useful | New way to get a callable. |
| `never` return type | Useful | Signals "this always throws or exits." |
| Fibers | Trivia (for now) | Async primitives, not used in Laravel yet. |
| `WeakMap` | Trivia | Edge case for caches and circular references. |
| `str_contains()`, `str_starts_with()`, `str_ends_with()` | Essential | Replaces `strpos() !== false` patterns. |
| `array_is_list()` | Useful | Check if array is sequential. |

Let's cover the ones that matter.

---

## Feature: Constructor Property Promotion

Reduces boilerplate.

```php
// Before
class Money {
    public int $amount;
    public string $currency;
    
    public function __construct(int $amount, string $currency) {
        $this->amount = $amount;
        $this->currency = $currency;
    }
}

// After (PHP 8.0+)
class Money {
    public function __construct(
        public int $amount,
        public string $currency,
    ) {}
}
```

> **Senior rule:** always use the promoted form. Save the old form only when the constructor needs to do real work beyond assignment.

---

## Feature: Named Arguments

```php
// Positional — fragile, especially with many params
new Student('Aisha', 16, 'aisha@x.com', true, 'S-001');

// Named — self-documenting, order doesn't matter
new Student(
    name: 'Aisha',
    age: 16,
    email: 'aisha@x.com',
    studentId: 'S-001',
);
```

> **Senior rule:** use named arguments when:
> - The function has 3+ parameters
> - The function has boolean parameters (true/false are unreadable)
> - You want to skip optional parameters
>
> Don't use them for 1-2 obvious parameters. Overuse makes the call site verbose.

---

## Feature: Match Expression (Replaces `switch`)

```php
// switch — old, error-prone
$grade = 'B';
switch ($grade) {
    case 'A':
        $gpa = 4.0;
        break;
    case 'B':
        $gpa = 3.0;
        break;
    case 'C':
        $gpa = 2.0;
        break;
    default:
        $gpa = 0.0;
}
```

```php
// match — strict comparison, returns a value, no fallthrough
$gpa = match($grade) {
    'A' => 4.0,
    'B' => 3.0,
    'C' => 2.0,
    default => 0.0,
};
```

Key differences:
- `match` uses **strict comparison** (`===`)
- `match` returns a value
- `match` has no fallthrough (no `break` needed)
- `match` throws `UnhandledMatchError` if no case matches (and no default)

> **Senior rule:** prefer `match` over `switch` always. The only time you need `switch` is when you have side effects per case AND no return value, which is rare.

### Multi-condition match

```php
$result = match(true) {
    $score >= 90 => 'A',
    $score >= 80 => 'B',
    $score >= 70 => 'C',
    $score >= 60 => 'D',
    default      => 'F',
};
```

---

## Feature: Nullsafe Operator (`?->`)

The "no need to check for null at every step" operator.

```php
// Before — verbose null checks
$country = null;
if ($user !== null) {
    if ($user->address !== null) {
        $country = $user->address->country;
    }
}

// After
$country = $user?->address?->country;
```

> **Senior rule:** use this for chained property access on possibly-null objects. Don't use it to hide actual null bugs — handle them properly.

---

## Feature: Union Types

A parameter or return type that can be one of several:

```php
function find(int|string $id): Student|null {
    // ...
}

function process(int|float $amount): string {
    return number_format($amount, 2);
}
```

Common patterns:

| Type | Use for |
|---|---|
| `int\|string` | IDs (sometimes int, sometimes UUID) |
| `Student\|null` (or `?Student`) | "might not be found" |
| `string\|Stringable` | Anything stringable |
| `array\|Collection` | Legacy code that returns both |

---

## Feature: Intersection Types (PHP 8.1+)

"All of these":

```php
function process(Countable&Iterator $collection): void {
    // $collection is BOTH Countable AND Iterator
}
```

Rare. Useful when working with specific interfaces.

---

## Feature: Readonly Properties and Classes (8.1 / 8.2)

```php
// Property readonly
class Money {
    public function __construct(
        public readonly int $amount,
        public readonly string $currency,
    ) {}
}

// Entire class readonly (PHP 8.2+)
readonly class ImmutableConfig {
    public function __construct(
        public string $appName,
        public string $env,
    ) {}
}
```

> **Senior rule:** if your class represents a value (Money, Email, Address, Coordinate), make it readonly. If it represents an entity with identity and state changes (User, Order), don't.

---

## Feature: `never` Return Type

```php
function redirectToLogin(): never {
    header('Location: /login');
    exit;
}

function throwIfInvalid(int $id): never {
    throw new \InvalidArgumentException("Bad ID");
}
```

> `never` tells the reader: "this function will not return normally. It either throws or exits."

---

## Feature: First-Class Callables (PHP 8.1+)

```php
// Old way
$upper = 'strtoupper';
$upper('hello');  // works, but the string is opaque

// New way
$upper = strtoupper(...);
$upper('hello');  // works, and IDEs can refactor
```

Same for instance methods:

```php
class Calculator {
    public function add(int $a, int $b): int { return $a + $b; }
}

$calc = new Calculator;
$add = $calc->add(...);
$add(2, 3);  // 5
```

> **Senior rule:** use this when you need to pass a method as a callable to array_map, array_filter, etc. The `...` form is greppable and IDE-friendly.

---

## Feature: New String Functions (PHP 8.0+)

```php
// Old
strpos($haystack, $needle) !== false
strpos($haystack, $needle) === 0
substr($haystack, -strlen($needle)) === $needle

// New (PHP 8.0+)
str_contains($haystack, $needle)         // any position
str_starts_with($haystack, $needle)      // beginning
str_ends_with($haystack, $needle)        // end
```

> **Senior rule:** if you see `strpos() !== false` in modern PHP code, that's a code smell. Refactor to `str_contains()`.

---

## Feature: `array_is_list()` (PHP 8.1+)

```php
array_is_list([1, 2, 3]);          // true
array_is_list([0 => 'a', 1 => 'b']); // true
array_is_list(['x' => 1, 'y' => 2]); // false
```

Useful when validating API input.

---

## Feature: Fibers (PHP 8.1+) — Trivia for Now

Fibers are **stackful coroutines.** They let you pause and resume execution. They're the building block for async PHP.

```php
$fiber = new Fiber(function (): void {
    $value = Fiber::suspend('paused');
    echo "Resumed with: $value\n";
});

$value = $fiber->start();
// $value === 'paused'
$fiber->resume('hello');
// Output: "Resumed with: hello"
```

> **You don't need to use Fibers directly** in 2026 Laravel. They're the foundation that tools like **ReactPHP** and **AmPHP** build on for async I/O. If you ever need WebSockets or thousands of concurrent connections in PHP, Fibers are how it works under the hood. For now, **just know they exist.**

---

## Feature: Readonly + Constructor Promotion (Combined)

```php
readonly class UserId {
    public function __construct(
        public string $value,
    ) {
        if (empty($value)) {
            throw new \InvalidArgumentException('UserId cannot be empty');
        }
    }
}
```

This is the **canonical value object pattern** in PHP 8.2+. You'll write a lot of these in Laravel (DTOs).

---

## Why It Matters

Modern PHP code looks like modern TypeScript. The features above are not trivia — they let you write:

- Less code (promotion, match, nullsafe)
- Safer code (readonly, strict types, enums)
- More expressive code (named args, match, union types)

A senior uses these to make their intent clear at a glance. A junior writes PHP 5 code in PHP 8 syntax.

> **The difference between a junior and senior PHP developer in 2026 is exactly these features.** Master them and you read modern Laravel fluently.

---

## Lab: Modernize an Old Codebase

### Step 1 — Create the lab file

```bash
cd G:\php-laravel
touch practice/07-modernize.php
```

### Step 2 — Old style

```php
<?php
// practice/07-old-style.php — what NOT to write in 2026
class Order {
    private $id;
    private $total;
    private $status;
    private $customer;
    
    public function __construct($id, $total, $status, $customer) {
        $this->id = $id;
        $this->total = $total;
        $this->status = $status;
        $this->customer = $customer;
    }
    
    public function getStatus() {
        if ($this->status == 1) {
            return 'pending';
        } elseif ($this->status == 2) {
            return 'paid';
        } elseif ($this->status == 3) {
            return 'shipped';
        } else {
            return 'unknown';
        }
    }
    
    public function getTotal() {
        if (strlen($this->total) > 0) {
            return $this->total;
        }
        return '0.00';
    }
}
```

> **Spot the smells:** untyped properties, no strict types, magic number status codes, `==` instead of `===`, `strlen` to check empty, getters everywhere.

### Step 3 — Modern style

```php
<?php
// practice/07-modernize.php
declare(strict_types=1);

enum OrderStatus: int
{
    case Pending = 1;
    case Paid = 2;
    case Shipped = 3;
    case Delivered = 4;
    case Cancelled = 5;
    
    public function label(): string
    {
        return match($this) {
            self::Pending   => 'Pending Payment',
            self::Paid      => 'Paid',
            self::Shipped   => 'Shipped',
            self::Delivered => 'Delivered',
            self::Cancelled => 'Cancelled',
        };
    }
}

readonly class Money
{
    public function __construct(
        public int $minorUnits,        // 100 = $1.00
        public string $currency,
    ) {}
    
    public function format(): string
    {
        $whole = intdiv($this->minorUnits, 100);
        $frac = str_pad((string)($this->minorUnits % 100), 2, '0', STR_PAD_LEFT);
        return "{$this->currency} {$whole}.{$frac}";
    }
}

class Order
{
    public function __construct(
        public readonly string $id,
        public readonly Money $total,
        public readonly OrderStatus $status,
        public readonly string $customerEmail,
    ) {}
    
    public function summary(): string
    {
        return "Order {$this->id}: {$this->status->label()} - {$this->total->format()}";
    }
    
    public function isCompleted(): bool
    {
        return match($this->status) {
            OrderStatus::Delivered, OrderStatus::Cancelled => true,
            default => false,
        };
    }
}

// Use it
$order = new Order(
    id: 'ORD-001',
    total: new Money(12500, 'USD'),
    status: OrderStatus::Shipped,
    customerEmail: 'aisha@example.com',
);

echo $order->summary() . PHP_EOL;          // Order ORD-001: Shipped - USD 125.00
echo $order->isCompleted() ? 'Yes' : 'No'; // No
```

### Step 4 — Compare the two files

| Old | Modern |
|---|---|
| Untyped properties | Typed, mostly readonly |
| Magic numbers | Enum with methods |
| `==` comparison | Enum === Enum comparison |
| String for status | Enum with `label()` method |
| `getStatus()` method | `$status->label()` |
| No validation | Validation in constructor |
| Mutable | Immutable (`readonly`) |
| String-based total | Money value object |
| Manual JSON | Not needed (typed) |

This is what modern PHP looks like. **Aim for this style always.**

### Step 5 — Try the nullsafe operator

```php
<?php
// practice/07-nullsafe.php
declare(strict_types=1);

class Address {
    public function __construct(
        public readonly ?string $street,
        public readonly ?string $city,
        public readonly ?string $country,
    ) {}
}

class Customer {
    public function __construct(
        public readonly string $name,
        public readonly ?Address $address = null,
    ) {}
}

$customerA = new Customer('Aisha', new Address('123 Main', 'Dhaka', 'BD'));
$customerB = new Customer('Bilal'); // no address

function getCountry(Customer $c): string {
    return $c->address?->country ?? 'Unknown';
}

echo getCountry($customerA) . PHP_EOL;  // BD
echo getCountry($customerB) . PHP_EOL;  // Unknown
```

### Step 6 — Try named arguments with a complex call

```php
function createInvoice(
    string $customerId,
    int $amount,
    string $currency = 'USD',
    ?string $notes = null,
    bool $sendEmail = true,
    ?\DateTime $dueDate = null,
): string {
    // Pretend this creates a DB record
    $email = $sendEmail ? ' (email will be sent)' : ' (no email)';
    $notesStr = $notes ? " | Notes: $notes" : '';
    return "Invoice for $customerId: $amount $currency$email$notesStr";
}

// Positional — what's that `true`? What's `null`?
$result1 = createInvoice('C-001', 5000, 'USD', 'Q1 fees', true, null);

// Named — crystal clear
$result2 = createInvoice(
    customerId: 'C-001',
    amount: 5000,
    currency: 'USD',
    notes: 'Q1 fees',
);

echo $result1 . PHP_EOL;
echo $result2 . PHP_EOL;
```

---

## Quiz

**Q1.** Refactor this using modern PHP 8 features:

```php
class User {
    private $name;
    private $email;
    private $role;
    
    public function __construct($name, $email, $role) {
        $this->name = $name;
        $this->email = $email;
        $this->role = $role;
    }
    
    public function getName() { return $this->name; }
    public function getEmail() { return $this->email; }
    public function getRole() { return $this->role; }
}
```

**Q2.** You see this in a Laravel codebase:

```php
if (strpos($user->email, '@gmail.com') !== false) {
    // ...
}
```

Refactor it. What's the senior's problem with the original?

**Q3.** When would you use a union type vs an interface? Give a concrete example for each.

**Q4.** A junior says "match is just like switch." Show why this is wrong with an example where the two diverge in behavior.

**Q5.** (Stretch) Look at this and explain the design:

```php
readonly class Email {
    public function __construct(public string $value) {
        if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
            throw new \InvalidArgumentException("Invalid: $value");
        }
    }
}

function sendWelcome(Email $email): void { ... }
```

Why `readonly class`? Why a class for an email instead of just a string?

---

## What You Now Have

✅ Mastery of the modern PHP 8 features that matter
✅ A sense of which features are trivia vs essential
✅ The skill to refactor old-style PHP into modern, safe code
✅ A complete OOP and functional vocabulary

## What's Next

Lesson 13 covers **error handling** — exceptions, try/catch, custom exceptions, and the senior's rule for when to throw, when to catch, and when to let it bubble.

📖 **[Lesson 13 — Error Handling: Exceptions and The Why](13-error-handling-exceptions-and-the-why.md)**
