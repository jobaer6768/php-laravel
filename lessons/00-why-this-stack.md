# Lesson 00 — Why This Stack, In This Order

> **Read this once carefully before Lesson 01.** It shapes how you'll think about everything that follows.

---

## The Senior's Question

You already build SaaS products with MERN. Why are you learning *another* stack? What's actually different, and what should you steal from the senior mindset — even if you keep coding in Node tomorrow?

---

## Theory: What Your SaaS Actually Is

Before tools, think about the workload. Your SaaS handles:

- Schools/coaches **signing up**, picking plans, paying monthly
- Admins **registering students**, assigning them to batches
- Teachers **taking attendance**, recording marks, posting notices
- Students/parents **viewing reports**, paying fees
- Bills, invoices, receipts — **money everywhere**
- Searchable records that must survive for **years**

This is what engineers call a **CRUD-heavy, form-heavy, business-logic-heavy application**. It is not a real-time chat app. It is not a video platform. It is not a game.

**That distinction matters**, because CRUD-heavy apps have well-known patterns, and the frameworks that own those patterns save you enormous time. Laravel owns them. So does Rails, so does Django. MERN doesn't — it gives you a router and an ORM and says "good luck."

## Theory: The Two Architectures Compared

```
┌─────────────── MERN (What you know) ────────────────┐
│  React SPA (everything renders in the browser)       │
│      ⇅ JSON                                        │
│  Express/Fastify (thin HTTP layer, business logic)   │
│      ⇅                                            │
│  Mongoose ODM                                       │
│      ⇅                                            │
│  MongoDB (document store)                            │
└────────────────────────────────────────────────────┘

┌─────── Laravel + Server-Rendered (Default) ────────┐
│  Browser gets HTML directly from the server         │
│      ⇅                                            │
│  PHP-FPM (one process per request, fast)            │
│      ⇅                                            │
│  Laravel (routing + auth + ORM + validation + ...)  │
│      ⇅                                            │
│  MySQL (relational, ACID, foreign keys, SQL)        │
└──────────────────────────────────────────────────┘
```

```
┌─────── Laravel + Inertia (Hybrid, what we'll use) ─────┐
│  React/Vue components, but rendered inside Laravel pages │
│      ⇅ (no separate API; same session, same routing)   │
│  PHP-FPM                                                  │
│      ⇄                                                   │
│  Laravel                                                  │
│      ⇄                                                   │
│  MySQL                                                   │
└─────────────────────────────────────────────────────────┘
```

For your SaaS the **Inertia + React** variant gives you React (familiar) without the bookkeeping cost of running a separate API server, handling CORS, double auth, etc.

## Why It Matters

| Concern | MERN | Laravel | Winner for YOUR SaaS |
|---|---|---|---|
| Time to ship a form | Build form + endpoint + validation + state + error display | Form Request class + Blade + old() input | **Laravel, by hours** |
| Auth & roles | Pick NextAuth/Auth0/roll-your-own | `php artisan make:auth` + built-in | **Laravel** |
| Subscription billing | Stripe SDK + custom webhook handler | `composer require laravel/cashier` | **Laravel** |
| Multi-tenant data isolation | Build it yourself, carefully | Patterns documented, packages available | **Laravel** |
| Admin dashboards | Pick a JS library, wire it all | Filament / Nova / Backpack exist | **Laravel** |
| Hiring later | Large MERN pool | Large Laravel pool | **Tie** |
| Real-time (chat, live updates) | Native via Socket.IO / WS | Possible but not idiomatic | **MERN** |
| Building a video call module | Same in either | Same in either | **Tie** |

**The bottom line:** for 90% of a coaching SaaS, Laravel gets you to production faster with fewer bugs because the boring decisions are already made.

## A Point About Bias

Your friend says "PHP is dead." Three things:

1. **78% of the web** (WordPress, Laravel, Symfony, etc.) runs on PHP.
2. **Laravel is the most-starred PHP framework** in 2026 and is downloaded 200M+ times a year.
3. **WordPress alone** powers 40%+ of all websites.

The "PHP is dead" meme is from people who learned PHP 5 in 2010 and never updated their view. Modern PHP (8.0+) is a strict-typed, performant, modern language. You will write code that looks nothing like the PHP your friend remembers.

## What You're Not Learning Here

You are not learning "PHP-the-glue-language." You are learning:

- **Modern PHP** (8.3 with strict types, attributes, enums, fibers)
- **Laravel** as an opinionated, batteries-included backend framework
- **SQL + relational modeling** (because that's what your data needs)
- **Testing as a habit** (so refactoring doesn't break production)
- **Deployment and scaling** (so going live isn't a guess)

## Lab

Do nothing. Just internalize the diagram above. When you start Lesson 01 and the shell commands start flying, you'll know *why* each piece is there.

## Quiz (no scrolling up)

**Q1.** You start building a real-time multiplayer quiz game. Would you still reach for Laravel? Why or why not?

**Q2.** Look at the architecture diagrams above. What's the *single biggest difference* in how data flows from browser to database between MERN and Laravel?

**Q3.** A team member argues: "Laravel is just a passing trend, real engineers use Node." Write a 3-sentence response that references your *specific* SaaS requirements.

**Q4.** What's the difference between "Laravel the framework" and "Laravel the ecosystem"? (Hint: think Filament, Nova, Forge, Vapor, Cashier.)

**Q5.** In one sentence: what's the most important *kind* of decision this course is going to teach you to make?

---

When you can answer those, move on:

📖 **[Lesson 01 — Install PHP + Composer](lessons/phase-0-environment/01-install-php-composer.md)**
