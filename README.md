# The Senior Engineer's Course on PHP, Laravel & Shipping a SaaS

Welcome. This is not a tutorial — it's a curriculum built by a senior engineer for someone who already ships code and wants to learn *how to think*, not just *what to type*.

## Who This Is For

You're a MERN-stack developer who's going to build a multi-tenant SaaS for coaching centers, schools, and kindergartens on PHP + Laravel. You've never used relational databases seriously, never written tests, never deployed anything, and want to level up to senior.

## How To Use This Course

Each lesson is one `.md` file. Every lesson follows the **exact same structure**:

1. **The Senior's Question** — the real-world problem that motivates the concept
2. **Theory** — what the thing is, in plain language with diagrams
3. **Why It Matters** — the reasoning a senior would give in a design meeting
4. **Lab** — concrete code you write, hands-on, in your own project
5. **Quiz** — 3–5 questions. Answer them before moving on.
6. **Stretch (optional)** — harder problems for when you're comfortable

### The Rules

- **Do the labs.** Don't just read them. Open your terminal, write the code, break it, fix it.
- **Do the quizzes.** Without scrolling up. If you can't answer a question, re-read the lesson.
- **Do them in order.** Some lessons depend on earlier ones. Skipping ruins the spiral.
- **Don't copy-paste labs.** Type them. Your fingers build muscle memory your eyes don't.
- **One lesson per sitting** is fine. Two is fine. All 56 in a weekend is a recipe for forgetting everything.

## The Curriculum Map

| Phase | What You'll Learn | # Lessons | Status |
|---|---|---|---|
| **0. Environment** | PHP, Composer, Laravel project skeleton, database | 5 | ✅ Complete |
| **1. PHP Fundamentals** | Modern PHP from zero (you're MERN-trained) | 8 | ✅ Complete |
| **2. Laravel Core** | Routing, Eloquent, Blade, Middleware, DI | 10 | ✅ Complete |
| **3. Architecture** | Multi-tenancy, Auth, Modeling, Events, Queues | 7 | ✅ Complete |
| **4. Quality** | Testing, TDD, Static Analysis, CI | 5 | ✅ Complete |
| **5. Production** | Caching, Billing, Mail, Storage, Search, API | 7 | ✅ Complete |
| **6. Deploy & Scale** | Environments, Hosting, Docker, Scaling | 10 | ✅ Complete |
| **7. Soft Skills** | Senior thinking, design docs, code review | 4 | ✅ Complete |

**Total: 56 lessons.** Each is 30–60 minutes of real study with the lab.

### All Lessons At a Glance

**Phase 0 — Environment**
1. [Install PHP + Composer](lessons/phase-0-environment/01-install-php-composer.md)
2. [Scaffold Your First Laravel Project](lessons/phase-0-environment/02-scaffold-first-laravel-project.md)
3. [Understand the Laravel Folder Structure](lessons/phase-0-environment/03-understand-the-laravel-folder-structure.md)
4. [Run Your First Artisan Command](lessons/phase-0-environment/04-run-your-first-artisan-command.md)
5. [Install Database and First Migration](lessons/phase-0-environment/05-install-database-and-first-migration.md)

**Phase 1 — PHP Fundamentals**
6. [PHP Syntax for MERN Developers](lessons/phase-1-php-fundamentals/06-php-syntax-for-mern-developers.md)
7. [Types, Strict Mode, and Type Juggling](lessons/phase-1-php-fundamentals/07-types-strict-mode-and-type-juggling.md)
8. [Arrays, Collections, and the List Pattern](lessons/phase-1-php-fundamentals/08-arrays-collections-and-the-list-pattern.md)
9. [Functions, Closures, and Arrow Functions](lessons/phase-1-php-fundamentals/09-functions-closures-and-arrow-functions.md)
10. [OOP: Classes, Objects, Namespaces](lessons/phase-1-php-fundamentals/10-oop-classes-objects-namespaces.md)
11. [Inheritance, Interfaces, Traits, Abstract](lessons/phase-1-php-fundamentals/11-inheritance-interfaces-traits-and-abstract.md)
12. [Modern PHP 8 Features](lessons/phase-1-php-fundamentals/12-modern-php-8-features.md)
13. [Error Handling, Exceptions, and the Why](lessons/phase-1-php-fundamentals/13-error-handling-exceptions-and-the-why.md)

**Phase 2 — Laravel Core**
14. [The Request Lifecycle](lessons/phase-2-laravel-core/14-the-request-lifecycle.md)
15. [Routing, Controllers, and Responses](lessons/phase-2-laravel-core/15-routing-controllers-and-responses.md)
16. [Blade Templating vs React Mental Model](lessons/phase-2-laravel-core/16-blade-templating-vs-react-mental-model.md)
17. [Eloquent ORM: The Active Record Pattern](lessons/phase-2-laravel-core/17-eloquent-orm-the-active-record-pattern.md)
18. [Migrations, Schema, and the Why of Versioned DB](lessons/phase-2-laravel-core/18-migrations-schema-and-the-why-of-versioned-db.md)
19. [Form Requests, Validation, and Authorization](lessons/phase-2-laravel-core/19-form-requests-validation-and-authorization.md)
20. [Middleware: The Pipeline Pattern](lessons/phase-2-laravel-core/20-middleware-the-pipeline-pattern.md)
21. [Service Container and Dependency Injection](lessons/phase-2-laravel-core/21-service-container-and-dependency-injection.md)
22. [Artisan: The CS:GO of Laravel](lessons/phase-2-laravel-core/22-artisan-the-csgo-of-laravel.md)
23. [Configuration and Environments](lessons/phase-2-laravel-core/23-configuration-and-environments.md)

**Phase 3 — Architecture**
24. [Multi-Tenancy: Shared Schema Strategy](lessons/phase-3-architecture/24-multi-tenancy-shared-schema-strategy.md)
25. [Authentication with Laravel Breeze](lessons/phase-3-architecture/25-authentication-with-laravel-breeze.md)
26. [Authorization: Roles, Policies, and Gates](lessons/phase-3-architecture/26-authorization-roles-policies-and-gates.md)
27. [Domain Modeling for a Coaching SaaS](lessons/phase-3-architecture/27-domain-modeling-for-coaching-saas.md)
28. [Repository Pattern or No Repository Pattern](lessons/phase-3-architecture/28-repository-pattern-or-no-repository-pattern.md)
29. [Events, Listeners, and Why Decoupling Matters](lessons/phase-3-architecture/29-events-listeners-and-why-decoupling-matters.md)
30. [Queues, Jobs, and Background Work](lessons/phase-3-architecture/30-queues-jobs-and-background-work.md)

**Phase 4 — Quality**
31. [Testing Mental Model and the Testing Pyramid](lessons/phase-4-quality/31-testing-mental-model-and-the-testing-pyramid.md)
32. [Pest Setup and Writing Your First Test](lessons/phase-4-quality/32-pest-setup-and-writing-your-first-test.md)
33. [Feature Tests vs Unit Tests in Depth](lessons/phase-4-quality/33-feature-tests-vs-unit-tests.md)
34. [Test-Driven Development for Real](lessons/phase-4-quality/34-test-driven-development-for-real.md)
35. [Code Quality, Static Analysis, and CI](lessons/phase-4-quality/35-code-quality-static-analysis-and-ci.md)

**Phase 5 — Production**
36. [Caching with Redis and Cache Invalidation](lessons/phase-5-production/36-caching-with-redis-and-cache-invalidation.md)
37. [Subscription Billing with Laravel Cashier](lessons/phase-5-production/37-subscription-billing-with-laravel-cashier.md)
38. [Email and Notifications](lessons/phase-5-production/38-email-and-notifications.md)
39. [File Storage and S3](lessons/phase-5-production/39-file-storage-and-s3.md)
40. [Logging, Monitoring, and Debugging](lessons/phase-5-production/40-logging-monitoring-and-debugging.md)
41. [Search with Scout](lessons/phase-5-production/41-search-with-scout.md)
42. [API Development vs Web Routes](lessons/phase-5-production/42-api-development-vs-web-routes.md)

**Phase 6 — Deploy & Scale**
43. [Environments: Dev, Staging, Production](lessons/phase-6-deploy-scale/43-environments-dev-staging-production.md)
44. [Hosting Choices: Forge, Vapor, and Raw](lessons/phase-6-deploy-scale/44-hosting-choices-forge-vapor-and-raw.md)
45. [Deploying on Laravel Forge](lessons/phase-6-deploy-scale/45-deploying-on-laravel-forge.md)
46. [Docker for PHP Applications](lessons/phase-6-deploy-scale/46-docker-for-php-applications.md)
47. [Queues and Workers in Production](lessons/phase-6-deploy-scale/47-queues-and-workers-in-production.md)
48. [Database at Scale](lessons/phase-6-deploy-scale/48-database-at-scale.md)
49. [Horizontal Scaling and When You Need K8s](lessons/phase-6-deploy-scale/49-horizontal-scaling-and-when-you-need-k8s.md)
50. [CDN, Static Assets, Edge Caching](lessons/phase-6-deploy-scale/50-cdn-static-assets-edge-caching.md)
51. [Backups, Disaster Recovery, RPO/RTO](lessons/phase-6-deploy-scale/51-backups-disaster-recovery-rpo-rto.md)
52. [The 72-Hour Launch Checklist](lessons/phase-6-deploy-scale/52-the-72-hour-launch-checklist.md)

**Phase 7 — Soft Skills**
53. [How Senior Developers Think Before Coding](lessons/phase-7-soft-skills/53-how-senior-developers-think-before-coding.md)
54. [Writing Design Docs and ADR](lessons/phase-7-soft-skills/54-writing-design-docs-and-adr.md)
55. [Code Review Culture](lessons/phase-7-soft-skills/55-code-review-culture.md)
56. [The SaaS Founder's Checklist Beyond Code](lessons/phase-7-soft-skills/56-the-saas-founders-checklist-beyond-code.md)

## The Mindset (Read This Twice)

A junior asks: *"How do I do X?"*
A senior asks: *"Why is X the right choice here, and what breaks if I'm wrong?"*

This course teaches you the second question. The labs give you the muscle memory; the "Why It Matters" sections give you the judgment.

If you finish all 56 lessons, do all the labs, and answer all the quizzes, you will not be a junior anymore. You will be a Laravel developer who can think like a senior.

That's the deal.

---

Ready? Start here:

📖 **[Phase 0, Lesson 1: Install PHP + Composer](lessons/phase-0-environment/01-install-php-composer.md)**

If you want the big-picture mental model first, read:

📖 **[Lesson 00: Why This Stack, In This Order](lessons/00-why-this-stack.md)**
