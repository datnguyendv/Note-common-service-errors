# Common MongoDB Issues (Standalone, ReplicaSet, Sharded Cluster): Troubleshooting, Commands, and Fixes

This guide describes frequent MongoDB errors, how to troubleshoot and resolve them, and includes relevant CLI commands for SRE/DevOps cases.

---

## 1. Standalone Deployment

**Typical Use:** Development, small/POC setups, single non-redundant server.

## Common Incidents

## 1.1 MongoDB Process Down

- **Symptoms:** Application cannot connect, "connection refused."
- bashsystemctl status mongodtail -f /var/log/mongodb/mongod.logps aux | grep mongod

## 1.2 Out of Disk Space

- **Symptoms:** Write errors, "No space left on device."
- bashdf -hdu -sh /var/lib/mongodb/\*rm -rf /var/log/mongodb/\*.log-old # Clear old logs with care!

## 1.3 Authentication or Permission Issues

- **Symptoms:** Authentication failed errors, cannot connect.
- bashmongo -u admin -p --authenticationDatabase admingrep "security:" /etc/mongod.confls -l /var/lib/mongodb/chown -R mongodb:mongodb /var/lib/mongodb

## 1.4 Corrupted Data Files

- **Symptoms:** Mongod won't start, “data corruption” logs.
- bashrm /var/lib/mongodb/mongod.lockmongod --repair --dbpath /var/lib/mongodb

## 2. Replica Set Deployment (RS)

**Typical Use:** Production systems needing high availability.

## Common Incidents

## 2.1 Primary Election Issues

- **Symptoms:** No primary, writes rejected by secondaries (not writable).
- bashmongo --eval "rs.status()"mongo --eval "rs.isMaster()"

## 2.2 Secondary Node Outage or Lag

- **Symptoms:** Secondary missing or lagging, alerts on lag.
- bashmongo --eval "rs.status()"mongo --eval "rs.printSlaveReplicationInfo()"

## 2.3 Replication Broken (Rollback)

- **Symptoms:** "Rollback" logs, inconsistent data upon node rejoin.
- bashmongo --eval "rs.status()"# Consider re-syncing node from scratch

## 2.4 Oplog or Disk Full

- **Symptoms:** "oplog out of sync", replication stops.
- bashdf -hmongo --eval "db.getReplicationInfo()"

## 2.5 Split Brain (Partitioned Replica Set)

- **Symptoms:** Multiple primaries (rare), or long period without primary.
- bashmongo --eval "rs.status()"# Fix network partition, consider reconfig with force if quorum lost:mongo --eval "rs.reconfig(cfg, {force: true})"

## 3. Sharded Cluster Deployment

**Typical Use:** Horizontal scaling for very large datasets/high throughput.

## Architecture

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`text[ mongos ] → [ config servers (CSRS) ] → [ shard replica sets ]`

## Common Incidents

## 3.1 Config Servers Down

- **Symptoms:** All sharding operations and metadata updates fail, cluster “frozen.”
- bashmongo --port 27019 --eval "rs.status()"tail -f /var/log/mongodb/configsvr.logmongo --port 27017 --eval "sh.status()"
- **Actions:** Restore config server majority, force reconfig if needed.

## 3.2 Mongos Router Failures

- **Symptoms:** Application cannot connect, route errors ("no primary found").
- bashsystemctl status mongosmongo --port 27017 --eval "db.adminCommand('ismaster')"db.adminCommand('listShards')
- **Actions:** Restart mongos, synchronize with config servers, check networking.

## 3.3 Shard or RS Down

- **Symptoms:** Data range is inaccessible, chunk errors in sh.status().
- bashmongo --port 27018 --eval "rs.status()"mongo --port 27017 --eval "sh.status()"
- **Actions:** Restore failed nodes, migrate chunks before removing any failed shard.

## 3.4 Balancer and Chunk Issues

- **Symptoms:** Write/read hotspots, chunk move failures, jumbo chunk warnings.
- bashmongo --port 27017 --eval "sh.getBalancerState()"db.getSiblingDB('config').chunks.aggregate(\[{$group: {\_id: '$shard', count: {$sum: 1}}}, {$sort: {count: -1}}\])mongo --port 27017 config --eval "db.chunks.find({jumbo: true})"
- **Actions:** Enable/disable balancer, split or refine shard keys, manual chunk moves.

## 3.5 Cross-Shard Query or Index Issues

- **Symptoms:** Slow aggregations, high cluster CPU/network.
- **Actions:** Review indices, make sure queries target shard key, use .explain().

## 4. Reference: Essential Health Check Bash Scripts

## 4.1 Standalone

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`bashsystemctl status mongod  mongo --eval "db.runCommand({serverStatus:1})"  df -h  free -h`

## 4.2 Replica Set

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`bashmongo --eval "rs.status()"  mongo --eval "rs.printSlaveReplicationInfo()"  df -h`

## 4.3 Sharded Cluster

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`bashmongo --port 27017 --eval "sh.status()"  mongo --port 27017 --eval "sh.getBalancerState()"  mongo --port 27019 --eval "rs.status()"   # Config server RS status  for SHARD in shard1-primary:27018 shard2-primary:27018; do    mongo $SHARD --eval "rs.status()"  done`

## 5. Key Monitoring & Alerting Points

- **Standalone:** Process down, disk full, memory high, slow queries.
- **Replica Set:** No primary, replication lag, node down, Oplog size/disk full.
- **Sharded Cluster:** Config server down, unbalanced shards/chunks, mongos down, balancer off, jumbo chunk counts, orphaned data.

**Tip:** Always script your cluster health-checks and practice failover in a test environment to avoid surprises during critical incidents.
