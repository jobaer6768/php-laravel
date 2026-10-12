# Lesson 53 — How Senior Developers Think Before Coding

> **Phase 7: Soft Skills** | You'll be here: ~25 min

## The Senior's Question

A junior sees a feature request and starts coding. A senior reads the feature request and **thinks for an hour before writing a line of code.** That hour saves a week. Let me show you the senior's mental process.

---

## Theory: The Junior vs Senior Loop

```
JUNIOR:                              SENIOR:
                                     
Read the ticket                      Read the ticket
   ↓                                    ↓
Start coding                         Understand the problem
   ↓                                    ↓
Hit a wall                           Ask "why" 5 times
   ↓                                    ↓
Try something                        Design the solution
   ↓                                    ↓
Hit another wall                     Write the test first
   ↓                                    ↓
Try something else                   Make it pass
   ↓                                    ↓
Get it working                       Refactor
   ↓                                    ↓
Move on                             Review own code
                                          ↓
                                      Move on
```

> **The senior's upfront thinking is 10x faster than the junior's trial-and-error.** Plus, the senior ships less code, with fewer bugs, that lasts longer.

---

## Theory: The Five Questions

Before writing any feature, the senior asks:

### 1. What problem are we actually solving?

```
"The user needs to export students to Excel"
   ↓
Why? "They want to upload to a system"
   ↓
Why? "The state government requires it"
   ↓
Why? "Compliance with regulation X"
   ↓
Aha — we need a CSV in the format the government system accepts.
      "Excel" is the wrong framing.
```

> **The first answer is rarely the right one.** Keep asking "why."

### 2. What does success look like?

```
Bad:  "It works" — what does that mean?
Good: "The user uploads the file, the government system accepts it, 
       no manual cleanup needed, in under 30 seconds."
```

> **Senior rule:** write the success criteria before writing the code. **If you can't describe success, you can't know if you've achieved it.**

### 3. What are the edge cases?

```
- 0 students (empty file)
- 10,000 students (large file)
- Student with special characters in name
- Student with no email
- Concurrent exports
- Export while data is being modified
```

> **Senior rule:** think about the bad paths first. The happy path is easy. The edge cases are where bugs hide.**

### 4. What could go wrong?

```
- File is too large → OOM
- Database query is slow → timeout
- Format is wrong → user gets angry
- Permissions wrong → user exports data they shouldn't
- GDPR violation → exporting all students' PII
```

> **Senior rule:** security and data integrity are not edge cases. They're the foundation.**

### 5. What's the simplest thing that could work?

```
Junior: builds a flexible export framework with 
        formats, filters, scheduling, email delivery

Senior: builds a CSV download button. 
        Ships in 1 hour. 
        Adds features later if asked.
```

> **Senior rule:** YAGNI. You aren't gonna need it. Build for today's problem, not tomorrow's hypothetical.**

---

## Theory: The 5 Whys

Toyota's famous technique. Used in software too.

```
Problem: The dashboard is slow.
Why?     The query takes 5 seconds.
Why?     It's doing a full table scan.
Why?     There's no index on (tenant_id, created_at).
Why?     We never had this many rows before.
Why?     Because we just signed a customer with 10k students.

Root cause: We didn't anticipate growth in this query.
Solution: Add the index. Plus, plan for scale.
```

> **The senior goes deep.** "Add an index" is a band-aid. "We didn't anticipate growth" is a process issue. **Fix the process, not just the symptom.**

---

## Theory: The Design Phase

After understanding, the senior **designs before coding.**

### Sketch the API

```php
// What's the function signature?
public function exportStudents(Tenant $tenant, string $format = 'csv'): StreamedResponse
{
    // ...
}

// What does it return? A file. A stream. A response.
// What does it take? A tenant. A format.
// What edge cases? Empty tenant. Permission denied.
```

### Sketch the data flow

```
User clicks "Export"
   ↓
Controller receives request
   ↓
Authorizes: can this user export?
   ↓
Validates: which format?
   ↓
Calls action: exportStudents($tenant)
   ↓
Action queries DB with cursor (no OOM)
   ↓
Streams CSV to response
   ↓
User downloads file
```

### Sketch the failure modes

```
- User not authorized → 403
- Tenant has no students → empty CSV
- DB connection drops mid-export → partial file
- File too large → timeout
```

> **The senior sees the whole flow before writing code.** Then the code is just translating the design.

---

## Theory: When to Stop Thinking and Start Coding

> **Senior rule:** thinking is good. Analysis paralysis is bad.** When you can answer:
> 
> - What am I building?
> - Why?
> - What does success look like?
> - What are the edge cases?
> - What's the simplest version?
> 
> **Start coding.** Don't wait for perfect information. **Iterate.**

---

## Theory: The Spikes

For things you don't understand, spike first:

```
"I don't know how Stripe webhooks work"
   ↓
Spike: build a quick prototype, read the docs, 
       play with the Stripe CLI
   ↓
Now I understand. Throw away the prototype.
   ↓
TDD the real implementation.
```

> **Senior rule:** spike to learn, throw away to ship.** Spikes in production code = technical debt.

---

## Theory: The Senior's PR Description

Before writing code, the senior writes a **pull request description.** Forces clarity.

```markdown
## What
Add student CSV export endpoint

## Why
Government compliance requires uploading student data
to the state system in a specific format. Without this,
our customers can't use the app for their reporting.

## How
- New endpoint: GET /students/export
- Returns CSV with columns: student_id, name, email, class, admission_date
- Streams the response (no OOM for large tenants)
- Authorization: admin or owner only

## Testing
- Unit test: CSV format is correct
- Feature test: admin can export
- Feature test: teacher cannot export
- Manual test: 10k students, completes in < 5s

## Edge cases
- Empty tenant → empty CSV with header
- Special characters in names → properly escaped
- Concurrent exports → no DB lock

## Out of scope (future)
- Excel format
- Scheduled exports
- Email delivery
```

> **The PR description is the design document.** If you can't write it, you don't understand the feature. **If you can write it, the code writes itself.**

---

## Theory: The Senior's "Definition of Done"

Before starting:

- [ ] Code is written
- [ ] Tests are written and pass
- [ ] Pint is clean
- [ ] PHPStan is clean
- [ ] Manual test on dev
- [ ] Manual test on staging
- [ ] Edge cases handled
- [ ] No `dd()`, `dump()`, `var_dump()` left
- [ ] No commented-out code
- [ ] Documentation updated
- [ ] Migrations work (up and down)
- [ ] Logs are appropriate (info, not debug, in prod)
- [ ] No N+1 queries
- [ ] No obvious security issues
- [ ] PR description filled out

> **If you can't check all of these, you're not done.** The junior says "it works." The senior says "it's done."

---

## Theory: The Senior's Calendar

```
9 AM:   Standup / email
10 AM:  Design and code reviews
11 AM:  Hardest work (most cognitive load)
12 PM:  Lunch
1 PM:   Hardest work continued
3 PM:   Meetings
4 PM:   Easier work (admin, docs, cleanup)
5 PM:   Plan tomorrow

Friday: ship / deploy / no meetings
```

> **Senior rule:** protect your maker time. Two 4-hour blocks of focused work beats eight 1-hour blocks of interrupted work.**

---

## Theory: The Senior's Reading List

A senior stays sharp by reading:

- **Source code** — Laravel, Symfony, packages you use
- **RFCs** — PHP RFC process, upcoming features
- **Architecture** — "Designing Data-Intensive Applications" (Kleppmann)
- **Operations** — "Site Reliability Engineering" (Google)
- **Practice** — "The Pragmatic Programmer" (Hunt & Thomas)
- **System design** — "System Design Interview" (Xu)

> **Senior rule:** read 30 minutes a day. After 5 years, you've read 100 books. **That's the difference between senior and senior+.**

---

## Theory: The Senior's Habits

Daily:
- [ ] Review own code before submitting PR
- [ ] Test before deploying
- [ ] Deploy small, often
- [ ] Write commit messages that explain "why"
- [ ] Communicate blockers immediately

Weekly:
- [ ] Look at error reports (Sentry)
- [ ] Review performance metrics
- [ ] Update dependencies
- [ ] Write a "what I learned" note
- [ ] Help a junior

Monthly:
- [ ] Review and refactor one area
- [ ] Update documentation
- [ ] Review hosting and infrastructure cost
- [ ] Talk to customers

> **The senior's habits compound.** A junior improves in fits and starts. **A senior improves continuously.**

---

## Theory: The Senior's Failure Modes

Even seniors fail. The difference is **how.**

### Failure mode 1: Analysis paralysis

```
"I'm not ready to code yet, I need to think more"
```

**Fix:** Set a time limit. 30 minutes of thinking, then code. **Done is better than perfect.**

### Failure mode 2: Bikeshedding

```
"Should we use tabs or spaces? Snake case or camel case?"
```

**Fix:** Have a style guide (Pint). Don't argue. **Decide once, automate.**

### Failure mode 3: Premature optimization

```
"This function is O(n²), we should make it O(n log n)"
```

**Fix:** Profile first. Optimize what matters. **99% of "slow" code doesn't need optimization.**

### Failure mode 4: Over-engineering

```
"Let me build a flexible plugin architecture for this one feature"
```

**Fix:** Build the simple version. Add abstraction when needed (Rule of Three). **YAGNI.**

### Failure mode 5: Hero complex

```
"I'll do it all myself, I can do it faster"
```

**Fix:** Delegate. Mentor. **Your job is to multiply the team's output, not to be the team.**

> **Senior rule:** recognize your own failure modes.** Self-awareness is the senior's super-power.

---

## Theory: The Senior's Decision Framework

When faced with a choice:

```
1. What are the options?
2. What are the trade-offs?
3. What does the data say? (if anything)
4. What would I regret more? (inaction vs action)
5. Can I reverse this decision later? (reversibility)
6. Make a decision. Move on.
```

> **Senior rule:** a decision you can reverse is a decision you can make quickly. **A decision you can't reverse (architecture, vendor) needs more thought.**

---

## Why It Matters

The senior's upfront thinking:
- Saves days of trial-and-error
- Catches bugs before they exist
- Builds simpler systems
- Communicates better with team
- Ships faster in the long run

> **A senior's biggest productivity hack is thinking before coding.** 30 minutes of design = 3 hours of coding avoided.

---

## Lab: Practice the Senior's Process

### Step 1 — Pick a feature you haven't built

For your coaching SaaS, pick something you haven't done. Examples:
- "Send bulk SMS to parents about a school closure"
- "Generate PDF report cards"
- "Bulk import students from CSV"
- "Multi-language support"

### Step 2 — Run the 5 Whys

What's the real problem? Document it.

### Step 3 — Write the success criteria

What does "done" look like?

### Step 4 — Sketch the API

```php
public function sendClosureNotice(Tenant $tenant, string $message, array $classIds = []): int
{
    // Returns number of SMS sent
}
```

### Step 5 — Sketch the data flow

```
Controller → Authorize → Validate → Action → SMS Service → Log
```

### Step 6 — List edge cases

- 0 parents to notify
- Parent with no phone number
- SMS service down
- Parent opted out
- International numbers (different format)
- Cost per SMS

### Step 7 — Write the PR description

Use the template from the lesson.

### Step 8 — Estimate

How long will this take? Be honest.

```
- 1 hour: design (you just did it)
- 2 hours: implementation
- 1 hour: tests
- 30 min: manual testing
- 30 min: code review
Total: ~5 hours
```

### Step 9 — Review with a friend

If possible, have a friend (or a senior on a forum) review your design. Where did you miss something?

### Step 10 — Then code

Now you can start coding. The design is done. The code is translation.

---

## Quiz

**Q1.** Why do seniors spend time "thinking" before coding? Isn't that wasted time?

**Q2.** A junior asks "should I use Stripe or LemonSqueezy for payments?" The senior says "what are you trying to achieve?" Why does the senior ask this?

**Q3.** A senior says "build the simplest thing that could work." A junior says "but what if we need X later?" Who's right?

**Q4.** A senior's "definition of done" has 15 items. A junior's has 1 ("it works"). Walk through what the senior would catch that the junior wouldn't.

**Q5.** (Stretch) You have a complex feature to build. Walk through your exact process from "ticket in inbox" to "PR merged." Be specific about what you do at each step.

---

## What You Now Have

✅ Senior's mental model for new features
✅ 5 Whys technique
✅ Edge case thinking
✅ Design before code
✅ PR description as design doc
✅ Definition of Done
✅ Failure mode awareness
✅ Decision framework

## What's Next

Lesson 54 covers **writing design docs and ADRs** — the artifacts that capture senior thinking in writing, so the team (and future you) can understand why decisions were made.

📖 **[Lesson 54 — Writing Design Docs and ADR](54-writing-design-docs-and-adr.md)**
