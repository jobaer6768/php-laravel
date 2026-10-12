# Lesson 48 — Database at Scale

> **Phase 6: Deploy & Scale** | You'll be here: ~30 min

## The Senior's Question

You have 1,000 coaching centers, 50,000 students. The dashboard takes 5 seconds. The report query times out. The junior adds more indexes. The senior knows **the database is the bottleneck at scale** — and the patterns that fix it without rewriting the app. Let me show you.

---

## Theory: The Database Reality

A single MySQL server has limits:

- **~10,000 reads/sec** (with SSD, well-tuned)
- **~1,000 writes/sec** (writes are slower, lock contention)
- **~100 GB** of comfortable storage
- **1 CPU core** ≈ 1,000 simple queries/sec

> **Senior rule:** the database doesn't scale vertically forever.** A 64-core server is expensive. **Read replicas are the answer.**

---

## Theory: Read Replicas

```
              ┌──────────┐
              │ Primary  │ ← writes
              │ (master) │
              └────┬─────┘
                   │ async replication
        ┌──────────┼──────────┐
        ▼          ▼          ▼
   ┌────────┐ ┌────────┐ ┌────────┐
   │Replica │ │Replica │ │Replica │ ← reads
   │  1     │ │  2     │ │  3     │
   └────────┘ └────────┘ └────────┘
```

- **Primary** handles all writes
- **Replicas** handle reads
- Replicas are slightly behind (async replication)
- You can have many replicas

> **Most SaaS apps are read-heavy.** 10 reads per write. Replicas scale reads infinitely.

---

## Theory: Laravel's Read/Write Splitting

Laravel supports this natively.

`config/database.php`:

```php
'mysql' => [
    'driver' => 'mysql',
    'host' => env('DB_HOST', '127.0.0.1'),
    'database' => env('DB_DATABASE', 'coaching'),
    'username' => env('DB_USERNAME', 'coaching'),
    'password' => env('DB_PASSWORD', ''),
    
    'read' => [
        'host' => [
            'replica1.coaching-saas.com',
            'replica2.coaching-saas.com',
        ],
    ],
    
    'write' => [
        'host' => 'primary.coaching-saas.com',
    ],
    
    'sticky' => true,  // Read your own writes for the same request
],
```

Laravel automatically:
- Routes `INSERT`, `UPDATE`, `DELETE` to **write**
- Routes `SELECT` to **read** (randomly distributed across replicas)

> **`sticky: true`** means: if you just wrote, the next read goes to the primary. Prevents "I saved but the page shows the old value."

---

## Theory: When Replicas Aren't Enough

Read replicas help reads. They don't help **writes** or **big queries**.

### If writes are slow

- **Connection pooling** — PgBouncer (for Postgres) or ProxySQL (for MySQL)
- **Batch writes** — instead of 1000 inserts, batch them
- **Async writes** — queue them
- **Sharding** — split data across multiple primary servers

### If a single query is slow

- **Indexes** (often missing)
- **Query optimization** (N+1, full table scans)
- **Caching** (Redis for hot data)
- **Materialized views** (pre-computed aggregates)

### If data is too big

- **Archive old data** (move 2-year-old records to cold storage)
- **Vertical partitioning** (split wide tables — `users` vs `user_profiles`)
- **Sharding** — split across DBs by tenant

> **Senior rule:** replicas first, then query optimization, then caching, then sharding.** Sharding is the last resort. **It's operationally complex.**

---

## Theory: Connection Pooling

PHP creates a new DB connection per request (mostly). With high traffic, this adds up.

ProxySQL sits between your app and MySQL:

```
App → ProxySQL → MySQL
```

- Maintains a pool of connections
- App asks for connection, gets one from pool
- Returns when done

For Postgres, use PgBouncer.

> **Senior rule:** in PHP, connection pooling is less important** because each request is short. But for long-running workers (queue), it matters.

---

## Theory: Query Optimization (The First Win)

Before scaling infrastructure, fix slow queries.

### Find them

Enable query log in `config/database.php`:

```php
'mysql' => [
    // ...
    'options' => [
        PDO::MYSQL_ATTR_INIT_COMMAND => 'SET GLOBAL slow_query_log = 1, long_query_time = 0.5',
    ],
],
```

Or use Telescope (dev) / Sentry Performance (prod) / MySQL slow query log.

### Fix the common ones

**N+1:**

```php
// BAD
$students = Student::all();
foreach ($students as $student) {
    echo $student->class->name;  // N queries
}

// GOOD
$students = Student::with('class')->get();  // 1 query
```

**Missing index:**

```sql
-- Find slow queries
EXPLAIN SELECT * FROM students WHERE email = 'aisha@x.com';

-- Add index
ALTER TABLE students ADD INDEX (email);
```

**Full table scan:**

```sql
EXPLAIN SELECT * FROM students WHERE name LIKE '%aisha%';
-- Type: ALL  ← full scan
-- Fix: full-text index or search engine
```

**Select *:**

```php
// BAD
Student::all();  // gets every column

// GOOD
Student::select('id', 'name', 'email')->get();
```

---

## Theory: Caching (The Second Win)

Most queries are reads of data that doesn't change often.

```php
$totalStudents = Cache::remember("tenant:{$id}:total_students", 3600, function () use ($id) {
    return Student::where('tenant_id', $id)->count();
});
```

> **Cache aggressively for read-heavy, slow-to-compute, rarely-changing data.** Counts, aggregates, dashboard stats.

For the senior's rule on cache invalidation, see Lesson 36.

---

## Theory: Materialized Views (When Aggregates Are Slow)

```sql
CREATE MATERIALIZED VIEW daily_attendance_summary AS
SELECT 
    tenant_id,
    date,
    COUNT(*) FILTER (WHERE status = 'present') as present,
    COUNT(*) FILTER (WHERE status = 'absent') as absent
FROM attendances
GROUP BY tenant_id, date;

CREATE UNIQUE INDEX ON daily_attendance_summary (tenant_id, date);
```

Now the dashboard query is instant.

Refresh:

```php
\DB::statement('REFRESH MATERIALIZED VIEW daily_attendance_summary');
```

Schedule daily in a job.

> **MySQL doesn't have materialized views** natively. Use a summary table updated by scheduled jobs. **Senior's pattern.**

---

## Theory: Sharding (Last Resort)

Split data across multiple primary servers:

```
tenant 1-1000   → shard_1
tenant 1001-2000 → shard_2
tenant 2001-3000 → shard_3
```

> **Most multi-tenant SaaS never needs sharding.** With 5,000 tenants, you fit on one server. **Don't shard prematurely.**

When you do need it:

```php
public function getShardForTenant(int $tenantId): string
{
    $shardNumber = $tenantId % 4;  // 4 shards
    
    return "shard_{$shardNumber}";
}
```

> **Senior rule:** sharding is a 6-month project.** Plan it. Don't do it on Friday.

---

## Theory: Database Backups at Scale

For 100k+ records:

- **Daily full backup** (mysqldump or Percona XtraBackup)
- **Hourly incremental** (binary log replication)
- **Offsite backup** (S3, with encryption)
- **Test restore** monthly

```bash
# Full backup
mysqldump --single-transaction --quick --routines --triggers \
    coaching_prod > backup_$(date +%F).sql

# Upload to S3
aws s3 cp backup_*.sql s3://coaching-backups/db/

# Encrypt
gpg --symmetric --cipher-algo AES256 backup_*.sql
```

> **Senior rule:** test restore in staging, not prod.** Every 3 months. **If you've never restored, you don't have backups.**

---

## Theory: Point-in-Time Recovery (PITR)

For serious data loss, you need binary logs:

```sql
-- Enable
SET GLOBAL log_bin = ON;
SET GLOBAL binlog_format = 'ROW';
SET GLOBAL expire_logs_days = 7;
```

Restore to a specific time:

```bash
# Restore full backup
mysql coaching_prod < backup_2023-10-12_0300.sql

# Apply binlogs from 03:00 to 14:23 (the disaster time)
mysqlbinlog --stop-datetime='2023-10-12 14:23:00' \
    binlog.000001 binlog.000002 | mysql coaching_prod
```

> **PITR is complex.** AWS RDS does it for you with one click. **Senior chooses managed databases when PITR matters.**

---

## Theory: Connection Management in Laravel

PHP-FPM creates a new request, uses a connection, returns it. With high concurrency:

- `max_connections` in MySQL: 151 (default), 1000+ for production
- `wait_timeout`: 28800 seconds (8 hours)
- Laravel's persistent connections: avoid unless you know why

```php
// config/database.php
'mysql' => [
    // ...
    'options' => [
        PDO::ATTR_PERSISTENT => false,  // default; rarely change
        PDO::ATTR_TIMEOUT => 5,
    ],
],
```

> **Persistent connections in PHP are usually wrong** because PHP-FPM recycles workers.

---

## Theory: Slow Query Log Analysis

In `my.cnf`:

```ini
slow_query_log = 1
long_query_time = 1
log_slow_extra = ON
```

Restart MySQL. Slow queries go to `/var/log/mysql/slow.log`.

Analyze with `pt-query-digest` (Percona Toolkit):

```bash
pt-query-digest /var/log/mysql/slow.log > report.txt
```

Shows top 10 slowest queries, their frequency, total time.

> **Senior rule:** review slow query log weekly.** Catches problems before users do.

---

## Theory: Connection Limits

Each Laravel worker holds a DB connection. With 100 workers:

```
100 workers × 1 connection = 100 connections to MySQL
```

If MySQL has `max_connections=151`, you're almost out. With replicas, replicas have their own limits.

> **Senior rule:** monitor `Threads_connected` and `max_connections`.** If approaching, raise MySQL's limit or add a connection pooler.

---

## Theory: The Senior's Database Checklist

Before scaling, ensure:

- [ ] All foreign keys indexed
- [ ] All `WHERE` columns indexed
- [ ] N+1 queries eliminated
- [ ] Slow query log enabled and reviewed
- [ ] Caching for hot data
- [ ] Connection limits set
- [ ] Backups tested
- [ ] Replication lag monitored (when using replicas)
- [ ] Old data archived

If all checked and still slow, then consider replicas, then sharding.

---

## Theory: AWS RDS / DigitalOcean Managed MySQL

> **Senior's default: managed database.**

RDS handles:
- Automated backups
- PITR (point-in-time recovery)
- Read replicas (one click)
- Failover
- Patching
- Monitoring

For a coaching SaaS at scale:

```
$50-500/mo for primary
$25-200/mo per replica
```

Worth it because: **the senior sleeps at night.**

---

## Theory: When Each Fix Matters

| Problem | Solution |
|---|---|
| One slow query | Index, query rewrite |
| Dashboard slow | Cache the result |
| Reports time out | Materialized view / archive |
| High CPU | Read replicas |
| Disk full | Archive, then bigger disk |
| Writes slow | Batch, async, sharding |
| Latency from region | Read replica in region |
| Disaster recovery | Backups, PITR, replicas in other region |

---

## Why It Matters

The database is the **single most common performance bottleneck.** Most "scale problems" are database problems.

Get it right:
- Sub-100ms p95 query times
- 10k+ concurrent users
- 99.9% uptime
- Quick recovery from failures

Get it wrong:
- Customers wait 5 seconds for pages
- Site goes down at peak
- Lost data, lost trust

> **A senior's database strategy is: optimize first, cache second, replicate third, shard never (if possible).**

---

## Lab: Set Up a Production Database

### Step 1 — Enable slow query log

In MySQL config (or RDS parameter group):

```sql
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;
```

Or in `my.cnf`:

```ini
slow_query_log = 1
long_query_time = 1
log_output = TABLE
```

### Step 2 — Find slow queries

```sql
SELECT * FROM mysql.slow_log ORDER BY start_time DESC LIMIT 20;
```

Or via MySQL CLI:

```bash
mysqldumpslow /var/log/mysql/slow.log -t 20
```

### Step 3 — Add missing indexes

Common missing indexes for a coaching SaaS:

```php
// Migration
Schema::table('students', function (Blueprint $table) {
    $table->index('tenant_id');
    $table->index(['tenant_id', 'is_active']);
    $table->index(['tenant_id', 'class_id']);
    $table->unique(['tenant_id', 'email']);
    $table->index('student_id');
});

Schema::table('attendances', function (Blueprint $table) {
    $table->index(['tenant_id', 'date']);
    $table->index(['student_id', 'date']);
    $table->index(['date', 'status']);
});

Schema::table('payments', function (Blueprint $table) {
    $table->index(['tenant_id', 'paid_at']);
    $table->index(['student_id', 'paid_at']);
});
```

### Step 4 — Eliminate N+1 in your codebase

```bash
# Find N+1
grep -r "::all()" app/ | grep -v "select"
grep -r "::find" app/Http/Controllers/ | head
```

In controllers:

```php
// BAD
$students = Student::all();
return view('students.index', compact('students'));

// GOOD
$students = Student::with('class')->get();
```

### Step 5 — Cache the dashboard

`app/Http/Controllers/DashboardController.php`:

```php
public function index()
{
    $tenant = app('current_tenant');
    
    $stats = Cache::remember("dashboard:tenant:{$tenant->id}", 300, function () use ($tenant) {
        return [
            'students' => $tenant->students()->count(),
            'active' => $tenant->students()->where('is_active', true)->count(),
            'today_attendance' => $this->todayAttendance($tenant),
            'pending_fees' => $tenant->students()->withSum('feeInvoices', 'due')->get()->sum('fee_invoices_sum_due'),
        ];
    });
    
    return view('dashboard', compact('stats'));
}
```

### Step 6 — Set up read replicas (if needed)

In `config/database.php`:

```php
'mysql' => [
    'driver' => 'mysql',
    'host' => env('DB_HOST'),
    'database' => env('DB_DATABASE'),
    
    'read' => [
        'host' => explode(',', env('DB_READ_HOSTS', '')),
    ],
    
    'write' => [
        'host' => env('DB_WRITE_HOST', env('DB_HOST')),
    ],
    
    'sticky' => true,
],
```

`.env`:

```env
DB_HOST=primary.coaching-saas.com
DB_WRITE_HOST=primary.coaching-saas.com
DB_READ_HOSTS=replica1.coaching-saas.com,replica2.coaching-saas.com
```

### Step 7 — Set up automated backups

`/home/forge/backup.sh`:

```bash
#!/bin/bash
TIMESTAMP=$(date +%F_%H%M)
BACKUP_DIR=/home/forge/backups
mkdir -p $BACKUP_DIR

# Dump
mysqldump --single-transaction --quick --routines --triggers \
    coaching_prod > $BACKUP_DIR/coaching_$TIMESTAMP.sql

# Compress
gzip $BACKUP_DIR/coaching_$TIMESTAMP.sql

# Encrypt
gpg --symmetric --cipher-algo AES256 \
    --passphrase-file /home/forge/.backup_passphrase \
    $BACKUP_DIR/coaching_$TIMESTAMP.sql.gz

# Upload to S3
aws s3 cp $BACKUP_DIR/coaching_$TIMESTAMP.sql.gz.gpg \
    s3://coaching-backups/db/

# Keep only last 7 local backups
ls -t $BACKUP_DIR/*.gz.gpg | tail -n +8 | xargs rm -f
```

Cron:

```bash
0 3 * * * /home/forge/backup.sh
```

### Step 8 — Test the backup

```bash
# Download
aws s3 cp s3://coaching-backups/db/coaching_2023-10-12_030000.sql.gz.gpg /tmp/

# Decrypt
gpg --decrypt /tmp/coaching_2023-10-12_030000.sql.gz.gpg > /tmp/coaching.sql.gz

# Decompress
gunzip /tmp/coaching.sql.gz

# Restore to staging
mysql coaching_staging < /tmp/coaching.sql
```

> **Test quarterly.** If you can't restore, you don't have backups.

### Step 9 — Add a query performance dashboard

In Telescope, the Queries tab shows all queries with timing. Watch for:
- Queries > 100ms
- Same query repeated (N+1)
- Total query count per request > 20

### Step 10 — Set up connection monitoring

`app/Console/Commands/DbMonitor.php`:

```php
public function handle()
{
    $connections = DB::select('SHOW STATUS LIKE "Threads_connected"');
    $maxConnections = DB::select('SHOW VARIABLES LIKE "max_connections"');
    
    $current = (int) $connections[0]->Value;
    $max = (int) $maxConnections[0]->Value;
    $percent = ($current / $max) * 100;
    
    if ($percent > 80) {
        Log::warning('db.connections.high', [
            'current' => $current,
            'max' => $max,
            'percent' => $percent,
        ]);
    }
    
    $this->info("DB connections: $current / $max ($percent%)");
}
```

Schedule hourly.

### Step 11 — Add a database health check

`/api/health/db`:

```php
public function db()
{
    try {
        $start = microtime(true);
        DB::select('SELECT 1');
        $latency = (microtime(true) - $start) * 1000;
        
        $healthy = $latency < 100;
        
        return response()->json([
            'healthy' => $healthy,
            'latency_ms' => $latency,
        ], $healthy ? 200 : 503);
    } catch (\Exception $e) {
        return response()->json(['healthy' => false, 'error' => $e->getMessage()], 503);
    }
}
```

> **Uptime monitor pings this every 30 seconds.**

### Step 12 — Archive old data

For a coaching SaaS, after 2 years, attendance data is rarely accessed.

`app/Console/Commands/ArchiveOldData.php`:

```php
public function handle()
{
    $cutoff = now()->subYears(2);
    
    $count = DB::table('attendances')
        ->where('date', '<', $cutoff)
        ->update(['archived' => true]);
    
    $this->info("Archived {$count} old attendance records");
}
```

Or move to a separate table:

```sql
CREATE TABLE attendances_archive LIKE attendances;
INSERT INTO attendances_archive SELECT * FROM attendances WHERE date < '2022-01-01';
DELETE FROM attendances WHERE date < '2022-01-01';
```

---

## Quiz

**Q1.** Your dashboard is slow. Walk through the debugging steps. What's the FIRST thing you check?

**Q2.** You add a read replica. The user's "I just saved" page shows old data. Why? How do you fix it?

**Q3.** A junior says "let's shard." You say "no, let's add indexes first." When is sharding actually necessary?

**Q4.** You have 200 connections to MySQL. `max_connections=200`. The site starts returning "too many connections." What's the fix? List 3 options.

**Q5.** (Stretch) Your SaaS is growing fast. You predict 100k tenants in 2 years. Walk through the database scaling roadmap from now to then.

---

## What You Now Have

✅ Read/write splitting with Laravel
✅ Query optimization (N+1, indexes, EXPLAIN)
✅ Caching for hot data
✅ Connection management
✅ Backup strategy
✅ Monitoring and alerting
✅ Scaling decision framework

## What's Next

Lesson 49 covers **horizontal scaling and when you need Kubernetes** — the patterns for running multiple app servers, load balancers, and the platform decisions that come with growth.

📖 **[Lesson 49 — Horizontal Scaling and When You Need K8s](49-horizontal-scaling-and-when-you-need-k8s.md)**
