# Lesson 55 — Code Review Culture

> **Phase 7: Soft Skills** | You'll be here: ~25 min

## The Senior's Question

A junior submits a PR. A senior says "LGTM" (Looks Good To Me) without reading. The bug ships. Another senior writes 200 comments, blocking the junior for 3 days. **Code review is the most important team skill — and most teams do it badly.** Let me show you the senior's way.

---

## Theory: The Two Sides of Code Review

### As the author

You wrote code. Now someone else looks at it. **This is the moment of truth.** Your code is being read by a human who will judge it.

### As the reviewer

Someone else wrote code. You're the second pair of eyes. **Your job is to find issues, share knowledge, and approve with confidence.**

> **Code review is not gatekeeping. It's collaboration.** Done right, both people learn. Done wrong, it's demoralizing and slow.

---

## Theory: The Reviewer's Mindset

The senior's mindset when reviewing:

```
"This is my colleague's work. They tried their best. 
I will help them make it better. 
I will be specific, kind, and useful.
I will approve when ready, not when perfect."
```

> **Senior rule:** review the code, not the person.** "This function is unclear" not "you wrote this badly."

---

## Theory: What to Look For (The Senior's Checklist)

### 1. Correctness

```
- Does it do what the PR description says?
- Are there obvious bugs?
- Are edge cases handled?
- Is the logic right?
```

> **Bugs are the #1 thing to catch.** A senior looks at the diff and asks "what could go wrong?"

### 2. Tests

```
- Are there tests for the new behavior?
- Do they cover edge cases?
- Are they meaningful (not "expect(true)->toBeTrue()")?
- Do existing tests still pass?
```

> **No tests = no approval.** The senior's rule.

### 3. Design

```
- Is this the right abstraction?
- Is it consistent with the rest of the codebase?
- Is the code in the right place (controller, action, model)?
- Are names clear and consistent?
```

> **A senior cares about design.** Not "does it work" but "is it the right way?"

### 4. Security

```
- Is user input validated?
- Is output escaped (XSS)?
- Are SQL queries parameterized?
- Is authorization checked?
- Are secrets logged?
- Multi-tenant isolation correct?
```

> **The senior's job is to be paranoid.** "What if the user is malicious?"

### 5. Performance

```
- Any obvious N+1?
- Are queries indexed?
- Are large operations queued?
- Are pages cached appropriately?
```

> **Don't bikeshed on micro-optimizations.** But flag the obvious stuff.

### 6. Readability

```
- Are names clear?
- Are functions small and focused?
- Are comments useful (not redundant)?
- Is the code formatted (Pint clean)?
```

> **Readable code is maintainable code.** A senior pushes back on clever-but-unclear code.

### 7. Documentation

```
- Is the PR description clear?
- Are public methods documented?
- Are complex things explained?
- Is the relevant doc updated?
```

> **Code without docs is incomplete.**

---

## Theory: The Reviewer's Comment Style

### Good comments

```php
// Specific, actionable, kind
// - "This query could be slow for large tenants. 
//    Consider adding an index on (tenant_id, class_id)."
// - "What happens if $student is null? The method signature 
//    says it's nullable, but we don't handle that case."
// - "Nice refactor! One small thing: let's extract the 
//    email-sending logic into an action for testability."
```

### Bad comments

```php
// Vague, harsh, or condescending
// - "This is wrong."
// - "Why would you do this?"
// - "You should know better."
// - "Just use X."  (without explaining why)
```

> **Senior rule:** every comment should be specific and actionable.** "This is wrong" tells the author nothing. **"This could fail when X is null because Y" tells them everything.**

---

## Theory: The Comment Prefixes

Some teams use prefixes for clarity:

```
nit:       minor style issue, optional
nitpick:   slightly opinionated
suggestion: a different approach
question:   I don't understand, please explain
issue:     this is a bug or problem
blocker:   must fix before merge
```

> **Prefixes set expectations.** "nit" says "ignore if you want." "blocker" says "this stops the merge."

---

## Theory: The Approval Spectrum

A senior's responses:

| Response | Meaning |
|---|---|
| `LGTM` | Looks good, approve |
| `LGTM with comments` | Approve, but please consider these |
| `A few small things` | Almost there, fix these and merge |
| `Needs work` | Significant issues, re-review after |
| `Let's discuss` | Need to talk before proceeding |
| `Blocker` | Cannot merge until this is fixed |

> **Senior rule:** don't use `LGTM` as a default.** Be specific about what you reviewed. **"LGTM, I checked the tests and security"** is better than just "LGTM."

---

## Theory: The Author's Mindset

Submitting a PR is a **request for review**, not a demand to merge.

The senior author:

- ✅ **Tests the code first** — before asking for review
- ✅ **Self-reviews the diff** — catches obvious issues
- ✅ **Writes a clear PR description** — what, why, how
- ✅ **Responds to comments promptly** — within 24h
- ✅ **Resolves conversations** — when addressed
- ✅ **Doesn't take it personally** — comments are about code

> **Senior rule:** before requesting review, walk through your own diff as if you're the reviewer.** You'll catch 30% of issues yourself.

---

## Theory: The PR Description Template

```markdown
## What
[One sentence: what does this PR do?]

## Why
[Why are we doing this? Link to ticket/issue.]

## How
[Brief technical description. Key changes.]

## Testing
- [ ] Unit tests added
- [ ] Feature tests added
- [ ] Manual testing done
- [ ] No regressions

## Screenshots (if UI)
[Before / after]

## Notes for Reviewer
[Anything specific to look at, or tricky parts]
```

> **A clear PR description makes review 5x faster.** The reviewer doesn't have to read the diff to understand the goal.

---

## Theory: Small PRs

A senior keeps PRs small:

```
Junior's PR: 50 files changed, 2000 lines added
Senior's PR: 3 files changed, 100 lines added
```

> **Why small PRs?**
> - Faster to review (15 min vs 3 hours)
> - Easier to find bugs
> - Easier to revert
> - Less merge conflict
> - Gets merged faster

> **Senior rule:** if your PR is more than 400 lines, split it.** Multiple small PRs > one big PR.

---

## Theory: The "Ship in Pieces" Pattern

```
Big feature: "Bulk SMS"
  ↓
PR 1: Database schema (10 min review)
PR 2: SmsService interface (15 min review)
PR 3: TwilioSmsService implementation (20 min review)
PR 4: SendBulkSmsAction (20 min review)
PR 5: HTTP endpoint (15 min review)
PR 6: Tests for isolation (15 min review)
```

> **Each PR is independently mergeable.** Each gets reviewed in 15-20 minutes. **Total review time is similar, but each PR is reviewable.**

---

## Theory: The Review Timing

A senior's review SLA:

```
Within 4 hours:  first response ("I'll review" or initial comments)
Within 24 hours: detailed review complete
```

> **Senior rule:** don't sit on PRs.** A PR waiting = a teammate blocked. **Review promptly.**

If you can't review in 4h, say "I'll review by EOD." That sets expectations.

---

## Theory: Disagreements in Review

When the author and reviewer disagree:

```
1. Discuss in PR comments
2. If unresolved, voice call
3. If still unresolved, escalate to tech lead
4. Tech lead makes final call
5. Document the decision (ADR if it's a big one)
```

> **Senior rule:** disagree and commit.** Once the decision is made, the team moves on. **No grudges.**

---

## Theory: The "Yes, And" Approach

When you disagree, the senior says:

```
"Your approach works. One concern: it doesn't handle the 
concurrent case. What if we add a row-level lock?"

Not: "Your approach is wrong, do it this way."
```

> **Senior rule:** build on the author's work, don't replace it.** "Yes, and..." not "No, but..."

---

## Theory: Code Review as Mentoring

A senior's review teaches:

```
- "I noticed you're not using eager loading here. 
   This causes N+1. Here's how to fix it."
   
- "Have you considered using a Form Request 
   for this validation? It's our pattern."

- "This is a great place to use a policy. 
   Let me show you how we usually do authorization."
```

> **The senior doesn't just approve.** They share patterns, explain why, and elevate the team.

---

## Theory: The Reviewer's Biases

A senior is aware of their own biases:

- **Nitpicking** — focusing on style instead of substance
- **"I would have done it differently"** — but is your way better?
- **Bikeshedding** — debating unimportant things
- **Personal preference** — confusing your taste with correctness

> **Senior rule:** "I would have done it differently" is not a reason to block.** Ask: "Is this actually wrong, or just different?"

---

## Theory: The "I Will Not Block" List

A senior approves even if:

- ✅ Naming is slightly different from their preference
- ✅ There's a slightly more elegant approach
- ✅ Tests could be more thorough (but they're meaningful)
- ✅ Comments are minimal (but code is clear)

> **The senior approves when the code is good enough.** Perfect is the enemy of shipped.

---

## Theory: The "I Will Block" List

A senior blocks when:

- ❌ Tests are missing
- ❌ There's a bug
- ❌ Security issue
- ❌ Breaks existing functionality
- ❌ Doesn't match the PR description
- ❌ Architectural violation (with no explanation)
- ❌ Performance regression

> **The senior's blocking is consistent.** Same standards for everyone, including themselves.

---

## Theory: The "Review Your Own Code First" Habit

Before requesting review:

```bash
# Open the diff yourself
git diff main..HEAD

# Walk through it line by line
# - Would I understand this if I were new?
# - Did I introduce any debugging code (dd, dump)?
# - Did I add tests?
# - Did I update docs?
# - Is the commit history clean?
```

> **The senior reviews their own code first.** Catches obvious issues, makes the reviewer's job easier, and is professional.

---

## Theory: The Squash Merge

A senior prefers squash merging:

```
5 commits in PR → 1 commit on main
```

```bash
# GitHub: "Squash and merge" button
# Result: "Add bulk SMS feature (#123)"
```

> **Why?** The main branch has clean, atomic commits. Each commit = one logical change. **History is clean.**

Some teams prefer merge commits to preserve history. **Senior's choice: squash for clean history.**

---

## Theory: Code Review Tools

- **GitHub PRs** — most common
- **GitLab MRs** — similar
- **Bitbucket** — Atlassian's tool
- **Phabricator** — Facebook's tool
- **Gerrit** — Google's tool
- **Graphite** — stacked PRs
- **Greptile** — AI review

> **Senior's default:** GitHub PRs. Familiar, good integrations, fine for most teams.

---

## Theory: AI-Assisted Code Review

A senior uses AI to assist (not replace) review:

```
# Things AI is good at:
- Catching style issues
- Suggesting improvements
- Explaining complex code
- Finding common patterns

# Things AI is bad at:
- Understanding business context
- Catching design issues
- Knowing the team's conventions
- Reviewing security/architecture
```

> **Senior rule:** AI is a tool, not a replacement.** "AI approved it" is not an approval. **A human must review.**

---

## Why It Matters

Code review is the **highest leverage team activity.** Done right:
- Catches bugs before prod
- Spreads knowledge across the team
- Maintains consistent quality
- Teaches juniors
- Documents decisions

Done wrong:
- Slows the team
- Demoralizes people
- Catches nothing (rubber stamps)
- Creates backlogs

> **A senior's code review is fast, kind, thorough, and consistent.** A junior's is "LGTM" or "rewrite this."

---

## Lab: Establish Your Code Review Practice

### Step 1 — Define your standards

Create `CODE_REVIEW.md`:

```markdown
# Code Review Standards

## Reviewer expectations
- First response within 4 working hours
- Detailed review within 24 hours
- Use comment prefixes: nit, suggestion, issue, blocker
- Approve when ready, not when perfect

## Author expectations
- PR description filled out
- Self-reviewed the diff
- Tests included
- No debug code
- PR < 400 lines (split if larger)

## What to look for
- Correctness
- Tests
- Design
- Security (especially multi-tenant)
- Performance
- Readability
- Documentation

## When to approve
- All blockers resolved
- Tests cover new behavior
- No obvious bugs
- Code is readable
```

### Step 2 — Practice giving review

Open a recent PR you submitted. Re-review it as if you're a senior. What would you flag?

### Step 3 — Practice receiving review

On your next PR, treat every comment as a learning opportunity. Don't argue. Either fix it or explain why you didn't.

### Step 4 — Use prefixes

In your next PR review, prefix your comments:

```
nit: this could be `auth()->id()` for clarity
suggestion: have you considered extracting this to an action?
issue: this N+1 will bite us with 1000 students
```

### Step 5 — Track review time

How long does your average review take? Set a goal to reduce it.

### Step 6 — Set up branch protection

In GitHub → Settings → Branches:

```
☑ Require a pull request before merging
  ☑ Require approvals: 1 (or 2 for production)
☑ Require status checks to pass before merging
```

> **No direct pushes to main. Always PR.**

### Step 7 — Make a checklist for the team

Post in your team channel:

```markdown
## PR Checklist (for authors)
- [ ] PR description filled out
- [ ] Self-reviewed the diff
- [ ] Tests added
- [ ] Pint clean
- [ ] PHPStan clean
- [ ] No debug code
- [ ] Docs updated
- [ ] PR < 400 lines

## PR Checklist (for reviewers)
- [ ] Reviewed within 24h
- [ ] Comments are specific
- [ ] Use prefixes
- [ ] Approve when ready
- [ ] Thank the author
```

### Step 8 — Try stacked PRs (advanced)

Use Graphite or a similar tool to break a big feature into multiple stacked PRs.

### Step 9 — Run a retrospective

After a month, ask the team:
- What's working in our reviews?
- What's not working?
- How can we improve?

### Step 10 — Document your style

In `CONTRIBUTING.md`:

```markdown
## Submitting a PR
1. Create a feature branch
2. Make changes
3. Test
4. Open PR with description
5. Wait for review
6. Address feedback
7. Squash and merge
```

---

## Quiz

**Q1.** A junior's PR has 50 files changed, 2000 lines. They want it reviewed today. What do you do?

**Q2.** A reviewer leaves "this is bad code" as a comment. As the author, how do you respond?

**Q3.** A reviewer wants a complete rewrite of your approach. You think the current approach is fine. Walk through the disagreement.

**Q4.** A senior's PR is missing tests. Another senior approves it anyway, saying "tests can come later." Why is this wrong?

**Q5.** (Stretch) You're a tech lead. Your team is doing 5 PRs a week, 100% approved. Senior says "too easy, not enough rigor." How do you balance speed and quality?

---

## What You Now Have

✅ Code review checklist
✅ Comment style and prefixes
✅ Approval spectrum
✅ PR description template
✅ Small PR discipline
✅ Author and reviewer mindsets
✅ Bias awareness

## What's Next

Lesson 56 — the final lesson — covers **the SaaS founder's checklist beyond code** — the business, legal, marketing, and operational things that turn a working app into a real company.

📖 **[Lesson 56 — The SaaS Founder's Checklist Beyond Code](56-the-saas-founders-checklist-beyond-code.md)**
