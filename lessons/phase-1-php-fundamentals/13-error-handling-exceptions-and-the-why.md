# Lesson 13 — Error Handling: Exceptions and The Why

> **Phase 1: PHP Fundamentals** | You'll be here: ~30 min

## The Senior's Question

A junior catches every exception. A senior catches almost none. Why is "catching exceptions" a code smell most of the time, and what's the right way to think about errors in modern PHP?

---

## Theory: PHP's Two Failure Modes

### 1. Errors (old-school)

```php
// PHP < 7
$result = file_get_contents('missing.txt');
// Returns false, emits a warning. Execution continues.
```

### 2. Exceptions (modern)

```php
// PHP 7+
try {
    $result = file_get_contents('missing.txt');
} catch (\Exception $e) {
    // Handle the failure
}
```

Since PHP 7, **most internal errors throw exceptions.** The old "return false and warn" behavior is mostly gone.

In PHP 8+, you have:

| Class | What it is |
|---|---|
| `\Throwable` | Interface — both Error and Exception implement it |
| `\Exception` | Things you should catch (recoverable problems) |
| `\Error` | Things you usually shouldn't catch (engine-level problems like calling undefined methods) |
| Custom subclasses | `App\Exceptions\StudentNotFound` etc. |

> **Senior rule:** catch `\Throwable` only at the top of the stack (controllers, queue workers). Catch specific `\Exception` subclasses at boundaries where you can recover.

---

## Theory: The Try/Catch/Finally Block

```php
try {
    // Code that might throw
} catch (SpecificException $e) {
    // Handle this specific exception
} catch (AnotherException $e) {
    // Handle a different one
} finally {
    // Always runs — cleanup
}
```

A real example:

```php
function transferFunds(Account $from, Account $to, Money $amount): bool
{
    try {
        DB::beginTransaction();
        
        $from->debit($amount);
        $to->credit($amount);
        
        DB::commit();
        return true;
    } catch (InsufficientFundsException $e) {
        DB::rollBack();
        Log::warning("Transfer failed: {$e->getMessage()}");
        return false;
    } catch (\Throwable $e) {
        DB::rollBack();
        Log::error("Unexpected error: {$e->getMessage()}");
        throw $e;  // re-throw — we don't know how to handle it
    }
}
```

> **Notice the re-throw at the end.** A senior catches specific exceptions they can handle, and lets everything else bubble up to whoever does.

---

## Theory: The Senior's Exception Strategy

### Rule 1: Throw early, catch late

Throw as close to the problem as possible. Catch as far away as makes sense to handle it.

```php
// Bad: catching too early hides the problem
function processPayment(Money $amount): void {
    try {
        $this->chargeCard($amount);
    } catch (\Throwable $e) {
        // "swallowing" the exception — caller never knows
        return;
    }
}

// Good: let it bubble to a place that knows what to do
function processPayment(Money $amount): void {
    $this->chargeCard($amount);
    // If chargeCard throws, this function throws.
    // Caller (controller) decides what to do.
}
```

### Rule 2: Don't catch generic `\Exception`

```php
// Bad
try { ... } catch (\Exception $e) { /* lost the specifics */ }

// Good
try { ... } catch (InsufficientFundsException $e) { ... }
try { ... } catch (CardDeclinedException $e) { ... }
```

Catch only what you can actually handle.

### Rule 3: Don't catch `\Error`

```php
// Usually wrong
try { ... } catch (\Error $e) { ... }
```

`\Error` is for engine-level problems (calling a method that doesn't exist, type errors). You should fix those, not catch them.

### Rule 4: Use `finally` for cleanup

```php
$file = fopen('data.csv', 'r');
try {
    while (($row = fgetcsv($file)) !== false) {
        processRow($row);
    }
} finally {
    fclose($file);  // Always runs, even if processRow throws
}
```

This is the **canonical pattern** for resource cleanup.

---

## Theory: Custom Exception Classes

For your coaching SaaS, you'll want exceptions like:

```php
namespace App\Exceptions;

class TenantNotFoundException extends \DomainException
{
    public static function withId(int $id): self
    {
        return new self("Tenant with ID $id not found");
    }
}

class InsufficientFundsException extends \DomainException
{
    public function __construct(
        public readonly Money $required,
        public readonly Money $available,
    ) {
        parent::__construct(
            "Required {$required->format()}, but only {$available->format()} available"
        );
    }
}

class SubscriptionExpiredException extends \DomainException {}
```

Notice the patterns:

| Pattern | Why |
|---|---|
| Extends `\DomainException` | `\DomainException` is the SPL class for "the domain rule was violated." Senior's choice over generic `\Exception`. |
| Named static constructors | `TenantNotFoundException::withId(42)` reads better than `new TenantNotFoundException(...)` |
| Carry data on the exception | `InsufficientFundsException` carries the amounts, so the catch site can use them |

### The SPL exception hierarchy

```
\Exception
├── \LogicException        ← bugs in code (programmer's fault)
│   ├── \BadFunctionCallException
│   ├── \BadMethodCallException
│   ├── \DomainException   ← business rule violated
│   ├── \InvalidArgumentException
│   ├── \LengthException
│   └── \OutOfRangeException
└── \RuntimeException      ← things outside your code (network, IO)
    ├── \OutOfBoundsException
    ├── \OverflowException
    ├── \RangeException
    ├── \UnderflowException
    └── \UnexpectedValueException
```

> **Senior rule:** choose the closest parent that fits:
> - Business rule violation → `\DomainException`
> - Bad input from user → `\InvalidArgumentException`
> - External service failed → `\RuntimeException` (or a custom subclass)

---

## Theory: Reporting vs Handling

A senior distinguishes between **reporting** and **handling** an exception.

**Handling** = you know what to do
```php
try {
    sendEmail($user);
} catch (SmtpConnectionException $e) {
    // We know: queue it for later
    dispatch(new RetryEmailJob($user))->delay(now()->addMinutes(5));
}
```

**Reporting** = you log it, then re-throw
```php
try {
    $this->chargeCard($amount);
} catch (\Throwable $e) {
    Log::error('Card charge failed', [
        'amount' => $amount->format(),
        'error'  => $e->getMessage(),
    ]);
    throw $e;  // someone else decides what to do
}
```

> **Senior rule:** if you can't recover, log and re-throw. Don't swallow.

---

## Theory: Laravel's Exception Handling

Laravel has a global exception handler at `app/Exceptions/Handler.php`. Every uncaught exception flows through it. You customize **how** exceptions are reported and rendered (what the user sees vs. what gets logged).

```php
// app/Exceptions/Handler.php
public function register(): void
{
    $this->renderable(function (TenantNotFoundException $e, Request $request) {
        if ($request->expectsJson()) {
            return response()->json(['error' => 'Tenant not found'], 404);
        }
        return response()->view('errors.tenant-not-found', [], 404);
    });
}
```

We'll cover this in detail in Lesson 15 (Controllers) and Lesson 19 (Form Requests).

---

## Why It Matters

Your SaaS will have hundreds of possible failure modes:
- Network timeouts to Stripe
- Database constraint violations
- Invalid user input
- Missing tenant context
- File upload too large
- SMS provider down

**How you handle errors is the difference between an app that crashes under stress and one that degrades gracefully.** A senior's code:

1. Throws specific exceptions with context
2. Catches only what it can handle
3. Logs everything
4. Re-throws what it can't handle
5. Has a global handler that converts exceptions to user-friendly errors

This is "defensive programming" — but done right, not paranoia.

---

## Lab: Build a Robust Money Transfer

### Step 1 — Create the lab file structure

```bash
cd G:\php-laravel
touch practice/08-account.php
touch practice/08-transfer.php
touch practice/08-demo.php
```

### Step 2 — Custom exceptions

`practice/08-account.php`:

```php
<?php
declare(strict_types=1);

namespace Practice;

class InsufficientFundsException extends \DomainException
{
    public function __construct(
        public readonly Money $required,
        public readonly Money $available,
    ) {
        parent::__construct(
            "Required {$required->format()}, available {$available->format()}"
        );
    }
}

class AccountNotFoundException extends \DomainException
{
    public static function withId(string $id): self
    {
        return new self("Account '$id' not found");
    }
}

class CurrencyMismatchException extends \DomainException {}
```

### Step 3 — An `Account` class that throws properly

Append to `practice/08-account.php`:

```php
class Account
{
    private Money $balance;
    
    public function __construct(
        public readonly string $id,
        string $currency,
        int $initialBalance = 0,
    ) {
        $this->balance = new Money($initialBalance, $currency);
    }
    
    public function balance(): Money
    {
        return $this->balance;
    }
    
    public function deposit(Money $amount): void
    {
        $this->assertSameCurrency($amount);
        $this->balance = new Money(
            $this->balance->amount + $amount->amount,
            $this->balance->currency,
        );
    }
    
    public function withdraw(Money $amount): void
    {
        $this->assertSameCurrency($amount);
        
        if ($amount->amount > $this->balance->amount) {
            throw new InsufficientFundsException($amount, $this->balance);
        }
        
        $this->balance = new Money(
            $this->balance->amount - $amount->amount,
            $this->balance->currency,
        );
    }
    
    private function assertSameCurrency(Money $other): void
    {
        if ($this->balance->currency !== $other->currency) {
            throw new CurrencyMismatchException(
                "Account is in {$this->balance->currency}, got {$other->currency}"
            );
        }
    }
}
```

### Step 4 — A transfer service that uses exceptions correctly

`practice/08-transfer.php`:

```php
<?php
declare(strict_types=1);

namespace Practice;

class TransferService
{
    /** @var array<string, callable> */
    private array $auditLog = [];
    
    /**
     * Transfer money between accounts atomically.
     * Throws specific exceptions on failure; logs and re-throws unexpected ones.
     */
    public function transfer(Account $from, Account $to, Money $amount): void
    {
        try {
            $from->withdraw($amount);
        } catch (InsufficientFundsException $e) {
            // We know how to handle this: log + re-throw with context
            $this->logFailure("Insufficient funds in {$from->id}", $e);
            throw $e;  // caller will see the failure
        } catch (CurrencyMismatchException $e) {
            $this->logFailure("Currency mismatch in {$from->id}", $e);
            throw $e;
        } catch (\Throwable $e) {
            // Unexpected — log with full context and re-throw
            $this->logFailure("Unexpected error in {$from->id}", $e);
            throw $e;
        }
        
        try {
            $to->deposit($amount);
        } catch (\Throwable $e) {
            // Compensating action: try to refund the source
            try {
                $from->deposit($amount);
            } catch (\Throwable $refundError) {
                $this->logFailure("REFUND FAILED for {$from->id}", $refundError);
            }
            $this->logFailure("Deposit to {$to->id} failed, refunded {$from->id}", $e);
            throw $e;
        }
        
        $this->logSuccess("Transferred {$amount->format()} from {$from->id} to {$to->id}");
    }
    
    private function logSuccess(string $msg): void
    {
        echo "[OK] $msg" . PHP_EOL;
    }
    
    private function logFailure(string $context, \Throwable $e): void
    {
        echo "[FAIL] $context: {$e->getMessage()}" . PHP_EOL;
    }
}
```

### Step 5 — Use it

`practice/08-demo.php`:

```php
<?php
declare(strict_types=1);

require __DIR__ . '/../coaching-saas/vendor/autoload.php';

use Practice\Account;
use Practice\Money;
use Practice\TransferService;
use Practice\InsufficientFundsException;

$checking = new Account('ACC-001', 'USD', 10000);  // $100.00
$savings  = new Account('ACC-002', 'USD', 5000);   // $50.00

$service = new TransferService();

// Successful transfer
try {
    $service->transfer($checking, $savings, new Money(2500, 'USD'));
    echo "After transfer: checking={$checking->balance()->format()}, savings={$savings->balance()->format()}" . PHP_EOL;
} catch (\Throwable $e) {
    echo "Transfer failed: {$e->getMessage()}" . PHP_EOL;
}

// Failed transfer — insufficient funds
try {
    $service->transfer($savings, $checking, new Money(999999, 'USD'));
} catch (InsufficientFundsException $e) {
    echo "Caught: required {$e->required->format()}, available {$e->available->format()}" . PHP_EOL;
}

// Failed transfer — currency mismatch
try {
    $service->transfer($checking, $savings, new Money(100, 'EUR'));
} catch (\Practice\CurrencyMismatchException $e) {
    echo "Caught: {$e->getMessage()}" . PHP_EOL;
}

echo "Final: checking={$checking->balance()->format()}, savings={$savings->balance()->format()}" . PHP_EOL;
```

Run:

```bash
php practice/08-demo.php
```

Expected output:

```
[OK] Transferred USD 25.00 from ACC-001 to ACC-002
After transfer: checking=USD 75.00, savings=USD 75.00
[FAIL] Insufficient funds in ACC-002: Required USD 9999.99, available USD 75.00
Caught: required USD 9999.99, available USD 75.00
[FAIL] Currency mismatch in ACC-001: Account is in USD, got EUR
Caught: Account is in USD, got EUR
Final: checking=USD 75.00, savings=USD 75.00
```

> **Notice:** each catch site knows exactly what to do. The exceptions carry context. The log shows what happened. The caller still gets a meaningful exception.

### Step 6 — Use `finally` for cleanup

`practice/08-finally-demo.php`:

```php
<?php
declare(strict_types=1);

require __DIR__ . '/../coaching-saas/vendor/autoload.php';

function readFirstLine(string $path): ?string
{
    $handle = fopen($path, 'r');
    if ($handle === false) {
        throw new \RuntimeException("Cannot open $path");
    }
    
    try {
        $line = fgets($handle);
        return $line === false ? null : trim($line);
    } finally {
        // Always close the file, even if reading threw
        fclose($handle);
    }
}

// Use it
try {
    $line = readFirstLine(__DIR__ . '/some-file.txt');
    echo "First line: $line" . PHP_EOL;
} catch (\Throwable $e) {
    echo "Failed: {$e->getMessage()}" . PHP_EOL;
}
```

> **The `finally` block runs even if the try block throws.** This is critical for file handles, database transactions, locks, and any other resource that must be released.

---

## Quiz

**Q1.** A junior writes:

```php
try {
    $user = User::find($id);
    $user->sendEmail();
    $user->updateLastSeen();
} catch (\Exception $e) {
    Log::error('Failed');
}
```

What's wrong with this? Refactor it to the senior's version.

**Q2.** You have a `ChargeCard` function that can throw `CardDeclined`, `NetworkTimeout`, or `InsufficientFunds`. You're in a `CheckoutController`. Which exceptions do you catch, and which do you let bubble?

**Q3.** Why is `extends \DomainException` better than `extends \Exception` for a custom business exception? What's the practical benefit?

**Q4.** A `finally` block runs even if you `return` from a `try` block, but **not** if you call `exit()`. Explain the second part — when would that matter?

**Q5.** (Stretch) The `TransferService` has a "compensating action" — if deposit fails, it refunds the source. This is a common pattern. What is this called, and when else have you seen it (even outside code)?

---

## What You Now Have

✅ Exception model: Throwable, Exception, Error, custom subclasses
✅ When to catch, when to throw, when to re-throw
✅ Custom exception classes with named constructors and data
✅ `finally` for cleanup
✅ The mental model of "report vs handle"

## What's Next

You've now finished **Phase 1: PHP Fundamentals.** You can read and write modern PHP.

Phase 2 begins: **Laravel Core.** You'll learn the request lifecycle, routing, Eloquent, Blade, and middleware — the building blocks of every Laravel feature.

📖 **[Phase 2 starts: Lesson 14 — The Request Lifecycle](phase-2-laravel-core/14-the-request-lifecycle.md)**
