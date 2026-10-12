# Lesson 51 — Backups, Disaster Recovery, RPO/RTO

> **Phase 6: Deploy & Scale** | You'll be here: ~25 min

## The Senior's Question

3 AM. PagerDuty. Your database is corrupted. You have 100 paying customers. "When was the last backup?" The junior freezes. **The senior already knows the answer, the restore time, and has tested it.** Let me show you the discipline.

---

## Theory: The Disaster Scenarios

Things that can go wrong:

1. **Server dies** — hardware failure
2. **Database corruption** — disk full, MySQL crash
3. **Accidental deletion** — `DROP TABLE` or wrong `DELETE`
4. **Bad migration** — schema change that breaks production
5. **Hacking** — ransomware, data exfiltration
6. **Region failure** — AWS us-east-1 goes down
7. **Provider bankruptcy** — your hosting company dies
8. **Bad code in prod** — deploys something that destroys data

> **Senior rule:** all of these will happen eventually. The question is: are you ready?**

---

## Theory: The Two Numbers You Must Know

### RPO — Recovery Point Objective

**How much data can you lose?** Measured in time.

```
If your last backup was 6 hours ago, RPO = 6 hours
You'll lose up to 6 hours of data.
```

- RPO of 1 hour = hourly backups
- RPO of 24 hours = daily backups
- RPO of 5 minutes = real-time replication

> **For a coaching SaaS, RPO of 1 hour is acceptable.** Losing an hour of attendance data is bad. Losing a day is catastrophic.

### RTO — Recovery Time Objective

**How fast can you be back online?** Measured in time.

```
If restore takes 4 hours, RTO = 4 hours
For 4 hours, your service is down.
```

- RTO of 1 hour = automated failover
- RTO of 4 hours = manual restore from backup
- RTO of 24 hours = rebuild from scratch

> **For a coaching SaaS, RTO of 4 hours is acceptable.** Total outage for 4 hours. Anything longer = churn.**

---

## Theory: The Backup Rule (3-2-1)

```
3 copies of your data
2 different media
1 offsite
```

Example:

1. **Production database** (live data)
2. **Daily backup on server** (local)
3. **Encrypted backup in S3** (offsite)

> **Senior rule:** never rely on one backup. Ever.** Hard drives fail. Cloud accounts get locked. **Have multiple, in multiple places.**

---

## Theory: Database Backup Strategies

### 1. `mysqldump` (logical)

```bash
mysqldump --single-transaction --quick --routines --triggers \
    --events --hex-blob \
    coaching_prod > backup_$(date +%F_%H%M).sql
```

- **Pros:** simple, portable, can restore to any version
- **Cons:** slow on large DBs (1TB+), locks during long transactions

### 2. `Percona XtraBackup` (physical)

```bash
xtrabackup --backup --target-dir=/backups/$(date +%F) --user=root --password=xxx
```

- **Pros:** fast, no locking, hot backup
- **Cons:** more complex setup, MySQL-specific

### 3. Managed snapshots (RDS, DigitalOcean)

Click a button, get a snapshot. **Senior's default.**

```bash
# DigitalOcean
doctl databases snapshot create coaching-prod --snapshot-name daily-$(date +%F)
```

- **Pros:** zero effort, automated
- **Cons:** tied to provider

### 4. Binary log replication (PITR)

```sql
SET GLOBAL binlog_format = 'ROW';
SET GLOBAL expire_logs_days = 7;
```

> **Binary logs** record every change. Combined with daily backups, you can restore to any second.

**RDS does this automatically.**

---

## Theory: What to Back Up

| Data | Backup strategy | Why |
|---|---|---|
| Database | Daily, encrypted, offsite | Most critical |
| User uploads (S3) | S3 versioning + cross-region replication | Customer data |
| `.env` file | Encrypted password manager | Need to restore config |
| Application code | Git (already backed up) | Code is in repo |
| Logs | Don't back up, archive to S3 | Operational, not critical |
| Redis data | Optional, can rebuild | Cache, not source of truth |

> **The senior's priorities:** database (critical), uploads (important), config (essential). Everything else is recoverable from code + DB.**

---

## Theory: Backup Schedule

For RPO of 1 hour:

```bash
# Every hour (binary logs, for PITR)
0 * * * * /scripts/rotate-binlogs.sh

# Daily at 3 AM (full backup)
0 3 * * * /scripts/full-backup.sh

# Weekly verification
0 4 * * 0 /scripts/verify-backups.sh
```

> **Always encrypt backups before uploading.** A leaked backup = leaked customer data.

```bash
gpg --symmetric --cipher-algo AES256 \
    --passphrase-file /root/.backup_passphrase \
    backup.sql
```

---

## Theory: The Restore Test

> **Senior rule:** test restore quarterly. If you've never restored, you don't have backups.**

```bash
# 1. Download latest backup
aws s3 cp s3://coaching-backups/db/latest.gpg /tmp/

# 2. Decrypt
gpg --decrypt /tmp/latest.gpg > /tmp/backup.sql

# 3. Create a fresh database
mysql -e "CREATE DATABASE coaching_restore_test;"

# 4. Restore
mysql coaching_restore_test < /tmp/backup.sql

# 5. Verify
mysql coaching_restore_test -e "SELECT COUNT(*) FROM users;"

# 6. Compare row counts to production
mysql coaching_prod -e "SELECT COUNT(*) FROM users;"
# Should match (or be very close, depending on time elapsed)

# 7. Drop the test database
mysql -e "DROP DATABASE coaching_restore_test;"
```

> **Schedule this.** Quarterly. With a calendar reminder. **Or it won't happen.**

---

## Theory: Application Data Recovery

Beyond the database, you need to back up:

### `.env` file

This isn't in your repo. Store in:
- **1Password / Bitwarden** — secure note
- **AWS Secrets Manager** — programmatic access
- **LastPass / KeePass** — if you must

### SSL certificates

If using Let's Encrypt, Forge reissues them. Manual? Save a copy.

### Cron jobs

```bash
crontab -l > /backups/crontab-$(date +%F).txt
```

### Server config

Nginx, supervisor, etc. — version-controlled or backed up to S3.

> **A senior's "recovery kit":** the database backup, the `.env` values, the deploy script, and the exact server config. With these, you can rebuild a working server in 4 hours.**

---

## Theory: Disaster Recovery Plan

Document it. In a file everyone can access.

`DISASTER-RECOVERY.md`:

```markdown
# Disaster Recovery Plan

## Scenarios

### Database corruption
1. Stop application (prevent further writes)
2. Restore latest backup
3. Apply binary logs (if available)
4. Verify data integrity
5. Restart application
6. RTO: 4 hours, RPO: 1 hour

### Server dies
1. Provision new server (DO API or manual)
2. Install dependencies (Laravel Forge, Ansible, or manual)
3. Deploy application
4. Restore database from backup
5. Point DNS at new server (or load balancer)
6. RTO: 4 hours, RPO: 1 hour

### Bad deploy
1. Identify the bad commit
2. Roll back to previous release (Forge: click "Rollback")
3. If DB migration is involved, run `migrate:rollback`
4. Verify application works
5. Investigate root cause
6. RTO: 30 minutes, RPO: 0 (no data loss)

### Ransomware / hack
1. Take affected servers offline
2. Provision fresh servers
3. Restore from clean backup
4. Rotate all credentials (DB, AWS, Stripe, etc.)
5. Notify customers (GDPR requirement: within 72 hours)
6. RTO: 24 hours, RPO: depends on backup age
```

> **Senior rule:** write the disaster plan when things are calm.** During a crisis, you can't think straight. **The plan must be ready.**

---

## Theory: Multi-Region Backup

For real disasters (region down), back up to a different region.

```bash
# Daily, copy to a different region
aws s3 cp s3://coaching-backups-us-east-1/latest.gpg \
         s3://coaching-backups-eu-west-1/latest.gpg
```

> **Most coaching SaaS apps don't need this.** But if your customers are in EU and your server is in US, EU region backup is GDPR-friendly.

---

## Theory: Off-Site Backup Providers

- **AWS S3** — gold standard, $0.023/GB/mo
- **Backblaze B2** — $0.005/GB/mo, 10x cheaper
- **Cloudflare R2** — $0.015/GB/mo, free egress
- **Wasabi** — $0.007/GB/mo, no egress fees

> **Senior default:** S3 for AWS, Backblaze for everything else. Encrypted, versioned, lifecycle-managed.

---

## Theory: S3 Backup Bucket Setup

```bash
# Create bucket
aws s3 mb s3://coaching-backups

# Enable versioning
aws s3api put-bucket-versioning \
    --bucket coaching-backups \
    --versioning-configuration Status=Enabled

# Lifecycle: delete old backups after 90 days
aws s3api put-bucket-lifecycle-configuration \
    --bucket coaching-backups \
    --lifecycle-configuration '{
        "Rules": [{
            "Id": "DeleteOldBackups",
            "Status": "Enabled",
            "Prefix": "",
            "Expiration": {"Days": 90}
        }]
    }'

# Server-side encryption
aws s3api put-bucket-encryption \
    --bucket coaching-backups \
    --server-side-encryption-configuration '{
        "Rules": [{
            "ApplyServerSideEncryptionByDefault": {
                "SSEAlgorithm": "AES256"
            }
        }]
    }'
```

> **Encrypted at rest, versioned, lifecycle-managed. Senior's default.**

---

## Theory: The Backup Monitoring

A backup that fails silently is worse than no backup. Monitor them.

`app/Console/Commands/VerifyBackups.php`:

```php
public function handle()
{
    $latest = \Storage::disk('s3')->lastModified('db/latest.gpg');
    
    if (!$latest) {
        Log::critical('backup.missing');
        return 1;
    }
    
    $age = now()->diffInHours($latest);
    
    if ($age > 26) {  // more than 26 hours since last backup
        Log::critical('backup.stale', ['age_hours' => $age]);
        return 1;
    }
    
    $this->info("Latest backup: {$age} hours old");
    return 0;
}
```

Schedule daily. Alert on failure.

---

## Theory: The First Time Disaster Hits

When something goes wrong (it will):

```
1. Take a breath
2. Assess: what exactly is broken? What data is at risk?
3. Communicate: tell the team / customers
4. Stabilize: stop the bleeding (rollback, take offline, etc.)
5. Recover: restore from backup, rebuild, etc.
6. Verify: is everything working? Did you lose data?
7. Communicate: tell customers what happened
8. Postmortem: write up what happened and how to prevent it
```

> **Senior rule:** don't panic, don't lie, don't hide.** Customers forgive downtime. They don't forgive dishonesty. **Tell them what happened and what you're doing about it.**

---

## Theory: The Postmortem (After Every Incident)

```markdown
# Incident Postmortem: Database Outage 2023-10-12

## Summary
Database crashed at 14:23 UTC due to disk full. Service down for 2 hours.

## Timeline
- 14:23: Database crashed
- 14:25: PagerDuty alert
- 14:30: On-call engineer investigating
- 14:45: Disk full identified
- 15:00: Cleared logs, restarted MySQL
- 15:30: Verified all data intact
- 16:00: Back to normal

## Root cause
Logs filled the disk. Disk monitoring wasn't set up.

## Impact
- 2 hours of downtime
- 100 customers affected
- No data loss

## Action items
- [ ] Set up disk monitoring (Owner: alice, Due: 2023-10-15)
- [ ] Add log rotation (Owner: bob, Due: 2023-10-15)
- [ ] Document recovery process (Owner: alice, Due: 2023-10-20)
```

> **Blameless postmortem.** Focus on systems, not people. **What failed, why, and how to prevent it next time.**

---

## Why It Matters

Disasters are **when, not if.** The question isn't whether you'll have one. It's whether you'll be ready.

Get it right:
- 4 hours RTO, 1 hour RPO
- Tested backups
- Documented recovery
- Calm execution

Get it wrong:
- Days of downtime
- Lost data
- Customer churn
- Career damage

> **A senior's disaster plan is written, tested, and calm.** A junior's is "we'll figure it out." **That's the difference between 4 hours and 4 days.**

---

## Lab: Set Up a Complete Backup Strategy

### Step 1 — Set up backup script

`/home/forge/backup.sh`:

```bash
#!/bin/bash
set -e

TIMESTAMP=$(date +%F_%H%M%S)
BACKUP_DIR=/home/forge/backups
S3_BUCKET=s3://coaching-backups/db
DB_NAME=coaching_prod

mkdir -p $BACKUP_DIR

# Dump
echo "Dumping database..."
mysqldump --single-transaction --quick --routines --triggers \
    --events --hex-blob $DB_NAME | gzip > $BACKUP_DIR/${DB_NAME}_${TIMESTAMP}.sql.gz

# Encrypt
echo "Encrypting..."
gpg --batch --yes --symmetric --cipher-algo AES256 \
    --passphrase-file /home/forge/.backup_passphrase \
    $BACKUP_DIR/${DB_NAME}_${TIMESTAMP}.sql.gz

# Upload
echo "Uploading..."
aws s3 cp $BACKUP_DIR/${DB_NAME}_${TIMESTAMP}.sql.gz.gpg \
    $S3_BUCKET/${DB_NAME}_${TIMESTAMP}.sql.gz.gpg \
    --storage-class STANDARD_IA

# Cleanup local (keep 7 days)
find $BACKUP_DIR -name "*.sql.gz.gpg" -mtime +7 -delete

# Cleanup S3 (keep 90 days)
aws s3 ls $S3_BUCKET/ | while read -r line; do
    fileDate=$(echo $line | awk '{print $1}')
    fileName=$(echo $line | awk '{print $4}')
    if [ -n "$fileName" ]; then
        if [[ $(date -d "$fileDate" +%s) -lt $(date -d "90 days ago" +%s) ]]; then
            aws s3 rm $S3_BUCKET/$fileName
        fi
    fi
done

echo "Backup complete: ${DB_NAME}_${TIMESTAMP}.sql.gz.gpg"
```

```bash
chmod +x /home/forge/backup.sh
```

### Step 2 — Generate the passphrase

```bash
# Generate a strong passphrase
openssl rand -base64 32 > /home/forge/.backup_passphrase
chmod 600 /home/forge/.backup_passphrase
```

> **Store this passphrase in your password manager.** Lose it = lose access to backups.

### Step 3 — Schedule the backup

```bash
crontab -e
# Add:
0 3 * * * /home/forge/backup.sh >> /home/forge/backup.log 2>&1
```

> **Daily at 3 AM, when traffic is lowest.**

### Step 4 — Set up S3 bucket

```bash
# Create bucket
aws s3 mb s3://coaching-backups

# Enable versioning
aws s3api put-bucket-versioning --bucket coaching-backups --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption --bucket coaching-backups --server-side-encryption-configuration '{
    "Rules": [{
        "ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}
    }]
}'
```

### Step 5 — Test the backup

```bash
/home/forge/backup.sh
# Watch the output

# Verify in S3
aws s3 ls s3://coaching-backups/db/
```

### Step 6 — Test the restore

```bash
# Download
aws s3 cp s3://coaching-backups/db/latest.gpg /tmp/

# Decrypt
gpg --decrypt /tmp/latest.gpg > /tmp/backup.sql.gz

# Decompress
gunzip /tmp/backup.sql.gz

# Verify
mysql -e "CREATE DATABASE coaching_restore_test;"
mysql coaching_restore_test < /tmp/backup.sql
mysql coaching_restore_test -e "SELECT COUNT(*) FROM users;"

# Cleanup
mysql -e "DROP DATABASE coaching_restore_test;"
```

> **Quarterly. Calendar reminder. Non-negotiable.**

### Step 7 — Document the recovery

`RECOVERY.md`:

```markdown
# How to Restore from Backup

## Latest daily backup
\`\`\`bash
# 1. Download latest
aws s3 cp s3://coaching-backups/db/$(aws s3 ls s3://coaching-backups/db/ | sort | tail -n 1 | awk '{print $4}') /tmp/latest.gpg

# 2. Decrypt (need passphrase from 1Password)
gpg --decrypt /tmp/latest.gpg > /tmp/backup.sql.gz

# 3. Decompress
gunzip /tmp/backup.sql.gz

# 4. Restore (CAUTION: overwrites current data)
mysql coaching_prod < /tmp/backup.sql

# 5. Verify
php artisan tinker
>>> User::count()
\`\`\`

## Time required: 30-60 minutes
```

### Step 8 — Set up backup monitoring

`app/Console/Commands/VerifyBackups.php`:

```php
public function handle()
{
    $files = \Storage::disk('s3')->files('db/');
    
    if (empty($files)) {
        Log::critical('backup.missing');
        $this->error('No backups found!');
        return 1;
    }
    
    $latest = collect($files)
        ->map(fn($f) => ['file' => $f, 'time' => \Storage::disk('s3')->lastModified($f)])
        ->sortByDesc('time')
        ->first();
    
    $age = now()->diffInHours(\Carbon\Carbon::createFromTimestamp($latest['time']));
    
    if ($age > 26) {
        Log::critical('backup.stale', [
            'age_hours' => $age,
            'file' => $latest['file'],
        ]);
        $this->error("Latest backup is {$age} hours old!");
        return 1;
    }
    
    $this->info("OK: Latest backup is {$age} hours old ({$latest['file']})");
    return 0;
}
```

Schedule:

```php
$schedule->command('verify:backups')->daily();
```

### Step 9 — Set up disk monitoring

In `bootstrap/app.php` or health check:

```php
public function disk()
{
    $free = disk_free_space('/');
    $total = disk_total_space('/');
    $used_percent = 100 - ($free / $total * 100);
    
    $healthy = $used_percent < 85;
    
    return response()->json([
        'healthy' => $healthy,
        'used_percent' => round($used_percent, 2),
    ], $healthy ? 200 : 503);
}
```

> **Uptime monitor pings this.** If disk > 85%, you get alerted.

### Step 10 — Document a DR drill

Schedule a quarterly DR drill:

```markdown
# Q4 2023 Disaster Recovery Drill

## What we test
- Can we restore the production database from S3?
- Can we rebuild the app server from scratch?
- Can we recover with 1 hour of data loss?

## How
1. Pick a Saturday morning
2. Provision a fresh staging server
3. Restore latest backup
4. Deploy the application
5. Verify the data
6. Document any issues

## Time taken: __ hours
## Issues found: __
## Follow-up actions: __
```

> **Run the drill. Document it. Quarterly.**

---

## Quiz

**Q1.** Define RPO and RTO. For a coaching SaaS, what's reasonable for each?

**Q2.** You back up daily at 3 AM. Database corrupts at 2 PM. How much data do you lose? How do you reduce this?

**Q3.** You test the backup, restore works. Three months later, you need to restore. The backup is corrupt. What went wrong?

**Q4.** Your hosting provider goes bankrupt. You have S3 backups, but your server is gone. Walk through recovery.

**Q5.** (Stretch) A junior deletes a production table by accident at 2 PM. Backups are daily. Walk through your response.

---

## What You Now Have

✅ RPO and RTO defined
✅ Automated daily backups
✅ Encrypted, offsite, versioned
✅ Restore tested quarterly
✅ Recovery documented
✅ Postmortem template
✅ Monitoring for backup failures

## What's Next

Lesson 52 covers **the 72-hour launch checklist** — the senior's last-72-hours-before-launch runbook. Everything you do in the final 3 days before going live with real customers.

📖 **[Lesson 52 — The 72-Hour Launch Checklist](52-the-72-hour-launch-checklist.md)**
