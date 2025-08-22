# Common Incidents When Self-Hosting PostgreSQL (All Deployment Modes)

This document summarizes the most frequent incidents encountered when self-hosting PostgreSQL, including standalone, replication, and clustered deployments (e.g., Patroni, Citus). For each incident, practical checks and remediation steps for SRE/DevOps are highlighted.

---

## 1. PostgreSQL Fails to Start

**Common Causes:**

- Incorrect file permissions or ownership (`/var/lib/postgresql`, data directory)
- Disk/full or inode exhaustion
- Syntax errors in `postgresql.conf` or `pg_hba.conf`
- Data corruption (log: "PANIC: could not locate...")

**How to Check & Fix:**

- Check logs

```
tail -n 50 /var/log/postgresql/postgresql-*.log
journalctl -u postgresql
```

- Inspect directory permissions

```
ls -ld /var/lib/postgresql/<version>/main
chown -R postgres:postgres /var/lib/postgresql/<version>/main
```

- Check disk space and inodes

```
df -h
df -i
```

- Test config file validity

```
postgres -D /var/lib/postgresql/<version>/main --check
```

- Fix permissions, clean up disk, correct configurations.
- If data corruption: use `pg_resetwal` (risk of data loss), recover from backup whenever possible.

---

## 2. Out of Connections

**Symptoms:**  
Log: "FATAL: remaining connection slots are reserved..."

**Causes:**

- Application leaks connections or unoptimized pooler
- No connection pooler (PgBouncer) in high-concurrency setups
- `max_connections` too low

**Checks & Solutions:**

```
SELECT * FROM pg_stat_activity WHERE state != 'idle';
SELECT max_connections FROM pg_settings;

-- Increase limit, requires restart:
ALTER SYSTEM SET max_connections = 200;

-- Terminate unnecessary connections (with caution):
SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE <condition>;
```

- Review pooling strategy, tune application and pooler setup.

---

## 3. Replication Lag or Failover Issues (Replication/Cluster Modes)

**Causes:**

- Network partition/latency
- Replication slot or WAL retention misconfigured
- Standby too slow to catch up (hardware, config bottleneck)

**Checks & Fix:**

```
SELECT * FROM pg_stat_replication;
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;
```

- Monitor disk I/O (`iostat -x -d 1`)
- For clusters (Patroni/Etcd): check health, logs, and role switching.
- Review slot and WAL configuration, manual switchover if needed.

---

## 4. Query Deadlock or Transaction Block

**Symptoms:**  
Queries hang; logs report "deadlock detected".

**How to Check:**

```
SELECT * FROM pg_locks WHERE granted = 'f';
SELECT pid, state, query FROM pg_stat_activity WHERE state != 'idle';
```

**Solution:**

- Kill stuck queries/transactions
- Review application logic on transaction boundaries and locking
- Set up blocking/lock alerts

---

## 5. Disk Full (WAL or Data Volumes)

**Symptoms:**  
Log: "No space left on device"

**Solution:**

- Clean up old WAL/archive, run full checkpoint, extend volume.
- Validate WAL archiving/retention; be careful in streaming setups.

```
du -sh /var/lib/postgresql
rm -rf <old_backup>
```

---

## 6. Data or Index Corruption

**Symptoms:**  
"PANIC" errors, "could not read block..." in logs.

**Remediation:**

- Stop app, dump/restore if possible
- Use `reindex` for index corruption
- Restore from backup if physical corruption

---

## 7. Performance Degradation (Slow Queries)

**Causes:**

- Missing indexes, large/bloated tables
- Vacuum/analyze not running frequently enough
- Long-running or idle-in-transaction blocking

**What to Do:**

```
VACUUM (VERBOSE, ANALYZE);
REINDEX DATABASE mydb;
EXPLAIN (ANALYZE, BUFFERS) <your query>;
```

- Tune autovacuum, optimize queries, monitor with Prometheus + Grafana.

---

## 8. Configuration Drift (Clustered Deployments)

**Causes:**  
Unsynchronized configs, version mismatch, DCS leader issues.

**How to Check:**

```
patronictl list
```

- Ensure uniform versions and configs
- Monitor DCS quorum and cluster state

---

## 9. WAL Archiving Failure

**Symptoms:**  
Log: "archive command failed", PITR/backup not working.

**Fix:**

- Check archive directory permissions and archiving scripts
- Test archive_command manually
- Set up alerts for WAL archiving backlog

---

## 10. Time/Clock Drift

**Symptoms:**  
Replication errors, out-of-sync timestamps, cluster shutdowns (Patroni).

**Remediation:**

```
systemctl status chronyd
timedatectl
```

- Enforce NTP/chrony on all nodes
- Alert on clock skew >5-10 seconds

---

## 11. Network Partition / Split-Brain (HA Clusters)

**Symptoms:**  
Multiple primaries (split-brain), writes to out-of-cluster nodes.

**Checks & Fix:**

- Validate DCS health (Etcd, Consul)
- Review fencing and failover safety settings
- Monitor quorum and network health

---

## 12. File System Corruption or Volume Loss

**Symptoms:**  
Random I/O errors, unexpected crashes.

**What to Do:**

- Check kernel logs (`dmesg`, `/var/log/syslog`)
- Use another system/volume to test backups/restores
- Plan for fast disaster recovery

---

## 13. OS Resource Exhaustion (CPU, Memory)

**Symptoms:**  
High CPU/memory, OOM killer stops PostgreSQL, slow queries.

**How to Check:**

```
top
htop
free -m
dmesg | grep -i kill
```

- Tune PostgreSQL memory params, add resource limits, analyze queries and app load

---

## 14. Security/Permission Issues

**Symptoms:**  
Connection errors, authentication failures.

**Checks:**

- Revalidate `pg_hba.conf`
- Audit password expiry and user management
- Monitor for suspicious logins

---

## 15. Backup Loss or Inconsistency

**Causes:**  
Unverified backups, incomplete WAL, or unsafe disk snapshots.

**Prevention:**

- Regularly test restores
- Use validated backup tools (`pgbackrest`, `barman`)
- Always verify successful backup jobs

---

## 16. OS Kernel or Package Incompatibility

**Symptoms:**  
Crashes after OS or extension upgrades.

**How to Check:**

- Review update logs
- Validate compatibility before/after upgrades
- Rebuild extensions if needed

---

## Recommendations & Best Practices

- Schedule regular health checks and failover drills
- Document and automate backup and restore playbooks
- Always verify failover, restore, and disaster recovery procedures under realistic scenarios
