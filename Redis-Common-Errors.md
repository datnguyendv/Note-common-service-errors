# Common Redis Issues (Standalone, Master-Slave, Cluster): Troubleshooting, Commands, and Fixes

This guide describes frequent Redis errors, how to troubleshoot and resolve them, and includes relevant CLI commands for SRE/DevOps cases.

---

## 1. Redis Standalone

### 1.1. Unable to Connect to Redis Server

- **Causes:** Server not running, wrong port, firewall, wrong bind config.
- **How to Check (Commands):**

  - Check process:

    ```
    ps aux | grep redis
    ```

  - Check open port:

    ```
    netstat -ntlp | grep 6379
    ```

  - Examine config:

    ```
    grep -E "bind|protected-mode|port" /etc/redis/redis.conf
    ```

  - Attempt to connect:

    ```
    redis-cli -h 127.0.0.1 -p 6379 ping
    ```

- **Fix:**

  - Start/Restart Redis:

    ```
    sudo systemctl start redis
    sudo systemctl restart redis
    ```

  - Edit `redis.conf` for appropriate `bind`/`port`.
  - Adjust firewall:

    ```
    sudo ufw allow 6379
    ```

---

### 1.2. Out of Memory (OOM)

- **How to Check:**

  - Logs:

    ```
    tail -f /var/log/redis/redis-server.log
    ```

  - Memory info:

    ```
    redis-cli info memory
    free -m
    ```

- **Fix:**

  - Update eviction policy in `redis.conf`, e.g.

    ```
    maxmemory-policy allkeys-lru
    ```

---

### 1.3. Persistence Errors (RDB/AOF failed)

- **How to Check:**

  - Disk usage:

    ```
    df -h
    ```

  - Check permissions:

    ```
    ls -l /var/lib/redis/
    ```

  - Logs for errors:

    ```
    tail -n 50 /var/log/redis/redis-server.log
    ```

- **Fix:**
  - Free up disk space or correct directory permissions.

---

### 1.4. AOF Failures and Repair

- **Symptoms:** Can't start, log shows `AOF file is not valid`.
- **How to Check:**
  - Examine logs for AOF corruption.
  - Verify presence of `appendonly.aof`.
- **Fix (Commands):**

  1. **Stop Redis:**

     ```
     sudo systemctl stop redis
     ```

  2. **Backup AOF:**

     ```
     cp /var/lib/redis/appendonly.aof /var/lib/redis/appendonly.aof.bak
     ```

  3. **Fix corrupted AOF:**

     ```
     redis-check-aof --fix /var/lib/redis/appendonly.aof
     ```

  4. **Start Redis:**

     ```
     sudo systemctl start redis
     ```

  5. **Check logs/data:**

     ```
     tail -f /var/log/redis/redis-server.log
     ```

---

## 2. Redis Master-Slave (Replication)

### 2.1. Slave Can't Sync to Master

- **How to Check:**

  - Read slave logs for error.
  - Replication info:

    ```
    redis-cli info replication
    ```

- **Fix:**

  - Ensure network is open:

    ```
    nc -vz <master_ip> 6379
    ```

  - Ensure correct password with:

    ```
    masterauth <password>
    ```

  - Replica setup (on slave):

    ```
    redis-cli replicaof <master_ip> 6379
    ```

---

### 2.2. High Replication Lag

- **How to Check:**

```
redis-cli info replication
```

Observe `master_link_status`, `slave_repl_offset`.

- **Fix:**
- Address network issues, enhance disk speed, or scale.

---

## 3. Redis Cluster

### 3.1. CLUSTERDOWN Error

- **How to Check:**
- Cluster info:

  ```
  redis-cli -c -h <node_ip> -p <port> cluster info
  ```

- List nodes:

  ```
  redis-cli -c -h <node_ip> -p <port> cluster nodes
  ```

- **Fix:**
- Start enough master nodes to reach quorum.
- Redistribute slots if needed:

  ```
  redis-cli --cluster fix <node_ip>:<port>
  ```

---

### 3.2. MOVED/ASK Errors

- **How to Check:**
- Client receives MOVED/ASK message.
- Inspect cluster slots:

  ```
  redis-cli -c -h <node_ip> -p <port> cluster slots
  ```

- **Fix:**
- Use cluster-aware client libraries.
- Hash tags for multi-key ops: Use keys like `user:{42}:data`.

---

### 3.3. Lost/Missing Slots or Node Failure

- **How to Check:**

```
redis-cli -c -h <node_ip> -p <port> cluster slots
```

- **Fix:**
- Repair cluster slots:

  ```
  redis-cli --cluster fix <node_ip>:<port>
  ```

- Add nodes and reshard if needed:

  ```
  redis-cli --cluster add-node <new_ip>:<port> <existing_ip>:<port>
  redis-cli --cluster reshard <existing_ip>:<port>
  ```

---

## Troubleshooting Table

| Mode         | Error                | Check Command(s)                   | Quick Fix Command(s)                         |
| ------------ | -------------------- | ---------------------------------- | -------------------------------------------- |
| Standalone   | Can't connect        | `netstat`, `ps`, `redis-cli ping`  | `systemctl start redis`, fix config/firewall |
| Standalone   | OOM                  | `redis-cli info memory`, `free -m` | Edit eviction policy/config                  |
| Standalone   | RDB/AOF persist fail | `tail log`, `df -h`                | Free disk, fix permissions                   |
| Standalone   | AOF failed           | `tail log`, check .aof             | `redis-check-aof --fix`, restore backup      |
| Master-Slave | Slave can't sync     | `info replication`, logs           | Setup correct replicaof, open port           |
| Master-Slave | Replication lag      | `info replication`                 | Scale/distribute load                        |
| Cluster      | CLUSTERDOWN          | `cluster info`, `cluster nodes`    | Start/recover masters, cluster fix           |
| Cluster      | MOVED/ASK/split slot | error message, `cluster slots`     | Use cluster client, `--cluster fix`          |
| Cluster      | Lost/missing slots   | `cluster slots`, logs              | `--cluster fix`, add/reshard node            |

---

## Best Practices & Monitoring

- Use health-checks and alerting on:
- Memory, persistence errors (`used_memory`, `rdb_last_bgsave_status`, `aof_last_write_status`)
- Replication lag (`master_link_status`, `slave_repl_offset`)
- Cluster state (`cluster_state` from `cluster info`)
- Always back up data (`appendonly.aof`, `dump.rdb`) before major changes.
- For AOF issues, never run fix command without backup!
- Use cluster-aware clients for Redis Cluster deployments.

---
