# Lesson 02 — Scaffold Your First Laravel Project

> **Phase 0: Environment** | You'll be here: ~30 min

## The Senior's Question

You have PHP and Composer. Before you write a line of code, you need a real project structure. What does a senior engineer look at *first* when they open a fresh Laravel repo, and why?

---

## Theory: What `composer create-project` Does

When you run `composer create-project laravel/laravel coaching-saas`, Composer does this:

1. Downloads the **Laravel skeleton** (a curated set of starter files)
2. Reads `composer.json` (the manifest)
3. Installs all of Laravel's dependencies (Symfony components, Eloquent, Carbon, etc.) into `vendor/`
4. Creates a `composer.lock` (pins exact versions)
5. Generates a `.env` file with a fresh app key
6. Leaves you with a runnable project

This is **not** like `git clone`. You're not just copying code — you're resolving a dependency graph. The fresh `vendor/` folder on your machine contains hundreds of packages, and Composer calculated a compatible version of each one.

### What "the skeleton" means

Laravel doesn't ship as a "library you import." It ships as a **starter project**. You `cd` into it and it *is* your app. You build your features inside it. There is no `node_modules` equivalent you re-build from scratch.

## Theory: The Laravel Folder Structure (Senior's Mental Map)

After you scaffold, you'll see this:

```
coaching-saas/
├── app/                    ← Your code lives here
│   ├── Http/
│   │   ├── Controllers/   ← Handle HTTP requests
│   │   └── Middleware/    ← Filter requests (auth, logging, etc.)
│   ├── Models/            ← Eloquent models (DB representations)
│   └── Providers/         ← Wire up services at boot
│
├── bootstrap/              ← Framework boot files
│   ├── app.php            ← The actual "start the framework" file
│   └── cache/             ← Compiled configs/routes (gitignored)
│
├── config/                 ← Every configurable thing, in PHP files
│   ├── app.php            ← App name, env, debug, timezone
│   ├── database.php       ← DB connections
│   ├── auth.php           ← Auth guards
│   └── ...                ← ~30+ config files
│
├── database/
│   ├── migrations/        ← Version-controlled DB schema
│   ├── seeders/           ← Test/initial data
│   └── factories/         ← Test data generators
│
├── public/                 ← The web root (Nginx points here)
│   ├── index.php          ← Single entry point for ALL requests
│   └── assets/            ← Compiled CSS/JS/images
│
├── resources/
│   ├── views/             ← Blade templates (server-rendered HTML)
│   ├── css/               ← Source styles
│   └── js/                ← Source scripts
│
├── routes/                 ← URL → handler mappings
│   ├── web.php            ← Browser routes (with sessions, CSRF)
│   ├── api.php            ← API routes (stateless, /api prefix)
│   └── console.php        ← Artisan command routes
│
├── storage/                ← Writable: logs, cache, uploads
│   ├── app/
│   ├── framework/
│   └── logs/
│
├── tests/                  ← Automated tests
│   ├── Feature/
│   └── Unit/
│
├── vendor/                 ← Composer dependencies (NEVER edit, NEVER commit)
│
├── .env                    ← Your secrets + local config (NEVER commit)
├── .env.example            ← Template showing what .env should contain
├── artisan                 ← The CLI entry point
├── composer.json           ← Project manifest
├── composer.lock           ← Pinned versions
└── package.json            ← For npm (Laravel uses Vite for assets)
```

## Why It Matters

A senior scans this structure in 30 seconds and knows:

| Folder | What lives there | When you'll touch it |
|---|---|---|
| `app/Http/Controllers` | Endpoints | Every feature |
| `app/Models` | Database tables as PHP objects | Every feature |
| `database/migrations` | DB schema changes | Every DB change |
| `routes/web.php` | URL → controller mappings | Every feature |
| `resources/views` | HTML templates | Every page |
| `config/` | Behavior knobs | When you need to change framework behavior |
| `tests/` | Automated checks | Continuously |
| `public/` | The only folder the web server exposes | Rarely (asset builds) |
| `storage/` | Writable runtime data | Rarely directly |
| `vendor/` | Third-party code | **Never** |

The **most important mental shift** from MERN: in MERN you can `node_modules` away and rebuild. In Laravel, **you also have a `vendor/` you can `rm -rf` and rebuild with `composer install`** because of `composer.lock`. The lock file is the contract.

### The Two Files You Must Understand Before Anything Else

1. **`public/index.php`** — Every web request enters here. It's ~20 lines. It boots the framework, dispatches the request to a route, and returns the response. **There is no other entry point for HTTP.** This is the "single front door" pattern.

2. **`artisan`** — The CLI entry point. Not a PHP file, not a folder. A small bash script that runs `php` with the right bootstrap. You'll use it hundreds of times a day.

## The `.env` vs `config/` Distinction

This trips up every Laravel beginner. Two layers:

- **`.env`** — Per-environment values. Database password on your laptop ≠ database password on production. **Never commit this file.**
- **`config/*.php`** — Reads from `.env` and exposes values to your code. Code calls `config('app.name')`, never `env('APP_NAME')` directly (except in config files themselves).

Why this split? Because you can change `.env` without redeploying, but you want one canonical place in code that reads it. Also, `php artisan config:cache` squashes config files into one for performance — direct `env()` calls get frozen at the wrong value.

**Rule you'll tattoo on your brain:** in app code → use `config('key')`. In config files → use `env('KEY')`. Never `env()` anywhere else.

---

## Lab: Scaffold and Boot Your Laravel Project

### Step 1 — Create the project

In your terminal (PowerShell, Git Bash, or whatever you use):

```bash
cd G:\php-laravel
composer create-project laravel/laravel coaching-saas
```

This downloads and installs Laravel. It will print progress. If it fails, **read the error** — usually PHP version mismatch, missing extension, or no internet.

### Step 2 — Enter the project

```bash
cd coaching-saas
```

### Step 3 — Verify the structure

```bash
ls -la
```

On Windows in PowerShell, use `dir` instead of `ls -la`. Confirm you see `app/`, `config/`, `database/`, `public/`, `routes/`, `vendor/`, `.env`, `artisan`.

### Step 4 — Check the Laravel version

```bash
php artisan --version
```

Expected:
```
Laravel Framework 11.x.x
```

(or 12.x if it's been released — same structure)

### Step 5 — Boot the dev server

```bash
php artisan serve
```

Expected output:
```
Starting Laravel development server: http://127.0.0.1:8000
[Thu ...] 127.0.0.1:8000 [200]: GET /
```

Open `http://127.0.0.1:8000` in your browser. You should see the default Laravel welcome page with the Laravel logo.

### Step 6 — Stop the server

Press `Ctrl+C` in the terminal.

### Step 7 — Look at the actual boot

Open these three files, in order, and just *read* them:

1. **`public/index.php`** (20 lines) — every request enters here
2. **`bootstrap/app.php`** (the new Laravel 11+ bootstrap)
3. **`routes/web.php`** — currently just one `Route::get('/', ...)` returning the welcome view

You don't need to understand everything. Just *see* the structure. Notice:

- `public/index.php` is tiny — it just loads the framework and dispatches
- `routes/web.php` is where URLs are defined
- The "welcome page" you saw is a Blade view at `resources/views/welcome.blade.php`

### Step 8 — Open the project in your editor

If you have VS Code:
```bash
code .
```

Take 5 minutes to click around. Don't change anything. Just orient yourself.

---

## What's Different From `create-react-app` / Vite (MERN)

| Step | MERN | Laravel |
|---|---|---|
| Create project | `npx create-next-app` or `npm create vite` | `composer create-project laravel/laravel` |
| Install deps | `npm install` | `composer install` (already done by create-project) |
| Run dev server | `npm run dev` (Vite/Next dev) | `php artisan serve` (built-in PHP server) |
| Frontend | You pick React/Next/Vue | Inertia + React, or Blade, or Livewire |
| Entry point | `src/main.tsx` or `pages/index.tsx` | `public/index.php` for web, `artisan` for CLI |

The `artisan serve` command is **only for development.** It runs PHP's built-in web server which is slow and single-threaded. In production you'll use Nginx + PHP-FPM (Lesson 43+).

---

## Quiz

**Q1.** A colleague says "Laravel is just like Node, `php artisan serve` is the same as `npm run dev`." Give two reasons why this comparison is misleading, especially around what `php artisan serve` is doing under the hood.

**Q2.** You need the app name in a controller. A junior writes `env('APP_NAME')`. What's wrong with that, and what should they write instead?

**Q3.** Look at the file tree above. If you wanted to add a new page (e.g. `/students`), which three folders will you likely touch? What goes in each?

**Q4.** Why does Laravel ship as a "starter project" you build inside, rather than as a library you `composer require` into an empty folder? (Hint: think about conventions.)

**Q5.** (Stretch) Open `bootstrap/app.php`. Find the line that registers the routing, middleware, and exception handling. What *design pattern* is this — and why is it better than a config-array approach (which is what older Laravel versions used)?

---

## What You Now Have

✅ A running Laravel project at `G:\php-laravel\coaching-saas`
✅ Verified you can boot it locally
✅ Mental map of the folder structure
✅ Understanding of `.env` vs `config/`

## What's Next

In Lesson 03, we'll go deep on the folder structure — what each piece is actually for, in detail — so when you start writing code, you know where it goes.

📖 **[Lesson 03 — Understand the Laravel Folder Structure (In Detail)](03-understand-the-laravel-folder-structure.md)**
