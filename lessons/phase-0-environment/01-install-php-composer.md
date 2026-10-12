# Lesson 01 — Install PHP 8.3 + Composer

> **Phase 0: Environment** | You'll be here: ~30 min

## The Senior's Question

You're about to install your first non-Node runtime in a long time. How do you avoid installing a version that's wrong, breaking something else on your machine, or making a choice you'll regret in 6 months?

---

## Theory: What Is PHP, Really?

PHP is a **server-side scripting language** that runs your code on the web server (not in the browser). When a request hits your server:

1. Web server (Nginx) sees a `.php` request
2. Hands the file to PHP
3. PHP executes the file top-to-bottom
4. Outputs whatever it printed (HTML, JSON, etc.)
5. Dies. Process ends.

That last point — *dies* — is the difference from Node. **PHP is per-request.** Every request is a fresh process with no shared memory. This sounds wasteful, but it's actually a feature:

- ✅ Memory leaks can't accumulate across requests
- ✅ Failed requests can't corrupt the next one
- ✅ You can scale by just adding more workers
- ⚠️ You can't do real-time, long-lived connections the way Node does

In 2026, modern PHP has an optional long-lived mode (Octane, RoadRunner) for when you need it. But by default, the per-request model is fine for your SaaS.

## Theory: What Is Composer?

Composer is to PHP what **npm is to Node**. But there's a philosophical difference worth knowing:

| | npm | Composer |
|---|---|---|
| Installs | Packages | Packages |
| But also | Nothing | **PSR-4 autoloading** (industry-standard PHP autoloader) |
| Lockfile | `package-lock.json` | `composer.lock` |
| Install command | `npm install` | `composer install` |
| Run script command | `npm run build` | `composer run-script` |

The key difference: **Composer assumes you're using namespaces + classes + autoloading.** The whole PHP ecosystem is built around this. If you've used npm but never thought about JavaScript's `import` resolution, you'll see the parallel — Composer is what makes `use App\Models\User;` actually find the file.

## Why It Matters

A junior installs the latest PHP they find, picks random extensions, doesn't pin a version, and 6 months later their project breaks because PHP 9 came out and Laravel 11 deprecated something. A senior:

1. Picks a **specific PHP version** that matches their framework's requirements
2. Uses a **version manager** so they can switch when needed
3. **Verifies** the install with a known command before doing anything else
4. Knows the **extensions** they need and why

We're going to do all four.

## Why PHP 8.3 Specifically?

Laravel 11 (the current major version) requires PHP 8.2+. PHP 8.3 is the latest stable, has better performance, and supports all the modern features (readonly classes, typed constants, `json_validate()`). We use 8.3 because:

- ✅ It's the current stable
- ✅ Laravel 11/12 are tested against it
- ✅ It has the most modern syntax you'll see in tutorials
- ⚠️ Don't use anything older. PHP 7.4 is end-of-life.

## Why Each PHP Extension Is Required

Laravel needs these. Here's what each one *actually does* in the context of a coaching SaaS:

| Extension | What It Does | Why Your SaaS Needs It |
|---|---|---|
| `openssl` | Encryption primitives, signed URLs | Sessions, encrypted cookies, password hashing, signing subscription webhooks |
| `pdo` + `pdo_mysql` | Database abstraction + MySQL driver | Every page that touches the DB |
| `mbstring` | Multi-byte string ops (UTF-8) | Student names in Bengali, Arabic, Chinese — single-byte ops corrupt them |
| `tokenizer` | Parse PHP code into tokens | Used by code-analysis packages, IDE plugins |
| `xml` | XML parsing | SMS/email provider integrations, SOAP APIs for payment gateways |
| `ctype` | Character type checks | Form validation ("is this input actually a number?") |
| `json` | JSON encode/decode | API responses, Inertia props, config files |
| `bcmath` | Arbitrary precision arithmetic | **Money math.** Never use floats for currency. |
| `fileinfo` | Detect file MIME types | Student photo uploads — prevents "fake.jpg is actually a .exe" attacks |
| `curl` | HTTP client | Talking to Stripe, payment gateways, SMS APIs |

### The `bcmath` Lesson (Read Twice)

```php
$total = 0.1 + 0.2;
echo $total;  // 0.30000000000000004
```

This is **floating point imprecision**. It exists in every language — JavaScript, Python, Java, PHP, all of them. For a coaching SaaS that handles fees, invoices, refunds, this bug will *eventually* produce a wrong invoice. The senior's fix:

```php
$total = bcadd('0.1', '0.2', 2);  // "0.30" — exact, 2 decimal places
```

If you remember nothing else from this lesson, remember: **money is `bcmath`**.

---

## Lab: Install PHP 8.3 + Composer on Windows

### Step 1 — Install Laravel Herd (Recommended Path)

**Herd** is Laravel's official Windows/macOS tool. It gives you PHP, Composer, Nginx, and a `herd` CLI in one install, and it lets you switch PHP versions later.

1. Open PowerShell **as Administrator**
2. Run:
   ```powershell
   winget install --id Laravel.Herd
   ```
3. Close and reopen PowerShell after install (PATH changes don't apply to live sessions)
4. Verify:
   ```powershell
   herd --version
   ```

### Step 2 — Activate a PHP Version

```powershell
herd use 8.3
```

This sets the default PHP to 8.3 for all your Laravel projects. You can switch later with `herd use 8.4` (when 8.4 ships).

### Step 3 — Verify PHP

```powershell
php -v
```

Expected output (yours may differ on patch version):
```
PHP 8.3.x (cli) (built: ...)
Copyright (c) The PHP Group
Zend Engine v4.3.x, Copyright (c) Zend Technologies
```

### Step 4 — Verify Composer

```powershell
composer --version
```

Expected output:
```
Composer version 2.x.x 2026-xx-xx
```

### Step 5 — Verify All Required Extensions

```powershell
php -m
```

This lists all loaded modules. **Confirm you see these in the output:**

```
[PHP Modules]
bcmath
ctype
curl
dom
fileinfo
filter
hash
json
mbstring
openssl
pdo
pdo_mysql
tokenizer
xml
```

If any are missing, Herd ships with them enabled. If you're using a different install, you'll need to enable them in your `php.ini` — search for the extension line and uncomment it.

### Step 6 — Sanity Check PHP Works

```powershell
php -r "echo 'Hello from PHP ' . PHP_VERSION . PHP_EOL;"
```

Expected:
```
Hello from PHP 8.3.x
```

If you see this, your PHP install is fully functional.

---

## Common Pitfalls (Things That Go Wrong)

| Symptom | Cause | Fix |
|---|---|---|
| `'php' is not recognized` | Terminal was open when Herd installed | Close and reopen PowerShell |
| Wrong PHP version (e.g. 7.4) | Old PHP in PATH before Herd's | `herd use 8.3`, then restart terminal |
| `pdo_mysql` missing | Manual install without checking extensions | Re-run Herd installer, or enable in `php.ini` |
| `composer` not found | Same as `php` — old session | Restart terminal |

---

## Quiz

**Q1.** You have two requests hit your Laravel app simultaneously. In Node, they're handled in the same process. In PHP, what happens at the process level? What does this mean for memory leaks and for scaling?

**Q2.** A junior writes `$balance = $user->paid - $user->refunded;` and stores it in a `DECIMAL(10,2)` column. Why might this still produce a wrong answer for some users, even though the column is `DECIMAL`? (Hint: when does the math actually happen?)

**Q3.** You `composer require` a package. It pulls in 5 dependencies. Why does Composer know where to put each one, and what file controls the mapping from `App\Models\User` → `app/Models/User.php`?

**Q4.** Why does Laravel need `mbstring` for a coaching SaaS, but a typical MERN + MongoDB app doesn't need an equivalent? (Hint: the answer is about how the data layer handles strings, not the framework.)

**Q5.** (Stretch) Herd installs Nginx too. You don't see it running yet. Why might that be, and how would you check?

---

## What You Now Have

✅ PHP 8.3 installed and verified
✅ Composer installed and verified
✅ All required extensions loaded
✅ The ability to run PHP scripts from the command line

## What's Next

In Lesson 02, we'll use Composer to scaffold your first Laravel project — `coaching-saas` — and see what gets created.

📖 **[Lesson 02 — Scaffold Your First Laravel Project](02-scaffold-first-laravel-project.md)**
