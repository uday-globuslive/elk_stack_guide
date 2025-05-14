# Disaster Recovery

This chapter covers disaster recovery (DR) strategies for the ELK Stack, ensuring your data and services can be recovered in the event of a major outage or catastrophic failure.

## Table of Contents
- [Introduction to Disaster Recovery](#introduction-to-disaster-recovery)
- [Disaster Recovery Planning](#disaster-recovery-planning)
- [Elasticsearch Backup Strategies](#elasticsearch-backup-strategies)
- [Cross-Cluster Replication](#cross-cluster-replication)
- [Multi-Region Architectures](#multi-region-architectures)
- [Logstash Disaster Recovery](#logstash-disaster-recovery)
- [Kibana Disaster Recovery](#kibana-disaster-recovery)
- [Data Validation and Testing](#data-validation-and-testing)
- [Disaster Recovery Automation](#disaster-recovery-automation)
- [Recovery Time Considerations](#recovery-time-considerations)
- [Operational Procedures](#operational-procedures)
- [Case Studies](#case-studies)

## Introduction to Disaster Recovery

Disaster recovery focuses on planning and processes to recover data and restore system functionality in the event of a catastrophic failure or disaster.

### Types of Disasters

Understanding the types of disasters that can affect your ELK deployment:

1. **Infrastructure Disasters**:
   - Data center outages
   - Cloud region failures
   - Network connectivity loss
   - Power failures

2. **Data Disasters**:
   - Data corruption
   - Accidental deletion
   - Index corruption
   - Mapping conflicts

3. **Software Disasters**:
   - Failed upgrades
   - Bug-induced data loss
   - Configuration errors
   - Incompatible plugin issues

4. **Security Incidents**:
   - Unauthorized access
   - Malicious data deletion
   - Ransomware attacks
   - Data breaches

### Key DR Metrics

Crucial metrics for disaster recovery planning:

1. **Recovery Point Objective (RPO)**:
   - Maximum acceptable data loss in a disaster
   - Measured in time (minutes, hours, days)
   - Determines backup frequency
   - Example: 15-minute RPO means maximum 15 minutes of data loss

2. **Recovery Time Objective (RTO)**:
   - Maximum acceptable downtime
   - Time to restore service after a disaster
   - Drives recovery method selection
   - Example: 4-hour RTO means service must be restored within 4 hours

3. **Recovery Consistency Objective (RCO)**:
   - Consistency level of recovered data
   - Can be measured in percentage or specific state
   - Example: Point-in-time consistency across services

### Disaster Recovery Tiers

Common disaster recovery tiers:

| Tier | RPO | RTO | Strategy | Typical Cost |
|------|-----|-----|----------|--------------|
| Tier 0 | 24+ hours | 24+ hours | Backup/restore | $ |
| Tier 1 | 12-24 hours | 12-24 hours | Cold site | $$ |
| Tier 2 | 4-12 hours | 4-12 hours | Warm site | $$$ |
| Tier 3 | 1-4 hours | 1-4 hours | Hot site | $$$$ |
| Tier 4 | < 1 hour | < 1 hour | Active-active | $$$$$ |

## Disaster Recovery Planning

Creating an effective disaster recovery plan for your ELK deployment.

### DR Plan Components

Essential components of an ELK Stack disaster recovery plan:

1. **Risk Assessment**:
   - Identify potential disaster scenarios
   - Evaluate impact on ELK components
   - Determine likelihood of each scenario
   - Prioritize based on risk and impact

2. **Recovery Objectives**:
   - Define RPO and RTO for each component
   - Set data preservation priorities
   - Determine acceptable service levels during recovery
   - Establish recovery sequence

3. **Recovery Strategies**:
   - Backup and restore procedures
   - Replication and failover mechanisms
   - Alternative site readiness
   - Data rebuild processes

4. **Recovery Team**:
   - Define roles and responsibilities
   - Identify key personnel
   - Establish communication channels
   - Provide necessary access and permissions

5. **Documentation**:
   - Detailed recovery procedures
   - Configuration information
   - Contact information
   - Dependencies and prerequisites

6. **Testing Schedule**:
   - Regular DR tests
   - Validation procedures
   - Test scenarios
   - Performance metrics

### DR Plan Example

Sample disaster recovery plan structure:

```
# ELK Stack Disaster Recovery Plan

## 1. Overview
   - Purpose and scope
   - Recovery priorities
   - RPO: 15 minutes
   - RTO: 4 hours

## 2. Disaster Recovery Team
   - Primary contact: Jane Smith, Lead DevOps (555-123-4567)
   - Secondary contact: John Doe, System Administrator (555-123-4568)
   - Elasticsearch specialist: Alice Johnson (555-123-4569)
   - Management contact: Bob Wilson, CTO (555-123-4570)

## 3. Recovery Infrastructure
   - Primary site: AWS us-east-1
   - DR site: AWS us-west-2
   - DR resources: (List of pre-provisioned resources)
   - Access methods: (VPN, bastion hosts, etc.)

## 4. Recovery Procedures
   4.1. Elasticsearch Recovery
      - Failover to DR cluster
      - Data validation
      - Client redirection
   
   4.2. Logstash Recovery
      - Deploy DR Logstash instances
      - Configure inputs and outputs
      - Validate data flow
   
   4.3. Kibana Recovery
      - Deploy DR Kibana instances
      - Configure security and access
      - Validate dashboards and visualizations

## 5. Communication Plan
   - Notification procedures
   - Status update frequency
   - Escalation paths
   - External communications

## 6. Testing and Validation
   - Testing schedule
   - Success criteria
   - Validation procedures
   - Improvement process
```

### Dependencies and External Systems

Identify dependencies that affect disaster recovery:

1. **Authentication Systems**:
   - LDAP/Active Directory
   - SAML identity providers
   - OAuth services
   - API key management

2. **Network Infrastructure**:
   - Load balancers
   - DNS services
   - Firewalls and security groups
   - VPN connections

3. **Storage Systems**:
   - S3 or equivalent object storage
   - NFS/shared file systems
   - SAN/block storage
   - Snapshot systems

4. **Monitoring and Alerting**:
   - External monitoring services
   - Alerting systems
   - Incident management platforms
   - Status page services

## Elasticsearch Backup Strategies

Configure backup solutions for Elasticsearch data protection.

### Snapshot Repository Configuration

Set up a snapshot repository for backups:

```json
// S3 Repository
PUT _snapshot/s3_repository
{
  "type": "s3",
  "settings": {
    "bucket": "elasticsearch-backups",
    "region": "us-east-1",
    "role_arn": "arn:aws:iam::123456789012:role/elasticsearch-backup-role",
    "compress": true,
    "server_side_encryption": true
  }
}

// Shared Filesystem Repository
PUT _snapshot/fs_repository
{
  "type": "fs",
  "settings": {
    "location": "/mnt/elasticsearch-backups",
    "compress": true
  }
}

// Azure Repository
PUT _snapshot/azure_repository
{
  "type": "azure",
  "settings": {
    "container": "elasticsearch-backups",
    "base_path": "elasticsearch",
    "compress": true
  }
}

// GCS Repository
PUT _snapshot/gcs_repository
{
  "type": "gcs",
  "settings": {
    "bucket": "elasticsearch-backups",
    "base_path": "elasticsearch",
    "compress": true
  }
}
```

### Snapshot Lifecycle Management

Configure automated snapshot management:

```json
// Create snapshot policy
PUT _slm/policy/daily
{
  "schedule": "0 30 1 * * ?", 
  "name": "<daily-snap-{now/d}>",
  "repository": "s3_repository",
  "config": {
    "indices": ["*"],
    "ignore_unavailable": true,
    "include_global_state": true
  },
  "retention": {
    "expire_after": "30d",
    "min_count": 5,
    "max_count": 50
  }
}

// Create different policies for different data types
PUT _slm/policy/logs-hourly
{
  "schedule": "0 0 * * * ?", 
  "name": "<logs-{now/h}>",
  "repository": "s3_repository",
  "config": {
    "indices": ["logs-*"],
    "ignore_unavailable": true,
    "include_global_state": false
  },
  "retention": {
    "expire_after": "7d",
    "min_count": 24,
    "max_count": 168
  }
}

// Create monthly snapshot for long-term retention
PUT _slm/policy/monthly
{
  "schedule": "0 0 1 * * ?", 
  "name": "<monthly-{now/M}>",
  "repository": "s3_repository",
  "config": {
    "indices": ["*"],
    "ignore_unavailable": true,
    "include_global_state": true
  },
  "retention": {
    "expire_after": "365d",
    "min_count": 12,
    "max_count": 24
  }
}
```

### Monitoring Snapshot Status

Monitor snapshot operations:

```json
// Check snapshot status
GET _snapshot/s3_repository/*?verbose=true

// Check SLM policy status
GET _slm/policy
GET _slm/stats

// Check snapshot status details
GET _cat/snapshots/s3_repository?v=true&s=state

// Monitor snapshot operations
GET _snapshot/_status

// Set up snapshot monitoring with Watcher
PUT _watcher/watch/snapshot_status
{
  "trigger": {
    "schedule": {
      "interval": "1h"
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
              "must": [
                {
                  "term": {
                    "type": "snapshot_stats"
                  }
                },
                {
                  "range": {
                    "timestamp": {
                      "gte": "now-24h"
                    }
                  }
                }
              ]
            }
          },
          "aggs": {
            "snapshots": {
              "terms": {
                "field": "snapshot_stats.repository",
                "size": 10
              },
              "aggs": {
                "snapshot_states": {
                  "terms": {
                    "field": "snapshot_stats.state"
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
      "source": "return ctx.payload.aggregations.snapshots.buckets.stream().anyMatch(bucket -> bucket.snapshot_states.buckets.stream().anyMatch(state -> state.key == 'FAILED'))"
    }
  },
  "actions": {
    "notify_email": {
      "email": {
        "to": ["admin@example.com"],
        "subject": "Elasticsearch Snapshot Failure Detected",
        "body": {
          "html": "Snapshot failures detected in the last 24 hours. Please investigate."
        }
      }
    }
  }
}
```

### Partial Snapshot Strategy

Configure partial snapshots for large clusters:

```json
// Indices-only snapshot (no global state)
PUT _snapshot/s3_repository/logs_snapshot?wait_for_completion=true
{
  "indices": "logs-*",
  "ignore_unavailable": true,
  "include_global_state": false,
  "metadata": {
    "taken_by": "system",
    "taken_because": "daily backup"
  }
}

// Snapshot specific indices with wildcards
PUT _snapshot/s3_repository/selected_indices?wait_for_completion=true
{
  "indices": "logs-2023.05.*,metrics-2023.05.*",
  "ignore_unavailable": true,
  "include_global_state": false
}

// Exclude certain indices
PUT _snapshot/s3_repository/exclude_temp?wait_for_completion=true
{
  "indices": "*,-temp-*,-test-*",
  "ignore_unavailable": true,
  "include_global_state": true
}
```

### Restoration Strategies

Develop strategies for different restore scenarios:

1. **Full Cluster Restore**:
   ```json
   // Restore entire snapshot to a new cluster
   POST _snapshot/s3_repository/daily-snap-2023.05.20/_restore
   {
     "indices": "*",
     "include_global_state": true
   }
   ```

2. **Selective Index Restore**:
   ```json
   // Restore specific indices
   POST _snapshot/s3_repository/daily-snap-2023.05.20/_restore
   {
     "indices": "logs-2023.05.20,metrics-2023.05.20",
     "ignore_unavailable": true,
     "include_global_state": false,
     "include_aliases": true
   }
   ```

3. **Restore with Rename**:
   ```json
   // Restore with index renaming
   POST _snapshot/s3_repository/daily-snap-2023.05.20/_restore
   {
     "indices": "logs-2023.05.20",
     "ignore_unavailable": true,
     "include_global_state": false,
     "rename_pattern": "(.+)",
     "rename_replacement": "restored_$1"
   }
   ```

4. **Partial Index Restore**:
   ```json
   // Restore with index settings override
   POST _snapshot/s3_repository/daily-snap-2023.05.20/_restore
   {
     "indices": "logs-*",
     "ignore_unavailable": true,
     "include_global_state": false,
     "index_settings": {
       "index.number_of_replicas": 0,
       "index.refresh_interval": "-1"
     },
     "ignore_index_settings": [
       "index.routing.allocation.*"
     ]
   }
   ```

## Cross-Cluster Replication

Configure cross-cluster replication (CCR) for disaster recovery.

### Remote Cluster Configuration

Set up remote cluster connections:

```json
// Configure remote cluster on the primary cluster
PUT _cluster/settings
{
  "persistent": {
    "cluster.remote.dr-cluster.seeds": [
      "es-dr-1.example.com:9300",
      "es-dr-2.example.com:9300",
      "es-dr-3.example.com:9300"
    ],
    "cluster.remote.dr-cluster.skip_unavailable": true
  }
}

// Verify remote cluster connection
GET _remote/info
```

### Follower Index Setup

Create follower indices for replication:

```json
// Create a follower index (executed on the DR cluster)
PUT logs-2023.05.20-follow
{
  "remote_cluster": "production-cluster",
  "leader_index": "logs-2023.05.20",
  "settings": {
    "index.number_of_replicas": 1
  }
}

// Configure advanced follower settings
PUT metrics-2023.05.20-follow
{
  "remote_cluster": "production-cluster",
  "leader_index": "metrics-2023.05.20",
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

### Auto-Follow Patterns

Configure automatic replication of new indices:

```json
// Create an auto-follow pattern for logs
PUT _ccr/auto_follow/logs-pattern
{
  "remote_cluster": "production-cluster",
  "leader_index_patterns": ["logs-*"],
  "follow_index_pattern": "{{leader_index}}-follow",
  "max_outstanding_read_requests": 12,
  "settings": {
    "index.number_of_replicas": 1
  }
}

// Create multiple auto-follow patterns for different index types
PUT _ccr/auto_follow/metrics-pattern
{
  "remote_cluster": "production-cluster",
  "leader_index_patterns": ["metrics-*"],
  "follow_index_pattern": "{{leader_index}}-follow",
  "max_outstanding_read_requests": 12,
  "settings": {
    "index.number_of_replicas": 1
  }
}
```

### Monitoring Replication

Monitor cross-cluster replication status:

```json
// Check auto-follow patterns
GET _ccr/auto_follow

// Check follower stats
GET _ccr/stats

// Check follower indices
GET _cat/follower

// Get detailed follower info
GET logs-2023.05.20-follow/_ccr/info

// Check follower stats for specific index
GET logs-2023.05.20-follow/_ccr/stats
```

### Failover Procedures

Develop procedures for CCR failover:

1. **Pausing Followers**:
   ```json
   // Pause a follower index
   POST logs-2023.05.20-follow/_ccr/pause_follow
   
   // Pause all followers (requires scripting)
   GET _cat/follower?format=json | jq -r '.[].index' | while read index; do
     curl -XPOST "http://localhost:9200/${index}/_ccr/pause_follow";
   done
   ```

2. **Converting Followers to Regular Indices**:
   ```json
   // Unfollow to convert to regular index
   POST logs-2023.05.20-follow/_ccr/unfollow
   
   // Unfollow all indices (requires scripting)
   GET _cat/follower?format=json | jq -r '.[].index' | while read index; do
     curl -XPOST "http://localhost:9200/${index}/_ccr/unfollow";
   done
   ```

3. **Redirecting Traffic**:
   - Update load balancers to point to DR cluster
   - Update client configurations
   - Change DNS entries

4. **Resuming Replication in Reverse**:
   - Configure reverse replication after primary cluster recovery
   - Set up remote cluster connection from DR to primary
   - Create follower indices on primary for any indices created during DR

## Multi-Region Architectures

Design disaster recovery architectures across multiple regions.

### Active-Passive Architecture

Create a standby system in a secondary region:

```
Region A (Active)                        Region B (Passive)
+--------------------+                  +--------------------+
| Elasticsearch      |                  | Elasticsearch      |
| Cluster            |                  | Cluster            |
+--------+-----------+                  +--------+-----------+
         |                CCR                    |
         +------------------------------------->+
         |                                      |
+--------+-----------+                  +--------+-----------+
| Kibana             |                  | Kibana             |
| Cluster            |                  | Cluster (Standby)  |
+--------+-----------+                  +--------+-----------+
         |                                      |
+--------+-----------+                  +--------+-----------+
| Logstash           |                  | Logstash           |
| Cluster            |                  | Cluster (Standby)  |
+--------+-----------+                  +--------+-----------+
         ^                                      ^
         |                                      |
+--------+-----------+                  +--------+-----------+
| Load               |                  | Load               |
| Balancer           |                  | Balancer           |
+--------+-----------+                  +--------+-----------+
         ^                                      ^
         |                                      |
+--------+-----------+                          |
| Beats/Agents       +--------------------------+
|                    |     Secondary Path
+--------------------+     (Used during failover)
```

Configuration elements:
- Cross-cluster replication from active to passive
- Regularly updated standby Kibana and Logstash
- Global DNS with region failover
- Dual-region Beat configurations

### Active-Active Architecture

Deploy active clusters in multiple regions:

```
Region A                                 Region B
+--------------------+                  +--------------------+
| Elasticsearch      |<---------------->| Elasticsearch      |
| Cluster A          |   Bidirectional  | Cluster B          |
+--------+-----------+     CCR          +--------+-----------+
         |                                      |
+--------+-----------+                  +--------+-----------+
| Kibana A           |                  | Kibana B           |
+--------+-----------+                  +--------+-----------+
         |                                      |
+--------+-----------+                  +--------+-----------+
| Logstash A         |                  | Logstash B         |
+--------+-----------+                  +--------+-----------+
         ^                                      ^
         |                                      |
+--------+-----------+                  +--------+-----------+
| Load Balancer A    |                  | Load Balancer B    |
+--------------------+                  +--------------------+
         ^                                      ^
         |                                      |
    +----+------------------------------------+
    |                                         |
+---+----------+                    +---------+-----+
| Beats/Agents |                    | Beats/Agents  |
| Region A     |                    | Region B      |
+--------------+                    +---------------+
```

Configuration elements:
- Primary-primary index allocation across regions
- Bidirectional cross-cluster replication
- Cross-region client routing (geo-based)
- Regional failover capability
- Coordinated configuration management

### Global Load Balancing

Implement global load balancing for multi-region deployments:

1. **AWS Global Accelerator**:
   ```json
   // CloudFormation template excerpt
   "EsAccelerator": {
     "Type": "AWS::GlobalAccelerator::Accelerator",
     "Properties": {
       "Name": "elasticsearch-accelerator",
       "Enabled": true,
       "IpAddressType": "IPV4"
     }
   },
   "EsListener": {
     "Type": "AWS::GlobalAccelerator::Listener",
     "Properties": {
       "AcceleratorArn": {"Ref": "EsAccelerator"},
       "Protocol": "TCP",
       "PortRanges": [
         {"FromPort": 9200, "ToPort": 9200},
         {"FromPort": 9300, "ToPort": 9300}
       ]
     }
   }
   ```

2. **Google Cloud Load Balancing**:
   ```yaml
   # Terraform configuration excerpt
   resource "google_compute_global_address" "elasticsearch" {
     name = "elasticsearch-address"
   }
   
   resource "google_compute_global_forwarding_rule" "elasticsearch" {
     name        = "elasticsearch-forwarding-rule"
     ip_address  = google_compute_global_address.elasticsearch.address
     port_range  = "9200"
     target      = google_compute_target_http_proxy.elasticsearch.self_link
   }
   ```

3. **Azure Traffic Manager**:
   ```json
   // ARM template excerpt
   {
     "type": "Microsoft.Network/trafficManagerProfiles",
     "apiVersion": "2018-08-01",
     "name": "elasticsearch-tm",
     "location": "global",
     "properties": {
       "profileStatus": "Enabled",
       "trafficRoutingMethod": "Performance",
       "dnsConfig": {
         "relativeName": "elasticsearch",
         "ttl": 60
       },
       "monitorConfig": {
         "protocol": "HTTPS",
         "port": 9200,
         "path": "/_cluster/health",
         "expectedStatusCodeRanges": [
           {
             "min": 200,
             "max": 299
           }
         ]
       }
     }
   }
   ```

### Data Consistency Challenges

Address data consistency challenges in multi-region deployments:

1. **Split-Brain Prevention**:
   - Use odd number of availability zones with quorum-based voting
   - Implement witness nodes in a third region
   - Configure proper minimum_master_nodes (Elasticsearch 6.x) or voting-only nodes (7.x+)

2. **Replication Lag Management**:
   ```json
   // Configure CCR with smaller batch sizes for reduced lag
   PUT metrics-follow
   {
     "remote_cluster": "primary",
     "leader_index": "metrics",
     "max_read_request_operation_count": 1000,
     "max_outstanding_read_requests": 4,
     "max_read_request_size": "8mb",
     "max_write_request_operation_count": 1000,
     "max_write_request_size": "8mb",
     "max_outstanding_write_requests": 4
   }
   ```

3. **Conflict Resolution**:
   - Implement versioning for documents
   - Use timestamp-based conflict resolution
   - Define clear write regions for specific data types

## Logstash Disaster Recovery

Design disaster recovery strategies for Logstash deployments.

### Logstash Configuration Management

Store and version Logstash configurations for disaster recovery:

1. **GitOps Approach**:
   ```bash
   # Store configurations in Git
   git init /etc/logstash/conf.d
   git -C /etc/logstash/conf.d add .
   git -C /etc/logstash/conf.d commit -m "Initial configuration"
   git -C /etc/logstash/conf.d remote add origin https://git-repo.example.com/logstash-configs.git
   git -C /etc/logstash/conf.d push -u origin master
   
   # Deploy configurations from Git
   git clone https://git-repo.example.com/logstash-configs.git /tmp/logstash-configs
   cp -R /tmp/logstash-configs/* /etc/logstash/conf.d/
   systemctl restart logstash
   ```

2. **Configuration Management Tools**:
   ```yaml
   # Ansible playbook excerpt
   - name: Deploy Logstash configurations
     ansible.builtin.template:
       src: "templates/logstash/{{ item }}.conf.j2"
       dest: "/etc/logstash/conf.d/{{ item }}.conf"
       owner: logstash
       group: logstash
       mode: '0644'
     with_items:
       - inputs
       - filters
       - outputs
     notify: restart logstash
   ```

### Input Redundancy

Configure redundant inputs for reliable data collection:

```ruby
# Multiple input methods for redundancy
input {
  # Primary collection via Beats
  beats {
    port => 5044
    ssl => true
    ssl_certificate => "/etc/logstash/certs/logstash.crt"
    ssl_key => "/etc/logstash/certs/logstash.key"
    ssl_certificate_authorities => ["/etc/logstash/certs/ca.crt"]
  }
  
  # Secondary collection via TCP
  tcp {
    port => 5140
    codec => "json"
  }
  
  # Tertiary collection via S3 for disaster recovery
  s3 {
    bucket => "logs-backup"
    region => "us-east-1"
    prefix => "logs/"
    role_arn => "arn:aws:iam::123456789012:role/logstash-s3-role"
    interval => 60
    codec => "json_lines"
    # Only process during DR
    tags => ["disaster_recovery"]
    sincedb_path => "/var/lib/logstash/s3_recovery_sincedb"
  }
}

# Use tags to control flow
filter {
  if "disaster_recovery" in [tags] {
    # Special processing for DR data
    mutate {
      add_field => { "recovered" => true }
    }
  }
}
```

### Output Redundancy

Configure multiple outputs for reliable data delivery:

```ruby
output {
  # Primary output
  elasticsearch {
    hosts => ["es1:9200", "es2:9200", "es3:9200"]
    index => "logs-%{+YYYY.MM.dd}"
    template_name => "logs"
    template_overwrite => false
    loadbalance => true
    retry_max_interval => 32
    retry_initial_interval => 2
    max_retries => 5
  }
  
  # Backup to S3 for DR
  s3 {
    bucket => "logs-backup"
    region => "us-east-1"
    prefix => "logs/%{+YYYY/MM/dd}/"
    time_file => 15
    codec => json_lines
    role_arn => "arn:aws:iam::123456789012:role/logstash-s3-role"
    # Reduce frequency for non-critical data
    size_file => 1048576
    rotation_strategy => "size_and_time"
  }
  
  # Dead letter queue for failed events
  if "_elasticsearch_failure" in [tags] {
    file {
      path => "/var/log/logstash/dead_letter/%{+YYYY-MM-dd}/failed_events_%{+HH}.log"
      codec => json
    }
  }
}
```

### Pipeline Recovery

Recover pipelines after a disaster:

```ruby
# Recovery pipeline for processing backed up data
input {
  s3 {
    bucket => "logs-backup"
    region => "us-east-1"
    prefix => "logs/"
    role_arn => "arn:aws:iam::123456789012:role/logstash-s3-role"
    sincedb_path => "/var/lib/logstash/s3_dr_sincedb"
    additional_settings => {
      "force_path_style" => true
    }
    codec => "json_lines"
  }
}

filter {
  # Ensure @timestamp is set correctly
  date {
    match => [ "original_timestamp", "ISO8601" ]
    target => "@timestamp"
  }
  
  # Add recovery metadata
  mutate {
    add_field => {
      "recovered_at" => "%{@timestamp}"
      "recovery_source" => "s3_backup"
    }
  }
}

output {
  elasticsearch {
    hosts => ["dr-es1:9200", "dr-es2:9200", "dr-es3:9200"]
    index => "recovered-%{+YYYY.MM.dd}"
    template_name => "logs"
    template_overwrite => false
    retry_max_interval => 64
    max_retries => 10
  }
}
```

### Stateful Pipeline Components

Handle stateful pipeline components in disaster recovery:

1. **Persistent Queues**:
   ```yaml
   # logstash.yml for disaster recovery
   queue.type: persisted
   queue.max_bytes: 4gb
   path.queue: /var/lib/logstash/queue
   queue.checkpoint.writes: 1024
   
   # Backup queue data as part of DR
   # In backup script
   tar -czf /backups/logstash-queue-$(date +%Y%m%d).tar.gz /var/lib/logstash/queue
   
   # Restore queue data
   systemctl stop logstash
   rm -rf /var/lib/logstash/queue/*
   tar -xzf /backups/logstash-queue-20230520.tar.gz -C /
   systemctl start logstash
   ```

2. **Stateful Filters**:
   ```ruby
   # Handle stateful filters like aggregate
   filter {
     aggregate {
       task_id => "%{transaction_id}"
       code => "map['items'] ||= 0; map['items'] += 1"
       push_map_as_event_on_timeout => true
       timeout => 120
       timeout_tags => ["_aggregatetimeout"]
       
       # Use external persistence for DR
       aggregate_maps_path => "/var/lib/logstash/aggregate_maps"
     }
   }
   ```

## Kibana Disaster Recovery

Implement disaster recovery for Kibana deployments.

### Saved Objects Management

Export and import saved objects for disaster recovery:

```bash
# Export all saved objects to a file
curl -X GET "localhost:5601/api/saved_objects/_export" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{"type":["index-pattern","visualization","dashboard","search","map"]}' \
  > kibana_saved_objects_$(date +%Y%m%d).ndjson

# Create a script for regular backups
cat <<EOF > /usr/local/bin/backup-kibana-objects.sh
#!/bin/bash
BACKUP_DIR="/backups/kibana"
mkdir -p \$BACKUP_DIR
curl -s -X GET "localhost:5601/api/saved_objects/_export" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{"type":["index-pattern","visualization","dashboard","search","map"]}' \
  > \$BACKUP_DIR/kibana_saved_objects_\$(date +%Y%m%d).ndjson
EOF
chmod +x /usr/local/bin/backup-kibana-objects.sh

# Schedule with cron
echo "0 2 * * * /usr/local/bin/backup-kibana-objects.sh" | crontab -

# Import saved objects during recovery
curl -X POST "localhost:5601/api/saved_objects/_import?overwrite=true" \
  -H "kbn-xsrf: true" \
  --form file=@kibana_saved_objects_20230520.ndjson
```

### Space-Based Recovery

Implement space-based backup and recovery:

```bash
# Export objects from a specific space
curl -X GET "localhost:5601/s/marketing/api/saved_objects/_export" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{"type":["index-pattern","visualization","dashboard","search"]}' \
  > kibana_marketing_space_$(date +%Y%m%d).ndjson

# Import objects to a specific space during recovery
curl -X POST "localhost:5601/s/marketing/api/saved_objects/_import?overwrite=true" \
  -H "kbn-xsrf: true" \
  --form file=@kibana_marketing_space_20230520.ndjson
```

### Kibana Configuration Management

Manage Kibana configuration files for disaster recovery:

```yaml
# Store kibana.yml in version control
# Example: kibana.yml with DR settings
server.name: "kibana"
server.host: "0.0.0.0"

# Elasticsearch connection
elasticsearch.hosts: ["https://es1:9200", "https://es2:9200", "https://es3:9200"]
elasticsearch.username: "${ES_USERNAME}"
elasticsearch.password: "${ES_PASSWORD}"
elasticsearch.ssl.verificationMode: certificate
elasticsearch.ssl.certificateAuthorities: ["/etc/kibana/certs/ca.crt"]

# Kibana encryption keys (must be preserved for DR)
xpack.security.encryptionKey: "${ENCRYPTION_KEY}"
xpack.reporting.encryptionKey: "${REPORTING_ENCRYPTION_KEY}"
xpack.encryptedSavedObjects.encryptionKey: "${SAVED_OBJECTS_ENCRYPTION_KEY}"

# Store encryption keys in a secure location
# Example: AWS Parameter Store
aws ssm put-parameter \
  --name "/kibana/security/encryptionKey" \
  --type "SecureString" \
  --value "your-encryption-key-here" \
  --key-id "alias/aws/ssm"

# Retrieve during recovery
ENCRYPTION_KEY=$(aws ssm get-parameter \
  --name "/kibana/security/encryptionKey" \
  --with-decryption \
  --query "Parameter.Value" \
  --output text)
```

### Session Management

Handle user sessions during disaster recovery:

```yaml
# Configure session settings
xpack.security.session.idleTimeout: "1h"
xpack.security.session.lifespan: "24h"

# During DR, users will need to re-authenticate
# Consider extending timeout during recovery period
xpack.security.session.idleTimeout: "8h"
```

### UI Settings Recovery

Recover user interface settings:

```bash
# Export UI settings to a file
curl -X GET "localhost:9200/.kibana/_search?q=type:ui-metric" \
  -H "Content-Type: application/json" \
  > kibana_ui_settings_$(date +%Y%m%d).json

# Import UI settings during recovery
# This requires a custom script to parse the JSON and make POST requests
```

## Data Validation and Testing

Validate data integrity and test recovery procedures.

### Data Integrity Checks

Verify Elasticsearch data integrity:

```bash
# Check cluster health
curl -X GET "localhost:9200/_cluster/health?pretty"

# Check indices
curl -X GET "localhost:9200/_cat/indices?v"

# Count documents in original and recovered indices
curl -X GET "localhost:9200/logs-2023.05.20/_count?pretty"
curl -X GET "localhost:9200/recovered-logs-2023.05.20/_count?pretty"

# Create a validation script
cat <<EOF > /usr/local/bin/validate-indices.sh
#!/bin/bash
INDEX_PATTERN=\$1
ORIGINAL_HOST=\$2
RECOVERED_HOST=\$3

original_count=\$(curl -s "\$ORIGINAL_HOST/\$INDEX_PATTERN/_count" | jq -r '.count')
recovered_count=\$(curl -s "\$RECOVERED_HOST/\$INDEX_PATTERN/_count" | jq -r '.count')

echo "Original count: \$original_count"
echo "Recovered count: \$recovered_count"

if [ "\$original_count" -eq "\$recovered_count" ]; then
  echo "Validation passed: counts match"
  exit 0
else
  echo "Validation failed: counts don't match"
  exit 1
fi
EOF
chmod +x /usr/local/bin/validate-indices.sh
```

### Recovery Testing

Implement regular recovery testing:

```bash
# Create a DR test script
cat <<EOF > /usr/local/bin/test-elasticsearch-dr.sh
#!/bin/bash
set -e

# 1. Take a snapshot of test indices
echo "Taking a snapshot..."
curl -X PUT "localhost:9200/_snapshot/backup_repository/dr_test_snapshot?wait_for_completion=true" \
  -H "Content-Type: application/json" \
  -d '{"indices": "test-*", "ignore_unavailable": true}'

# 2. Delete test indices
echo "Deleting test indices..."
curl -X DELETE "localhost:9200/test-*"

# 3. Restore from snapshot
echo "Restoring from snapshot..."
curl -X POST "localhost:9200/_snapshot/backup_repository/dr_test_snapshot/_restore?wait_for_completion=true" \
  -H "Content-Type: application/json" \
  -d '{"indices": "test-*", "ignore_unavailable": true, "rename_pattern": "(.+)", "rename_replacement": "recovered-\$1"}'

# 4. Validate restoration
echo "Validating restoration..."
./validate-indices.sh "test-index" "localhost:9200" "localhost:9200/recovered"

echo "DR test completed"
EOF
chmod +x /usr/local/bin/test-elasticsearch-dr.sh
```

### Recovery Documentation

Create detailed recovery documentation:

```markdown
# Elasticsearch Disaster Recovery Procedure

## Prerequisites
- Access to DR environment
- AWS CLI configured with appropriate permissions
- Elasticsearch credentials
- Snapshot repository information

## Recovery Steps

### 1. Verify DR Environment Readiness
```bash
# Check DR cluster health
curl -X GET "https://dr-elasticsearch:9200/_cluster/health?pretty"
```

### 2. Identify Most Recent Snapshot
```bash
# List available snapshots
curl -X GET "https://dr-elasticsearch:9200/_snapshot/s3_repository/_all?pretty"
```

### 3. Restore Indices
```bash
# Restore all indices from the most recent snapshot
curl -X POST "https://dr-elasticsearch:9200/_snapshot/s3_repository/daily-snap-2023.05.20/_restore" \
  -H "Content-Type: application/json" \
  -d '{
    "indices": "*",
    "ignore_unavailable": true,
    "include_global_state": true
  }'
```

### 4. Validate Restoration
```bash
# Check cluster health
curl -X GET "https://dr-elasticsearch:9200/_cluster/health?pretty"

# Verify indices are restored
curl -X GET "https://dr-elasticsearch:9200/_cat/indices?v"
```

### 5. Update DNS/Load Balancers
- Update DNS records to point to DR environment
- Update load balancer configurations

### 6. Validate Application Access
- Verify Kibana can connect to restored indices
- Confirm visualizations and dashboards are working
- Verify log ingestion is functioning

### 7. Document Recovery
- Record the time recovery was completed
- Document any issues encountered
- Update the DR plan with improvements
```

## Disaster Recovery Automation

Automate disaster recovery processes for faster recovery.

### Automated Recovery Scripts

Create scripts for automated recovery:

```bash
#!/bin/bash
# elasticsearch-dr-recovery.sh
set -e

# Load configuration
source /etc/elasticsearch-dr/config.sh

# Log recovery start
log_message "Starting Elasticsearch disaster recovery procedure"

# Check DR cluster health
log_message "Checking DR cluster health"
cluster_health=$(curl -s -X GET "$DR_ES_URL/_cluster/health")
status=$(echo $cluster_health | jq -r '.status')

if [[ "$status" == "red" ]]; then
  log_message "ERROR: DR cluster is in red status. Manual intervention required." >&2
  exit 1
fi

# Find latest snapshot
log_message "Finding the latest snapshot"
latest_snapshot=$(curl -s -X GET "$DR_ES_URL/_snapshot/$SNAPSHOT_REPO/_all" | \
  jq -r '.snapshots | sort_by(.start_time) | last | .snapshot')

if [[ -z "$latest_snapshot" ]]; then
  log_message "ERROR: No snapshots found in repository $SNAPSHOT_REPO" >&2
  exit 1
fi

log_message "Latest snapshot: $latest_snapshot"

# Restore from snapshot
log_message "Starting restoration from snapshot $latest_snapshot"
curl -X POST "$DR_ES_URL/_snapshot/$SNAPSHOT_REPO/$latest_snapshot/_restore" \
  -H "Content-Type: application/json" \
  -d "{
    \"indices\": \"*\",
    \"ignore_unavailable\": true,
    \"include_global_state\": true
  }"

# Wait for restoration to complete
log_message "Waiting for restoration to complete"
while true; do
  recovery_status=$(curl -s -X GET "$DR_ES_URL/_recovery" | jq -r '.[] | .[] | .stage')
  if echo "$recovery_status" | grep -q "done"; then
    log_message "Recovery completed"
    break
  fi
  log_message "Recovery in progress, waiting..."
  sleep 30
done

# Validate restoration
log_message "Validating restoration"
index_count=$(curl -s -X GET "$DR_ES_URL/_cat/indices?v" | grep -v "^health" | wc -l)
log_message "$index_count indices restored"

# Update DNS if configured
if [[ "$UPDATE_DNS" == "true" ]]; then
  log_message "Updating DNS records"
  # AWS Route53 example
  aws route53 change-resource-record-sets \
    --hosted-zone-id $ROUTE53_ZONE_ID \
    --change-batch file://$DNS_CHANGE_JSON
fi

log_message "Disaster recovery procedure completed successfully"
```

### Recovery Orchestration

Implement orchestration for complex recoveries:

```yaml
# Ansible playbook for ELK disaster recovery
---
- name: Elasticsearch Disaster Recovery
  hosts: dr_elasticsearch
  become: yes
  tasks:
    - name: Check DR cluster health
      uri:
        url: "https://{{ dr_elasticsearch_host }}:9200/_cluster/health"
        method: GET
        return_content: yes
        headers:
          Authorization: "Basic {{ elasticsearch_auth | b64encode }}"
      register: cluster_health
      
    - name: Find latest snapshot
      uri:
        url: "https://{{ dr_elasticsearch_host }}:9200/_snapshot/{{ snapshot_repo }}/_all"
        method: GET
        return_content: yes
        headers:
          Authorization: "Basic {{ elasticsearch_auth | b64encode }}"
      register: snapshots
      
    - name: Set latest snapshot fact
      set_fact:
        latest_snapshot: "{{ (snapshots.json.snapshots | sort(attribute='start_time'))[-1].snapshot }}"
      
    - name: Restore from snapshot
      uri:
        url: "https://{{ dr_elasticsearch_host }}:9200/_snapshot/{{ snapshot_repo }}/{{ latest_snapshot }}/_restore"
        method: POST
        body_format: json
        body:
          indices: "*"
          ignore_unavailable: true
          include_global_state: true
        headers:
          Authorization: "Basic {{ elasticsearch_auth | b64encode }}"
      
    - name: Wait for restoration to complete
      uri:
        url: "https://{{ dr_elasticsearch_host }}:9200/_cluster/health"
        method: GET
        return_content: yes
        headers:
          Authorization: "Basic {{ elasticsearch_auth | b64encode }}"
      register: restoration_status
      until: restoration_status.json.status == "green" or restoration_status.json.status == "yellow"
      retries: 30
      delay: 30

- name: Kibana Disaster Recovery
  hosts: dr_kibana
  become: yes
  tasks:
    - name: Deploy Kibana configuration
      template:
        src: templates/kibana.yml.j2
        dest: /etc/kibana/kibana.yml
        owner: kibana
        group: kibana
        mode: '0644'
      
    - name: Import saved objects
      uri:
        url: "http://localhost:5601/api/saved_objects/_import?overwrite=true"
        method: POST
        headers:
          kbn-xsrf: "true"
        body_format: form-multipart
        body:
          file: "{{ lookup('file', 'files/kibana_saved_objects.ndjson') }}"
      
    - name: Restart Kibana
      systemd:
        name: kibana
        state: restarted
        
- name: Update DNS Records
  hosts: localhost
  tasks:
    - name: Update Route53 DNS record
      route53:
        state: present
        zone: "example.com"
        record: "elasticsearch.example.com"
        type: A
        value: "{{ dr_elasticsearch_ip }}"
        ttl: 60
      when: update_dns | bool
```

### AWS-Specific Recovery Automation

Automate recovery in AWS:

```python
# AWS Lambda function for ELK disaster recovery
import boto3
import json
import requests
import time
from base64 import b64encode
from datetime import datetime

def lambda_handler(event, context):
    # Configuration
    dr_elasticsearch_url = "https://dr-elasticsearch.example.com:9200"
    snapshot_repo = "s3_repository"
    route53_hosted_zone_id = "Z1234567890ABC"
    domain_name = "elasticsearch.example.com"
    dr_elasticsearch_ip = "10.0.1.100"
    
    # AWS clients
    route53 = boto3.client('route53')
    ssm = boto3.client('ssm')
    
    # Get credentials from Parameter Store
    es_username = ssm.get_parameter(Name='/elk/elasticsearch/username', WithDecryption=True)['Parameter']['Value']
    es_password = ssm.get_parameter(Name='/elk/elasticsearch/password', WithDecryption=True)['Parameter']['Value']
    auth_header = "Basic " + b64encode(f"{es_username}:{es_password}".encode()).decode()
    
    # Check DR cluster health
    health_response = requests.get(
        f"{dr_elasticsearch_url}/_cluster/health",
        headers={"Authorization": auth_header},
        verify=True
    )
    
    if health_response.status_code != 200:
        return {
            'statusCode': 500,
            'body': json.dumps('Failed to check DR cluster health')
        }
    
    health_data = health_response.json()
    if health_data['status'] == 'red':
        return {
            'statusCode': 500,
            'body': json.dumps('DR cluster is in red status')
        }
    
    # Find latest snapshot
    snapshots_response = requests.get(
        f"{dr_elasticsearch_url}/_snapshot/{snapshot_repo}/_all",
        headers={"Authorization": auth_header},
        verify=True
    )
    
    if snapshots_response.status_code != 200:
        return {
            'statusCode': 500,
            'body': json.dumps('Failed to get snapshots')
        }
    
    snapshots_data = snapshots_response.json()
    snapshots = sorted(snapshots_data['snapshots'], key=lambda x: x['start_time'])
    latest_snapshot = snapshots[-1]['snapshot']
    
    # Restore from snapshot
    restore_body = {
        "indices": "*",
        "ignore_unavailable": True,
        "include_global_state": True
    }
    
    restore_response = requests.post(
        f"{dr_elasticsearch_url}/_snapshot/{snapshot_repo}/{latest_snapshot}/_restore",
        headers={"Authorization": auth_header, "Content-Type": "application/json"},
        data=json.dumps(restore_body),
        verify=True
    )
    
    if restore_response.status_code not in [200, 202]:
        return {
            'statusCode': 500,
            'body': json.dumps('Failed to start restoration')
        }
    
    # Update DNS records
    dns_change = {
        'Changes': [
            {
                'Action': 'UPSERT',
                'ResourceRecordSet': {
                    'Name': domain_name,
                    'Type': 'A',
                    'TTL': 60,
                    'ResourceRecords': [
                        {
                            'Value': dr_elasticsearch_ip
                        }
                    ]
                }
            }
        ]
    }
    
    route53_response = route53.change_resource_record_sets(
        HostedZoneId=route53_hosted_zone_id,
        ChangeBatch=dns_change
    )
    
    return {
        'statusCode': 200,
        'body': json.dumps('Disaster recovery process initiated successfully')
    }
```

### Automated Testing

Implement automated DR testing:

```yaml
# Jenkins pipeline for DR testing
pipeline {
    agent any
    
    environment {
        PRIMARY_ES = "https://elasticsearch-primary.example.com:9200"
        DR_ES = "https://elasticsearch-dr.example.com:9200"
        SNAPSHOT_REPO = "s3_repository"
        TEST_INDEX = "dr-test-index"
    }
    
    stages {
        stage('Prepare Test Data') {
            steps {
                sh """
                # Create test index with known data
                curl -X PUT "${PRIMARY_ES}/${TEST_INDEX}" -H "Content-Type: application/json" -d '
                {
                  "settings": {
                    "number_of_shards": 1,
                    "number_of_replicas": 1
                  }
                }'
                
                # Add test documents
                for i in {1..1000}; do
                  curl -X POST "${PRIMARY_ES}/${TEST_INDEX}/_doc" -H "Content-Type: application/json" -d '
                  {
                    "title": "Test Document ${i}",
                    "content": "This is a test document for DR testing",
                    "created_at": "'$(date -Iseconds)'",
                    "test_id": '${i}'
                  }'
                done
                
                # Force refresh
                curl -X POST "${PRIMARY_ES}/${TEST_INDEX}/_refresh"
                
                # Count documents
                curl -X GET "${PRIMARY_ES}/${TEST_INDEX}/_count"
                """
            }
        }
        
        stage('Create Snapshot') {
            steps {
                sh """
                # Create snapshot
                curl -X PUT "${PRIMARY_ES}/_snapshot/${SNAPSHOT_REPO}/dr-test-snapshot?wait_for_completion=true" \
                  -H "Content-Type: application/json" \
                  -d '{
                    "indices": "${TEST_INDEX}",
                    "ignore_unavailable": true,
                    "include_global_state": false
                  }'
                """
            }
        }
        
        stage('Restore to DR Cluster') {
            steps {
                sh """
                # Delete index if it exists in DR
                curl -X DELETE "${DR_ES}/${TEST_INDEX}"
                
                # Restore from snapshot
                curl -X POST "${DR_ES}/_snapshot/${SNAPSHOT_REPO}/dr-test-snapshot/_restore?wait_for_completion=true" \
                  -H "Content-Type: application/json" \
                  -d '{
                    "indices": "${TEST_INDEX}",
                    "ignore_unavailable": true,
                    "include_global_state": false
                  }'
                
                # Force refresh
                curl -X POST "${DR_ES}/${TEST_INDEX}/_refresh"
                """
            }
        }
        
        stage('Validate Restoration') {
            steps {
                sh """
                # Get original count
                ORIGINAL_COUNT=\$(curl -s -X GET "${PRIMARY_ES}/${TEST_INDEX}/_count" | jq -r '.count')
                
                # Get restored count
                RESTORED_COUNT=\$(curl -s -X GET "${DR_ES}/${TEST_INDEX}/_count" | jq -r '.count')
                
                echo "Original count: \${ORIGINAL_COUNT}"
                echo "Restored count: \${RESTORED_COUNT}"
                
                if [ "\${ORIGINAL_COUNT}" -eq "\${RESTORED_COUNT}" ]; then
                  echo "Validation passed: counts match"
                else
                  echo "Validation failed: counts don't match"
                  exit 1
                fi
                
                # Sample document validation
                ORIGINAL_DOC=\$(curl -s -X GET "${PRIMARY_ES}/${TEST_INDEX}/_doc/1")
                RESTORED_DOC=\$(curl -s -X GET "${DR_ES}/${TEST_INDEX}/_doc/1")
                
                if [ "\$(echo \${ORIGINAL_DOC} | jq -r '._source')" == "\$(echo \${RESTORED_DOC} | jq -r '._source')" ]; then
                  echo "Validation passed: document content matches"
                else
                  echo "Validation failed: document content doesn't match"
                  exit 1
                fi
                """
            }
        }
        
        stage('Cleanup') {
            steps {
                sh """
                # Delete test index
                curl -X DELETE "${PRIMARY_ES}/${TEST_INDEX}"
                curl -X DELETE "${DR_ES}/${TEST_INDEX}"
                
                # Delete snapshot
                curl -X DELETE "${PRIMARY_ES}/_snapshot/${SNAPSHOT_REPO}/dr-test-snapshot"
                """
            }
        }
    }
    
    post {
        success {
            echo 'DR test completed successfully'
        }
        failure {
            echo 'DR test failed'
        }
    }
}
```

## Recovery Time Considerations

Optimize recovery time for faster disaster recovery.

### Optimizing Snapshot Recovery

Tune snapshot recovery for faster restores:

```json
// Increase recovery threads
PUT _cluster/settings
{
  "transient": {
    "indices.recovery.max_bytes_per_sec": "500mb",
    "indices.recovery.concurrent_streams": 8,
    "indices.recovery.concurrent_small_file_streams": 16,
    "cluster.routing.allocation.node_concurrent_recoveries": 8,
    "cluster.routing.allocation.node_initial_primaries_recoveries": 16
  }
}

// Optimize recovery settings during restore
POST _snapshot/s3_repository/snapshot_name/_restore
{
  "indices": "*",
  "ignore_unavailable": true,
  "include_global_state": false,
  "index_settings": {
    "index.number_of_replicas": 0,
    "index.refresh_interval": "-1"
  }
}
```

### Prioritizing Critical Indices

Restore critical indices first:

```bash
# Restore critical indices first
curl -X POST "https://dr-elasticsearch:9200/_snapshot/s3_repository/daily-snap-2023.05.20/_restore" \
  -H "Content-Type: application/json" \
  -d '{
    "indices": "critical-*",
    "ignore_unavailable": true,
    "include_global_state": false
  }'

# After critical indices are restored, restore the rest
curl -X POST "https://dr-elasticsearch:9200/_snapshot/s3_repository/daily-snap-2023.05.20/_restore" \
  -H "Content-Type: application/json" \
  -d '{
    "indices": "*,-critical-*",
    "ignore_unavailable": true,
    "include_global_state": false
  }'
```

### Parallel Recovery

Implement parallel recovery for multiple components:

```bash
#!/bin/bash
# parallel-dr-recovery.sh

# Start Elasticsearch recovery
echo "Starting Elasticsearch recovery..."
./elasticsearch-recovery.sh &
ES_PID=$!

# Start Logstash recovery
echo "Starting Logstash recovery..."
./logstash-recovery.sh &
LS_PID=$!

# Start Kibana recovery
echo "Starting Kibana recovery..."
./kibana-recovery.sh &
KB_PID=$!

# Wait for Elasticsearch recovery to complete
wait $ES_PID
echo "Elasticsearch recovery completed"

# Wait for other components
wait $LS_PID
echo "Logstash recovery completed"

wait $KB_PID
echo "Kibana recovery completed"

echo "All recovery procedures completed"
```

### Tiered Recovery Approach

Implement a tiered recovery approach:

1. **Tier 1 (Immediate Recovery)**:
   - Critical indices with current data (last 24 hours)
   - Minimal Kibana dashboards
   - Essential Logstash pipelines

2. **Tier 2 (Short-term Recovery)**:
   - Recent indices (last 7 days)
   - Most Kibana dashboards
   - All production Logstash pipelines

3. **Tier 3 (Complete Recovery)**:
   - All historical indices
   - All saved objects
   - All pipelines and configurations

## Operational Procedures

Document operational procedures for disaster recovery.

### DR Activation Criteria

Define clear criteria for DR activation:

```markdown
# Disaster Recovery Activation Criteria

DR procedures should be initiated when ANY of the following conditions are met:

## Primary Site Availability
- Complete loss of access to primary site for >30 minutes
- Network connectivity to primary site <50% for >1 hour
- Power outage at primary site with expected duration >2 hours

## Elasticsearch Cluster Health
- Cluster status RED for >15 minutes
- More than 50% of data nodes unreachable for >15 minutes
- Cluster availability <80% for >30 minutes

## Data Loss Events
- Confirmed data corruption affecting critical indices
- Unauthorized deletion of indices
- Failed upgrade resulting in data unavailability

## Security Incidents
- Confirmed unauthorized access to Elasticsearch
- Ransomware or malware affecting the ELK infrastructure
- Data breach requiring system isolation

## Authorization
- DR activation must be approved by at least one of:
  - CTO
  - Head of Infrastructure
  - Lead DevOps Engineer (on-call)
- Document activation time, reason, and approver
```

### Communication Plan

Develop a communication plan for DR events:

```markdown
# Disaster Recovery Communication Plan

## Initial Notification
- **Primary Contact**: On-call Engineer
- **Immediate Notification** (within 15 minutes of incident detection):
  - DevOps Team: Slack #devops-alerts channel + PagerDuty
  - IT Management: Email + SMS
  - Application Teams: Slack #service-status channel

## Status Updates
- **Frequency**: Every 30 minutes or upon significant developments
- **Channels**:
  - Internal: Slack #incident-updates channel
  - External: Status page + Email notifications
- **Content**:
  - Current status
  - Estimated time to recovery
  - Affected services
  - Workarounds if available

## Recovery Notification
- **Upon Successful Recovery**:
  - All stakeholders: Email + Slack
  - End users: Status page update
  - Management: Detailed report within 24 hours

## Communication Templates

### Initial DR Activation
```
SUBJECT: [CRITICAL] ELK Stack Disaster Recovery Activated

The ELK Stack disaster recovery plan has been activated at [TIME] due to [REASON].

Current Status:
- Services affected: [LIST]
- Estimated impact: [DESCRIPTION]
- Current actions: DR environment activation in progress

Next update will follow in 30 minutes or sooner if significant developments occur.

Contact [NAME] at [PHONE] for urgent inquiries.
```

### Status Update
```
SUBJECT: [UPDATE] ELK Stack Disaster Recovery Status

Update as of [TIME]:

- Recovery status: [% complete]
- Systems recovered: [LIST]
- Systems pending: [LIST]
- Current actions: [DESCRIPTION]
- ETA for full recovery: [TIME ESTIMATE]

Next update will follow in 30 minutes.
```

### Recovery Complete
```
SUBJECT: [RESOLVED] ELK Stack Recovery Completed

The ELK Stack has been successfully recovered as of [TIME].

- All services have been restored
- Data has been verified and validated
- Systems are operating normally

Actions required from users:
- [ANY REQUIRED ACTIONS]

A detailed incident report will be provided within 24 hours.
```
```

### Recovery Runbooks

Create detailed runbooks for recovery:

```markdown
# Elasticsearch Disaster Recovery Runbook

## Prerequisites
- DR cluster is operational
- Snapshot repository is accessible
- Authentication credentials are available
- Network access to DR environment is established

## Recovery Procedure

### 1. Assess the Situation
- [ ] Confirm DR activation criteria are met
- [ ] Document the nature and scope of the disaster
- [ ] Assign DR team roles and responsibilities
- [ ] Notify stakeholders according to communication plan

### 2. Prepare the DR Environment
- [ ] Verify DR cluster health
```bash
curl -X GET "https://dr-elasticsearch:9200/_cluster/health?pretty"
```
- [ ] Check available resources
```bash
curl -X GET "https://dr-elasticsearch:9200/_cat/nodes?v"
```
- [ ] Adjust cluster settings for recovery
```bash
curl -X PUT "https://dr-elasticsearch:9200/_cluster/settings" -H "Content-Type: application/json" -d'
{
  "transient": {
    "indices.recovery.max_bytes_per_sec": "500mb",
    "cluster.routing.allocation.node_concurrent_recoveries": 8
  }
}'
```

### 3. Identify Recovery Point
- [ ] Find the most recent successful snapshot
```bash
curl -X GET "https://dr-elasticsearch:9200/_snapshot/s3_repository/_all?pretty"
```
- [ ] Verify snapshot status and contents
```bash
curl -X GET "https://dr-elasticsearch:9200/_snapshot/s3_repository/daily-snap-2023.05.20?verbose=true"
```
- [ ] Document the selected recovery point

### 4. Restore Critical Indices
- [ ] Restore critical indices with priority
```bash
curl -X POST "https://dr-elasticsearch:9200/_snapshot/s3_repository/daily-snap-2023.05.20/_restore" -H "Content-Type: application/json" -d'
{
  "indices": "critical-*",
  "ignore_unavailable": true,
  "include_global_state": false,
  "index_settings": {
    "index.number_of_replicas": 0
  }
}'
```
- [ ] Monitor restoration progress
```bash
curl -X GET "https://dr-elasticsearch:9200/_cat/recovery?v&active_only=true"
```
- [ ] Verify critical indices are accessible
```bash
curl -X GET "https://dr-elasticsearch:9200/critical-*/_search?size=0"
```

### 5. Restore Remaining Indices
- [ ] Restore non-critical indices
```bash
curl -X POST "https://dr-elasticsearch:9200/_snapshot/s3_repository/daily-snap-2023.05.20/_restore" -H "Content-Type: application/json" -d'
{
  "indices": "*,-critical-*",
  "ignore_unavailable": true,
  "include_global_state": false,
  "index_settings": {
    "index.number_of_replicas": 0
  }
}'
```
- [ ] Monitor restoration progress
- [ ] Adjust replica count after recovery
```bash
curl -X PUT "https://dr-elasticsearch:9200/_settings" -H "Content-Type: application/json" -d'
{
  "index": {
    "number_of_replicas": 1
  }
}'
```

### 6. Restore Kibana
- [ ] Deploy Kibana with DR configuration
- [ ] Import saved objects
```bash
curl -X POST "https://dr-kibana:5601/api/saved_objects/_import?overwrite=true" -H "kbn-xsrf: true" --form file=@kibana_saved_objects.ndjson
```
- [ ] Verify Kibana can access restored indices
- [ ] Test critical dashboards

### 7. Redirect Traffic
- [ ] Update DNS records
```bash
aws route53 change-resource-record-sets --hosted-zone-id Z123456789 --change-batch file://dns-changes.json
```
- [ ] Update load balancer configuration
- [ ] Test client connectivity to DR environment

### 8. Validate Recovery
- [ ] Verify data integrity
- [ ] Confirm service functionality
- [ ] Document any data or functionality gaps
- [ ] Notify stakeholders of recovery status

### 9. Post-Recovery Actions
- [ ] Set up monitoring for DR environment
- [ ] Establish ongoing replication if primary is still accessible
- [ ] Plan for return to primary site (if applicable)
- [ ] Schedule post-incident review
```

## Case Studies

Real-world examples of ELK Stack disaster recovery.

### Case Study 1: Cloud Provider Region Outage

```markdown
# DR Case Study: Cloud Provider Region Outage

## Scenario
A major cloud provider experienced a prolonged outage in the US-East region, affecting an ELK deployment used for application logging and monitoring.

## Environment
- ELK Stack v7.10
- 12-node Elasticsearch cluster (3 dedicated master, 9 data nodes)
- 6 Logstash instances
- 3 Kibana instances
- Processing ~10TB of log data daily
- RPO: 15 minutes
- RTO: 4 hours

## DR Strategy
- Cross-region replication to US-West region
- Hourly snapshots to S3
- Auto-follow patterns for new indices
- Global load balancer with health checks
- Automated failover scripts

## Incident Timeline
- **09:15 AM**: Cloud provider reports issues in US-East
- **09:30 AM**: Monitoring alerts trigger for Elasticsearch cluster health
- **09:45 AM**: DR team convenes and confirms region-wide outage
- **10:00 AM**: Decision to activate DR plan
- **10:15 AM**: DR activation script executed
- **10:30 AM**: DNS records updated to point to DR environment
- **10:45 AM**: Core services operational in DR region
- **11:30 AM**: All services verified operational
- **12:00 PM**: All teams notified of successful failover

## Recovery Process
1. DR activation script:
   - Verified DR cluster health
   - Promoted follower indices to leaders
   - Updated DNS records
   - Deployed additional Logstash instances
   - Reconfigured Beat agents

2. Data Validation:
   - Verified recent data was available (~10 minutes of data loss)
   - Confirmed index integrity
   - Validated Kibana dashboards

3. Performance Tuning:
   - Scaled DR cluster to handle full production load
   - Optimized query caching and refresh intervals
   - Monitored and adjusted resources as needed

## Lessons Learned
1. **What Worked Well**:
   - Cross-region replication ensured minimal data loss
   - Automated recovery scripts reduced recovery time
   - Regular DR testing ensured smooth failover
   - Clear roles and responsibilities enabled efficient response

2. **Challenges Faced**:
   - Some Beat agents needed manual reconfiguration
   - Custom Logstash plugins had region-specific dependencies
   - Initial performance in DR region was suboptimal

3. **Improvements Made**:
   - Enhanced Beat agent configuration for multi-region awareness
   - Removed region-specific dependencies in Logstash configurations
   - Increased DR cluster capacity to match production
   - Improved monitoring for replication lag
   - Updated DR runbooks with lessons learned
```

### Case Study 2: Data Corruption Recovery

```markdown
# DR Case Study: Elasticsearch Data Corruption

## Scenario
A mapping conflict resulted in widespread data corruption in production Elasticsearch cluster, affecting recent indices and preventing new data from being indexed properly.

## Environment
- ELK Stack v7.16
- 6-node Elasticsearch cluster
- 4 Logstash instances
- Handling financial transaction logs
- RPO: 5 minutes
- RTO: 2 hours

## DR Strategy
- Point-in-time snapshots every 15 minutes
- Snapshot lifecycle management policies
- Duplicate Logstash pipelines with message queueing
- Manually triggered recovery procedures

## Incident Timeline
- **14:20 PM**: First mapping conflict errors appear in logs
- **14:35 PM**: Alerts trigger for high indexing failure rate
- **14:40 PM**: Engineering team identifies index corruption
- **14:50 PM**: Attempt to fix mapping fails
- **15:00 PM**: Decision to perform recovery from snapshot
- **15:10 PM**: Most recent clean snapshot identified
- **15:15 PM**: Recovery process initiated
- **16:45 PM**: Recovery verified complete
- **17:00 PM**: Normal operations resumed

## Recovery Process
1. Identification:
   - Located the last snapshot before corruption occurred
   - Verified snapshot contents and integrity
   - Identified affected indices

2. Recovery Execution:
   - Closed corrupted indices
   - Restored from last clean snapshot
   - Reindexed data from message queue for the gap period
   - Verified restored indices

3. Service Restoration:
   - Tested indexing with new documents
   - Verified Kibana dashboards and visualizations
   - Gradually redirected traffic to restored services

## Lessons Learned
1. **What Worked Well**:
   - Frequent snapshots limited data loss
   - Message queuing preserved data during outage
   - Clear recovery procedures reduced confusion

2. **Challenges Faced**:
   - Identifying the exact time of corruption was difficult
   - Some in-flight transactions were lost
   - Index aliases had to be manually reassigned

3. **Improvements Made**:
   - Implemented stricter mapping validation in CI/CD pipeline
   - Added pre-production validation for mapping changes
   - Improved monitoring for mapping conflicts
   - Enhanced index alias management in recovery procedures
   - Implemented dual-write pattern for critical indices
```

### Case Study 3: Planned Migration with Zero Downtime

```markdown
# DR Case Study: Zero-Downtime Migration

## Scenario
A planned data center migration required moving a large ELK Stack deployment with minimal disruption to services, using DR procedures to ensure continuity.

## Environment
- ELK Stack v7.14
- 15-node Elasticsearch cluster
- 8 Logstash instances
- 4 Kibana instances
- Over 20TB of indexed data
- RPO: Zero data loss
- RTO: Zero downtime

## DR Strategy
- Cross-cluster replication to new data center
- Blue-green deployment approach
- Dual-write configuration for Logstash
- Staged cutover process with rollback capability

## Migration Timeline
- **Week 1-2**: Set up target infrastructure
- **Week 3-4**: Configure cross-cluster replication
- **Week 5**: Synchronize historical data
- **Day 1**: Enable dual-write for Logstash
- **Day 2**: Verify data consistency
- **Day 3**: Migrate Kibana configuration
- **Day 4**: Test target environment
- **Day 5**: Cutover clients to new environment
- **Week 6**: Decommission old environment

## Migration Process
1. Preparation:
   - Built target environment with enhanced capacity
   - Established network connectivity between environments
   - Set up cross-cluster replication
   - Transferred historical data via snapshots

2. Synchronization:
   - Configured CCR for active indices
   - Set up auto-follow patterns
   - Validated replication status and latency
   - Ensured follower indices were up to date

3. Dual-Write Implementation:
   - Modified Logstash to write to both clusters
   - Verified data consistency across clusters
   - Monitored for any performance impact

4. Kibana Migration:
   - Exported all saved objects
   - Imported to new environment
   - Verified dashboards and visualizations
   - Migrated user settings and roles

5. Cutover:
   - Updated load balancers to direct traffic to new environment
   - Monitored for any issues
   - Kept old environment running as fallback

6. Validation:
   - Verified all components functioning
   - Confirmed data integrity
   - Checked all integrations

## Lessons Learned
1. **What Worked Well**:
   - CCR provided reliable data synchronization
   - Dual-write strategy ensured zero data loss
   - Staged approach reduced risk

2. **Challenges Faced**:
   - Replicating custom plugins required additional steps
   - Some Kibana objects needed manual recreation
   - Coordinating client reconfiguration was complex

3. **Key Takeaways**:
   - DR procedures provided an effective framework for migration
   - Blue-green approach minimized risk
   - Thorough testing was critical to success
   - Communication plan was essential for coordination
```

## Conclusion

A comprehensive disaster recovery strategy is essential for ensuring business continuity and protecting data in your ELK Stack deployment. By implementing regular backups, cross-cluster replication, automated recovery procedures, and thorough testing, you can minimize downtime and data loss when disasters occur.

Remember that disaster recovery is an ongoing process, not a one-time setup. Regularly review and update your DR plans, test recovery procedures, and incorporate lessons learned from incidents and tests to continuously improve your preparedness.

In the next chapter, we'll explore data lifecycle management strategies for the ELK Stack, focusing on how to efficiently manage the complete lifecycle of your data from ingestion to archival or deletion.