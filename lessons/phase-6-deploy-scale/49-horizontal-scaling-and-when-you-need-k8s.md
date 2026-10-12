# Lesson 49 — Horizontal Scaling and When You Need K8s

> **Phase 6: Deploy & Scale** | You'll be here: ~30 min

## The Senior's Question

Your single server is at 80% CPU. You have 50 customers. The junior buys a bigger server. The senior asks: "do you need more capacity, or do you need reliability?" The answer determines whether you scale up, scale out, or orchestrate. Let me show you the framework.

---

## Theory: Vertical vs Horizontal Scaling

### Vertical (scale up)

Bigger server. More CPU, more RAM, more disk.

```
1 server × 32GB RAM = 1 server with 32GB
```

- **Pros:** simple, no code changes
- **Cons:** expensive at the top, single point of failure, hard limits

### Horizontal (scale out)

More servers. Same or smaller.

```
4 servers × 8GB RAM = 4 servers with 8GB each, 32GB total
```

- **Pros:** cheaper, redundant, scales infinitely
- **Cons:** complex, need load balancer, need shared state (DB, cache)

> **Senior rule:** vertical first, then horizontal.** Start with one big box. When you can't go bigger, scale out.

---

## Theory: When to Scale Vertically

- Single server under 70% CPU/memory consistently
- Cost of bigger server < cost of multiple + complexity
- Low traffic / low value
- You have other things to do

> **For a coaching SaaS at 1,000 customers, a $48/mo DigitalOcean 8GB server is enough.** Don't add complexity you don't need.

---

## Theory: When to Scale Horizontally

- Server at 80%+ CPU consistently
- Need zero-downtime deploys (rolling updates)
- Need reliability (one server can die without outage)
- Geographic distribution (servers in multiple regions)
- Cost of bigger server >> cost of multiple

> **For 10,000+ customers, horizontal becomes the only option.** A single server can't handle that load.

---

## Theory: The Components of Horizontal Scaling

```
                        ┌──────────────┐
                        │ Load Balancer│
                        │ (Nginx/ELB)  │
                        └──────┬───────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
         ┌────────┐       ┌────────┐       ┌────────┐
         │ App 1  │       │ App 2  │       │ App 3  │
         │ PHP-FPM│       │ PHP-FPM│       │ PHP-FPM│
         └────────┘       └────────┘       └────────┘
              │                │                │
              └────────────────┼────────────────┘
                               ▼
                    ┌─────────────────────┐
                    │   Shared services   │
                    │  MySQL, Redis, S3   │
                    └─────────────────────┘
```

Components you need:

- **Load balancer** — distributes traffic
- **Multiple app servers** — stateless
- **Shared DB** — single primary (with replicas for reads)
- **Shared cache** — Redis (one or cluster)
- **Shared storage** — S3 (not local disk)
- **Session storage** — Redis, not files

> **The app servers must be stateless.** No local files. No local sessions. No local cache. **Everything in shared services.**

---

## Theory: Sessions in Laravel (Stateless Apps)

`config/session.php`:

```php
'driver' => env('SESSION_DRIVER', 'redis'),
```

- `file` — local, won't work across servers
- `cookie` — small data only, no server-side
- `database` — works but slow
- `redis` — fast, shared, the senior's default

> **Always use Redis (or cookie/database) for sessions in production.** File-based sessions break horizontal scaling.

---

## Theory: Local Files in Stateless Apps

Storing uploads to `storage/app/`? **Breaks horizontal scaling.** Files on one server, request hits another.

**Always use S3.**

```php
// config/filesystems.php
'default' => env('FILESYSTEM_DISK', 's3'),
```

> **Senior rule:** stateless apps scale. Stateful apps don't.**

---

## Theory: Sticky Sessions (Anti-Pattern)

Load balancers can "stick" a user to one server for the session:

```
User A → Server 1 (for entire session)
User B → Server 2
User C → Server 1
```

This "works" but defeats horizontal scaling. If Server 1 dies, those users lose their session.

> **Don't use sticky sessions.** Use shared session storage (Redis). Any server can serve any request. **Real horizontal scaling.**

---

## Theory: Load Balancer Options

### AWS ELB (Elastic Load Balancer)

- Managed, scales automatically
- $20/mo + per-GB
- Integrates with ECS, EC2

### DigitalOcean Load Balancer

- $12/mo
- Simple, good for small setups

### HAProxy (self-hosted)

- Free, very fast
- You manage it
- More control

### Nginx (self-hosted)

- Can do load balancing
- Common for small setups
- Single point of failure unless clustered

### Cloudflare Load Balancer

- $5/mo, geo-routing
- Active health checks
- Senior's choice for global apps

> **Senior default for AWS:** ELB or ALB. **For DO:** their load balancer. **For global:** Cloudflare.

---

## Theory: Health Checks

The load balancer pings each app server. If it fails, the server is removed from the pool.

```php
// /api/health
public function health()
{
    $checks = [
        'app' => 'ok',
        'database' => 'unknown',
        'cache' => 'unknown',
    ];
    
    try {
        DB::select('SELECT 1');
        $checks['database'] = 'ok';
    } catch (\Exception $e) {
        $checks['database'] = 'down';
    }
    
    try {
        Cache::put('health', '1', 1);
        $checks['cache'] = 'ok';
    } catch (\Exception $e) {
        $checks['cache'] = 'down';
    }
    
    $healthy = !in_array('down', $checks, true);
    
    return response()->json($checks, $healthy ? 200 : 503);
}
```

Configure the load balancer:

- Path: `/api/health`
- Interval: 10 seconds
- Failure threshold: 3 (30 seconds of failure → remove)
- Success threshold: 2

> **Health checks are how horizontal scaling works.** Without them, the load balancer sends traffic to dead servers.

---

## Theory: Auto-Scaling

Cloud-based:

- **AWS:** target tracking on CPU (e.g., add 1 instance when avg CPU > 70%)
- **DO:** simple auto-scaler
- **DigitalOcean:** manual
- **GCP:** managed instance groups

> **Auto-scaling is for spiky traffic.** Steady traffic = fixed size.

For most SaaS, fixed-size + over-provisioned is simpler.

---

## Theory: When You Need Kubernetes

Kubernetes (k8s) is a container orchestrator. It:

- Manages hundreds of containers across many servers
- Auto-scales based on metrics
- Self-heals (restarts dead containers)
- Rolling deploys with zero downtime
- Service discovery (containers find each other)

> **K8s is for 10+ servers and a dedicated DevOps team.** Below that, it's over-engineering.

### When you need K8s

- 10+ app servers
- Multiple services (microservices)
- 100+ deploys per week
- Dedicated DevOps / SRE team
- Specific compliance needs

### When you don't need K8s

- 1-3 app servers
- One monolith
- < 10 deploys per week
- No dedicated DevOps
- Want to ship features, not infrastructure

> **Senior rule:** don't use K8s because it's cool. Use it because nothing else handles your scale.** Most SaaS apps never need it.

---

## Theory: The Alternative to K8s (Senior's Pragmatic Path)

Most SaaS apps use:

```
Phase 1: Single server, Forge-managed       (0-1k customers)
Phase 2: Bigger single server + workers     (1k-5k)
Phase 3: 2-3 app servers + load balancer    (5k-20k)
Phase 4: Auto-scaling group, RDS, ElastiCache (20k-100k)
Phase 5: ECS or EKS, multi-region           (100k+)
```

> **Each step is justified by real load.** K8s is Phase 5. Don't jump to it.

---

## Theory: Container Orchestration Without K8s

If you need orchestration but not K8s:

- **AWS ECS** — Amazon's simpler container service
- **AWS Fargate** — serverless containers
- **Google Cloud Run** — serverless containers
- **Fly.io** — simple container deployment
- **Railway** — opinionated container PaaS
- **Render** — Heroku-style containers

> **Fly.io and Render are the senior's choice for small-to-medium SaaS.** Simpler than K8s, more flexible than Heroku.

---

## Theory: Multi-Region

For global apps, you want servers close to users.

```
US users → US-East servers
EU users → EU servers
APAC users → APAC servers
```

Each region:
- Has its own app servers
- Has its own DB (or read replica)
- Has its own cache

Sync across regions is async. **A user in Tokyo might see data from 5 seconds ago.**

> **Multi-region is a 6-month project.** Don't do it before you have customers in multiple regions complaining about latency.

---

## Theory: The Capacity Planning Exercise

Before scaling, calculate:

```
Daily active users:        1,000
Average requests per user: 100
Peak requests per second:  1,000 × 100 / 86400 × 5 (peak factor) = ~6 req/s

Each app server handles:   ~50 req/s
Number of app servers:     6 / 50 = 1 (with headroom, 2)

DB connections needed:     1,000 users × 0.05 = 50 connections
DB can handle:             500 connections
```

> **Plan capacity before you need it.** "We'll add servers when we need them" is reactive. **"We need 3 servers for Q4 launch" is proactive.**

---

## Theory: The Single Server vs Multi-Server Trade-offs

| | Single Server | Multi-Server |
|---|---|---|
| Cost | Lower | Higher (more servers, load balancer) |
| Complexity | Lower | Higher |
| Reliability | One failure = outage | One failure = no outage |
| Deploys | Some downtime possible | Zero-downtime |
| Capacity | Limited by hardware | Scales infinitely |
| Best for | < 1,000 customers | 1,000+ customers |

> **Senior rule:** single server is fine until it isn't.** When you hit the wall, scale out. **Don't over-engineer before.**

---

## Theory: The "Right Size" Server

For a coaching SaaS:

| Customers | Server size | Cost (DO) | Notes |
|---|---|---|---|
| 0-100 | 2GB | $12/mo | Hobby |
| 100-500 | 4GB | $24/mo | Single server |
| 500-2k | 8GB | $48/mo | Add worker server |
| 2k-5k | 16GB | $96/mo | + load balancer |
| 5k-20k | 2-3 × 8GB | $100-200/mo | + load balancer + managed DB |
| 20k+ | Multi-region, K8s | $500+/mo | DevOps team needed |

> **Match server size to actual load.** Don't pay for 16GB if 4GB is enough.

---

## Why It Matters

Scaling is a **judgment call.** Get it right:
- Cost-effective
- Reliable
- Simple to operate
- Ready to grow

Get it wrong:
- Over-engineering (K8s for 100 users)
- Under-engineering (single server for 50k users)
- Premature optimization (sharding at 100 customers)

> **A senior scales for the load they have, with a plan for the load they expect.** Not for the load they hope for.

---

## Lab: Plan Your Scaling Roadmap

> **This is a planning lab.** You don't need to actually scale — just understand the path.

### Step 1 — Estimate your Phase 1 needs

```
Year 1 target: 100 customers
Average students per tenant: 50
Total students: 5,000
Daily active users: 1,000
Average requests per user per day: 50
Peak RPS: ~3

Server: 4GB, 2 vCPU, 80GB SSD
Cost: $24/mo (DO) + $12 (Forge) = $36/mo
```

### Step 2 — Define Phase 2 triggers

When do you move from Phase 1 to Phase 2?

- CPU sustained > 70%
- Memory > 80%
- Response time p95 > 1s
- Queue wait time > 1 min
- Database CPU > 50%

### Step 3 — Plan Phase 2

Move to a bigger server:

```
Server: 8GB, 4 vCPU, 160GB SSD
Cost: $48/mo
Move DB to managed: +$15/mo
Total: ~$75/mo
```

### Step 4 — Plan Phase 3 (Multi-server)

When single server isn't enough:

```
App server 1: 8GB, 4 vCPU
App server 2: 8GB, 4 vCPU
Load balancer: $12/mo
Managed MySQL: $30/mo
Redis (managed): $15/mo
Worker server: 4GB, 2 vCPU
Total: ~$200/mo
```

### Step 5 — Plan Phase 4 (Auto-scaling)

For spiky or fast growth:

```
3-10 app servers (auto-scaling)
ELB: $20/mo
RDS MySQL with read replica: $100-300/mo
ElastiCache Redis: $50/mo
S3: $5-20/mo
CloudFront: $10-50/mo
Total: $300-1000/mo
```

### Step 6 — Plan Phase 5 (Multi-region / K8s)

When global or massive:

```
Multi-region: US, EU, APAC
ECS Fargate for app (serverless containers)
RDS Aurora Global
ElastiCache Global
CloudFront
Total: $1000-5000/mo
```

### Step 7 — Build the trigger system

Add a command that checks and alerts:

`app/Console/Commands/ScaleCheck.php`:

```php
public function handle()
{
    $cpu = sys_getloadavg()[0];  // 1-min load avg
    $cpuThreshold = 0.7 * swoole_cpu_num();
    
    if ($cpu > $cpuThreshold) {
        Log::warning('scale.cpu_high', ['load' => $cpu, 'threshold' => $cpuThreshold]);
    }
    
    $memory = memory_get_usage(true) / memory_get_total() * 100;
    if ($memory > 80) {
        Log::warning('scale.memory_high', ['percent' => $memory]);
    }
    
    $dbThreads = DB::selectOne('SHOW STATUS LIKE "Threads_connected"')->Value;
    if ($dbThreads > 80) {
        Log::warning('scale.db_connections_high', ['count' => $dbThreads]);
    }
    
    $this->info("CPU: $cpu, Memory: $memory%, DB connections: $dbThreads");
}
```

Schedule hourly.

### Step 8 — Document the scaling decision

`SCALING.md`:

```markdown
# Scaling Plan

## Current state
- Server: 4GB DO
- Cost: $36/mo
- Capacity: ~3k customers

## Triggers to scale
- CPU > 70% sustained
- p95 response time > 1s
- DB CPU > 50%

## Phase 2 (when triggers hit)
- Upgrade to 8GB DO
- Move DB to managed
- Cost: $75/mo
- Capacity: ~10k customers

## Phase 3 (when Phase 2 is full)
- 2 app servers + load balancer
- Separate worker server
- Cost: $200/mo
- Capacity: ~50k customers
```

### Step 9 — Set up the actual current state

Provision your first server (if you want to actually deploy):

```bash
# On Forge
Server: coaching-saas-prod
Size: 4GB
Region: closest to you

# Verify
ssh forge@your-server-ip
php artisan about
```

### Step 10 — Review

Answer:

1. Where am I now? (probably Phase 0 / local)
2. Where do I want to be in 6 months? (probably Phase 1)
3. What's my trigger to scale? (defined above)
4. What does it cost? (calculated above)
5. Who maintains it? (you + Forge)

---

## Quiz

**Q1.** When does single server stop being enough? List 3 specific signals.

**Q2.** You scale to 3 app servers. Sessions break for users mid-session. Why? How do you fix it?

**Q3.** A junior says "we should use Kubernetes." You say "we have 200 users." When is K8s actually justified?

**Q4.** Your traffic is steady at 100 RPS. Should you use auto-scaling or fixed-size? Why?

**Q5.** (Stretch) You have customers in the US and EU. US page load is 200ms; EU is 1.5s. Walk through the multi-region plan.

---

## What You Now Have

✅ Vertical vs horizontal decision framework
✅ Load balancer + health checks
✅ Stateless app patterns
✅ Capacity planning
✅ When to consider K8s
✅ Scaling roadmap for 0-100k customers

## What's Next

Lesson 50 covers **CDN, static assets, edge caching** — making your app fast globally with CloudFront, Cloudflare, and edge caching strategies.

📖 **[Lesson 50 — CDN, Static Assets, Edge Caching](50-cdn-static-assets-edge-caching.md)**
