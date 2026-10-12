# Lesson 35 — Code Quality, Static Analysis, and CI

> **Phase 4: Quality** | You'll be here: ~30 min

## The Senior's Question

Tests catch runtime bugs. But what about bugs that exist at code-review time? Type mismatches. Unused variables. Deprecated functions. Inconsistent style. A senior runs **static analysis** and **CI** so these never reach production.

---

## Theory: The Quality Pipeline

```
Code → Style Check → Static Analysis → Tests → Review → Merge
        (Pint)        (PHPStan)        (Pest)    (Human)
```

Each gate catches different issues:

| Tool | Catches |
|---|---|
| **Pint** (Laravel's code style fixer) | Inconsistent style, wrong indentation |
| **PHPStan** (static analysis) | Type errors, dead code, wrong API usage |
| **Pest/PHPUnit** (tests) | Logic bugs, regression |
| **Human review** | Design issues, business logic |
| **CI** | Automates the first three |

---

## Theory: Pint — Laravel's Code Style Fixer

```bash
composer require laravel/pint --dev
```

Run it:

```bash
./vendor/bin/pint           # Fix all files
./vendor/bin/pint --test    # Check without changing
```

Pint uses **Laravel's coding style** by default (PSR-12 + Laravel conventions). It auto-fixes:

- Indentation
- Spacing
- Import order
- Unused imports
- Trailing whitespace
- And more

> **Senior rule:** run Pint on every commit. **Auto-fix and move on.** Style debates are a waste of time.

Configure in `pint.json`:

```json
{
    "preset": "laravel",
    "rules": {
        "simplified_null_return": true,
        "no_unused_imports": true,
        "not_operator_with_successor_space": true,
        "ordered_imports": {
            "sort_algorithm": "alpha"
        }
    }
}
```

---

## Theory: PHPStan — Static Analysis

PHPStan reads your code without running it. It catches:

- Calling a method that doesn't exist
- Wrong argument types
- Undefined variables
- Dead code
- Missing return types
- Wrong return types

```bash
composer require --dev phpstan/phpstan larastan/larastan
```

Create `phpstan.neon`:

```neon
includes:
    - vendor/larastan/larastan/extension.neon

parameters:
    paths:
        - app/
    
    level: 5   # 0-9, higher is stricter
    
    ignoreErrors:
        # Add specific false positives here
```

Run it:

```bash
./vendor/bin/phpstan analyse
```

### Levels explained

| Level | Strictness |
|---|---|
| 0 | Basic checks |
| 1 | Possibly undefined variables |
| 2 | Unknown methods on $this |
| 3 | Unknown methods on non-$this |
| 4 | Dead code, always falsy |
| 5 | Function/method arg types |
| 6 | Missing typehints |
| 7 | Partially wrong union types |
| 8 | Stricter nullability checks |
| 9 | Maximum strictness |

> **Senior rule:** start at level 5. Bump to 7+ as your codebase matures. The stricter, the more bugs caught.

### Example errors

```php
// PHPStan error: "Parameter #1 $student of method execute() expects Student, App\Models\Student|null given."
$student = Student::find($id);
app(RecordPayment::class)->execute($student, 100, 'cash');  // $student could be null
```

> **PHPStan catches this at code review time.** The runtime equivalent: a `null` passed to `execute()` crashes on the first property access.

---

## Theory: Rector — Automated Refactoring

Rector reads your code and applies modernizations:

```bash
composer require --dev rector/rector
```

Create `rector.php`:

```php
<?php

use Rector\Config\RectorConfig;
use Rector\Set\ValueObject\SetList;

return RectorConfig::configure()
    ->withPaths(['app', 'tests'])
    ->withSets([
        SetList::CODE_QUALITY,
        SetList::DEAD_CODE,
        SetList::TYPE_DECLARATION,
    ]);
```

Run it:

```bash
./vendor/bin/rector process        # Apply changes
./vendor/bin/rector process --dry-run  # See what would change
```

> **Rector can transform your code:** `array() → []`, `public function foo(): void` for type hints, modern PHP 8 syntax. **Powerful but review its output carefully.**

---

## Theory: CI — Continuous Integration

Every push to GitHub should automatically:

1. Run Pint
2. Run PHPStan
3. Run tests
4. Report results

This is **CI.** Without it, a junior can break tests without noticing.

### GitHub Actions

Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      mysql:
        image: mysql:8
        env:
          MYSQL_ROOT_PASSWORD: secret
          MYSQL_DATABASE: coaching_test
        ports:
          - 3306:3306
        options: --health-cmd="mysqladmin ping" --health-interval=10s --health-timeout=5s --health-retries=5
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          extensions: mbstring, pdo_mysql, bcmath
          coverage: xdebug
      
      - name: Cache Composer
        uses: actions/cache@v3
        with:
          path: vendor
          key: ${{ runner.os }}-php-${{ hashFiles('composer.json') }}
      
      - name: Install dependencies
        run: composer install --prefer-dist --no-interaction
      
      - name: Copy .env
        run: cp .env.example .env
      
      - name: Generate key
        run: php artisan key:generate
      
      - name: Run migrations
        run: php artisan migrate --force
        env:
          DB_CONNECTION: mysql
          DB_HOST: 127.0.0.1
          DB_DATABASE: coaching_test
          DB_USERNAME: root
          DB_PASSWORD: secret
      
      - name: Run Pint
        run: ./vendor/bin/pint --test
      
      - name: Run PHPStan
        run: ./vendor/bin/phpstan analyse --no-progress
      
      - name: Run tests
        run: php artisan test --parallel
        env:
          DB_CONNECTION: mysql
          DB_HOST: 127.0.0.1
          DB_DATABASE: coaching_test
          DB_USERNAME: root
          DB_PASSWORD: secret
```

> **This runs on every push.** It blocks merging if anything fails. **Senior discipline.**

---

## Theory: Pre-commit Hooks

Catch issues **before** they hit CI:

```bash
composer require --dev brainmaestro/composer-git-hooks
```

In `composer.json`:

```json
"scripts": {
    "post-install-cmd": [
        "cghooks add"
    ]
},
"extra": {
    "hooks": {
        "pre-commit": [
            "composer pint",
            "composer test"
        ],
        "pre-push": [
            "composer pint-test",
            "composer phpstan",
            "composer test"
        ]
    }
}
```

> **Now when you commit, Pint and tests run automatically.** If they fail, the commit is blocked.

---

## Theory: EditorConfig and Pint Together

`.editorconfig`:

```ini
root = true

[*]
indent_style = space
indent_size = 4
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

[*.md]
trim_trailing_whitespace = false
```

> **Ensures consistent editor behavior across the team.**

---

## Theory: Code Review Discipline

Even with great tools, humans review code. A senior's checklist:

### Before submitting a PR

- [ ] Tests cover the new behavior
- [ ] Existing tests still pass
- [ ] Pint is clean
- [ ] PHPStan is clean
- [ ] No debug code (`dd`, `dump`, `var_dump`)
- [ ] No commented-out code (delete it; git remembers)
- [ ] Migration handles the down case
- [ ] New env vars are in `.env.example`
- [ ] Documentation is updated if public-facing

### Reviewing someone else's PR

- [ ] Does the code do what the PR description says?
- [ ] Are tests meaningful (not just `expect(true)->toBeTrue()`)?
- [ ] Are there obvious bugs?
- [ ] Is the design consistent with the rest of the codebase?
- [ ] Is the naming clear?
- [ ] Are there security issues?

> **Senior rule:** if you wouldn't be comfortable deploying this Friday at 5pm, don't approve it.

---

## Why It Matters

CI and static analysis are the **safety net beneath the tests.** They catch:

- Things tests miss (wrong types)
- Things humans miss (style, dead code)
- Things that take 5 minutes to find manually (but 0 seconds with a tool)

> **A senior's code always runs Pint + PHPStan + Tests before being reviewed.** The human reviewer focuses on design, not style.

---

## Lab: Set Up the Full Quality Pipeline

### Step 1 — Install Pint

```bash
cd G:\php-laravel\coaching-saas
composer require laravel/pint --dev
```

Configure `pint.json`:

```json
{
    "preset": "laravel",
    "rules": {
        "no_unused_imports": true,
        "ordered_imports": {
            "sort_algorithm": "alpha"
        },
        "trailing_comma_in_multiline": true
    }
}
```

Run it:

```bash
./vendor/bin/pint
```

> **Pint auto-fixes your code style.** Commit the changes.

### Step 2 — Install PHPStan

```bash
composer require --dev phpstan/phpstan larastan/larastan
```

Create `phpstan.neon`:

```neon
includes:
    - vendor/larastan/larastan/extension.neon

parameters:
    paths:
        - app/
    
    level: 5
    
    ignoreErrors:
        # 'Access to an undefined property Illuminate\Database\Eloquent\Model::$tenant_id.'
        - "#Access to an undefined property .+::\\$tenant_id#"
```

Run it:

```bash
./vendor/bin/phpstan analyse
```

> **You may see errors. Fix them.** Some are false positives; add to `ignoreErrors` with a comment explaining why.

### Step 3 — Add scripts to composer.json

```json
"scripts": {
    "test": "php artisan test",
    "test-coverage": "php artisan test --coverage",
    "pint": "./vendor/bin/pint",
    "pint-test": "./vendor/bin/pint --test",
    "phpstan": "./vendor/bin/phpstan analyse",
    "analyse": [
        "@pint-test",
        "@phpstan",
        "@test"
    ]
}
```

Now you can run:

```bash
composer analyse    # runs all three
```

### Step 4 — Set up GitHub Actions

Create `.github/workflows/ci.yml` (use the template from above).

> **Push to GitHub** to trigger the workflow. Check the "Actions" tab.

### Step 5 — Add pre-commit hooks

```bash
composer require --dev brainmaestro/composer-git-hooks
```

Add to `composer.json`:

```json
"extra": {
    "hooks": {
        "pre-commit": "composer pint-test"
    }
}
```

```bash
composer cghooks add
```

> **Now every commit runs Pint.** Bad code can't enter the repo.

### Step 6 — Add a README badge

```markdown
[![CI](https://github.com/yourname/coaching-saas/actions/workflows/ci.yml/badge.svg)](https://github.com/yourname/coaching-saas/actions)
```

> **Public badge shows your CI passes.** Employers, contributors, future-you all see it.

---

## Quiz

**Q1.** Why run static analysis when you have tests? What does PHPStan catch that tests don't?

**Q2.** You have PHPStan level 5. You want to upgrade to 7. Walk through the steps to do this safely.

**Q3.** A junior commits code with `dd($variable)` left in. The CI passes. A senior is angry. Why?

**Q4.** Why have a pre-commit hook AND a CI workflow? Aren't they redundant?

**Q5.** (Stretch) PHPStan ignores "mixed" type by default. Your codebase has lots of mixed (especially from JSON). How would you incrementally improve types?

---

## What You Now Have

✅ Pint for code style
✅ PHPStan for static analysis
✅ GitHub Actions for CI
✅ Pre-commit hooks for local safety
✅ A senior's quality pipeline

## What's Next

**Phase 4 complete!** You can write tested, type-safe, auto-formatted code.

**Phase 5: Production** begins with **caching** — making your app fast under load. Redis, cache invalidation, the patterns that prevent "the system is slow" complaints.

📖 **[Phase 5 starts: Lesson 36 — Caching with Redis and Cache Invalidation](phase-5-production/36-caching-with-redis-and-cache-invalidation.md)**
