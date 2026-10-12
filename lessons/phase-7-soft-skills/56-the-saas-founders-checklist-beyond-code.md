# Lesson 56 — The SaaS Founder's Checklist Beyond Code

> **Phase 7: Soft Skills** | You'll be here: ~25 min

## The Senior's Question

You've built a working SaaS. Customers can sign up, pay, use the product. **But code is 30% of a SaaS.** The other 70% is legal, marketing, support, accounting, and operations. The senior knows what they are — and tackles them. Let me show you the full picture.

---

## Theory: The Founder's Iceberg

```
What customers see:
    [ Web app, marketing site, support ]
              ↑
———————————————————————————————
What founders do:
    Legal    Marketing  Support   Accounting  Operations  Tax
    Privacy  Content    Docs      Billing     Hiring     Insurance
    ToS      SEO        Community Bank        Sales      Compliance
    Contracts         Email       Payroll   Pricing     Localization
    GDPR               Social      Bookkeeping
```

> **Senior rule:** "I'll figure out the legal stuff after I get customers."** Wrong. You need at least the basics before launch. **Get them done now.**

---

## Theory: The Legal Basics

### 1. Business Entity

You need a legal entity to:
- Sign contracts
- Receive payments
- Get a bank account
- Pay taxes
- Limit personal liability

```
Options:
- Sole proprietorship   (just you, no protection)
- LLC                  (US, simple, $100-500 to form)
- Limited company      (UK, simple)
- Pvt Ltd              (India, simple)
- C-Corp / Inc          (US, for investors)
```

> **Senior default for solo founders: LLC (US) or Ltd (UK) or Pvt Ltd (India).** Simple, cheap, liability protection.

### 2. Terms of Service

What you and the customer agree to. Includes:

- What you provide
- What you don't guarantee (no SLA)
- Payment terms
- Cancellation and refund policy
- Limitation of liability
- Dispute resolution
- Governing law

> **Don't write this yourself.** Use a template from a lawyer or service like TermsFeed / iubenda. **Costs $100-500, saves you from lawsuits.**

### 3. Privacy Policy

Required by GDPR (EU) and similar laws. Includes:

- What data you collect
- Why you collect it
- How you protect it
- How users can access, export, delete their data
- Where data is stored
- Who you share it with (Stripe, Postmark, etc.)

> **Senior default: use a generator (iubenda, TermsFeed).** Update yearly.

### 4. Cookie Policy

If you use any cookies (analytics, etc.), you need a cookie banner.

Tools: CookieBot, CookieScript, or implement manually with a simple consent banner.

> **Senior rule:** for EU customers, you NEED a cookie banner.** No banner = €10M fine under GDPR. **Just do it.**

### 5. Data Processing Agreement (DPA)

If you process EU customer data, your customers may need a DPA from you (because their data subjects are EU).

If you use Stripe, they have a DPA you can use.

> **For B2B SaaS serving EU customers, you need this.**

### 6. GDPR Compliance (EU Customers)

If you have EU customers:
- [ ] Privacy policy linked from every page
- [ ] Cookie banner
- [ ] User can export their data
- [ ] User can delete their account
- [ ] Data breach notification process (within 72 hours)
- [ ] DPA available on request
- [ ] Lawful basis for processing (usually "contract" or "consent")

### 7. CCPA Compliance (California)

If you have California customers:
- [ ] "Do Not Sell My Info" link
- [ ] User can request their data
- [ ] User can request deletion

> **Most US states have copycat laws. Senior's default: GDPR-compliant globally.** Easier than per-state.

---

## Theory: The Marketing Basics

### 1. Domain Name

Get a `.com` (or your country TLD). Use Cloudflare Registrar for cheap (no markup).

### 2. Email Addresses

```
hello@coaching-saas.com      for general inquiries
support@coaching-saas.com    for customer support
billing@coaching-saas.com    for billing issues
noreply@coaching-saas.com   for system emails
```

> **Senior rule:** use a helpdesk (Helpdesk, HelpDesk, Front, etc.) for support@.** Personal email doesn't scale.

### 3. Marketing Site

Landing page that explains:
- What the product does
- Who it's for
- Pricing
- Sign-up CTA

For early stage, a single page is fine. Use Laravel + Blade, or a static site generator, or Carrd / Framer.

### 4. SEO Basics

- [ ] Page titles unique per page
- [ ] Meta descriptions
- [ ] Open Graph tags (for social sharing)
- [ ] Sitemap.xml
- [ ] robots.txt
- [ ] Schema.org markup (Organization, Product)
- [ ] Fast page load (Lighthouse 90+)
- [ ] Mobile-friendly

### 5. Analytics

- [ ] Plausible or Fathom (privacy-friendly)
- [ ] Or Google Analytics 4
- [ ] Track: visitors, sign-ups, conversion, churn

> **Senior default for EU customers: Plausible.** Privacy-friendly, no cookie banner needed.

### 6. Social Presence

- [ ] Twitter / X account
- [ ] LinkedIn page
- [ ] Facebook page (if relevant)
- [ ] YouTube (for tutorials)

> **Senior rule:** pick 1-2 platforms.** Don't spread thin. **Where are your customers?**

### 7. Launch Channels

- [ ] Hacker News (for tech audience)
- [ ] Product Hunt
- [ ] Reddit (relevant subreddits)
- [ ] Indie Hackers
- [ ] Twitter launch thread
- [ ] Email to waitlist

> **Senior rule:** launch once, well.** Don't spam. **Make a launch that fits the audience.**

---

## Theory: The Support Basics

### 1. Help Documentation

A help center with:
- Getting started guide
- Common tasks
- FAQ
- Troubleshooting

Use a tool like:
- **HelpDesk** (Zendesk) — paid
- **HelpScout Docs** — paid
- **HelpCenter** (free, Gitbook-style)
- **Plain text README** — for very early stage

### 2. Support Email

`support@coaching-saas.com`. Goes to a helpdesk or shared inbox.

> **Senior rule:** set expectations.** "We respond within 24 hours" or "within 1 business day." **Then meet them.**

### 3. Status Page

Public status of your service. Already covered in Lesson 52.

### 4. In-App Help

Tooltip, "?" icons, link to docs.

### 5. Onboarding

The first-run experience:
- Welcome screen
- Quick tour
- Sample data
- First task guided

> **Senior rule:** make the first 5 minutes magical.** Most users churn in the first 5 minutes. **Show them value immediately.**

---

## Theory: The Financial Basics

### 1. Bank Account

A business bank account, separate from personal.

Options: Mercury (US, popular for SaaS), Wise, or your local bank.

### 2. Accounting Software

- **QuickBooks** — popular in US
- **Xero** — popular in UK/AU
- **Wave** — free for basic
- **FreshBooks** — simple

Track: revenue, expenses, invoices.

### 3. Tax Registration

In most countries, you need to register for taxes (VAT, GST, sales tax).

**Hire an accountant.** Even part-time. They save you 10x their cost.

### 4. Pricing Strategy

```
Common SaaS pricing:
- Free trial (14 days)
- Free tier (limited features)
- Starter ($X/mo)
- Pro ($Y/mo)
- Enterprise (custom)

Annual discount: 17-20% off monthly
Per-seat discounts:
- Volume discounts for larger teams
```

> **Senior rule:** price for value, not cost.** If you save a customer 10 hours/month, charge 1-10% of that value.

### 5. Refund Policy

Decide upfront. Common:
- 30-day money-back, no questions
- Pro-rated refund for cancellations
- No refunds (risky)

> **Senior default:** 30-day money-back for the first month.** Builds trust. The cost is low.

---

## Theory: The Sales Basics

### 1. The Sales Funnel

```
Visitor → Lead → Trial → Paid → Customer → Advocate
   ↓        ↓       ↓       ↓         ↓
 Awareness Interest Demo  Purchase Retention
```

### 2. Conversion Metrics to Track

```
Visitor → Sign-up:        target 2-5%
Sign-up → Activation:     target 20-40%
Trial → Paid:             target 5-15%
Monthly churn:            target < 5%
```

### 3. The First Customers

For B2B SaaS, the first 10 customers come from:
- Your network
- Cold emails
- Direct outreach
- Communities (Reddit, IndieHackers)

> **Senior rule:** the founder does sales until it doesn't scale.** Then hire a sales person.

### 4. The Sales Process

```
1. Lead signs up
2. Email: "How can we help?"
3. (Optional) Onboarding call
4. Usage monitoring
5. At day 7, day 14: check-in
6. At trial end: "Your trial is ending, want to subscribe?"
7. If churn: "What could we have done better?"
```

---

## Theory: The Operational Basics

### 1. Customer Support Hours

Even if it's just you:
- Monday-Friday, 9-5 your timezone
- Or 24/7 if global

> **Senior rule:** set hours.** "Always on" means always exhausted.

### 2. Incident Response

Already covered. Plan + execute.

### 3. Customer Feedback Loop

```
Customer feedback → Product backlog → Roadmap → Ship → Tell customer
```

> **Senior rule:** tell the customer when their feedback ships.** They feel heard. **They become advocates.**

### 4. Hiring (Eventually)

When you hire:
- Hire for slope, not intercept (learning ability, not current skill)
- Hire someone better than you at their role
- Hire slow, fire fast
- First hires: customer success, then engineering

> **Don't hire before you can afford 6 months of their salary.** Runway matters.

---

## Theory: The Founder's Time Allocation

Year 1 (solo):
```
40%   Building product
20%   Talking to customers
15%   Marketing
10%   Support
10%   Operations (legal, accounting)
5%    Misc
```

Year 2 (with first hire):
```
30%   Building product
25%   Sales / customer success
15%   Marketing
15%   Hiring / ops
10%   Support
5%    Misc
```

> **Senior rule:** customer time compounds.** Talk to customers every week. **Every other activity supports this.**

---

## Theory: The Founder's Habits

Daily:
- [ ] Check support inbox
- [ ] Check error reports (Sentry)
- [ ] Reply to customer feedback
- [ ] Ship one thing

Weekly:
- [ ] Customer call (5-10 calls a week, 15 min each)
- [ ] Update roadmap
- [ ] Review metrics (sign-ups, churn, MRR)
- [ ] Review hosting / infra costs
- [ ] Write a "what I learned" note

Monthly:
- [ ] Financial review (revenue, expenses, runway)
- [ ] Update pricing / plans if needed
- [ ] Backup verification
- [ ] Tax / accounting
- [ ] Self-review

Quarterly:
- [ ] Strategic review (am I on track?)
- [ ] Update docs
- [ ] Update legal pages if needed
- [ ] Plan next quarter

> **The senior founder has a rhythm.** Daily, weekly, monthly, quarterly. **No chaos.**

---

## Theory: The Founder's Mindset

The hard truths:

1. **Most SaaS startups fail.** Not because of bad code.
2. **Distribution is harder than product.** Building is fun. Selling is hard.
3. **Customer feedback is more valuable than your opinion.** The customer is always partly right.
4. **Speed matters more than perfection.** Ship, learn, iterate.
5. **Burnout is real.** Take breaks. Set boundaries.

> **Senior rule:** building the product is 30% of the work.** Marketing, sales, support, operations are the other 70%. **Don't underestimate them.**

---

## Theory: The Founder's Resources

### Books

- **"The Mom Test"** — how to talk to customers
- **"Traction"** — marketing channels
- **"Lean Startup"** — build-measure-learn
- **"Hooked"** — habit-forming products
- **"From Impossible to Inevitable"** — SaaS metrics
- **"Obviously Awesome"** — positioning

### Communities

- **IndieHackers** — solo founders
- **MicroConf** — SaaS conferences
- **Twitter SaaS community** — share and learn
- **Reddit r/SaaS** — questions and answers

### Tools

- **Stripe** — payments
- **Postmark** — email
- **HelpDesk** — support
- **Linear** — issue tracking
- **Notion** — docs
- **Plausible** — analytics
- **Loom** — videos for customers

---

## Why It Matters

The senior founder knows:
- Code is necessary, not sufficient
- Legal and ops compound over time (do them early)
- Customer conversations drive everything
- Money, runway, and metrics are real
- Burnout is a choice (mostly)

> **A senior founder's business has a moat.** Not technology — operations. **Customer trust, processes, and quality compound.**

---

## Lab: Build Your Founder's Checklist

### Step 1 — Legal basics

- [ ] Business entity formed
- [ ] Bank account opened
- [ ] Terms of Service published
- [ ] Privacy Policy published
- [ ] Cookie banner (if EU)
- [ ] Tax registration done
- [ ] Accountant hired

### Step 2 — Marketing basics

- [ ] Domain registered
- [ ] Email addresses set up (support, hello, noreply)
- [ ] Marketing site live
- [ ] Analytics tracking
- [ ] Social accounts created
- [ ] SEO basics in place

### Step 3 — Support basics

- [ ] Help docs written
- [ ] Support email configured
- [ ] Status page set up
- [ ] Onboarding flow designed

### Step 4 — Financial basics

- [ ] Accounting software set up
- [ ] Pricing page live
- [ ] Refund policy published
- [ ] Stripe webhook working

### Step 5 — Operational basics

- [ ] On-call plan defined
- [ ] Feedback collection in place
- [ ] Roadmap documented
- [ ] Founder habits established

### Step 6 — Run a customer interview

Find a potential customer (or current one). 15-minute call:
- What do they do today for this problem?
- What have they tried?
- What worked, what didn't?
- Would they pay for a solution? How much?

> **Document the conversation.** Quote them (with permission). Use the quotes in your marketing.

### Step 7 — Set up your metrics dashboard

Track weekly:
- Visitors
- Sign-ups
- Activations (first valuable action)
- Paying customers
- MRR
- Churn
- NPS (or simple "would you recommend?")

### Step 8 — Plan your first hire (eventually)

When? When you can afford 6 months salary + benefits + tools.

What role? Customer success, then engineering.

How? From your network, or a recruiter (after you can't find people yourself).

### Step 9 — Write a "next 90 days" plan

```
Q2 2024
- [ ] Launch publicly
- [ ] Get 10 paying customers
- [ ] Hit $500 MRR
- [ ] Talk to 20 customers
- [ ] Ship 3 major features based on feedback
- [ ] Hire a part-time contractor
```

### Step 10 — Reflect

```
- What did I learn this week?
- What's blocking me?
- What's the next decision I need to make?
- Am I taking care of myself?
```

---

## Quiz

**Q1.** A founder says "I'll do legal stuff after I get 100 customers." Why is this wrong?

**Q2.** Your SaaS is GDPR-non-compliant (no cookie banner, no data export). What are the risks? Walk through the fix.

**Q3.** You have 50 paying customers. When should you hire your first employee? What role?

**Q4.** A junior founder spends 80% of time coding. A senior founder spends 40% coding. Who's right?

**Q5.** (Stretch) You're 1 year in, $5k MRR, 100 customers. What's your next 6 months look like?

---

## What You Now Have

✅ Legal basics checklist
✅ Support basics
✅ Financial basics
✅ Operational basics
✅ Founder's habits and time allocation
✅ Customer feedback loop
✅ 90-day planning

## Course Complete

**You have finished all 56 lessons.**

✅ **Phase 0:** Environment setup (5 lessons)
✅ **Phase 1:** PHP fundamentals (8 lessons)
✅ **Phase 2:** Laravel core (10 lessons)
✅ **Phase 3:** Architecture (7 lessons)
✅ **Phase 4:** Quality (5 lessons)
✅ **Phase 5:** Production (7 lessons)
✅ **Phase 6:** Deploy & Scale (10 lessons)
✅ **Phase 7:** Soft Skills (4 lessons)

**What you can now do:**
- Build a multi-tenant Laravel SaaS
- Write tested, type-safe, formatted code
- Deploy to production with confidence
- Scale to thousands of users
- Operate it like a senior engineer
- Build the business around it

**What's next is up to you.** The code is just the beginning. **Talk to customers. Ship. Iterate. Compound.**

🎉 **Go build your coaching SaaS.**