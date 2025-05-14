# Cross-Cluster Replication

This chapter covers cross-cluster replication (CCR) in Elasticsearch, enabling you to replicate indices across multiple clusters for disaster recovery, geographic distribution, and data locality.

## Table of Contents
- [Introduction to Cross-Cluster Replication](#introduction-to-cross-cluster-replication)
- [Cross-Cluster Replication Architecture](#cross-cluster-replication-architecture)
- [Setting Up Remote Clusters](#setting-up-remote-clusters)
- [Configuring Cross-Cluster Replication](#configuring-cross-cluster-replication)
- [Auto-Follow Patterns](#auto-follow-patterns)
- [Monitoring Cross-Cluster Replication](#monitoring-cross-cluster-replication)
- [Failover and Recovery](#failover-and-recovery)
- [CCR Performance Optimization](#ccr-performance-optimization)
- [Security Considerations](#security-considerations)
- [Advanced Use Cases](#advanced-use-cases)
- [Limitations and Considerations](#limitations-and-considerations)
- [Cross-Cluster Search](#cross-cluster-search)
- [Multi-Cluster Architectures](#multi-cluster-architectures)

## Introduction to Cross-Cluster Replication

Cross-Cluster Replication (CCR) is a feature of Elasticsearch that allows you to replicate indices from one cluster to another, providing capabilities for disaster recovery, data locality, and centralized reporting.

### Key Benefits

1. **Disaster Recovery**: Replicate data to standby clusters for rapid recovery
2. **Geographic Distribution**: Place data closer to users across multiple regions
3. **Central Reporting**: Aggregate data from multiple clusters to a central reporting cluster
4. **High Availability**: Ensure data availability across multiple data centers
5. **Data Locality**: Maintain local copies of data for low-latency access
6. **Load Distribution**: Distribute read queries across multiple clusters

### Core Concepts

1. **Leader Index**: The source index that is being replicated
2. **Follower Index**: The target index that receives updates from the leader
3. **Remote Cluster**: A connection to another Elasticsearch cluster
4. **Auto-Follow Pattern**: A pattern to automatically replicate new indices
5. **Soft Deletes**: A retention mechanism that preserves deleted documents for replication
6. **Soft Delete Retention Period**: How long deleted documents are kept for replication

### Use Cases

1. **Disaster Recovery**: Replicate data to a secondary data center or cloud region
2. **Multi-Region Deployment**: Replicate indices to multiple regions for local access
3. **Central Reporting**: Aggregate data from edge clusters to a central cluster
4. **Data Segregation**: Maintain isolated environments with shared data
5. **Compliance**: Keep data in specific geographic regions for regulatory compliance
6. **Upgrades**: Perform zero-downtime upgrades with cross-version compatibility

## Cross-Cluster Replication Architecture

Understand the architecture and components of cross-cluster replication.

### Replication Process

The replication process follows these steps:

1. **Initial Sync**: Follower indexes all documents from the leader
2. **Change Detection**: Leader tracks changes through soft deletes
3. **Change Shipping**: Changes are shipped to follower via the replication API
4. **Applying Changes**: Follower applies changes to maintain consistency
5. **Retention**: Leader maintains soft deletes for the retention period

```
   Leader Cluster                        Follower Cluster
   +--------------+                      +--------------+
   |              |                      |              |
   | Leader Index |  ----------------→   | Follower Index|
   |              |   Change Shipping    |              |
   +--------------+                      +--------------+
          |                                     |
          | Soft Delete                         | Apply
          | Retention                           | Changes
          ↓                                     ↓
   +--------------+                      +--------------+
   |   Translog   |                      |   Translog   |
   +--------------+                      +--------------+
```

### Replication Modes

CCR supports two primary replication modes:

1. **Active-Passive**: Leader handles writes, follower serves as a read-only backup
   ```
   Write → Leader Cluster → Replication → Follower Cluster → Read (during failover)
   ```

2. **Active-Active**: Bidirectional replication between clusters
   ```
   Cluster A ← → Cluster B
   (requires careful conflict management)
   ```

### Component Interactions

Key interactions between components:

1. **Transport Layer**: CCR uses the transport layer for replication
2. **Translog Operations**: Changes are tracked via translog
3. **Mapping Propagation**: Mappings are replicated from leader to follower
4. **Settings Propagation**: Certain settings are replicated automatically
5. **Sequence Numbers**: Used to track the progress of replication
6. **Remote Recovery**: Initializes follower shards from leader shards

## Setting Up Remote Clusters

Configure connections between Elasticsearch clusters.

### Remote Cluster Settings

Configure remote cluster connections:

```json
// Using seed nodes (Elasticsearch 7.x+)
PUT _cluster/settings
{
  "persistent": {
    "cluster.remote.cluster-west.seeds": [
      "192.168.1.10:9300",
      "192.168.1.11:9300",
      "192.168.1.12:9300"
    ],
    "cluster.remote.cluster-west.skip_unavailable": true
  }
}

// Using a proxy (Elasticsearch 7.7+)
PUT _cluster/settings
{
  "persistent": {
    "cluster.remote.cluster-east.mode": "proxy",
    "cluster.remote.cluster-east.proxy_address": "proxy-address:9300",
    "cluster.remote.cluster-east.proxy_socket_connections": 18,
    "cluster.remote.cluster-east.server_name": "cluster-east.example.com"
  }
}
```

### Verifying Remote Cluster Connections

Check the status of remote cluster connections:

```json
// Check remote cluster info
GET _remote/info

// Check remote cluster health
GET _remote/cluster-west/_health

// List remote clusters
GET _cluster/settings?include_defaults=true&filter_path=defaults.cluster.remote
```

### Connection Troubleshooting

Troubleshoot common remote cluster connection issues:

```json
// Check transport connectivity
GET _nodes/transport

// Verify node roles
GET _cat/nodes?v&h=name,role,ip,port

// Update connection settings
PUT _cluster/settings
{
  "persistent": {
    "cluster.remote.cluster-west.skip_unavailable": true,
    "cluster.remote.cluster-west.transport.ping_schedule": "5s",
    "cluster.remote.cluster-west.transport.compress": true
  }
}
```

### Remote Cluster Network Requirements

Ensure proper network configuration:

1. **Transport Port Access**: Remote clusters need access to transport ports (default 9300)
2. **Bidirectional Communication**: Both clusters must be able to communicate if using seed mode
3. **Network Latency**: Low latency is preferable for efficient replication
4. **Bandwidth**: Sufficient bandwidth for the expected volume of changes
5. **Firewall Rules**: Allow appropriate traffic between clusters

```
Firewall Configuration Example:
  - Allow TCP 9300 from follower cluster to leader cluster
  - Allow TCP 9300 from leader cluster to follower cluster (if using seed mode)
  - Consider using dedicated network interfaces for cross-cluster traffic
```

## Configuring Cross-Cluster Replication

Set up cross-cluster replication for indices.

### Prerequisite Configuration

Ensure prerequisites are met before setting up CCR:

```json
// Ensure soft deletes are enabled on leader indices
PUT leader-index
{
  "settings": {
    "index": {
      "soft_deletes.enabled": true,
      "soft_deletes.retention.operations": 1024,
      "number_of_shards": 3,
      "number_of_replicas": 1
    }
  }
}

// Set up an index lifecycle policy for leader indices
PUT _ilm/policy/leader-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "50gb",
            "max_age": "30d"
          },
          "set_priority": {
            "priority": 100
          }
        }
      }
    }
  }
}

// Create an index template for leader indices
PUT _index_template/leader-template
{
  "index_patterns": ["leader-logs-*"],
  "template": {
    "settings": {
      "index": {
        "soft_deletes.enabled": true,
        "soft_deletes.retention.operations": 1024,
        "number_of_shards": 3,
        "number_of_replicas": 1,
        "lifecycle.name": "leader-policy",
        "lifecycle.rollover_alias": "leader-logs"
      }
    }
  }
}
```

### Creating Follower Indices

Create follower indices that replicate from leader indices:

```json
// Create a follower index
PUT follower-logs-000001
{
  "remote_cluster": "cluster-west",
  "leader_index": "leader-logs-000001",
  "settings": {
    "index.number_of_replicas": 1
  }
}

// Create a follower index with advanced settings
PUT follower-logs-000002
{
  "remote_cluster": "cluster-west",
  "leader_index": "leader-logs-000002",
  "settings": {
    "index.number_of_replicas": 1
  },
  "max_read_request_operation_count": 5000,
  "max_outstanding_read_requests": 12,
  "max_read_request_size": "32mb",
  "max_write_request_operation_count": 5000,
  "max_write_request_size": "32mb",
  "max_outstanding_write_requests": 9,
  "max_write_buffer_count": 2147483647,
  "max_write_buffer_size": "512mb",
  "max_retry_delay": "500ms",
  "read_poll_timeout": "1m"
}
```

### Managing Follower Indices

Perform operations on follower indices:

```json
// Pause a follower index
POST follower-logs-000001/_ccr/pause_follow

// Resume a paused follower index
POST follower-logs-000001/_ccr/resume_follow
{
  "max_read_request_operation_count": 5000,
  "max_outstanding_read_requests": 12
}

// Unfollow an index (convert to regular index)
POST follower-logs-000001/_ccr/unfollow

// Check follower stats
GET follower-logs-000001/_ccr/stats

// Check follower info
GET follower-logs-000001/_ccr/info
```

### Handling Index Templates

Manage index templates for CCR:

```json
// Create a template for follower indices
PUT _index_template/follower-template
{
  "index_patterns": ["follower-logs-*"],
  "template": {
    "settings": {
      "index": {
        "number_of_replicas": 1
      }
    }
  }
}
```

## Auto-Follow Patterns

Configure automatic replication of new indices.

### Creating Auto-Follow Patterns

Set up patterns to automatically replicate new indices:

```json
// Create an auto-follow pattern
PUT _ccr/auto_follow/logs-pattern
{
  "remote_cluster": "cluster-west",
  "leader_index_patterns": ["leader-logs-*"],
  "follow_index_pattern": "{{leader_index}}",
  "max_read_request_operation_count": 5000,
  "max_outstanding_read_requests": 12,
  "max_read_request_size": "32mb",
  "max_write_request_operation_count": 5000,
  "max_write_request_size": "32mb",
  "max_outstanding_write_requests": 9,
  "max_write_buffer_count": 2147483647,
  "max_write_buffer_size": "512mb",
  "max_retry_delay": "500ms",
  "read_poll_timeout": "1m"
}

// Create multiple auto-follow patterns
PUT _ccr/auto_follow/metrics-pattern
{
  "remote_cluster": "cluster-west",
  "leader_index_patterns": ["leader-metrics-*"],
  "follow_index_pattern": "{{leader_index}}",
  "settings": {
    "index.number_of_replicas": 1
  }
}
```

### Managing Auto-Follow Patterns

Manage existing auto-follow patterns:

```json
// Get all auto-follow patterns
GET _ccr/auto_follow

// Get a specific auto-follow pattern
GET _ccr/auto_follow/logs-pattern

// Delete an auto-follow pattern
DELETE _ccr/auto_follow/logs-pattern

// Pause all auto-follow patterns
POST _ccr/auto_follow/logs-pattern/pause

// Resume paused auto-follow patterns
POST _ccr/auto_follow/logs-pattern/resume
```

### Auto-Follow Pattern Best Practices

Best practices for auto-follow patterns:

1. **Pattern Precision**: Use specific patterns to control which indices are followed
2. **Index Naming**: Use consistent naming conventions for leader indices
3. **Template Consistency**: Ensure follower templates handle follower indices properly
4. **Resource Control**: Set appropriate limits for replication resources
5. **Monitoring**: Monitor auto-follow stats to detect issues

```json
// Get auto-follow stats
GET _ccr/stats

// Check for auto-follow errors
GET _ccr/stats?filter_path=auto_follow_stats.auto_follow_errors
```

## Monitoring Cross-Cluster Replication

Monitor the health and status of cross-cluster replication.

### Follower Index Stats

Check the status of follower indices:

```json
// Get stats for all follower indices
GET _ccr/stats

// Get stats for specific follower index
GET follower-logs-000001/_ccr/stats

// Check detailed leader and follower info
GET follower-logs-000001/_ccr/info
```

Key metrics to monitor:

1. **Replication Lag**: Time behind the leader
   ```json
   // Check time lag
   GET _cat/follower?v&h=index,leader_index,leader_lag_time,follower_index_uuid
   ```

2. **Operations Lag**: Number of operations behind
   ```json
   // Check operations lag
   GET _cat/follower?v&h=index,leader_index,leader_operations_lag
   ```

3. **Failed Operations**: Count of replication failures
   ```json
   // Check failed operations
   GET follower-logs-000001/_ccr/stats?filter_path=indices.*.fatal_exception,indices.*.failed_*
   ```

### Setting Up Monitoring

Configure monitoring for CCR:

```json
// Enable X-Pack monitoring
PUT _cluster/settings
{
  "persistent": {
    "xpack.monitoring.collection.enabled": true,
    "xpack.monitoring.collection.ccr.enabled": true
  }
}
```

Create monitoring dashboards in Kibana:

```json
// Example visualization for replication lag
{
  "aggs": {
    "indices": {
      "terms": {
        "field": "indices.key",
        "size": 10
      },
      "aggs": {
        "lag": {
          "max": {
            "field": "indices.leader_lag_time_millis"
          }
        }
      }
    }
  },
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        {
          "term": {
            "type": "ccr_stats"
          }
        }
      ]
    }
  }
}
```

### Alerting on Replication Issues

Set up alerts for replication problems:

```json
// Watcher for replication lag
PUT _watcher/watch/ccr_replication_lag
{
  "trigger": {
    "schedule": {
      "interval": "1m"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": [".monitoring-es-*"],
        "body": {
          "size": 0,
          "query": {
            "bool": {
              "filter": [
                {
                  "term": {
                    "type": "ccr_stats"
                  }
                },
                {
                  "range": {
                    "timestamp": {
                      "gte": "now-2m"
                    }
                  }
                }
              ]
            }
          },
          "aggs": {
            "indices": {
              "terms": {
                "field": "indices.key",
                "size": 100
              },
              "aggs": {
                "max_lag": {
                  "max": {
                    "field": "indices.leader_lag_time_millis"
                  }
                }
              }
            }
          }
        }
      }
    }
  },
  "condition": {
    "script": {
      "source": "return ctx.payload.aggregations.indices.buckets.stream().anyMatch(bucket -> bucket.max_lag.value > 300000);"
    }
  },
  "actions": {
    "notify_email": {
      "email": {
        "to": ["admin@example.com"],
        "subject": "CCR Replication Lag Alert",
        "body": {
          "html": """
          <h2>CCR Replication Lag Detected</h2>
          <p>The following follower indices have a replication lag greater than 5 minutes:</p>
          <ul>
          {{#ctx.payload.aggregations.indices.buckets}}
            {{#max_lag}}
              {{#value}}
                {{#compare . "gt" 300000.0}}
                <li>{{../../../key}} - {{display ../value "0.00"}} seconds</li>
                {{/compare}}
              {{/value}}
            {{/max_lag}}
          {{/ctx.payload.aggregations.indices.buckets}}
          </ul>
          """
        }
      }
    }
  }
}
```

## Failover and Recovery

Manage failover and recovery procedures for CCR.

### Failover Procedures

Procedures for failing over to a follower cluster:

1. **Pause Replication**:
   ```json
   // Pause all follower indices
   GET _cat/follower?h=index | xargs -I{} curl -X POST "localhost:9200/{}/_ccr/pause_follow"
   ```

2. **Convert Followers to Regular Indices**:
   ```json
   // Unfollow all follower indices
   GET _cat/follower?h=index | xargs -I{} curl -X POST "localhost:9200/{}/_ccr/unfollow"
   ```

3. **Update Aliases**:
   ```json
   // Update aliases to point to follower indices
   POST _aliases
   {
     "actions": [
       {
         "remove": {
           "index": "leader-logs*",
           "alias": "logs"
         }
       },
       {
         "add": {
           "index": "follower-logs*",
           "alias": "logs"
         }
       }
     ]
   }
   ```

4. **Redirect Traffic**:
   - Update load balancers
   - Update client configurations
   - Update DNS entries

### Resuming Normal Operations

Procedures for resuming normal operations:

1. **Reverse Replication** (if primary cluster recovers):
   ```json
   // Set up remote cluster connection from former leader to follower
   PUT _cluster/settings
   {
     "persistent": {
       "cluster.remote.former-follower.seeds": [
         "192.168.2.10:9300", 
         "192.168.2.11:9300"
       ]
     }
   }

   // Create leader indices on former follower
   PUT leader-logs-new
   {
     "settings": {
       "index": {
         "soft_deletes.enabled": true,
         "number_of_shards": 3,
         "number_of_replicas": 1
       }
     }
   }

   // Set up follower indices on former leader
   PUT follower-logs-new
   {
     "remote_cluster": "former-follower",
     "leader_index": "leader-logs-new"
   }
   ```

2. **Data Migration**:
   ```json
   // Use reindex to migrate data
   POST _reindex
   {
     "source": {
       "index": "leader-logs*"
     },
     "dest": {
       "index": "leader-logs-migrated",
       "op_type": "create"
     }
   }
   ```

### Testing Failover

Regularly test failover procedures:

```bash
#!/bin/bash
# Test failover script

# 1. Pause replication
echo "Pausing follower indices..."
indices=$(curl -s "http://follower-cluster:9200/_cat/follower?h=index" | tr -d '[:space:]')
for index in $indices; do
  echo "Pausing $index"
  curl -X POST "http://follower-cluster:9200/${index}/_ccr/pause_follow"
done

# 2. Convert followers to regular indices
echo "Converting follower indices to regular indices..."
for index in $indices; do
  echo "Unfollowing $index"
  curl -X POST "http://follower-cluster:9200/${index}/_ccr/unfollow"
done

# 3. Update aliases
echo "Updating aliases..."
curl -X POST "http://follower-cluster:9200/_aliases" -H 'Content-Type: application/json' -d '
{
  "actions": [
    {
      "add": {
        "index": "follower-logs*",
        "alias": "logs-active"
      }
    }
  ]
}
'

# 4. Test search functionality
echo "Testing search functionality..."
search_result=$(curl -s "http://follower-cluster:9200/logs-active/_search?size=0")
docs=$(echo $search_result | jq -r '.hits.total.value')
echo "Found $docs documents in the follower indices"

# 5. Verify functionality
echo "Verifying application functionality..."
# Add application-specific verification here

echo "Failover test completed"
```

## CCR Performance Optimization

Optimize cross-cluster replication performance.

### Replication Parameters

Tune replication parameters for optimal performance:

```json
// Adjust replication parameters for high throughput
PUT follower-logs-000001
{
  "remote_cluster": "cluster-west",
  "leader_index": "leader-logs-000001",
  "max_read_request_operation_count": 10000,    // Increased batch size
  "max_outstanding_read_requests": 20,          // More concurrent requests
  "max_read_request_size": "64mb",              // Larger request size
  "max_write_request_operation_count": 10000,   // Larger write batches
  "max_write_request_size": "64mb",             // Larger write size
  "max_outstanding_write_requests": 15,         // More concurrent writes
  "max_write_buffer_count": 2147483647,
  "max_write_buffer_size": "1gb",               // Larger buffer
  "max_retry_delay": "1s",
  "read_poll_timeout": "1m"
}

// Adjust replication parameters for low latency
PUT follower-logs-000002
{
  "remote_cluster": "cluster-west",
  "leader_index": "leader-logs-000002",
  "max_read_request_operation_count": 1000,     // Smaller batch size
  "max_outstanding_read_requests": 30,          // More concurrent requests
  "max_read_request_size": "16mb",
  "max_write_request_operation_count": 1000,    // Smaller write batches
  "max_write_request_size": "16mb",
  "max_outstanding_write_requests": 30,         // More concurrent writes
  "max_write_buffer_count": 2147483647,
  "max_write_buffer_size": "512mb",
  "max_retry_delay": "500ms",                   // Lower retry delay
  "read_poll_timeout": "30s"                    // Shorter timeout
}
```

### Network Optimization

Optimize network performance for CCR:

```json
// Configure compression for remote cluster connections
PUT _cluster/settings
{
  "persistent": {
    "cluster.remote.cluster-west.transport.compress": true
  }
}

// Adjust ping schedule for remote cluster
PUT _cluster/settings
{
  "persistent": {
    "cluster.remote.cluster-west.transport.ping_schedule": "30s"
  }
}
```

### Shard Allocation

Optimize shard allocation for replication:

```json
// Configure shard allocation for leader indices
PUT leader-logs-000001/_settings
{
  "index": {
    "routing.allocation.include.data": "hot",
    "routing.allocation.total_shards_per_node": 3,
    "priority": 100
  }
}

// Configure shard allocation for follower indices
PUT follower-logs-000001/_settings
{
  "index": {
    "routing.allocation.include.data": "warm",
    "routing.allocation.total_shards_per_node": 3,
    "priority": 50
  }
}
```

### Resource Allocation

Allocate system resources appropriately:

1. **Network Bandwidth**:
   - Ensure sufficient bandwidth between clusters
   - Consider dedicated network links for replication

2. **CPU Resources**:
   - Ensure sufficient CPU resources for replication threads
   - Monitor CPU usage during peak replication periods

3. **Memory Allocation**:
   - Configure appropriate heap sizes for high replication loads
   - Monitor GC patterns during replication

4. **Disk I/O**:
   - Use high-performance storage for leader indices
   - Monitor disk I/O patterns during replication

## Security Considerations

Secure cross-cluster replication deployments.

### Authentication and Authorization

Configure authentication for remote clusters:

```json
// Set up user for CCR
POST _security/user/ccr_user
{
  "password": "secure-password",
  "roles": ["ccr_role"],
  "full_name": "CCR User",
  "email": "ccr@example.com"
}

// Create role for CCR
POST _security/role/ccr_role
{
  "cluster": [
    "manage_ccr",
    "read_ccr"
  ],
  "indices": [
    {
      "names": ["leader-logs-*"],
      "privileges": ["monitor", "read", "read_cross_cluster"]
    }
  ]
}

// Configure user for remote cluster
PUT _cluster/settings
{
  "persistent": {
    "cluster.remote.cluster-west.credentials": {
      "username": "ccr_user"
    }
  }
}
```

### Encryption

Configure encryption for cross-cluster communication:

```yaml
# elasticsearch.yml on all nodes
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.client_authentication: required
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
```

### Cross-Cluster Security Considerations

Additional security considerations:

1. **Network Security**:
   - Use VPNs or private networks for cross-cluster traffic
   - Implement firewall rules to restrict access
   - Consider using proxy mode for restricted environments

2. **Certificate Management**:
   - Manage certificates across clusters
   - Plan for certificate rotation
   - Monitor certificate expiration

3. **Audit Logging**:
   - Enable audit logging for CCR operations
   ```yaml
   # elasticsearch.yml
   xpack.security.audit.enabled: true
   xpack.security.audit.logfile.events.include: ["ccr", "remote_cluster"]
   ```

## Advanced Use Cases

Explore advanced use cases for cross-cluster replication.

### Active-Active Configuration

Configure bidirectional replication between clusters:

```
Cluster A                               Cluster B
+------------------+                   +------------------+
| Index A1 (Leader)| ----------------> | Index A1 (Follower)|
+------------------+                   +------------------+
|                  |                   |                  |
| Index B1 (Follower)| <---------------- | Index B1 (Leader)|
+------------------+                   +------------------+
```

Implementation approach:

1. **Index Naming Strategy**:
   - Use region-specific prefixes for leader indices
   - Create follower indices in other regions

   ```json
   // In Cluster A
   PUT us-logs-000001 {...}  // Leader index
   
   // In Cluster B
   PUT follower-us-logs-000001
   {
     "remote_cluster": "cluster-a",
     "leader_index": "us-logs-000001"
   }
   
   // In Cluster B
   PUT eu-logs-000001 {...}  // Leader index
   
   // In Cluster A
   PUT follower-eu-logs-000001
   {
     "remote_cluster": "cluster-b",
     "leader_index": "eu-logs-000001"
   }
   ```

2. **Conflict Management**:
   - Implement application-level conflict resolution
   - Use versioning for documents
   - Consider using a conflict resolution strategy

### Multi-Cluster Replication

Configure replication across multiple clusters:

```
Cluster A (Primary)
       |
       | Replication
       v
Cluster B (DR)
       |
       | Replication
       v
Cluster C (Reporting)
```

Implementation:

```json
// Set up remote cluster connections on Cluster B
PUT _cluster/settings
{
  "persistent": {
    "cluster.remote.cluster-a.seeds": ["192.168.1.10:9300"],
    "cluster.remote.cluster-c.seeds": ["192.168.3.10:9300"]
  }
}

// Follow indices from Cluster A on Cluster B
PUT follower-logs-000001
{
  "remote_cluster": "cluster-a",
  "leader_index": "leader-logs-000001"
}

// Set up auto-follow on Cluster C from Cluster B
PUT _ccr/auto_follow/reporting-pattern
{
  "remote_cluster": "cluster-b",
  "leader_index_patterns": ["follower-logs-*"],
  "follow_index_pattern": "reporting-{{leader_index}}"
}
```

### Data Transformation During Replication

Transform data during replication using pipelines:

```json
// Create an ingest pipeline on the follower cluster
PUT _ingest/pipeline/enrich-logs
{
  "description": "Enrich log data with additional information",
  "processors": [
    {
      "set": {
        "field": "datacenter",
        "value": "us-east"
      }
    },
    {
      "set": {
        "field": "environment",
        "value": "dr"
      }
    },
    {
      "date": {
        "field": "timestamp",
        "formats": ["ISO8601"],
        "target_field": "timestamp_processed"
      }
    }
  ]
}

// Create an index template to apply the pipeline
PUT _index_template/follower-template
{
  "index_patterns": ["follower-*"],
  "template": {
    "settings": {
      "index.default_pipeline": "enrich-logs"
    }
  }
}
```

### Time-Based Failover

Implement time-based failover to the follower cluster:

```bash
#!/bin/bash
# Scheduled failover script for maintenance windows

# Configuration
PRIMARY_CLUSTER="https://cluster-a:9200"
DR_CLUSTER="https://cluster-b:9200"
AUTH_HEADER="Authorization: Basic $(echo -n 'elastic:password' | base64)"
MAINTENANCE_WINDOW_START=$(date -d "2023-05-20 22:00:00" +%s)
MAINTENANCE_WINDOW_END=$(date -d "2023-05-21 02:00:00" +%s)
CURRENT_TIME=$(date +%s)

# Check if we're in the maintenance window
if [ $CURRENT_TIME -ge $MAINTENANCE_WINDOW_START ] && [ $CURRENT_TIME -le $MAINTENANCE_WINDOW_END ]; then
  echo "Maintenance window active, failing over to DR cluster"
  
  # 1. Pause replication
  follower_indices=$(curl -s -H "$AUTH_HEADER" "$DR_CLUSTER/_cat/follower?h=index" | tr -d '[:space:]')
  for index in $follower_indices; do
    echo "Pausing $index"
    curl -X POST -H "$AUTH_HEADER" "$DR_CLUSTER/${index}/_ccr/pause_follow"
  done
  
  # 2. Convert followers to regular indices
  for index in $follower_indices; do
    echo "Unfollowing $index"
    curl -X POST -H "$AUTH_HEADER" "$DR_CLUSTER/${index}/_ccr/unfollow"
  done
  
  # 3. Update aliases
  curl -X POST -H "$AUTH_HEADER" -H 'Content-Type: application/json' "$DR_CLUSTER/_aliases" -d '
  {
    "actions": [
      {
        "add": {
          "index": "follower-*",
          "alias": "active-logs"
        }
      }
    ]
  }'
  
  # 4. Update DNS or load balancer
  # This would typically call an API to update your load balancer or DNS
  # echo "Updating load balancer configuration..."
  # aws elbv2 modify-listener ...
  
  echo "Failover completed"
else
  echo "Outside maintenance window, no action needed"
fi
```

## Limitations and Considerations

Understand the limitations and considerations for CCR.

### Replication Constraints

Key constraints to consider:

1. **Soft Deletes Requirement**:
   - Leader indices must have soft deletes enabled
   - Cannot replicate existing indices without soft deletes

2. **Index Settings Compatibility**:
   - Number of shards must be the same
   - Certain settings must match between leader and follower

3. **Version Compatibility**:
   - Follower cluster version must be the same or newer than leader
   - Maximum of one major version difference supported

4. **Write Restrictions**:
   - Follower indices are read-only
   - Cannot write to follower indices directly

5. **Mapping Limitations**:
   - Leader controls the mapping
   - Cannot modify follower index mapping independently

### Performance Considerations

Performance factors to consider:

1. **Network Latency**:
   - Higher latency increases replication lag
   - Consider geographic distance between clusters

2. **Throughput Requirements**:
   - High write volume requires more bandwidth
   - Configure appropriate batch sizes for your write patterns

3. **Resource Consumption**:
   - CCR uses resources on both leader and follower clusters
   - Additional CPU, memory, and network usage

4. **Shard Count Impact**:
   - More shards increase parallelism but also overhead
   - Balance shard count for replication performance

### Data Consistency Considerations

Data consistency factors to consider:

1. **Eventual Consistency**:
   - CCR provides eventual consistency, not immediate replication
   - Plan for replication lag in your applications

2. **Soft Delete Retention**:
   - Configure appropriate soft delete retention period
   - Too short can cause replication failures
   - Too long can increase storage requirements

3. **Sequence Numbers**:
   - CCR uses sequence numbers for tracking replication progress
   - If sequence numbers are missing, replication can fail

4. **Conflict Resolution**:
   - In active-active setups, conflicts can occur
   - Implement appropriate conflict resolution strategies

## Cross-Cluster Search

Use cross-cluster search (CCS) with CCR deployments.

### Configuring Cross-Cluster Search

Set up cross-cluster search:

```json
// Configure remote clusters for search
PUT _cluster/settings
{
  "persistent": {
    "cluster.remote.cluster-west.seeds": [
      "192.168.1.10:9300",
      "192.168.1.11:9300"
    ],
    "cluster.remote.cluster-east.seeds": [
      "192.168.2.10:9300",
      "192.168.2.11:9300"
    ]
  }
}

// Perform a cross-cluster search
GET cluster-west:logs-*,cluster-east:logs-*,logs-*/_search
{
  "query": {
    "match": {
      "message": "error"
    }
  }
}
```

### Cross-Cluster Aliases

Create aliases that span multiple clusters:

```json
// Create a local alias
POST _aliases
{
  "actions": [
    {
      "add": {
        "index": "logs-*",
        "alias": "all-logs"
      }
    }
  ]
}

// Query across clusters using cluster prefixes
GET all-logs,cluster-west:logs-*,cluster-east:logs-*/_search
{
  "query": {
    "match_all": {}
  }
}
```

### Setting Up Cross-Cluster Search in Kibana

Configure Kibana for cross-cluster search:

```yaml
# kibana.yml
elasticsearch.hosts: ["http://localhost:9200"]
elasticsearch.requestHeadersWhitelist: ["Authorization", "X-Custom-Header", "X-Cluster"]
```

Create cross-cluster search patterns in Kibana:
1. Navigate to Stack Management > Index Patterns
2. Create a new index pattern with cluster prefix: `cluster-west:logs-*`
3. Configure the pattern with appropriate time field
4. Use the pattern in visualizations and dashboards

### Cross-Cluster Search vs CCR

Compare cross-cluster search and cross-cluster replication:

| Feature | Cross-Cluster Search | Cross-Cluster Replication |
|---------|----------------------|---------------------------|
| Data Location | Data stays in original cluster | Data is copied to follower cluster |
| Query Performance | Depends on network latency | Local query performance |
| Network Dependency | Required for every query | Only required for replication |
| Data Freshness | Always up-to-date | Subject to replication lag |
| Query Complexity | Can impact performance | No impact on query performance |
| Resource Usage | Uses resources during queries | Uses resources during replication |
| Use Case | Ad-hoc queries across clusters | Local query performance, DR |

## Multi-Cluster Architectures

Design multi-cluster architectures with CCR.

### Regional Deployment Pattern

Deploy clusters in multiple geographic regions:

```
                +------------------+
                | Global Load      |
                | Balancer         |
                +------------------+
                         |
         +---------------+----------------+
         |                                |
+--------v---------+            +--------v---------+
| US Region        |            | EU Region        |
| Cluster          |            | Cluster          |
|                  |            |                  |
| Leader Indices   | <--CCR---> | Follower Indices |
| (US data)        |            | (US data)        |
|                  |            |                  |
| Follower Indices | <--CCR---> | Leader Indices   |
| (EU data)        |            | (EU data)        |
+------------------+            +------------------+
```

Configuration example:

```json
// US Cluster - Set up remote cluster connection
PUT _cluster/settings
{
  "persistent": {
    "cluster.remote.eu-cluster.seeds": ["eu-es1:9300", "eu-es2:9300"]
  }
}

// US Cluster - Create auto-follow pattern for EU data
PUT _ccr/auto_follow/eu-logs
{
  "remote_cluster": "eu-cluster",
  "leader_index_patterns": ["eu-logs-*"],
  "follow_index_pattern": "{{leader_index}}"
}

// EU Cluster - Set up remote cluster connection
PUT _cluster/settings
{
  "persistent": {
    "cluster.remote.us-cluster.seeds": ["us-es1:9300", "us-es2:9300"]
  }
}

// EU Cluster - Create auto-follow pattern for US data
PUT _ccr/auto_follow/us-logs
{
  "remote_cluster": "us-cluster",
  "leader_index_patterns": ["us-logs-*"],
  "follow_index_pattern": "{{leader_index}}"
}
```

### Tiered Deployment Pattern

Create a tiered architecture with specialized clusters:

```
   +------------------+         +------------------+
   | Data Ingestion   |         | Data Processing  |
   | Cluster          |         | Cluster          |
   |                  | --CCR-> |                  |
   | Leader Indices   |         | Follower Indices |
   +------------------+         +------------------+
                                         |
                                         | CCR
                                         v
                               +------------------+
                               | Analytics        |
                               | Cluster          |
                               |                  |
                               | Follower Indices |
                               +------------------+
```

Configuration example:

```json
// Data Ingestion Cluster - Create leader indices
PUT logs-000001
{
  "settings": {
    "index": {
      "soft_deletes.enabled": true,
      "number_of_shards": 5,
      "number_of_replicas": 1
    }
  }
}

// Data Processing Cluster - Follow indices from ingestion
PUT logs-000001
{
  "remote_cluster": "ingest-cluster",
  "leader_index": "logs-000001"
}

// Data Processing Cluster - Configure processing pipeline
PUT _ingest/pipeline/process-logs
{
  "processors": [
    {
      "grok": {
        "field": "message",
        "patterns": ["%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} %{GREEDYDATA:content}"]
      }
    },
    {
      "date": {
        "field": "timestamp",
        "formats": ["ISO8601"]
      }
    }
  ]
}

// Analytics Cluster - Follow indices from processing
PUT logs-000001
{
  "remote_cluster": "processing-cluster",
  "leader_index": "logs-000001"
}

// Analytics Cluster - Create rollup jobs
PUT _rollup/job/logs_rollup
{
  "index_pattern": "logs-*",
  "rollup_index": "logs-rollup",
  "cron": "0 0 * * * ?",
  "page_size": 1000,
  "groups": {
    "date_histogram": {
      "field": "@timestamp",
      "fixed_interval": "1h"
    },
    "terms": {
      "fields": ["level", "service"]
    }
  },
  "metrics": [
    {
      "field": "duration",
      "metrics": ["min", "max", "avg", "sum"]
    }
  ]
}
```

### Disaster Recovery Pattern

Implement a comprehensive disaster recovery pattern:

```
   Production Region                     DR Region
   +------------------+                +------------------+
   | Primary Cluster  |                | DR Cluster       |
   |                  | -----CCR-----> |                  |
   | Leader Indices   |                | Follower Indices |
   +------------------+                +------------------+
            |                                    |
            v                                    v
   +------------------+                +------------------+
   | Kibana           |                | Kibana (Standby) |
   +------------------+                +------------------+
            |                                    |
            v                                    v
   +------------------+                +------------------+
   | Applications     |                | Applications     |
   | (Active)         |                | (Standby)        |
   +------------------+                +------------------+
```

Configuration example:

```json
// Primary Cluster - Set up DR remote cluster
PUT _cluster/settings
{
  "persistent": {
    "cluster.remote.dr-cluster.seeds": ["dr-es1:9300", "dr-es2:9300"]
  }
}

// Primary Cluster - Create index template with CCR settings
PUT _index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "index": {
        "soft_deletes.enabled": true,
        "soft_deletes.retention.operations": 1024,
        "number_of_shards": 3,
        "number_of_replicas": 1
      }
    }
  }
}

// Primary Cluster - Create auto-follow pattern
PUT _ccr/auto_follow/dr-logs
{
  "remote_cluster": "dr-cluster",
  "leader_index_patterns": ["logs-*"],
  "follow_index_pattern": "{{leader_index}}"
}

// DR Cluster - Set up monitoring for replication lag
PUT _watcher/watch/replication_lag
{
  "trigger": {
    "schedule": {
      "interval": "1m"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": [".monitoring-es-*"],
        "body": {
          "query": {
            "bool": {
              "filter": [
                {
                  "term": {
                    "type": "ccr_stats"
                  }
                }
              ]
            }
          },
          "aggs": {
            "max_lag": {
              "max": {
                "field": "ccr_stats.leader_lag_time_millis"
              }
            }
          }
        }
      }
    }
  },
  "condition": {
    "script": {
      "source": "return ctx.payload.aggregations.max_lag.value > 300000"
    }
  },
  "actions": {
    "notify_team": {
      "email": {
        "to": "team@example.com",
        "subject": "CCR Replication Lag Alert",
        "body": "Replication lag exceeds 5 minutes. Current lag: {{ctx.payload.aggregations.max_lag.value}}ms"
      }
    }
  }
}
```

## Conclusion

Cross-cluster replication is a powerful feature in Elasticsearch that enables data replication across clusters for disaster recovery, data locality, and centralized reporting. By properly configuring and monitoring CCR, you can achieve high availability and resilience for your Elasticsearch deployment.

When implementing CCR, carefully consider your requirements for replication lag, resource utilization, and security. Design your multi-cluster architecture to align with your specific use cases, whether that's disaster recovery, geographic distribution, or specialized cluster roles.

Regular testing of your failover procedures and monitoring of replication status are essential to ensure your CCR setup functions correctly when needed. With proper planning and implementation, CCR can significantly enhance the reliability and performance of your Elasticsearch deployment.

In the next chapter, we'll explore deployment strategies for the ELK Stack, covering cloud deployments, on-premises installations, and hybrid approaches.