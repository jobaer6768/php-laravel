# Lesson 08 — Arrays, Collections, and the List Pattern

> **Phase 1: PHP Fundamentals** | You'll be here: ~35 min

## The Senior's Question

PHP has a single data structure called "array" that does what JavaScript needs two (`Array` and `Map`) for. That's either a strength or a trap. How do you use it cleanly, and when do you graduate to Laravel Collections?

---

## Theory: PHP Arrays Are Maps

In JavaScript, you have:
- `Array` — ordered, indexed
- `Map` — keyed
- `Object` — keyed, prototype-based

In PHP, you have **one structure** that does all three: `array`. It's an **ordered map** — both indexed and associative.

```php
// Indexed (like JS array)
$fruits = ['apple', 'banana', 'cherry'];
echo $fruits[0];  // apple

// Associative (like JS object literal)
$user = ['name' => 'John', 'age' => 25];
echo $user['name'];  // John

// Mixed (this is unique to PHP)
$mixed = [
    'name' => 'John',
    'hobbies' => ['reading', 'coding'],
    'age' => 25,
    0 => 'first',
    1 => 'second',
];
```

> **Senior note:** mixed arrays are a smell. If you have string keys and integer keys in the same array, restructure. Use a class instead.

## Theory: Two Kinds of PHP Arrays

PHP internally calls them "arrays" but they're really two implementations under one type:

| Implementation | Use case | Example |
|---|---|---|
| **List** (packed) | Sequential integer keys, 0-indexed | `[1, 2, 3]` |
| **Hash table** | String or mixed keys | `['name' => 'John']` |

You don't usually care which one PHP uses internally. The language handles it. But knowing this explains why some operations are fast and some are slow:

- Accessing by integer key on a list: **O(1)**
- Accessing by string key on a hash table: **O(1)**
- Searching for a value: **O(n)**
- `array_unique`, `array_diff`: **O(n²)** in some cases

---

## Theory: Array Operations (The Essentials)

### Add to array
```php
$fruits = ['apple', 'banana'];
$fruits[] = 'cherry';              // append
array_push($fruits, 'date', 'fig');  // push multiple
array_unshift($fruits, 'apricot');   // prepend
```

### Remove from array
```php
$last = array_pop($fruits);        // remove and return last
$first = array_shift($fruits);     // remove and return first
unset($fruits[1]);                 // remove by index
```

### Check existence
```php
$fruits = ['apple', 'banana'];

in_array('apple', $fruits);          // true
array_key_exists('name', $user);     // true (for assoc arrays)
isset($fruits[0]);                   // true (also checks not null)
```

### Count, sort, filter, map
```php
$fruits = ['banana', 'apple', 'cherry'];

count($fruits);                      // 3
sort($fruits);                       // ['apple', 'banana', 'cherry'] — mutates!
rsort($fruits);                      // reverse sort
asort($user);                        // sort assoc by value
ksort($user);                        // sort assoc by key
```

> **Senior warning:** PHP's `sort()`, `rsort()`, `asort()` all **mutate the original array.** If you want a new array, copy first: `sorted = $fruits; sort($sorted);`

### Filter, Map, Reduce (functional style)

PHP has dedicated functions for these:

```php
$numbers = [1, 2, 3, 4, 5, 6];

// Filter (returns array of values where callback is true)
$evens = array_filter($numbers, fn($n) => $n % 2 === 0);
// [2, 4, 6]

// Map (returns array of callback results)
$squares = array_map(fn($n) => $n * $n, $numbers);
// [1, 4, 9, 16, 25, 36]

// Reduce (returns single value)
$sum = array_reduce($numbers, fn($carry, $n) => $carry + $n, 0);
// 21
```

> **The `fn` keyword is PHP 7.4+ arrow functions** — equivalent to JS arrow functions. Use them.

---

## Theory: Spread, Destructuring, and List (PHP 7.1+)

### Spread operator
```php
$first = [1, 2, 3];
$second = [4, 5, 6];
$combined = [...$first, ...$second];  // [1, 2, 3, 4, 5, 6]
```

### Array destructuring
```php
[$first, $second, $third] = [1, 2, 3];
// $first = 1, $second = 2, $third = 3

// Skip elements
[, $second, $third] = [1, 2, 3];

// Associative destructuring (PHP 7.1+)
['name' => $name, 'age' => $age] = $user;
```

> **Senior note:** in Laravel, you'll see destructuring used a lot in `foreach` loops over key-value pairs.

---

## Theory: When to Use Laravel Collections

For simple data, PHP arrays are perfect. For **transforming, filtering, and querying data**, Laravel's Collection class is dramatically better.

```php
// Using arrays
$activeStudents = array_filter($students, fn($s) => $s['is_active']);
$names = array_map(fn($s) => $s['name'], $activeStudents);

// Using Collections
$activeStudents = collect($students)
    ->filter(fn($s) => $s->is_active)
    ->map(fn($s) => $s->name);
```

> **Senior rule:** when an Eloquent query returns multiple models, you get a Collection automatically (`Student::all()` returns a Collection, not an array). When you build a list of data in your app, ask "do I need to chain transforms?" If yes, use a Collection.

### Key Collection methods

```php
$students = collect([
    ['name' => 'Aisha', 'grade' => 85, 'class' => '10A'],
    ['name' => 'Bilal', 'grade' => 72, 'class' => '10B'],
    ['name' => 'Chen',  'grade' => 91, 'class' => '10A'],
    ['name' => 'Daria', 'grade' => 68, 'class' => '10C'],
]);

// Filter
$topStudents = $students->filter(fn($s) => $s['grade'] >= 80);

// Map
$names = $students->map(fn($s) => $s['name']);

// Pluck (extract a column)
$grades = $students->pluck('grade');     // [85, 72, 91, 68]
$byName = $students->pluck('grade', 'name');  // ['Aisha' => 85, ...]

// Group by
$byClass = $students->groupBy('class');
// ['10A' => [...], '10B' => [...], '10C' => [...]]

// Sum, avg, max, min
$students->sum('grade');
$students->avg('grade');
$students->max('grade');
$students->min('grade');

// Sort
$students->sortBy('grade');
$students->sortByDesc('grade');

// First, last
$students->first();
$students->last();
$students->firstWhere('class', '10A');

// Chaining
$topStudentNames = $students
    ->filter(fn($s) => $s['grade'] >= 80)
    ->sortByDesc('grade')
    ->pluck('name')
    ->take(3)
    ->values();
```

> **This is why seniors prefer Collections.** Read that last chain top-to-bottom. It says: "filter top students, sort by grade descending, take their names, take the top 3, reset keys." With raw arrays and `array_*` functions, the same logic takes 4-5 lines and intermediate variables.

---

## Why It Matters

Arrays in PHP are the most-used type. You'll process them constantly:
- Form input arrives as an array
- Database results are arrays of arrays until cast to Models
- Configuration is arrays
- API responses are arrays

A senior:
- Knows the difference between indexed and associative
- Uses `===` and `isset()` properly
- Reaches for Collections when there's more than one transform
- Never uses nested arrays beyond 2 levels (otherwise use a class)

---

## Lab: Array Mastery

### Step 1 — Create the practice file

```bash
cd G:\php-laravel
touch practice/03-arrays.php
```

### Step 2 — Master indexing and iteration

```php
<?php
// practice/03-arrays.php
declare(strict_types=1);

$students = [
    ['id' => 1, 'name' => 'Aisha',  'class' => '10A', 'marks' => 85],
    ['id' => 2, 'name' => 'Bilal',  'class' => '10B', 'marks' => 72],
    ['id' => 3, 'name' => 'Chen',   'class' => '10A', 'marks' => 91],
    ['id' => 4, 'name' => 'Daria',  'class' => '10C', 'marks' => 68],
    ['id' => 5, 'name' => 'Esha',   'class' => '10A', 'marks' => 78],
];

// 1. Loop and print
foreach ($students as $student) {
    echo "{$student['name']} ({$student['class']}): {$student['marks']}\n";
}

// 2. Filter
$topPerformers = array_filter($students, fn($s) => $s['marks'] >= 80);
echo "Top performers: " . count($topPerformers) . PHP_EOL;

// 3. Map to names
$names = array_map(fn($s) => $s['name'], $students);
print_r($names);

// 4. Reduce to total
$total = array_reduce($students, fn($sum, $s) => $sum + $s['marks'], 0);
echo "Total: $total" . PHP_EOL;

// 5. Sort by marks desc — careful, this mutates!
$ranked = $students;
usort($ranked, fn($a, $b) => $b['marks'] <=> $a['marks']);
// Note: <=> is the "spaceship operator" for three-way comparison
foreach ($ranked as $i => $student) {
    echo ($i + 1) . ". {$student['name']} - {$student['marks']}\n";
}
```

Run:
```bash
php practice/03-arrays.php
```

> **The `<=>` operator is PHP's "spaceship operator":**
> - Returns -1 if left < right
> - Returns 0 if equal
> - Returns 1 if left > right
>
> It's the cleanest way to write comparison functions for `usort`. You'll use it often.

### Step 3 — Now do it with Laravel Collections

The same operations through Collection API:

```php
<?php
// practice/03b-collections.php
declare(strict_types=1);

require __DIR__ . '/../coaching-saas/vendor/autoload.php';

use Illuminate\Support\Collection;

$students = collect([
    ['name' => 'Aisha',  'class' => '10A', 'marks' => 85],
    ['name' => 'Bilal',  'class' => '10B', 'marks' => 72],
    ['name' => 'Chen',   'class' => '10A', 'marks' => 91],
    ['name' => 'Daria',  'class' => '10C', 'marks' => 68],
    ['name' => 'Esha',   'class' => '10A', 'marks' => 78],
]);

// Class average
$avg = $students->avg('marks');
echo "Class average: $avg" . PHP_EOL;

// Top 3 by marks
$top3 = $students
    ->sortByDesc('marks')
    ->take(3)
    ->pluck('name')
    ->implode(', ');
echo "Top 3: $top3" . PHP_EOL;

// Group by class
$byClass = $students->groupBy('class');
foreach ($byClass as $class => $group) {
    $avg = $group->avg('marks');
    echo "Class $class average: $avg" . PHP_EOL;
}

// Pass / fail
$stats = $students->reduce(function ($acc, $s) {
    $acc[$s['marks'] >= 50 ? 'pass' : 'fail'][] = $s['name'];
    return $acc;
}, ['pass' => [], 'fail' => []]);

print_r($stats);
```

> **Why does this work outside Laravel?** Because we required Composer's autoloader. The Illuminate components are installed in `coaching-saas/vendor/` and are reusable.

### Step 4 — Try `pluck` deeply

```php
// Nested plucking
$students = collect([
    ['name' => 'Aisha', 'marks' => ['math' => 90, 'english' => 80]],
    ['name' => 'Bilal', 'marks' => ['math' => 70, 'english' => 75]],
]);

$mathMarks = $students->pluck('marks.math');
// [90, 70]

$firstStudent = $students->firstWhere('marks.math', 90);
echo $firstStudent['name'];  // Aisha
```

### Step 5 — Map to a different shape

```php
$report = $students->map(function ($s) {
    return [
        'name' => $s['name'],
        'grade' => match (true) {
            $s['marks'] >= 90 => 'A+',
            $s['marks'] >= 80 => 'A',
            $s['marks'] >= 70 => 'B',
            $s['marks'] >= 60 => 'C',
            default          => 'F',
        },
    ];
});

print_r($report->toArray());
```

> **The `match` expression is PHP 8.0+.** Like `switch` but strict and returns a value. Much cleaner.

### Step 6 — Use `each` for side effects

```php
$students->each(function ($s) {
    echo "Sending report to {$s['name']}..." . PHP_EOL;
    // Imagine: Mail::to($s['email'])->send(...)
});
```

> **Note:** `each` doesn't return a new collection. For transformations, use `map`. For side effects, use `each`.

---

## Theory: When NOT to Use Collections

- **Tiny data (1-5 items):** just use the array directly
- **Performance-critical hot loops:** Collection methods have overhead. PHP arrays are faster.
- **Data you don't transform:** if you just iterate once, a `foreach` is fine

> **Senior rule:** if you find yourself using 3+ `array_*` calls in a row, switch to a Collection.

---

## Quiz

**Q1.** What's the difference between `array_filter($arr, $fn)` and a `foreach` with an `if` inside? In what situation would you prefer each?

**Q2.** Translate this to PHP using the spaceship operator. Then explain the order it sorts in.

```php
$sorted = $students->sortByDesc('marks');
```

**Q3.** Why does `array_filter()` preserve keys, and why does this sometimes surprise people? (Hint: try `->values()` after `->filter()`.)

**Q4.** You have a deeply nested array: `['data' => ['students' => [...]]]`. A senior says "this is a smell." Why? What would they refactor it to?

**Q5.** (Stretch) Look at the `match` expression in step 5. Why is it better than the equivalent `switch`? List two reasons.

---

## What You Now Have

✅ Confident with PHP arrays: indexed, associative, mixed
✅ Functional-style operations: `array_filter`, `array_map`, `array_reduce`
✅ The `<=>` spaceship operator and `match` expression
✅ Laravel Collections fluency
✅ Pattern recognition: when to use arrays vs Collections

## What's Next

Lesson 09 covers **functions, closures, and arrow functions** — the first-class function support in PHP, including how Laravel uses closures for routing, middleware, and queries.

📖 **[Lesson 09 — Functions, Closures, and Arrow Functions](09-functions-closures-and-arrow-functions.md)**
