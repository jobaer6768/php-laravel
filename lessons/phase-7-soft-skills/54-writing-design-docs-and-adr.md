# Lesson 54 — Writing Design Docs and ADR

> **Phase 7: Soft Skills** | You'll be here: ~25 min

## The Senior's Question

A junior codes and ships. A senior codes, ships, and **documents why.** Six months later, when someone asks "why is this built this way?" the senior's answer is in a file. The junior's answer is "I don't know, it just is." Let me show you how to write the documents that compound.

---

## Theory: The Two Documents

### Design Document (Design Doc)

Captures the **what and how** of a feature or system.

Written **before** building. Used to align the team and catch design issues early.

### Architecture Decision Record (ADR)

Captures a **single decision** — what was chosen, why, and what was rejected.

Written **when** the decision is made. Used to remember why we did this.

> **Design doc = plan. ADR = history.** Both are essential.

---

## Theory: When to Write a Design Doc

| Decision size | Write a design doc? |
|---|---|
| 1-line bug fix | No |
| New field on a model | No |
| New endpoint | Maybe |
| New feature (a day of work) | Yes |
| New feature (a week of work) | Yes |
| New microservice | Definitely |
| New data model | Yes |
| Cross-team change | Definitely |

> **Senior rule:** if the decision is hard to reverse, write a design doc.** If it's easy to change, just do it.

---

## Theory: The Design Doc Template

A senior's design doc has 7 sections:

```markdown
# [Feature Name] Design Doc

## Author
[Your name], [date]

## Status
Draft | In Review | Approved | Implemented | Deprecated

## Context
What problem are we solving? Why now?
- Background information
- Links to related issues, PRs, discussions
- Constraints (technical, business, time)

## Goals
What does success look like? List 3-5 outcomes.

## Non-Goals
What are we explicitly NOT doing? Just as important.

## Proposal
The actual design. Code snippets, diagrams, schemas.
- High-level architecture
- API surface
- Data model changes
- Migration plan
- Rollout plan

## Alternatives Considered
What other approaches did we consider? Why didn't we pick them?

## Open Questions
What are we still uncertain about? List them.

## Testing Plan
How will we know it works?
- Unit tests
- Feature tests
- Manual smoke tests
- Performance benchmarks

## Rollout Plan
How do we ship this safely?
- Feature flag?
- Canary deploy?
- Rollback plan?

## Risks
What could go wrong? How do we mitigate?
```

> **The senior's design doc is 1-3 pages.** Not 20. Concise but complete.

---

## Theory: A Real Example

Let's write a design doc for "Bulk SMS to parents on school closure day."

```markdown
# Bulk SMS for School Closures

## Author
Alice, 2024-03-15

## Status
Approved

## Context
Coaching centers need to notify parents quickly when the school 
is closed (weather, holiday, emergency). Currently, admins 
manually SMS each parent or use WhatsApp broadcast. Both are 
slow and error-prone.

Customers have asked for this in 8 support tickets over the last 
3 months. Two have switched to competitors because of it.

## Goals
- Admin can send a single SMS to all parents of selected classes
- Delivered within 5 minutes for typical school (500 parents)
- Tenant-scoped — no cross-tenant leakage
- Audit log of who sent what, when

## Non-Goals
- Two-way SMS (parents can't reply)
- WhatsApp integration (later)
- Scheduled SMS (just immediate)
- Marketing SMS (separate feature)

## Proposal

### Architecture
```
Admin UI → SMSController → SendBulkSms action → SmsService (interface)
                                                    ↓
                                            TwilioSmsService (production)
                                            LogSmsService (dev/test)
```

### API
```php
POST /api/sms/bulk
{
    "class_ids": [1, 2, 3],     // empty = all classes
    "message": "School closed due to weather",
    "tenant_id": 1              // implicit from auth
}

Response 202:
{
    "queued": true,
    "estimated_recipients": 450
}
```

### Data Model
```php
Schema::create('sms_messages', function (Blueprint $table) {
    $table->id();
    $table->foreignId('tenant_id');
    $table->foreignId('sent_by');
    $table->text('message');
    $table->integer('recipient_count');
    $table->enum('status', ['queued', 'sending', 'sent', 'failed']);
    $table->timestamp('sent_at')->nullable();
    $table->timestamps();
});

Schema::create('sms_recipients', function (Blueprint $table) {
    $table->id();
    $table->foreignId('sms_message_id');
    $table->foreignId('parent_id');
    $table->string('phone');
    $table->enum('status', ['pending', 'sent', 'failed']);
    $table->text('error')->nullable();
    $table->timestamps();
});
```

### Async Processing
Send the SMS in a queued job (`SendBulkSmsJob`) to avoid request timeouts.

### Rate Limiting
- Twilio: 1 SMS per second per number
- For 500 parents: 8-10 minutes
- Use Twilio's bulk API (sends 100/sec)

### Cost
- Twilio: $0.0079 per SMS
- For 500 parents: $3.95 per closure
- Pass cost to customer? Or absorb? (See Open Questions)

## Alternatives Considered

### 1. WhatsApp Business API
- Pros: cheaper, more familiar to parents
- Cons: more complex setup, requires customer opt-in
- Decision: defer to v2

### 2. Email instead
- Pros: free, no integration
- Cons: not immediate, not always read
- Decision: SMS for emergencies, email for non-urgent

### 3. Per-tenant rate limiting
- Pros: prevents one tenant from spamming
- Cons: complicates the design
- Decision: use Twilio's bulk API which has its own limits

## Open Questions
- Should we charge customers per SMS or include in plan?
- What's the max message length? (160 chars = 1 SMS)
- Do we need Unicode support (Hindi, Arabic)?
- Should we add unsubscribe mechanism (CAN-SPAM)?

## Testing Plan
- Unit test: SendBulkSms action creates records correctly
- Feature test: admin can send, teacher cannot
- Feature test: empty class list works
- Integration test: Twilio mock receives correct payload
- Performance test: 1000 parents in < 1 min

## Rollout Plan
1. Deploy to staging, test with internal team
2. Enable for one beta customer
3. Monitor for 1 week
4. Enable for all customers
5. Document in user guide

## Risks
- Twilio outage → SMS don't send
  - Mitigation: retry logic, alert on failures
- Customer spam (sending 1000s of SMS)
  - Mitigation: rate limit, audit log, alert on anomaly
- Cost overrun
  - Mitigation: cap per tenant per day
```

> **This is a real design doc.** Real companies use this format. **It takes 1-2 hours to write. Saves 1-2 weeks of rework.**

---

## Theory: The ADR (Architecture Decision Record)

ADRs are smaller than design docs. They capture **one decision.**

```markdown
# ADR-007: Use Redis for Caching and Sessions

## Status
Accepted, 2024-03-15

## Context
We need a fast cache and session store. Options:
1. File-based (Laravel default)
2. Database
3. Redis
4. Memcached

## Decision
Use Redis for both.

## Consequences
Positive:
- Sub-millisecond reads
- Cross-server sessions (enables horizontal scaling)
- Industry-standard, well-supported

Negative:
- One more service to manage
- Adds a dependency
- Costs memory (but small)

## Alternatives Considered
File-based: doesn't work across servers
Database: too slow for cache, puts pressure on DB
Memcached: similar to Redis, but Redis has more features (pub/sub, 
streams, sorted sets)

## Notes
Reviewed 2024-06-01: still happy with this decision.
```

> **The ADR is the historical record.** "Why did we choose Redis?" The answer is in ADR-007.

---

## Theory: When to Write an ADR

| Decision | Write ADR? |
|---|---|
| Use Redis for cache | Yes |
| Use Spatie/Laravel-Permission for roles | Yes |
| Use Tailwind for CSS | Maybe |
| Variable name change | No |
| Database column added | No |
| Switch from MySQL to Postgres | Definitely |

> **Senior rule:** if the decision shapes the system, write an ADR.** If a future engineer would ask "why?", write an ADR.

---

## Theory: ADRs in Practice

Store ADRs in `/docs/adr/`:

```
docs/adr/
├── 0001-record-architecture-decisions.md
├── 0002-use-laravel-framework.md
├── 0003-use-mysql-for-database.md
├── 0004-use-shared-schema-multi-tenancy.md
├── 0005-use-stripe-for-payments.md
├── 0006-use-cashier-for-subscription-management.md
├── 0007-use-redis-for-cache-and-sessions.md
├── 0008-use-pest-for-testing.md
└── README.md
```

`0001-record-architecture-decisions.md`:

```markdown
# ADR-0001: Record Architecture Decisions

## Status
Accepted

## Context
We need to record the architectural decisions made on this project.

## Decision
We will use Architecture Decision Records (ADRs) as described by 
Michael Nygard. Each significant decision will be recorded as a 
separate ADR.

## Consequences
- Future engineers can understand why we made choices
- Decisions are reviewed and discussed
- We can revisit and revise decisions
```

> **The first ADR is "we use ADRs."** Self-referential but necessary.

---

## Theory: ADR Lifecycle

```
Proposed → Accepted → Deprecated → Superseded
```

A deprecated ADR is one that's no longer relevant. A superseded ADR is replaced by a newer one.

```markdown
## Status
Superseded by ADR-015 (2024-09-01)

We switched from MySQL to Postgres. The reasoning in ADR-003 
no longer applies. See ADR-015 for the current approach.
```

> **ADRs are immutable once accepted.** If you change your mind, write a new ADR. **History is preserved.**

---

## Theory: Tools for Writing

- **Markdown files in `/docs/adr/`** — simple, version-controlled
- **adr-tools** (npm) — CLI for creating ADRs
- **Log4brains** — web-based ADR viewer
- **Notion / Confluence** — for non-engineers to read

> **Senior default: markdown in repo.** Version-controlled, code-reviewed, searchable.

---

## Theory: The Senior's Documentation Hierarchy

```
README.md            → What is this project? How do I run it?
DEPLOY.md            → How do I deploy?
RECOVERY.md          → How do I recover from disaster?
docs/adr/            → Why did we make these decisions?
docs/design/         → How is this big feature built?
docs/architecture/   → What's the high-level architecture?
```

> **Each doc answers a different question.** "How do I run it?" vs "Why is it built this way?" vs "How do I fix it?"

---

## Theory: When Docs Get Stale

Docs go stale. The senior prevents this by:

1. **PRs update docs** — "if you change code, update the doc"
2. **Quarterly doc review** — calendar reminder, 1 hour
3. **Doc ownership** — someone is responsible for each doc
4. **Search first** — when answering a question, if doc is missing, write it
5. **Test the docs** — "if I follow the README, does it work?" quarterly

> **The senior's rule:** if a doc is wrong, fix it. Don't work around it.

---

## Theory: The Design Doc Review Process

1. **Author writes** the design doc
2. **Author shares** in team channel or PR
3. **Team reviews** (1-3 days)
4. **Author revises** based on feedback
5. **Tech lead approves**
6. **Author starts implementation**

> **Don't skip the review.** The whole point is to catch issues before code is written. **If no one reviews, the doc is theater.**

---

## Theory: The Senior's Writing Style

A senior's design doc:

- ✅ **Concise** — 1-3 pages
- ✅ **Concrete** — code snippets, not vague descriptions
- ✅ **Decision-oriented** — choices and trade-offs
- ✅ **Honest** — admits unknowns, lists open questions
- ✅ **Visual** — diagrams, tables, code blocks
- ❌ Not a novel
- ❌ Not a wishlist
- ❌ Not a sales pitch

> **A senior's design doc is the shortest document that fully describes the decision.**

---

## Theory: The Junior vs Senior Document

| | Junior's doc | Senior's doc |
|---|---|---|
| Length | 20 pages | 2-3 pages |
| Diagrams | Maybe | Yes |
| Code snippets | Few | Many |
| Alternatives | "We considered X, Y, Z" | Why each was rejected |
| Trade-offs | Not discussed | Explicit |
| Open questions | Hidden | Listed |
| Decision | "Let's do X" | "We're doing X because Y" |

> **The senior's doc helps the team decide together.** The junior's doc is a monologue.

---

## Why It Matters

Design docs and ADRs:
- **Compound** — every doc is leverage for the next decision
- **Onboard** — new team members read and understand
- **Defend** — "why did we do this?" is in the doc
- **Improve** — you can revisit decisions with context
- **Communicate** — team alignment before code

> **A senior's documentation is the most leveraged thing they do.** 1 hour writing = 100 hours saved across the team.

---

## Lab: Write Your First ADR and Design Doc

### Step 1 — Pick a decision for an ADR

For your coaching SaaS, pick a decision you've made or need to make:
- "Use shared schema with tenant_id for multi-tenancy"
- "Use Meilisearch for search"
- "Use Stripe for payments"
- "Use Forge for hosting"

### Step 2 — Write the ADR

Create `docs/adr/0001-use-shared-schema.md`:

```markdown
# ADR-0001: Use Shared Schema with Tenant ID for Multi-Tenancy

## Status
Accepted

## Context
We need to support multiple coaching centers (tenants) in one 
database. Options:
1. Database per tenant
2. Schema per tenant
3. Shared schema with tenant_id column

## Decision
Use shared schema with tenant_id column on every tenant-scoped table.

## Consequences
Positive:
- Cheap (one DB, one connection pool)
- Easy to migrate (single ALTER TABLE)
- Cross-tenant analytics possible
- Standard pattern (Shopify, Slack, etc.)

Negative:
- Tenant isolation must be enforced in code
- One DB failure = all tenants affected
- Less isolation for compliance (HIPAA)

## Alternatives Considered
Database per tenant:
- Pro: complete isolation
- Con: thousands of DBs, expensive, hard to maintain
- Con: cross-tenant analytics impossible

Schema per tenant:
- Pro: isolation, single DB
- Con: complex migrations (one per schema)
- Con: tooling doesn't support well

## Mitigation
- Global scope on all tenant models (BelongsToTenant trait)
- Comprehensive test coverage of isolation
- Database backups per tenant via filter
```

### Step 3 — Pick a feature for a design doc

A non-trivial feature. Examples:
- "Bulk SMS to parents"
- "PDF report cards"
- "Multi-language support"
- "Tenant onboarding wizard"

### Step 4 — Write the design doc

Create `docs/design/bulk-sms.md`. Use the template from the lesson.

### Step 5 — Get feedback

If you have a friend, mentor, or forum, share the doc. Ask: "What's missing? What would you push back on?"

### Step 6 — Revise

Update the doc based on feedback. Note what changed and why.

### Step 7 — Implement

Now you have a plan. Implement it. Reference the design doc in your PR.

### Step 8 — Update the doc

When the implementation diverges from the design, update the doc. **The doc must match reality.**

### Step 9 — Set up a docs folder

```
docs/
├── README.md           # index of all docs
├── adr/
│   ├── README.md
│   └── 0001-use-shared-schema.md
├── design/
│   ├── bulk-sms.md
│   └── ...
└── architecture/
    └── overview.md
```

### Step 10 — Review quarterly

Calendar reminder: "Review docs." 1 hour. Fix what's stale. Add what's missing.

---

## Quiz

**Q1.** Why write a design doc before code? Isn't that just bureaucracy?

**Q2.** The team writes ADRs for every decision. The list grows to 200. Is this good or bad?

**Q3.** A junior writes a 30-page design doc. A senior says "cut it to 2 pages." Who's right and why?

**Q4.** You wrote an ADR saying "use MySQL." A year later, you switch to Postgres. What do you do?

**Q5.** (Stretch) You're starting a new project. List the first 5 ADRs you'd write, in order.

---

## What You Now Have

✅ Design doc template
✅ ADR template and process
✅ When to write what
✅ Writing style for technical docs
✅ Documentation hierarchy
✅ Doc maintenance practice

## What's Next

Lesson 55 covers **code review culture** — how to give and receive code review, what to look for, how to make it a positive experience for the team.

📖 **[Lesson 55 — Code Review Culture](55-code-review-culture.md)**
