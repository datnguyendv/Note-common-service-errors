# MongoDB Troubleshooting Guide: Standalone, Replica Set, Sharded Cluster

---

## 1. Standalone Deployment

**Typical Use:** Development, testing, single-node PoC systems.

### Common Incidents

#### 1.1 MongoDB Process Down

- **Symptoms:** Application cannot connect, "connection refused".
- **Actions:**

  ```
  systemctl status mongod
  tail -f /var/log/mongodb/mongod.log
  ps aux | grep mongod
  ```

#### 1.2 Out of Disk Space

- **Symptoms:** Write errors, "No space left on device".
- **Actions:**

  ```
  df -h
  du -sh /var/lib/mongodb/*
  rm -rf /var/log/mongodb/*.log-old    # Be careful when removing!
  ```

#### 1.3 Authentication or Permission Issues

- **Symptoms:** "Authentication failed" errors, cannot connect.
- **Actions:**

  ```
  mongo -u admin -p --authenticationDatabase admin
  grep "security:" /etc/mongod.conf
  ls -l /var/lib/mongodb/
  chown -R mongodb:mongodb /var/lib/mongodb
  ```

#### 1.4 Data Corruption, mongod Won't Start

- **Symptoms:** mongod process exits, "corruption" in log.
- **Actions:**

  ```
  rm /var/lib/mongodb/mongod.lock
  mongod --repair --dbpath /var/lib/mongodb
  ```

---

## 2. Replica Set Deployment

**Typical Use:** Production environments requiring high availability.

### Common Incidents

#### 2.1 Primary Election Problems

- **Symptoms:** No primary present, writes rejected.
- **Actions:**

  ```
  mongo --eval "rs.status()"
  mongo --eval "rs.isMaster()"
  ```

#### 2.2 Secondary Node Outage or Lag

- **Symptoms:** Secondary node missing or lagging.
- **Actions:**

  ```
  mongo --eval "rs.status()"
  mongo --eval "rs.printSlaveReplicationInfo()"
  ```

#### 2.3 Replication Broken / Rollback

- **Symptoms:** Replication errors, inconsistent data after downtime.
- **Actions:**

  ```
  mongo --eval "rs.status()"
  # Re-sync failed secondary if needed
  ```

#### 2.4 Oplog or Disk Full

- **Symptoms:** Replication stops, out of sync.
- **Actions:**

  ```
  df -h
  mongo --eval "db.getReplicationInfo()"
  ```

#### 2.5 Split Brain (Partitioned Replica Set)

- **Symptoms:** Multiple primaries, or long periods with no primary.
- **Actions:**

  ```
  mongo --eval "rs.status()"
  # After resolving partition
  mongo --eval "rs.reconfig(cfg, {force: true})"
  ```

---

## 3. Sharded Cluster Deployment

**Typical Use:** Large scale, high throughput, horizontal scalability.

### Architecture

```
[mongos] --> [config servers (CSRS)] --> [sharded replica sets]
```

### Common Incidents

#### 3.1 Config Servers Down

- **Symptoms:** Sharding/metadata operations fail.
- **Checks:**

  ```
  mongo --port 27019 --eval "rs.status()"
  tail -f /var/log/mongodb/configsvr.log
  mongo --port 27017 --eval "sh.status()"
  ```

- **Actions:** Restore majority, force reconfig if needed.

#### 3.2 Mongos Router Failure

- **Symptoms:** Cannot connect, "no primary found".
- **Actions:**

  ```
  systemctl status mongos
  mongo --port 27017 --eval "db.adminCommand('ismaster')"
  ```

#### 3.3 Shard or Replica Set Down

- **Symptoms:** Data is unavailable, chunk errors in `sh.status()`.
- **Checks:**

  ```
  mongo --port 27018 --eval "rs.status()"
  mongo --port 27017 --eval "sh.status()"
  ```

#### 3.4 Balancer and Chunk Issues

- **Symptoms:** Imbalanced data, chunk move failures.
- **Checks:**

  ```
  mongo --port 27017 --eval "sh.getBalancerState()"
  mongo --port 27017 --eval "db.getSiblingDB('config').chunks.aggregate([{ $group: {_id: '$shard', count: {$sum: 1}} }, { $sort: {count: -1} }])"
  mongo --port 27017 config --eval "db.chunks.find({jumbo: true})"
  ```

#### 3.5 Cross-Shard Query or Index Issues

- **Symptoms:** Slow aggregations, cluster performance drops.
- **Actions:** Review indexes and queries, ensure they use the shard key.

---

## 4. Reference: Essential Health Check Bash Scripts

### 4.1 Standalone

```
systemctl status mongod
mongo --eval "db.runCommand({serverStatus:1})"
df -h
free -h
```

### 4.2 Replica Set

```
mongo --eval "rs.status()"
mongo --eval "rs.printSlaveReplicationInfo()"
df -h
```

### 4.3 Sharded Cluster

```
mongo --port 27017 --eval "sh.status()"
mongo --port 27017 --eval "sh.getBalancerState()"
mongo --port 27019 --eval "rs.status()" # Config server status
for SHARD in shard1-primary:27018 shard2-primary:27018; do
mongo $SHARD --eval "rs.status()"
done
```

---

## 5. Key Monitoring & Alerting Points

- **Standalone:** mongod down, disk/memory full, slow queries.
- **Replica Set:** Primary lost, lagging secondary, oplog/disk full, node unreachable.
- **Sharded Cluster:** Config server unavailable, chunk imbalance, mongos down, balancer off, jumbo chunk presence.

---

**Tip:** Automate health checks and failover testing. Regular monitoring and documented runbooks prevent most emergencies!
