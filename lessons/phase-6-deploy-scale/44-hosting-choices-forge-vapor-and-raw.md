# Lesson 44 — Hosting Choices: Forge, Vapor, and Raw

> **Phase 6: Deploy & Scale** | You'll be here: ~30 min

## The Senior's Question

You're ready to deploy. You Google "Laravel hosting" and find 50 options. The junior picks the cheapest. The senior picks based on **who manages what, what they cost at scale, and what fits the team's skills.** Let me show you the framework for choosing.

---

## Theory: The Hosting Decision Framework

Five questions a senior asks:

1. **Who manages the server?** (you / partial / fully)
2. **What's the traffic pattern?** (steady / spiky / unknown)
3. **What's the team's DevOps skill?** (high / medium / low)
4. **What's the budget?** ($50 / $500 / $5000+ per month)
5. **What are the compliance needs?** (none / GDPR / HIPAA / SOC 2)

Your answers map to one of three paths:

```
Low ops / low skill → Managed (Vapor, Heroku, Render)
Medium ops / medium skill → Semi-managed (Forge, Ploi)
High ops / high skill → Raw (DigitalOcean, AWS, Hetzner)
```

---

## Theory: Option 1 — Laravel Forge ($12-249/mo + server)

**What it is:** Server management panel by Laravel's creator. You bring a server (DigitalOcean, AWS, Hetzner, Linode). Forge installs PHP, Nginx, MySQL, Redis, sets up deploys, SSL, queues.

**Who manages what:**
- Forge: server provisioning, deploys, SSL, monitoring basics
- You: app code, database, scaling decisions
- Server provider: hardware, network

**Pros:**
- ✅ Purpose-built for Laravel
- ✅ One-click PHP/MySQL/Redis setup
- ✅ Git push to deploy
- ✅ Free SSL via Let's Encrypt
- ✅ Queue worker management
- ✅ Cron job management
- ✅ Zero-downtime deploys

**Cons:**
- ❌ Single server (scale by upgrading, not horizontal)
- ❌ You manage the server
- ❌ Not serverless (pays even at zero traffic)

**Best for:** Solo devs, small teams, traditional SaaS, predictable traffic.

**Cost example:**
- DO server: $24/mo (4GB RAM, 2 vCPU)
- Forge: $12/mo (Basic plan)
- Total: ~$36/mo for production

> **Senior default for solo / small teams.** Forge is the right balance of automation and control.

---

## Theory: Option 2 — Laravel Vapor ($39/mo + AWS)

**What it is:** Serverless platform by Laravel's creator. Your app runs on AWS Lambda. No servers to manage. Auto-scales to zero.

**Who manages what:**
- Vapor: server management, scaling, certificates
- AWS: hardware
- You: app code

**Pros:**
- ✅ True auto-scaling (0 to millions of requests)
- ✅ Pay only for what you use
- ✅ No server maintenance
- ✅ Built-in CDN (CloudFront)
- ✅ Built-in cache (DynamoDB)
- ✅ Built-in queues (SQS)
- ✅ Native Laravel support

**Cons:**
- ❌ Lambda cold starts (first request ~1s)
- ❌ AWS lock-in
- ❌ Different mental model (stateless functions)
- ❌ Some Laravel features need workarounds
- ❌ Debugging is harder
- ❌ More expensive at steady high traffic

**Best for:** Apps with spiky traffic, large teams with AWS expertise, $1M+ ARR.

**Cost example:**
- Vapor: $39/mo (Team plan)
- AWS Lambda: ~$0-100/mo at small scale
- DynamoDB: $1-25/mo
- CloudFront: $1-10/mo
- Total: $40-200/mo at small scale

> **Vapor is the senior's choice for B2C with unpredictable traffic.** When your SaaS hits Product Hunt and you get 50k signups in a day, Vapor scales. Forge on a $24 server dies.

---

## Theory: Option 3 — Raw VPS ($5-200/mo)

**What it is:** You rent a server and run everything yourself. DigitalOcean, Hetzner, Vultr, AWS EC2, Linode.

**Who manages what:**
- You: everything
- Provider: hardware, network

**Pros:**
- ✅ Cheapest
- ✅ Full control
- ✅ No vendor lock-in
- ✅ Any stack you want

**Cons:**
- ❌ You do everything (provisioning, security, updates)
- ❌ You handle scaling
- ❌ You handle backups
- ❌ You handle SSL
- ❌ You handle monitoring
- ❌ Mistakes are catastrophic

**Best for:** Large teams with DevOps expertise, specific compliance needs, full custom stacks.

**Cost example:**
- Hetzner: €5/mo (2GB RAM)
- DO: $6/mo (1GB)
- Time to set up: 4-8 hours
- Time to maintain: 2-4 hours/month

> **Senior only with a DevOps team.** A solo founder should never run a raw server for production. Too many things to forget.

---

## Theory: Other Options (Quick Survey)

| Service | Type | Best for |
|---|---|---|
| **Ploi** | Like Forge, similar pricing | EU users, simpler UI |
| **Heroku** | PaaS | Quick deploys, expensive at scale |
| **Render** | PaaS | Heroku alternative, generous free tier |
| **Railway** | PaaS | Hobby projects, quick deploys |
| **DigitalOcean App Platform** | PaaS | Simple Laravel deploys |
| **Cloudways** | Managed VPS | WordPress + Laravel |
| **RunCloud** | Like Forge | Custom control panel |
| **AWS Lightsail** | VPS | AWS ecosystem users |

---

## Theory: When to Migrate

```
   Solo / MVP          Growth / stable         Big / spiky
       │                      │                      │
       ▼                      ▼                      ▼
   Hetzner $5       DO + Forge $36           Vapor $200+
   (DIY)            (semi-managed)           (fully managed)
```

Migrate when:
- **From raw to Forge:** you spend more than 1 day/month on server issues
- **From Forge to Vapor:** you have spiky traffic that costs you money OR you outgrow a single server
- **Stay put:** your current setup works and you have other things to do

> **Senior rule:** optimize for your time, not the hosting bill.** If you're spending 10 hours/month on raw server maintenance, the $30 you save isn't worth it.

---

## Theory: The Total Cost of Ownership

Hosting isn't just the server bill. Total cost includes:

```
Server: $24/mo
Forge: $12/mo
Backups: included
Monitoring (Sentry): $26/mo
Email (Postmark): $10/mo
Error budget: $10/mo
Logging (Papertrail): $5/mo
Uptime monitoring: free
---
Total: ~$87/mo
```

A solo founder should expect to spend $50-200/mo for a production SaaS. **Don't cut corners on monitoring and backups.**

---

## Theory: Multi-Region Considerations

Where is your customer?

- **US customers** → US-East (Virginia) or US-West (Oregon)
- **EU customers** → Frankfurt, Ireland
- **APAC customers** → Singapore, Tokyo
- **Global** → multi-region CDN (CloudFront) + single region app

> **Senior rule:** put the app close to the database.** Latency is dominated by DB roundtrips, not static assets. Static assets go on CDN.

---

## Theory: Compliance

| Compliance | Hosting implication |
|---|---|
| None | Any |
| GDPR (EU) | Data in EU, GDPR DPA with provider |
| SOC 2 | Audit trail, access logs, encrypted at rest |
| HIPAA | BAA with provider, no public clouds without BAA |
| PCI DSS | Tokenize cards (don't store), use Stripe Elements |

> **A coaching SaaS with EU customers needs GDPR.** Choose providers with EU regions and GDPR Data Processing Agreements.

---

## Theory: The Migration Path

When your SaaS grows, you'll move hosts. Plan for it now:

- **Use env vars for ALL config** (no hard-coded values)
- **Containerize your app** (Docker makes migration easy)
- **Keep DB schema migrations in code** (no manual SQL)
- **Use S3-compatible storage** (S3, R2, DO Spaces all work the same)
- **External services for state** (Postmark, Stripe, Sentry — not self-hosted)

> **Senior rule:** make the app portable. Don't paint yourself into a corner.**

---

## Theory: My Recommendation for a Coaching SaaS

**Phase 1 (0-100 customers):**
- Hetzner $5/mo + Forge $12/mo
- Single server, MySQL + Redis on same box
- Daily backups to S3
- Total: ~$25-50/mo

**Phase 2 (100-1000 customers):**
- DigitalOcean $24-48/mo + Forge $12/mo
- Bigger server, separate Redis
- Managed DB option
- Total: ~$50-100/mo

**Phase 3 (1000+ customers):**
- DO or AWS $100-500/mo + Forge $39/mo
- Multiple servers, load balancer
- Read replicas
- Or migrate to Vapor
- Total: ~$200-1000/mo

> **Start with Phase 1.** Move to Phase 2 when the server struggles. Move to Phase 3 when money allows. **Senior scales by need, not by anticipation.**

---

## Why It Matters

Hosting is a **time vs money vs control** decision. Get it wrong:
- Cheap but unmaintained → outages
- Expensive but underused → wasted money
- Right-sized but misconfigured → still outages

Get it right:
- $25-50/mo is enough for 100 customers
- Forge gives you 95% of automation with 5% of vendor lock-in
- You can migrate when you outgrow it

> **A senior picks the boring, reliable option. Boring is good. Boring means you can sleep at night.**

---

## Lab: Plan Your Hosting

This is a planning lab, not a setup lab. We're not actually deploying — just choosing.

### Step 1 — Estimate your needs

Answer:

1. How many customers in year 1? (e.g., 50)
2. How many students total? (e.g., 2,500)
3. Where are they geographically? (e.g., South Asia)
4. What's your monthly hosting budget? (e.g., $50)
5. Do you have DevOps experience? (likely no, you're a MERN dev learning PHP)

### Step 2 — Pick your stack

Based on your answers:

```
Recommended for solo founder / first year:
  → Hetzner CX21 (4GB, €10/mo) or DigitalOcean 4GB ($24/mo)
  → Laravel Forge ($12/mo Basic plan)
  → MySQL on same box (or managed DB for +$15/mo)
  → Redis on same box
  → Backups to Backblaze B2 ($5/mo for 100GB)
  → Sentry free tier (5k errors/mo)
  → Postmark $10/mo (10k emails)
  → Total: ~$50-75/mo
```

### Step 3 — Plan the migration

If you outgrow Phase 1, your migration plan:

1. Move DB to managed (DigitalOcean Managed MySQL)
2. Move Redis to managed (Redis Cloud or Upstash)
3. Move to bigger server (DO 8GB $48/mo)
4. Eventually: add second server + load balancer

### Step 4 — Set up an actual account (Forge)

If you want to follow along:

1. Sign up at https://forge.laravel.com
2. Connect a server provider (DigitalOcean, Hetzner, etc.)
3. Provision a server (Ubuntu 22.04, 4GB)
4. Forge installs PHP 8.3, Nginx, MySQL, Redis
5. Add your SSH key
6. Create a site
7. Connect to GitHub
8. Set up a deploy script

> **Don't actually do this in the lab — just understand the steps.** Real deploy happens in the next lessons.

### Step 5 — Cost calculator

Build a spreadsheet:

| Item | Provider | Cost | Notes |
|---|---|---|---|
| Server | Hetzner CX21 | €10/mo | 4GB, 2 vCPU, 40GB SSD |
| Server management | Forge Basic | $12/mo | |
| Database | Same box | $0 | Until you need managed |
| Email | Postmark | $10/mo | 10k emails |
| Error tracking | Sentry | $0 | Free tier |
| File storage | S3 or R2 | $1-5/mo | Pay per use |
| Search | Meilisearch Cloud | $0-30/mo | Free tier covers most |
| Backups | S3 lifecycle | $1-3/mo | Daily DB snapshots |
| **Total** | | **~$35-70/mo** | |

### Step 6 — Define your "scale triggers"

When do you migrate to bigger infrastructure?

```
- CPU sustained > 70%       → upgrade server
- Memory > 80%              → upgrade server
- DB CPU > 70%              → managed DB
- Queue backlog > 10k       → dedicated queue worker
- Response time p95 > 1s    → add caching layer
- Monthly cost > $500       → evaluate Vapor
- Single region feels slow  → add CDN
```

> **Define the triggers before you need them.** "We'll scale when we need to" is a junior's answer. **"We'll scale when X hits Y" is a senior's.**

---

## Quiz

**Q1.** Three hosting options: Forge+VPS, Vapor, Raw VPS. For a solo founder's coaching SaaS in year 1, which is best? Why?

**Q2.** Vapor is "auto-scaling to zero." What does that mean, and when is it a bad thing?

**Q3.** You pick Hetzner + Forge. List 3 things Forge does for you, and 3 things you still need to handle.

**Q4.** Your traffic spikes 10x for 2 days after a news mention. Which hosting model handles this best? Why?

**Q5.** (Stretch) You're at 5,000 customers, 200k students, paying $300/mo for hosting. Should you migrate to Vapor? Walk through the analysis.

---

## What You Now Have

✅ Framework for choosing hosting
✅ Trade-offs: Forge vs Vapor vs Raw
✅ Total cost awareness
✅ Migration path planned
✅ Scale triggers defined

## What's Next

Lesson 45 covers **deploying on Laravel Forge** — the senior's actual workflow. Server provisioning, deploy script, zero-downtime deploys, queue workers, monitoring.

📖 **[Lesson 45 — Deploying on Laravel Forge](45-deploying-on-laravel-forge.md)**
