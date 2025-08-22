# Elasticsearch Troubleshooting Guide

This document lists the most common Elasticsearch errors and step-by-step solutions for SRE and DevOps teams.

---

## 1. Cluster Health Issues

### 🔴 Cluster is RED or YELLOW

- **Symptom:** One or more unassigned shards, unavailable indices, or degraded search/indexing.
- **How to Fix:**
  1. Check cluster health and unassigned reasons:
     ```
     GET _cluster/health?level=shards
     GET _cat/shards?v&h=index,shard,prirep,state,unassigned.reason
     ```
  2. Enable shard allocation and retry:
     ```
     PUT _cluster/settings
     {
       "persistent": { "cluster.routing.allocation.enable": "all" }
     }
     POST _cluster/reroute?retry_failed=true
     ```
  3. Analyze allocation explanations for root cause.
  4. Free up disk space if disk watermark exceeded.

---

## 2. Node/Discovery Issues

### ⚡ Master Not Discovered

- **Symptom:** Errors about `master_not_discovered_exception`.
- **How to Fix:**
  1. Check node connectivity and configuration.
  2. Ensure `discovery.seed_hosts` and `cluster.initial_master_nodes` are set correctly in `elasticsearch.yml`.
  3. Ensure all nodes are on same cluster name and correct network settings.
  4. Check logs for network binding/permission errors.

---

## 3. Memory and Performance Problems

### 💾 OutOfMemoryError / High JVM Heap Usage

- **Symptom:** Frequent GC pauses, circuit breaking, slowness.
- **How to Fix:**
  1. Monitor heap usage:
     ```
     GET _cat/nodes?v&h=name,heap.percent,ram.percent
     ```
  2. Adjust JVM heap: Set `-Xms` and `-Xmx` no more than 50% of system RAM.
  3. Optimize queries and aggregations.
  4. Increase fielddata or request circuit breaker limits if necessary:
     ```
     PUT _cluster/settings
     {
       "persistent": {
         "indices.breaker.fielddata.limit": "60%",
         "indices.breaker.request.limit": "60%"
       }
     }
     ```

---

## 4. Index and Mapping Issues

### 🗺️ Mapper Parsing Exception

- **Symptom:** Errors when indexing documents due to incompatible mappings.
- **How to Fix:**
  1. Inspect current index mappings:
     ```
     GET /your_index/_mapping
     ```
  2. Update mapping schema or create a new index with correct mapping.
  3. Reindex from old to new index.

---

### 📊 Bulk Indexing Errors

- **Symptom:** Partial or failed bulk insert, e.g., version conflicts.
- **How to Fix:**
  1. Reduce bulk size.
  2. Fix any version conflicts by ensuring the correct `_id` or using versioning strategies.
  3. Increase thread pool write queue size if needed.

---

## 5. Search and Query Issues

### ⏰ Search Timeout / All Shards Failed

- **Symptom:** Slow or failed queries, timeouts, partial results.
- **How to Fix:**
  1. Optimize queries (prefer filters, limit returned fields, avoid wildcards).
  2. Tune timeout parameters:
     ```
     GET /index/_search?timeout=60s
     ```
  3. Investigate shards' health:
     ```
     GET _cat/shards/your_index?v
     ```

---

## 6. Disk and Storage Alerts

### 💽 Disk Watermark Exceeded

- **Symptom:** Cluster blocks writes, unassigned shards due to full disk.
- **How to Fix:**
  1. Check disk usage:
     ```
     GET _cat/nodes?v&h=name,disk.used_percent
     ```
  2. Increase or adjust disk watermark settings if needed.
  3. Move or delete old indices to free up space.
  4. Relocate shards manually if required.

---

## 7. Network/Connectivity Issues

### 🌐 Connection Refused / Transport Error

- **Symptom:** Cannot connect to ES REST API or transport port.
- **How to Fix:**
  1. Check that Elasticsearch process is running.
  2. Verify `network.host`, `http.port`, and firewall rules.
  3. Allow relevant ports (`9200` for REST, `9300` for node comms).
  4. Test with `curl` or `telnet` from client to cluster node.

---

## 8. Security/Access Issues

### 🔒 Security Exception / Read-Only Block

- **Symptom:** Index blocks all writes (`FORBIDDEN/12/index read-only / allow delete (api)`).
- **How to Fix:**
  1. Remove read-only block:
     ```
     PUT /your_index/_settings
     {
       "index.blocks.read_only_allow_delete": null
     }
     ```
  2. Check permissions, credentials, or API tokens.

---

## 9. Index Lifecycle Management (ILM) Errors

### ♻️ ILM not working (no deletes, rollovers, etc.)

- **Symptom:** Indices don't rollover or delete as expected.
- **How to Fix:**
  1. Check ILM status and policies:
     ```
     GET _ilm/policy
     GET _ilm/status
     ```
  2. Make sure rollover alias is set and write indices are correct.
  3. Review policy for syntax or logic problems.

---

## 10. Startup and Configuration Issues

### 🔧 Bootstrap Check Failures

- **Symptom:** Node fails to start, logs show "bootstrap checks failed".
- **How to Fix:**

  1. Set file descriptor/virtual memory limits:

     ```
     # /etc/security/limits.conf
     elasticsearch soft nofile 65536
     elasticsearch hard nofile 65536

     # /etc/sysctl.conf
     vm.max_map_count=262144
     sudo sysctl -p
     ```

  2. Enable `bootstrap.memory_lock: true` if setting mlock.
  3. Always restart the service after configuration changes.

---

## Monitoring and Prevention Best Practices

- Regularly check:
  - `GET _cluster/health`
  - `GET _cat/nodes?v`
  - `GET _cat/indices?v`
- Setup alerts for:
  - Cluster health not green
  - JVM heap > 75%
  - Disk usage > 85%
  - Slow queries or search latency > 1s
  - Unassigned shards > 0
- Use slow logs and centralized logging for incident root cause analysis.

---

**Tip:** Always test fixes in staging and have a current backup or snapshot before making changes on production clusters.
