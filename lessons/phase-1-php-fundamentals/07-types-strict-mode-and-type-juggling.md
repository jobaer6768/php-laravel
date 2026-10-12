# Lesson 07 — Types, Strict Mode, and Type Juggling

> **Phase 1: PHP Fundamentals** | You'll be here: ~35 min

## The Senior's Question

PHP was famously "loosely typed." Numbers became strings, strings became booleans, and your code did the wrong thing silently. Modern PHP fixed this — but only if you turn it on. What is strict mode, and why is it non-negotiable for a senior's code?

---

## Theory: PHP Has Two Modes

### Weak mode (default if you don't say otherwise)

```php
<?php
function add(int $a, int $b): int {
    return $a + $b;
}

echo add("5", "10");  // 15 — strings coerced to ints
echo add(5.7, 3);     // 8 — float truncated
echo add(true, 1);    // 2 — true is 1
```

PHP **coerces** types. It tries to do what you probably meant.

### Strict mode

```php
<?php
declare(strict_types=1);

function add(int $a, int $b): int {
    return $a + $b;
}

echo add("5", "10");  // TypeError — must be int
echo add(5.7, 3);     // TypeError — must be int
```

> **In strict mode, PHP refuses to coerce.** It throws a `TypeError` exception.

### What `declare(strict_types=1)` actually does

It's a **per-file directive.** Affects only the file where it's declared. Place it at the top, right after `<?php`. Some teams enable it globally with a code style rule. Laravel itself does **not** declare it by default (yet), so you'll add it yourself.

## Why It Matters

This is the **#1 source of "weird PHP bugs."** A junior writes:

```php
function getStudent(int $id): Student {
    return Student::find($id);
}
```

Then in tinker:
```php
getStudent("5");  // works in weak mode, fails in strict
```

In strict mode, the type mismatch is caught immediately. In weak mode, it works "by accident" — and one day, when `$id` is `"5abc"`, it silently becomes `5` and you get the wrong student.

> **Senior rule:** **`declare(strict_types=1);` in every PHP file you write.** No exceptions.

---

## Theory: PHP's Type System (What Types Exist)

### Scalar types (singular values)

| Type | Example | Notes |
|---|---|---|
| `string` | `"John"` | UTF-8 by default in modern PHP |
| `int` | `42` | 64-bit on most systems |
| `float` | `3.14` | Double precision. Same gotchas as JS floats. |
| `bool` | `true` / `false` | Lowercase only |

### Compound types

| Type | Example | Notes |
|---|---|---|
| `array` | `[1, 2, 3]` | Ordered map; can be list or assoc |
| `object` | `new stdClass()` | Any object |
| `callable` | A function name as string, or closure | For callbacks |
| `iterable` | Anything you can `foreach` over | Array or Traversable |

### Special types

| Type | Example | Notes |
|---|---|---|
| `mixed` | Anything | "I don't care what you pass." Use sparingly. |
| `void` | (return only) | "Returns nothing" |
| `never` | (return only) | "Always throws or exits" |
| `null` | `null` | Single value: `null` |
| `false` | (return only, PHP 8.0+) | "Always returns false" |

### Union types (PHP 8.0+)

```php
function find(int|string $id): Student|null {
    return Student::find($id);
}
```

A parameter or return can be **multiple types.** Very common for IDs (sometimes int, sometimes UUID string).

---

## Theory: Type Juggling — The Land Mines

Even in modern PHP, **operators** still do coercion. This is the trap:

```php
// Comparisons
0 == "abc"      // true (in weak mode!) — both are falsy
0 === "abc"     // false — strict comparison

"1" == 1        // true (weak) — strings can be numeric
"1" === 1       // false (strict) — different types

null == 0       // true (weak) — null is falsy
null === 0      // false

[] == false     // true (weak) — empty array is falsy
[] === false    // false

true == "1"     // true (weak)
true === "1"    // false
```

> **Senior rule:** use `===` and `!==` always. **Always.** Triple-equals is the default in a senior's code. `==` is a code smell.

The "0 == 'abc'" thing is the most famous PHP WTF. **Never happens in strict mode** for typed parameters, but it can still happen in `if` conditions and `==` comparisons.

### When does type juggling still bite you in strict mode?

In **conditional expressions**. The `if` statement still applies truthy/falsy rules:

```php
declare(strict_types=1);

$value = 0;
if ($value) { /* NOT executed */ }     // 0 is falsy
if ($value === 0) { /* executed */ }   // explicit check

$name = "";
if ($name) { /* NOT executed */ }       // empty string is falsy
if (empty($name)) { /* executed */ }    // empty() is the helper
```

### The "falsy" values in PHP

| Value | Falsy? |
|---|---|
| `false` | ✅ |
| `0`, `0.0`, `"0"` | ✅ |
| `""` empty string | ✅ |
| `null` | ✅ |
| `[]` empty array | ✅ |
| **Everything else** | ❌ (truthy) |

> **Note:** unlike JavaScript, `"0"` is falsy in PHP. The string `" "` (space) is **truthy.**

---

## Theory: Nullable Types and the `?` Prefix

In PHP, a parameter without a default can't be `null`:

```php
function greet(string $name): string {
    return "Hello $name";
}

greet(null);  // TypeError — must be string
```

To allow null, prefix with `?`:

```php
function greet(?string $name): string {
    return "Hello " . ($name ?? 'stranger');
}

greet(null);  // "Hello stranger"
greet("Aisha");  // "Hello Aisha"
```

The `?` is shorthand for `string|null` in the parameter list.

---

## Theory: Return Types and the `void` / `never` Keywords

```php
function log(string $message): void {
    file_put_contents('log.txt', $message);
    // No return statement
}

function redirect(string $url): never {
    header("Location: $url");
    exit;
}

function findOrFail(int $id): Student {
    $student = Student::find($id);
    if (!$student) {
        throw new \RuntimeException("Not found");
    }
    return $student;
}
```

> **Senior rule:** **always declare return types.** Even for `void`. The type signature is your function's contract. Other developers read it and know what to expect.

---

## Lab: Type Experiments

### Step 1 — Create the practice file

```bash
cd G:\php-laravel
touch practice/02-types.php
```

### Step 2 — Try the difference between weak and strict mode

```php
<?php
// File 1: weak mode (no declare)
// Save as practice/02a-weak.php

function multiply(int $a, int $b): int {
    return $a * $b;
}

echo multiply("3", "4") . PHP_EOL;   // 12
echo multiply(3.7, 2) . PHP_EOL;     // 6 (truncated)
echo multiply("0x1A", 1) . PHP_EOL;   // 26 (hex parsed!)
```

Run:
```bash
php practice/02a-weak.php
```

```php
<?php
// File 2: strict mode
// Save as practice/02b-strict.php

declare(strict_types=1);

function multiply(int $a, int $b): int {
    return $a * $b;
}

echo multiply("3", "4") . PHP_EOL;   // TypeError
```

Run:
```bash
php practice/02b-strict.php
```

> **Senior observation:** even in strict mode, `multiply(3.7, 2)` would throw because `3.7` is `float`, not `int`. The integer 3 and float 3.7 are different types. PHP doesn't auto-truncate.

### Step 3 — Try union types

```php
<?php
// practice/02c-union.php
declare(strict_types=1);

function formatId(int|string $id): string {
    if (is_int($id)) {
        return "ID-$id";
    }
    return "ID-" . strtoupper($id);
}

echo formatId(42) . PHP_EOL;        // ID-42
echo formatId("abc123") . PHP_EOL;  // ID-ABC123
```

### Step 4 — Try nullable and null coalescing

```php
<?php
// practice/02d-null.php
declare(strict_types=1);

function greet(?string $name): string {
    return "Hello, " . ($name ?? 'stranger') . "!";
}

echo greet("Aisha") . PHP_EOL;  // Hello, Aisha!
echo greet(null) . PHP_EOL;     // Hello, stranger!
```

> **Note the `??` operator:** the null-coalescing operator. Returns left if not null, else right. Equivalent to `$name !== null ? $name : 'stranger'`. You'll use this constantly.

### Step 5 — Try strict comparisons

```php
<?php
// practice/02e-strict-comparison.php
declare(strict_types=1);

$tests = [
    [0, "0"],
    [0, ""],
    [null, false],
    ["1", 1],
    [[], false],
];

foreach ($tests as [$a, $b]) {
    $weak = $a == $b ? 'true' : 'false';
    $strict = $a === $b ? 'true' : 'false';
    echo "loose: $weak | strict: $strict" . PHP_EOL;
}
```

Run it. Look at the table. Notice how often `==` lies.

### Step 6 — Try type-casting functions

```php
<?php
// practice/02f-casts.php
declare(strict_types=1);

$input = "42 students";

echo (int)$input . PHP_EOL;       // 42 — cast string to int
echo (float)"3.14" . PHP_EOL;     // 3.14
echo (string)true . PHP_EOL;      // "1"
echo (bool)0 . PHP_EOL;           // "" (empty — bool false renders empty)
echo (array)"hello" . PHP_EOL;    // Array ( [0] => hello )
```

> **Senior rule:** cast early, at the boundary where untrusted data enters your code (form input, API response, file read). Inside your app, types should be enforced by signatures.

---

## Theory: Where Strict Mode Doesn't Help (Yet)

Even with `strict_types=1`, these still happen:

1. **`==` vs `===`** — strict mode doesn't change comparison operators
2. **Conditional truthiness** — `if ($value)` still applies falsy rules
3. **JSON decoding** — `json_decode()` returns mixed types
4. **Database results** — come back as strings unless you cast in the model

You'll handle these with:
- Always use `===`
- Use explicit `is_*` functions: `is_int()`, `is_string()`, `is_null()`
- Cast JSON manually: `(int)json_decode($input)`
- Use Eloquent's `$casts` (we saw this in Lesson 04)

---

## Quiz

**Q1.** Predict the output of each line. Then run it and check:

```php
declare(strict_types=1);

echo "0" == false ? "yes" : "no";   // ?
echo "0" === false ? "yes" : "no";  // ?
echo [] == 0 ? "yes" : "no";        // ?
echo [] === 0 ? "yes" : "no";       // ?
```

**Q2.** Why is `declare(strict_types=1)` not enough to prevent all type confusion? Give two concrete examples of bugs that strict mode doesn't catch.

**Q3.** A function `processPayment(float $amount)` is called with `"99.99"`. In strict mode, what happens? Without strict mode, what might happen that's dangerous for money?

**Q4.** Write a function signature for `findStudent` that:
- Takes either an `int` ID or a `string` email
- Returns either a `Student` or `null`
- Doesn't accept `null` for the parameter

**Q5.** (Stretch) Look at this line and explain why it's a senior-vs-junior moment:

```php
// Junior
if ($user->is_active == 1) { ... }

// Senior
if ($user->is_active === true) { ... }
```

---

## What You Now Have

✅ Deep understanding of PHP's type system
✅ Comfort with `declare(strict_types=1)` and when it helps
✅ The habit of using `===` over `==`
✅ Knowledge of nullable types and union types
✅ Type-casting skills

## What's Next

Lesson 08 covers **arrays and collections** — the workhorse of PHP. This is the most important data structure in the language, and Laravel's Collection class wraps it with a beautiful API.

📖 **[Lesson 08 — Arrays, Collections, and the List Pattern](08-arrays-collections-and-the-list-pattern.md)**
