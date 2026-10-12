# Lesson 09 — Functions, Closures, and Arrow Functions

> **Phase 1: PHP Fundamentals** | You'll be here: ~30 min

## The Senior's Question

Laravel is built on closures. Routes are closures. Middleware is closures. Query scopes are closures. Migration up/down are callbacks. You must understand PHP's function flavors — named, anonymous, arrow — and how they capture variables — before you can read Laravel code.

---

## Theory: Three Function Flavors

### 1. Named functions

```php
function add(int $a, int $b): int {
    return $a + $b;
}
```

Top-level, named, callable by name. **Available globally** within the namespace (after the declaration line in PHP — unlike JS hoisting).

### 2. Anonymous functions (closures)

```php
$greet = function (string $name): string {
    return "Hello $name";
};

echo $greet("Aisha");  // Hello Aisha
```

A function value. Assign to a variable, pass to other functions, store in arrays.

### 3. Arrow functions (PHP 7.4+)

```php
$double = fn(int $n): int => $n * 2;

echo $double(5);  // 10
```

One-liner. Implicit return. No `function` keyword. **Closest to JS arrow functions.**

```js
// JS
const double = (n) => n * 2;
```

```php
// PHP
$double = fn($n) => $n * 2;
```

> **Senior rule:** prefer arrow functions for short transforms. Use named functions for anything with more than 1-2 expressions.

---

## Theory: The `use` Keyword — Variable Capture

This is the part that surprises everyone. In PHP, **closures don't automatically see outer variables.** You must declare them with `use`:

```php
$prefix = "Mr.";

$greet = function (string $name) use ($prefix): string {
    return "$prefix $name";
};

echo $greet("Aisha");  // "Mr. Aisha"
```

> **Arrow functions capture automatically — no `use` needed:**

```php
$prefix = "Mr.";
$greet = fn(string $name) => "$prefix $name";
echo $greet("Aisha");  // "Mr. Aisha"
```

This is a major quality-of-life improvement. **Use arrow functions whenever you need variable capture in a one-liner.**

### Capturing by value vs by reference

```php
$count = 0;

$counter = function () use ($count) {
    return $count + 1;
};
echo $counter();  // 1 — $count is 0
// $count is still 0, the closure has a copy

$mutator = function () use (&$count) {
    $count++;
};
$mutator();
$mutator();
echo $count;  // 2 — by reference
```

> **Senior rule:** capture by reference is rare and usually a smell. If you need to mutate state, use a class.

---

## Theory: Type Hints on Functions

PHP 8.0+ lets you type-hint everything:

```php
function process(
    string $name,
    int $age,
    ?array $tags = null,
    callable $callback = null,
): string {
    // ...
}
```

And in PHP 8.1+ — **never** return type:

```php
function redirect(string $url): never {
    header("Location: $url");
    exit;
}
```

`never` tells readers "this function always throws or exits."

---

## Theory: Variadic Functions

```php
function sum(int ...$numbers): int {
    return array_sum($numbers);
}

echo sum(1, 2, 3, 4, 5);  // 15
```

> **You'll use this when building flexible APIs.** For example, a `Mail::to()` that takes any number of recipients.

---

## Theory: First-Class Callables (PHP 8.1+)

You can pass a function by name as a callable:

```php
$double = double(...);  // first-class callable syntax

function double(int $n): int { return $n * 2; }

array_map($double, [1, 2, 3]);  // [2, 4, 6]
```

> **Easier than the old way** (`'double'` as a string, or `[new SomeClass, 'method']`).

---

## Theory: How Laravel Uses Closures (Preview)

```php
// Route
Route::get('/students', function () {
    return view('students.index');
});

// Middleware
$middleware->append(function ($request, $next) {
    // ... do something ...
    return $next($request);
});

// Query scope
Builder::macro('active', function () {
    return $this->where('is_active', true);
});

// Migration
public function up(): void {
    Schema::create('students', function (Blueprint $table) {
        // ...
    });
}
```

Closures are everywhere. **Master them now.**

---

## Why It Matters

In MERN, you pass functions constantly: `array.map(x => x * 2)`, `.then(res => ...)`, event handlers. PHP has the same model but with different syntax. Once you internalize:

- `function ($x) { ... }` — explicit body
- `fn($x) => ...` — implicit return, auto-capture
- `use ($var)` — explicit capture (only in non-arrow)

…you can read 90% of Laravel code without stumbling.

---

## Lab: Function Patterns

### Step 1 — Create the practice file

```bash
cd G:\php-laravel
touch practice/04-functions.php
```

### Step 2 — Practice all three flavors

```php
<?php
// practice/04-functions.php
declare(strict_types=1);

// 1. Named function
function describeStudent(string $name, int $grade): string
{
    return "$name scored $grade";
}

echo describeStudent('Aisha', 92) . PHP_EOL;

// 2. Anonymous function assigned to variable
$greet = function (string $name): string {
    return "Hello, $name!";
};
echo $greet('Bilal') . PHP_EOL;

// 3. Arrow function
$upper = fn(string $s) => strtoupper($s);
echo $upper('chen') . PHP_EOL;
```

### Step 3 — Practice variable capture

```php
// Closure: must use `use` to capture
$institution = "ABC Coaching";

$welcome = function (string $name) use ($institution): string {
    return "Welcome to $institution, $name!";
};
echo $welcome('Aisha') . PHP_EOL;

// Arrow function: auto-captures
$goodbye = fn(string $name) => "Goodbye from $institution, $name.";
echo $goodbye('Aisha') . PHP_EOL;
```

### Step 4 — Use closures as callbacks

```php
$numbers = [1, 2, 3, 4, 5];

// Pass closure
$squared = array_map(function (int $n): int {
    return $n * $n;
}, $numbers);
print_r($squared);

// Pass arrow
$doubled = array_map(fn($n) => $n * 2, $numbers);
print_r($doubled);
```

### Step 5 — Use closures in `usort`

```php
$students = [
    ['name' => 'Aisha',  'grade' => 85],
    ['name' => 'Bilal',  'grade' => 72],
    ['name' => 'Chen',   'grade' => 91],
];

usort($students, fn($a, $b) => $b['grade'] <=> $a['grade']);
print_r($students);
```

> **Notice the `<=>` spaceship operator** from Lesson 08, used inside an arrow function. This is the canonical PHP way to write comparators.

### Step 6 — Practice type declarations

```php
function processPayment(
    float $amount,
    string $currency = 'USD',
    ?string $notes = null,
    callable $onSuccess = null,
): string {
    $result = "Processed $amount $currency";
    if ($notes) {
        $result .= " (Note: $notes)";
    }
    if ($onSuccess) {
        $onSuccess($result);
    }
    return $result;
}

echo processPayment(99.99, 'BDT', 'Q1 fees') . PHP_EOL;
echo processPayment(50.00, onSuccess: fn($r) => print("OK: $r" . PHP_EOL)) . PHP_EOL;
```

> **Notice the named argument:** `onSuccess: fn(...)`. PHP 8.0+ named arguments. Makes calls self-documenting.

### Step 7 — Practice variadics

```php
function average(int ...$numbers): float {
    if (count($numbers) === 0) return 0.0;
    return array_sum($numbers) / count($numbers);
}

echo average(85, 90, 78) . PHP_EOL;       // 84.333...
echo average(100) . PHP_EOL;              // 100
echo average() . PHP_EOL;                 // 0
```

### Step 8 — Try first-class callables

```php
$str = 'Aisha';
$upper = strtoupper(...);  // first-class callable

echo $upper('chen') . PHP_EOL;  // CHEN
```

### Step 9 — Build a "pipeline" (real-world pattern)

A senior's pattern: chain a value through a series of functions.

```php
function pipeline(mixed $input, callable ...$stages): mixed
{
    return array_reduce(
        $stages,
        fn($carry, $stage) => $stage($carry),
        $input
    );
}

$result = pipeline(
    '  hello world  ',
    fn($s) => trim($s),
    fn($s) => strtoupper($s),
    fn($s) => str_replace(' ', '_', $s),
);

echo $result . PHP_EOL;  // HELLO_WORLD
```

> **This is the same pattern Laravel uses for middleware.** Each stage transforms the request, then passes it to the next.

---

## Quiz

**Q1.** Translate this JS to PHP using the closest equivalent. There are multiple valid answers; pick the one a senior would prefer.

```javascript
const process = (items) => items
  .filter(x => x > 10)
  .map(x => x * 2);
```

**Q2.** What's wrong with this code, and why does it produce unexpected output?

```php
$multiplier = 2;
$double = function (int $n): int {
    return $n * $multiplier;
};
echo $double(5);  // ?
```

**Q3.** When would you use a named function instead of an arrow function? When would you use an anonymous function instead of either?

**Q4.** Write a function `retry(callable $operation, int $times): mixed` that calls `$operation` up to `$times` times, catching exceptions, and returns the result if any call succeeds. If all fail, throw the last exception.

**Q5.** (Stretch) Read this Laravel code and explain each closure's purpose:

```php
Route::middleware(['auth', 'tenant'])->group(function () {
    Route::get('/students', [StudentController::class, 'index']);
    Route::post('/students', [StudentController::class, 'store']);
});
```

What does the outer closure do? What does `['auth', 'tenant']` mean? What is `[StudentController::class, 'index']`?

---

## What You Now Have

✅ Mastery of PHP's three function flavors
✅ Variable capture rules (use `use` vs auto-capture)
✅ Type hints, variadics, named arguments, first-class callables
✅ Read-to-understand Laravel closure patterns

## What's Next

Lesson 10 covers **OOP** — classes, objects, namespaces, autoloading. This is the biggest mental shift from JavaScript, where OOP is "bolted on." PHP was designed for OOP. Embrace it.

📖 **[Lesson 10 — OOP: Classes, Objects, Namespaces](10-oop-classes-objects-namespaces.md)**
