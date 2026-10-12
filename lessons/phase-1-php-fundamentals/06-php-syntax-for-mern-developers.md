# Lesson 06 — PHP Syntax for MERN Developers

> **Phase 1: PHP Fundamentals** | You'll be here: ~30 min

## The Senior's Question

You're switching languages. What carries over from JavaScript/TypeScript, what breaks, and what should you refuse to do (because it's a PHP-ism from 2010 that modern PHP abandoned)?

---

## Theory: What Carries Over (A Lot)

Modern PHP and modern JavaScript are **closer than you think.** Both are:
- Dynamically typed (with optional strict mode)
- C-family syntax (`{}` blocks, `;` statements, `if`/`for`/`while`)
- Garbage-collected
- First-class functions
- Closures

You'll feel at home in 30 minutes. The differences are real but small.

## Theory: What Breaks (5 Big Things)

### 1. Variables start with `$`

```php
$name = "John";
$age = 25;
$isAdmin = true;
```

No `$` in JS. In PHP, **always.** Even in string interpolation:

```php
echo "Hello $name";          // works
echo "Hello {$name}";        // also works, safer
echo 'Hello $name';          // single quotes = NO interpolation
```

> **Senior rule:** use single quotes unless you need interpolation. Single quotes are marginally faster and signal "this is a literal."

### 2. Strings: single vs double quotes (real difference)

```php
$name = "John";

// Double quotes interpolate
echo "Hello $name";           // Hello John

// Single quotes are literal
echo 'Hello $name';           // Hello $name
```

In JavaScript, `'a'` and `"a"` are identical. In PHP, **they differ.** This is the first place you'll silently get bugs.

### 3. Array literal syntax: `[]` vs `array()`

```php
// Modern (PHP 5.4+) — use this
$nums = [1, 2, 3];
$user = ['name' => 'John', 'age' => 25];

// Old — you'll see in old code, never write it
$nums = array(1, 2, 3);
$user = array('name' => 'John', 'age' => 25);
```

The `=>` arrow for key-value pairs is the equivalent of TypeScript's `Record<string, T>` literal.

### 4. Concatenation with `.` (not `+`)

```php
$fullName = "John " . "Doe";          // "John Doe"
$count = 5;
echo "You have " . $count . " items"; // "You have 5 items"
```

In JS you can use `+` for both string concat and math. **PHP forces you to be explicit** — this is actually a feature. No more `"5" + 3` producing `"53"` or `8` depending on context.

### 5. Semicolons are mandatory

```php
$name = "John"   // ERROR — missing semicolon
$name = "John";  // correct
```

PHP will throw a `Parse error` and refuse to run. JS engines are more forgiving. PHP is strict. **End every statement with `;`.**

---

## Theory: The Mental Model Difference

### JS: Everything is a value, including functions
### PHP: Functions are values, but the language is **statement-oriented**

In PHP, a file is read top-to-bottom. There's no "hoisting" like JS. This is why **order matters** in PHP more than in JS:

```php
<?php
// greet();  // ERROR — function not yet defined
// In PHP, function declarations are also hoisted in most cases, but:
// variables, classes, constants are NOT.
```

### One file, one purpose

A PHP file usually:
- Declares a class
- Or defines functions
- Or runs procedural code (top-to-bottom)

In MERN, a `.ts` file often has many exports. In PHP, a file typically has **one main declaration** plus its imports.

---

## Theory: PHP Tags

A PHP file starts with `<?php` and ends with `?>` (the closing tag is optional if the file is pure PHP). Anything outside the tags is treated as raw output.

```php
<?php
// PHP code here
echo "Hello";
?>
HTML outside PHP tags
<?php
echo "More PHP";
?>
```

**Modern best practice:** omit the closing `?>` at the end of pure-PHP files. This prevents accidental whitespace from leaking into output (which causes "headers already sent" errors).

```php
<?php
// pure PHP file — no closing tag
```

---

## Theory: Comments (Three Styles)

```php
// Single-line (C++ style)

/* Multi-line (C style) */

/**
 * DocBlock — used for documentation
 * @param string $name
 * @return string
 */
function greet(string $name): string {
    return "Hello $name";
}
```

In modern Laravel code, you'll mostly use:
- `//` for inline
- `/** */` for documenting functions/classes

---

## Why It Matters

You are going to read **thousands of lines of PHP** written by other developers. The first time you see `$this->name` or `->method()` instead of `.method()` or `this.method`, you need to understand it without thinking.

The biggest mental shift: **PHP is older than you, and a lot of "PHP is bad" reputation comes from code written in 2005.** Modern PHP, especially 8.0+, reads like a modern, type-safe language. If you keep your style modern, you write code that's clean, fast, and maintainable.

---

## Lab: Translate JS to PHP

Open `G:\php-laravel\coaching-saas`. We'll write a small PHP file outside the Laravel app to practice.

### Step 1 — Create a practice file

```bash
# From your project root
cd G:\php-laravel
mkdir practice
touch practice/01-translate.php
```

(Windows PowerShell: `New-Item -ItemType File -Path practice/01-translate.php`)

### Step 2 — Write this JavaScript first, in your head

```javascript
const student = {
  name: "Aisha",
  age: 16,
  courses: ["Math", "Physics", "English"],
  greet() {
    return `Hello, I'm ${this.name} and I take ${this.courses.length} courses.`;
  }
};
console.log(student.greet());
```

### Step 3 — Now write the PHP equivalent

Open `practice/01-translate.php` and write:

```php
<?php
declare(strict_types=1);

$student = [
    'name' => 'Aisha',
    'age' => 16,
    'courses' => ['Math', 'Physics', 'English'],
];

function greet(array $student): string
{
    $count = count($student['courses']);
    return "Hello, I'm {$student['name']} and I take $count courses.";
}

echo greet($student) . PHP_EOL;
```

> **Run it:** `php practice/01-translate.php`

Expected output:
```
Hello, I'm Aisha and I take 3 courses.
```

### Step 4 — Read the differences

| JS | PHP | Why |
|---|---|---|
| `const` | (no const for variables) | PHP variables are always mutable. Use `const` for class constants, `define()` for global constants |
| `=>` in object literal | `=>` in array | Same syntax, different meaning |
| `${this.name}` | `{$student['name']}` | PHP interpolates `{}` blocks |
| `this.courses.length` | `count($student['courses'])` | PHP function instead of property |
| `console.log()` | `echo` (or `print`) | `echo` is a language construct, faster than `print` |
| ` ` ` template literal | `"..."` double-quoted string | Single quotes don't interpolate |
| `;` optional | `;` mandatory | PHP enforces |
| Top of file: nothing | `<?php` and `declare(strict_types=1);` | PHP needs the open tag; strict types is best practice |

### Step 5 — Try breaking it

Try each of these, one at a time, and read the error:

```php
// 1. Remove the semicolon
echo "test"

// 2. Use single quotes
echo 'Hello $student[name]';

// 3. Use + for string concat
echo "Hello " + $student['name'];

// 4. Forget the $ on a variable
echo name;
```

> **Senior habit:** every error you trigger intentionally teaches you the language's grammar. Don't fear errors.

### Step 6 — Try it with an `array()` instead of `[]`

```php
$nums = array(1, 2, 3);
print_r($nums);
```

Notice: it works. But every modern code style guide says use `[]`. This is how you read old code.

### Step 7 — Try variable variables (you'll see this in old code)

```php
$foo = 'bar';
$$foo = 'baz';

echo $bar;  // "baz" — variable variable. Don't do this. But know it exists.
```

> **Senior rule:** if you see `$$name` in old code, it's a sign the author wanted to dynamically build variable names. Modern code uses arrays for this. Never write this yourself.

---

## Theory: PHP's `echo` vs `print` vs `var_dump` vs `print_r`

You'll use all four:

| Function | Returns | Use for |
|---|---|---|
| `echo` | void | Quick output. Can take multiple args. Fastest. |
| `print` | 1 (always) | Same as echo, but returns a value. Rarely used. |
| `var_dump($x)` | void | Debugging. Prints type, length, value. Stops execution. |
| `print_r($x)` | void (or returns string) | Pretty-prints arrays/objects. |

Try them all:

```php
<?php
$student = ['name' => 'Aisha', 'age' => 16];

echo $student['name'] . PHP_EOL;          // Aisha
print $student['name'] . PHP_EOL;          // Aisha
var_dump($student);                        // array(2) { ["name"]=> string(5) "Aisha" ["age"]=> int(16) }
print_r($student);                         // Array ( [name] => Aisha [age] => 16 )
```

> **Senior rule:** `var_dump` and `print_r` are your debugging friends. Use them in tinker all the time.

---

## Theory: PHP_EOL — The Newline Constant

PHP runs on Windows, Linux, macOS. Each has a different newline character:
- Windows: `\r\n`
- Linux: `\n`
- Old Mac: `\r`

`PHP_EOL` is the right newline **for the current OS.** Always use it instead of hardcoding `\n` in strings meant for output.

For HTML, use `<br>` or `nl2br()`. For JSON, use `\n` literally. Different contexts, different newlines.

---

## Quiz

**Q1.** You write `echo "Hello $user['name']";` and PHP throws a parse error. Why? How do you fix it?

**Q2.** What's the difference between these two? Why does the second produce `Hello $name`?

```php
$name = "John";
echo "Hello $name";
echo 'Hello $name';
```

**Q3.** Translate this JS to PHP — both the array and the function:

```javascript
const prices = [10, 20, 30, 40];
const total = prices.reduce((sum, p) => sum + p, 0);
console.log(`Total: ${total}`);
```

**Q4.** A junior says "PHP is just like JavaScript, they both have `$` in shell scripts." Why is this a confused statement?

**Q5.** (Stretch) Read this PHP line and predict the output:

```php
$count = 5;
echo "Count: $count" . " items" . PHP_EOL;
```

What if the second string were `'Count: $count'`? Why?

---

## What You Now Have

✅ PHP syntax confidence
✅ Translation skills from JS
✅ Understanding of PHP's "statement-oriented" model
✅ First practice file written

## What's Next

Lesson 07 dives into **types** — strict mode, type juggling, the things that bite you when PHP silently converts between types. This is where most PHP bugs come from, and modern PHP's type system is your shield.

📖 **[Lesson 07 — Types, Strict Mode, and Type Juggling](07-types-strict-mode-and-type-juggling.md)**
