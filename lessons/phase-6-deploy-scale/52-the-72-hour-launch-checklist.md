# Lesson 52 — The 72-Hour Launch Checklist

> **Phase 6: Deploy & Scale** | You'll be here: ~25 min

## The Senior's Question

You've built the SaaS. You have customers waiting. The junior pushes to prod on Wednesday and hopes. The senior uses a **72-hour checklist** — every detail verified, every failure mode covered, every customer touchpoint polished. Let me give you the runbook.

---

## Theory: The Three Phases of Launch

```
T-72h: Final preps       (server, security, data)
T-24h:  Pre-flight       (smoke tests, monitoring)
T-0:    Launch           (deploy, communicate, watch)
T+24h:  Stabilize        (monitor, fix, listen)
T+72h:  Reflect          (postmortem, optimize)
```

> **The senior's rule: don't skip steps. Each phase has a purpose. Skip one = something will break.**

---

## Theory: The Mindset

The 72 hours before launch are **not for new features.** They are for:
- ✅ Verifying what exists works
- ✅ Removing risk
- ✅ Setting up detection
- ✅ Preparing for failure

> **Senior rule:** "I'll fix it after launch" is a lie.** "After launch" means "when customers complain." **Fix it now.**

---

## T-72h: The Final Preps

### Code

- [ ] All features complete and tested
- [ ] No `dd()`, `dump()`, `var_dump()`, `console.log()` in code
- [ ] No commented-out code (delete it; git remembers)
- [ ] No TODO without an issue ticket
- [ ] All tests passing
- [ ] Code coverage on critical paths (auth, multi-tenancy, payments)
- [ ] Pint is clean
- [ ] PHPStan is clean
- [ ] No high-severity vulnerabilities (`composer audit`)

### Environment

- [ ] Production server provisioned and configured
- [ ] All environment variables set in production
- [ ] `APP_DEBUG=false` in production `.env`
- [ ] `APP_ENV=production` in production `.env`
- [ ] `APP_KEY` is set and not the default
- [ ] Database is created and migrated
- [ ] SSL certificate installed and auto-renewing
- [ ] DNS pointing at the server
- [ ] CDN configured (Cloudflare, etc.)
- [ ] Email service configured and tested
- [ ] File storage (S3) configured
- [ ] Search (Meilisearch) configured
- [ ] Caching (Redis) configured
- [ ] Queue workers running

### Security

- [ ] HTTPS only (no HTTP fallback)
- [ ] Security headers set (CSP, X-Frame-Options, etc.)
- [ ] CSRF protection on all forms
- [ ] SQL injection prevention (Eloquent or parameterized queries)
- [ ] XSS prevention (Blade `{{ }}` escapes by default)
- [ ] Rate limiting on auth endpoints
- [ ] Brute force protection (throttle)
- [ ] Sensitive data not in logs
- [ ] `.env` not in repo
- [ ] File permissions correct (no 777 anywhere)
- [ ] Firewall rules (only 22, 80, 443 open)
- [ ] SSH key-only auth (no passwords)
- [ ] Fail2ban enabled
- [ ] Composer audit: no high-severity issues

### Business

- [ ] Stripe account in live mode
- [ ] Stripe webhook URL points to production
- [ ] Stripe products and prices created
- [ ] Pricing page live
- [ ] Terms of Service published
- [ ] Privacy Policy published (GDPR requirement)
- [ ] Refund Policy published
- [ ] Contact email works
- [ ] Support process defined (where do customers email?)

### Data

- [ ] Anonymized production-like data in staging
- [ ] All migrations run on production database
- [ ] Indexes created
- [ ] Search indexes imported
- [ ] Default admin user created (you)
- [ ] First real customer (yourself or beta tester) can sign up

---

## T-48h: Mid-Prep

### Documentation

- [ ] README.md with setup instructions
- [ ] DEPLOY.md with deploy process
- [ ] RECOVERY.md with disaster recovery
- [ ] SCALING.md with capacity plan
- [ ] API docs (Scribe or similar)
- [ ] Help docs / FAQ

### Monitoring

- [ ] Uptime monitoring (UptimeRobot, Pingdom)
- [ ] Error tracking (Sentry)
- [ ] Performance monitoring (Sentry Performance, Telescope in dev)
- [ ] Server monitoring (CPU, memory, disk)
- [ ] Database monitoring
- [ ] Queue monitoring
- [ ] Alerts going to right people (Slack, email)

### Backups

- [ ] Daily database backups scheduled
- [ ] Backups encrypted
- [ ] Backups stored offsite (S3)
- [ ] Backup restore tested
- [ ] Disaster recovery plan written

### Performance

- [ ] Lighthouse score 90+ on landing page
- [ ] Page load < 2s on broadband
- [ ] Time to First Byte < 200ms
- [ ] Database queries < 100ms p95
- [ ] CDN serving static assets
- [ ] Images optimized (WebP, lazy loading)
- [ ] Brotli compression enabled

### Legal

- [ ] GDPR compliance (if EU customers)
- [ ] Cookie consent banner
- [ ] Data deletion process
- [ ] Data export process
- [ ] Privacy policy linked in footer
- [ ] Terms of service linked in footer
- [ ] Business entity registered
- [ ] Tax collection configured (if applicable)

---

## T-24h: Pre-Flight

### Smoke tests

Run these manually on production:

- [ ] Landing page loads
- [ ] Sign up form works (with real email)
- [ ] Email verification works
- [ ] Login works
- [ ] Password reset works
- [ ] Create student works
- [ ] List students works
- [ ] Edit student works
- [ ] Delete student works
- [ ] Multi-tenant isolation works (sign up as tenant B, can't see tenant A's data)
- [ ] Record payment works
- [ ] Stripe checkout works (test with real card, refund immediately)
- [ ] Webhook works (Stripe sends events)
- [ ] Search works
- [ ] File upload works
- [ ] Email send works (real email arrives)
- [ ] Queue worker processes jobs
- [ ] Scheduled tasks run

### Performance check

- [ ] Lighthouse score on production
- [ ] Page load time from real device
- [ ] Load test with simulated users (optional)

### Final review

- [ ] No `git status` uncommitted changes
- [ ] All branches merged
- [ ] Latest commit is on main
- [ ] All env vars are production values
- [ ] All third-party services in production mode (not test)
- [ ] Stripe: live keys
- [ ] Postmark: production server
- [ ] Sentry: production project
- [ ] Meilisearch: production instance

---

## T-0: Launch Day

### The Deploy

```bash
# 1. Final deploy
git push origin main
# Or via Forge: click Deploy

# 2. Watch the deploy logs
# No errors? Continue.

# 3. Verify the site
curl https://coaching-saas.com
# Returns 200? Continue.

# 4. Check the queue
php artisan queue:size
# 0? Continue.
```

### The Communication

- [ ] Status page: operational
- [ ] Email to waitlist: "We're live!"
- [ ] Social media: announcement
- [ ] Slack/Discord: notify community

### The Monitoring

For the first 4 hours, watch:

```bash
# Logs
tail -f storage/logs/laravel.log

# Errors
# Check Sentry dashboard

# Queue
php artisan queue:size  # should be 0
php artisan queue:failed  # should be 0

# Database
php artisan tinker
>>> User::count()  # growing?
>>> Tenant::count()  # new signups?

# Server
top  # CPU, memory

# Response time
# Check Sentry Performance / Telescope
```

> **Senior rule:** the founder watches for the first 4 hours.** No lunch. No meetings. **Watch.**

### Common launch day issues

**1. Stripe webhook fails**
- Check webhook URL points to production
- Check signing secret matches
- Test with `stripe trigger invoice.payment_succeeded`

**2. Emails don't send**
- Check Postmark API key
- Check from address is verified
- Test with `php artisan tinker`

**3. Database slow**
- Check indexes
- Check slow query log
- Add caching

**4. High CPU**
- Check for runaway loops
- Check for N+1 queries
- Add caching

**5. Customer can't sign up**
- Check email verification
- Check error logs
- Reproduce manually

---

## T+24h: First Day After

- [ ] All alerts reviewed
- [ ] Failed jobs reviewed
- [ ] Customer feedback reviewed
- [ ] Critical bugs fixed immediately
- [ ] Response time verified
- [ ] Uptime verified (target 99.9%)
- [ ] Backup verified (last night's backup exists)

> **Senior rule:** the first 24 hours are the highest risk.** Customers are signing up, payments are flowing, things are breaking. **Stay vigilant.**

---

## T+72h: First Week

- [ ] All issues from first day resolved
- [ ] Performance verified under real load
- [ ] Customer feedback prioritized
- [ ] First real metrics gathered
- [ ] Postmortem written (what broke, what worked)
- [ ] Marketing metrics tracked
- [ ] Plan for next iteration

---

## Theory: The Launch Email

A senior's launch email:

```
Subject: We're live — Coaching SaaS is open for sign-ups

Hi there,

Three months ago, we set out to build the simplest coaching management 
software on the planet. Today, [SaaS name] is open for everyone.

What you get:
✅ Manage students, classes, attendance, fees
✅ Beautiful dashboard, mobile-friendly
✅ Free 14-day trial, no credit card required
✅ Cancel anytime, no questions asked

[Start your free trial →]

Thanks for being part of the journey.

— [Founder name]
```

> **Keep it short. Clear CTA. No hype. No exclamation marks on every line.**

---

## Theory: The Status Page

Customers need to know "is the system down or is it me?"

Use a status page service:

- **Statuspage.io** (free for hobby) — the senior's choice
- **Instatus** — simple, free
- **Cachet** — self-hosted
- **Better Uptime** — alternative

Show:
- ✅ Operational
- ⚠️ Degraded performance
- ❌ Major outage
- 🔧 Maintenance

> **Status page builds trust.** When things break (and they will), customers see you're transparent.

---

## Theory: The On-Call Plan

For a solo founder, **you are the on-call.** For a team, rotate.

```
Week 1: alice
Week 2: bob
Week 3: alice
```

Each on-call:
- Has the production SSH keys
- Has the recovery plan
- Is reachable 24/7
- Acknowledges pages within 15 minutes

> **Senior rule:** if you can't be on call, you can't be in production.** Solo founders accept this. **Don't launch if you can't respond to 3 AM pages.**

---

## Theory: The Bug Triage

When bugs come in (and they will):

```
Severity 1: Service down, all customers affected
   → Fix within 1 hour
   → Examples: DB down, app returns 500, can't login

Severity 2: Major feature broken, some customers affected
   → Fix within 24 hours
   → Examples: Can't record payment, can't see students

Severity 3: Minor feature broken, workaround exists
   → Fix within 1 week
   → Examples: Email formatting wrong, button misaligned

Severity 4: Cosmetic
   → Fix when convenient
   → Examples: typo, color slightly off
```

> **Senior rule:** fix Sev 1 immediately, communicate.** Sev 2 by EOD, communicate. **Sev 3 in next sprint. **Sev 4 in backlog.**

---

## Theory: The First Customer Wins

Your first 10 customers are special. The senior treats them like VIPs.

- [ ] Personal email follow-up after sign-up
- [ ] Personal email after first payment
- [ ] Onboarding call (15 min) to make sure they get value
- [ ] Bug reports from them get priority
- [ ] Feature requests from them get considered first

> **The first 10 customers tell you what to build next.** Listen to them. They are your product roadmap.

---

## Theory: The Launch Metrics

Track from day 1:

```
Sign-ups:       how many people register
Activation:     how many reach "first valuable action"
Conversion:     trial → paid
Retention:      day 1, day 7, day 30
Churn:          % who cancel each month
MRR:            monthly recurring revenue
ARPU:           average revenue per user
NPS:            net promoter score
```

> **The senior tracks these daily.** First week tells you if you have a product. **First month tells you if you have a business.**

---

## Why It Matters

The 72 hours before launch set the tone:
- **Boring launch = good launch.** No drama. No outages. No apologies.
- **Exciting launch = bad launch.** Customers tweeting about bugs, founders apologizing.

> **A senior's launch is invisible.** Customers sign up, pay, use the product, tell their friends. **That's it.**

---

## Lab: Build Your Launch Checklist

> **This is a planning lab.** Customize for your SaaS.

### Step 1 — Build your checklist

Copy the T-72h, T-48h, T-24h, T-0, T+24h, T+72h checklists above into `LAUNCH.md` in your repo.

### Step 2 — Set up the monitoring services

- [ ] Sentry — production project
- [ ] UptimeRobot or similar — uptime monitoring
- [ ] Slack — alerts channel
- [ ] Status page — public status

### Step 3 — Write the launch email

Draft it. Have a friend review. Send it.

### Step 4 — Run the smoke tests

Actually run them. Don't skip.

### Step 5 — Set up on-call

Solo? Just you. Team? Schedule.

### Step 6 — Define the success criteria

```
Day 1: 10 sign-ups
Day 7: 3 paying customers
Day 30: 10 paying customers, 90% retention
```

> **Without targets, you can't tell if launch worked.**

### Step 7 — Plan the postmortem

After launch, write:

```markdown
# Launch Postmortem

## What worked
- ...

## What didn't
- ...

## What we'd do differently
- ...

## Metrics
- Sign-ups: X
- Conversion: X%
- MRR: $X
- NPS: X
```

### Step 8 — Set the launch date

Pick a date. Communicate it. Stick to it.

> **A launch date that's always "next week" never happens.** Pick one. Hold yourself to it.**

---

## Quiz

**Q1.** Why do the 72 hours before launch matter? What's the worst that can happen if you skip pre-flight?

**Q2.** You launch at 9 AM. By 10 AM, 50 sign-ups but payments are failing. Walk through your response.

**Q3.** Your first paying customer says "the dashboard is slow." It's T+1 day. Walk through your response.

**Q4.** Stripe webhook works in test mode but not production. What did you forget?

**Q5.** (Stretch) You're 6 months post-launch, 50 paying customers, $2k MRR. Walk through your next 6 months. What changes about the operations?

---

## What You Now Have

✅ Complete launch checklist
✅ Pre-flight, launch day, post-launch protocols
✅ On-call and incident response
✅ Launch communication plan
✅ First customer playbook
✅ Metrics framework
✅ Postmortem template

## What's Next

**Phase 6 complete!** You now have a deployable, scalable, monitored, backed-up SaaS with a launch runbook.

**Phase 7: Soft Skills** — the things that separate a senior engineer from a senior+ engineer. How to think, how to write, how to review, how to build a company.

📖 **[Phase 7 starts: Lesson 53 — How Senior Developers Think Before Coding](phase-7-soft-skills/53-how-senior-developers-think-before-coding.md)**
