# Production Deployment Considerations

## Introduction

Deploying the ELK Stack in production requires careful planning and consideration of various factors to ensure optimal performance, reliability, and security. This chapter covers essential considerations for moving from development to production environments, helping you create a robust ELK Stack deployment that meets your organization's needs.

## Hardware Requirements

### Elasticsearch Hardware Considerations

Elasticsearch is the most resource-intensive component of the ELK Stack and requires careful hardware planning:

#### Memory Requirements

- **JVM Heap Size**: Allocate 50% of available RAM to JVM heap, but not more than 31GB
  - For smaller nodes: `-Xms4g -Xmx4g`
  - For larger nodes: `-Xms16g -Xmx16g` or `-Xms31g -Xmx31g`
- **Remaining Memory**: Reserved for OS file cache (crucial for performance)

#### CPU Requirements

- **Cores**: More cores provide better concurrency
  - Master nodes: 4-8 cores
  - Data nodes: 8-16 cores
  - Coordinating nodes: 4-8 cores
- **CPU Quality**: Higher clock speed and newer generations improve single-thread performance

#### Storage Requirements

- **Type**: SSD strongly recommended for all production deployments
  - Hot nodes: NVMe SSD
  - Warm/cold nodes: SATA SSD or HDD
- **Capacity**: Plan for:
  - Raw data size
  - Replication factor (typically 1x or 2x)
  - Index overhead (typically 10-15%)
  - Growth buffer (typically 30% extra)

Example calculation for 1TB raw data:
```
Base size: 1TB
With replication (1x): 2TB
With overhead (10%): 2.2TB
With growth buffer (30%): ~3TB total storage needed
```

#### Network Requirements

- **Bandwidth**: 10 Gbps recommended for production clusters
- **Latency**: Low latency network essential for cluster communication
- **Isolation**: Separate networks for client and cluster communication when possible

### Logstash Hardware Considerations

Logstash requires significant CPU and memory to process data:

- **CPU**: 4-8 cores for moderate workloads
- **Memory**: 4-8GB (2-4GB for JVM heap)
- **Disk**: Fast local storage for persistent queues (if enabled)
- **Network**: Sufficient bandwidth to handle anticipated data flow

### Kibana Hardware Considerations

Kibana is less resource-intensive but still requires adequate resources:

- **CPU**: 2-4 cores
- **Memory**: 2-4GB
- **Disk**: Only needed for logging (minimal)
- **Network**: Low latency connection to Elasticsearch

## Capacity Planning

### Data Volume Considerations

Understand your data to properly size your deployment:

- **Daily Ingest Volume**: How much data you need to process per day
- **Total Data Volume**: How much data you need to store (including retention period)
- **Growth Rate**: How quickly your data volume is increasing
- **Query Patterns**: Types and frequency of searches and aggregations

Example sizing matrix:

| Data Volume | Recommended Deployment |
|-------------|------------------------|
| < 25GB/day  | 3 nodes, 16GB RAM, 4 cores |
| 25-100GB/day | 3-5 nodes, 32GB RAM, 8 cores |
| 100-500GB/day | 5-10 nodes, 32-64GB RAM, 16 cores |
| > 500GB/day | 10+ nodes, 64GB RAM, 16+ cores |

### Concurrent User Considerations

Scale your deployment based on user load:

- **Kibana Users**: Number of concurrent Kibana users affects Kibana and Elasticsearch
- **API Clients**: Applications making direct API calls to Elasticsearch
- **Query Complexity**: Complex queries require more resources

Recommendations:
- 1 Kibana instance per 50-100 concurrent users
- Scale coordinating nodes based on API client load
- Implement caching for repetitive queries

### Workload Type Considerations

Different workloads require different optimizations:

- **Search-Heavy Workloads**: More replica shards and search-optimized nodes
- **Indexing-Heavy Workloads**: More primary shards and indexing-optimized nodes
- **Aggregation-Heavy Workloads**: More memory for data nodes
- **Mixed Workloads**: Separation of concerns with dedicated node types

## Deployment Architecture

### Single-Node Deployment

Not recommended for production but described for completeness:

- All components on a single server
- Limited redundancy and scalability
- Suitable only for small proof-of-concepts

### Small Production Deployment

For smaller organizations or departments:

- 3-node Elasticsearch cluster
  - Each node with master, data, and ingest roles
- 1-2 Logstash instances
- 1-2 Kibana instances behind a load balancer
- Filebeat installed on all source servers

Example server specs:
- Each Elasticsearch node: 32GB RAM, 8 cores, 2TB SSD
- Logstash: 16GB RAM, 4 cores
- Kibana: 8GB RAM, 2 cores

### Medium Production Deployment

For medium-sized organizations:

- 3 dedicated master nodes
- 4-8 data nodes
- 2-4 coordinating nodes
- 2-4 Logstash instances
- 2+ Kibana instances behind a load balancer
- Beats installed on all source servers

Example server specs:
- Master nodes: 16GB RAM, 4 cores, 100GB SSD
- Data nodes: 64GB RAM, 16 cores, 4TB SSD
- Coordinating nodes: 32GB RAM, 8 cores
- Logstash: 32GB RAM, 8 cores
- Kibana: 16GB RAM, 4 cores

### Large Enterprise Deployment

For large enterprises:

- 3-5 dedicated master nodes
- Hot-warm-cold architecture:
  - 5-10+ hot nodes (SSD)
  - 10-20+ warm nodes (SSD/HDD)
  - 5-10+ cold nodes (HDD)
- 5-10+ coordinating nodes
- 5-10+ ingest nodes
- Logstash server farm with load balancing
- Multiple Kibana instances with load balancing
- Cross-cluster search for multiple use cases
- Centralized Beats management

Example server specs:
- Master nodes: 32GB RAM, 8 cores, 100GB SSD
- Hot nodes: 64GB RAM, 16 cores, 2TB NVMe SSD
- Warm nodes: 64GB RAM, 16 cores, 10TB SSD
- Cold nodes: 64GB RAM, 8 cores, 20TB HDD
- Coordinating nodes: 32GB RAM, 16 cores
- Ingest nodes: 32GB RAM, 8 cores
- Logstash: 32GB RAM, 16 cores
- Kibana: 16GB RAM, 8 cores

### Geographic Distribution

For globally distributed organizations:

- Regional Elasticsearch clusters
- Cross-cluster replication (CCR) for data redundancy
- Cross-cluster search (CCS) for unified search experience
- Local Kibana instances connecting to regional clusters
- Global load balancing

## System Configuration

### Operating System Settings

#### File Descriptors

Elasticsearch requires a high number of file descriptors:

```bash
# /etc/security/limits.conf
elasticsearch soft nofile 65536
elasticsearch hard nofile 65536
```

#### Virtual Memory

Set vm.max_map_count to allow Elasticsearch to use memory-mapped files:

```bash
# /etc/sysctl.conf
vm.max_map_count = 262144
```

Apply the setting:
```bash
sudo sysctl -w vm.max_map_count=262144
```

#### Swapping

Disable swapping to prevent performance degradation:

```bash
# Option 1: Disable swap completely
sudo swapoff -a

# Option 2: Minimize swapping
# /etc/sysctl.conf
vm.swappiness = 1
```

Ensure Elasticsearch memory is locked:

```yaml
# elasticsearch.yml
bootstrap.memory_lock: true
```

#### Filesystem Cache

Ensure sufficient filesystem cache is available:

- Reserve 50% of memory for filesystem cache
- Monitor cache hit rates with Elasticsearch monitoring tools

### JVM Configuration

Configure JVM settings to optimize performance:

```
# jvm.options

# Heap size
-Xms16g
-Xmx16g

# GC settings
-XX:+UseG1GC
-XX:G1ReservePercent=25
-XX:InitiatingHeapOccupancyPercent=30

# GC logging
-Xlog:gc*,gc+age=trace,safepoint:file=logs/gc.log:utctime,pid,tags:filecount=32,filesize=64m
```

JVM tuning tips:
- Use G1GC for heaps larger than 4GB
- Keep min and max heap sizes identical
- Don't exceed 31GB heap size due to JVM pointer compression limits
- Monitor GC frequency and duration

### Network Configuration

Configure networking for optimal performance:

- **Bind to specific interfaces**:
  ```yaml
  # elasticsearch.yml
  network.host: 192.168.1.10
  ```

- **Configure discovery**:
  ```yaml
  # elasticsearch.yml
  discovery.seed_hosts: ["192.168.1.10", "192.168.1.11", "192.168.1.12"]
  cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]
  ```

- **Configure transport and HTTP ports**:
  ```yaml
  # elasticsearch.yml
  http.port: 9200
  transport.port: 9300
  ```

### Disk I/O Configuration

Optimize storage for Elasticsearch:

- **I/O Scheduler**: Use the 'noop' or 'deadline' scheduler for SSDs
  ```bash
  echo noop > /sys/block/sda/queue/scheduler
  ```

- **Mount Options**: Use appropriate mount options for your filesystem
  ```
  # /etc/fstab
  /dev/sda1 /var/lib/elasticsearch ext4 noatime,data=writeback,discard 0 0
  ```

## Elasticsearch Configuration

### Cluster Configuration

Configure the cluster to match your architecture:

```yaml
# elasticsearch.yml

# Cluster name
cluster.name: production-elk-cluster

# Node roles
node.roles: [ master, data, ingest ]  # For combined nodes
# OR
node.roles: [ master ]  # For dedicated master nodes
# OR
node.roles: [ data ]  # For dedicated data nodes

# Node attributes for tiered storage
node.attr.data: hot  # For hot nodes
# OR
node.attr.data: warm  # For warm nodes
```

### Shard Configuration

Plan your sharding strategy:

- **Primary Shards**: Determined at index creation
  - Rule of thumb: Aim for shards between 20-40GB
  - Too many small shards create overhead
  - Too few large shards limit parallelism

- **Replica Shards**: Can be changed at any time
  - At least 1 replica for production
  - More replicas improve search performance but increase storage needs

Example template for time-based indices:

```
PUT _index_template/logs_template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 5,
      "number_of_replicas": 1
    }
  }
}
```

### Memory Configuration

Configure memory settings for optimal performance:

```yaml
# elasticsearch.yml

# Field data cache
indices.fielddata.cache.size: 20%

# Shard request cache
indices.requests.cache.size: 2%

# Query cache
indices.queries.cache.size: 5%
```

### Recovery Configuration

Configure recovery settings to handle node restarts and failures:

```yaml
# elasticsearch.yml

# Recovery throttling
cluster.routing.allocation.node_concurrent_recoveries: 4
indices.recovery.max_bytes_per_sec: 100mb

# Allocation settings
cluster.routing.allocation.cluster_concurrent_rebalance: 2
```

### Thread Pool Configuration

Configure thread pools based on workload:

```yaml
# elasticsearch.yml

# Search thread pool
thread_pool.search.size: 12
thread_pool.search.queue_size: 1000

# Write thread pool
thread_pool.write.size: 8
thread_pool.write.queue_size: 1000
```

## Logstash Configuration

### Pipeline Configuration

Configure Logstash for optimal performance:

```yaml
# logstash.yml

# Pipeline settings
pipeline.workers: 4  # Number of threads (typically number of cores)
pipeline.batch.size: 1000  # Events per worker
pipeline.batch.delay: 50  # Wait time in milliseconds

# Queue settings (for persistent queues)
queue.type: persisted
queue.max_bytes: 4gb
```

### Multiple Pipelines

Organize complex workflows with multiple pipelines:

```yaml
# pipelines.yml

- pipeline.id: apache
  path.config: "/etc/logstash/conf.d/apache.conf"
  pipeline.workers: 2

- pipeline.id: mysql
  path.config: "/etc/logstash/conf.d/mysql.conf"
  pipeline.workers: 2

- pipeline.id: syslog
  path.config: "/etc/logstash/conf.d/syslog.conf"
  pipeline.workers: 1
```

### Input Configuration

Configure inputs for reliability:

```ruby
input {
  beats {
    port => 5044
    client_inactivity_timeout => 3600
    ssl => true
    ssl_certificate => "/etc/logstash/ssl/logstash.crt"
    ssl_key => "/etc/logstash/ssl/logstash.key"
  }
}
```

### Filter Optimization

Optimize filters for better performance:

```ruby
filter {
  # Process conditionally to avoid unnecessary work
  if [type] == "apache" {
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
      tag_on_failure => ["_grokparsefailure", "apache_parse_failure"]
    }
  }
  
  # Use efficient patterns
  grok {
    patterns_dir => ["/etc/logstash/patterns"]
    match => { "message" => "%{MY_CUSTOM_PATTERN}" }
  }
  
  # Drop unnecessary fields early
  mutate {
    remove_field => ["field1", "field2"]
  }
}
```

### Output Configuration

Configure outputs for reliability and performance:

```ruby
output {
  elasticsearch {
    hosts => ["es1:9200", "es2:9200", "es3:9200"]
    manage_template => false
    index => "logs-%{+YYYY.MM.dd}"
    
    # Bulk settings
    bulk_size => 5000
    flush_size => 10000
    
    # Retry settings
    retry_initial_interval => 2
    retry_max_interval => 64
    retry_on_conflict => 5
    
    # Connection pool
    pool_size => 10
  }
}
```

## Kibana Configuration

### Basic Configuration

Configure Kibana for production:

```yaml
# kibana.yml

# Server settings
server.host: "0.0.0.0"
server.port: 5601
server.name: "kibana-1"

# Elasticsearch connection
elasticsearch.hosts: ["http://es1:9200", "http://es2:9200", "http://es3:9200"]
elasticsearch.sniffOnStart: true
elasticsearch.requestTimeout: 60000

# Security settings (when X-Pack security is enabled)
elasticsearch.username: "kibana_system"
elasticsearch.password: "${KIBANA_PASSWORD}"

# SSL settings
server.ssl.enabled: true
server.ssl.certificate: "/etc/kibana/ssl/kibana.crt"
server.ssl.key: "/etc/kibana/ssl/kibana.key"
```

### Memory Optimization

Optimize Node.js memory for Kibana:

```bash
NODE_OPTIONS="--max-old-space-size=4096" bin/kibana
```

Or in `kibana.yml`:

```yaml
node.options: "--max-old-space-size=4096"
```

### Load Balancing

Deploy multiple Kibana instances behind a load balancer:

- Use sticky sessions to maintain user context
- Health check path: `/api/status`
- Configure session timeouts appropriately

Example NGINX configuration:

```nginx
upstream kibana {
    server kibana1:5601 max_fails=3 fail_timeout=30s;
    server kibana2:5601 max_fails=3 fail_timeout=30s;
}

server {
    listen 443 ssl;
    server_name kibana.example.com;

    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    location / {
        proxy_pass http://kibana;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## Security Considerations

### Authentication and Authorization

Implement proper authentication and authorization:

```yaml
# elasticsearch.yml
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.http.ssl.enabled: true
```

Create appropriate roles and users:

```
# Create roles
POST /_security/role/logs_reader
{
  "cluster": ["monitor"],
  "indices": [
    {
      "names": ["logs-*"],
      "privileges": ["read", "view_index_metadata"]
    }
  ]
}

# Create users
POST /_security/user/logs_user
{
  "password": "secure_password",
  "roles": ["logs_reader"],
  "full_name": "Logs Reader"
}
```

### Network Security

Secure your network communication:

- Use private networks for cluster communication
- Implement firewalls to restrict access
- Use VPN or bastion hosts for remote administration
- Configure TLS for all communication

Example firewall rules:

```bash
# Allow Elasticsearch HTTP API
sudo iptables -A INPUT -p tcp --dport 9200 -s 10.0.0.0/24 -j ACCEPT

# Allow Elasticsearch transport
sudo iptables -A INPUT -p tcp --dport 9300 -s 10.0.0.0/24 -j ACCEPT

# Allow Logstash Beats input
sudo iptables -A INPUT -p tcp --dport 5044 -s 10.0.0.0/24 -j ACCEPT

# Allow Kibana
sudo iptables -A INPUT -p tcp --dport 5601 -s 10.0.0.0/24 -j ACCEPT
```

### Encryption

Enable encryption for data in transit:

```yaml
# elasticsearch.yml
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12

xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: http.p12
```

For Logstash:

```ruby
input {
  beats {
    port => 5044
    ssl => true
    ssl_certificate => "/etc/logstash/ssl/logstash.crt"
    ssl_key => "/etc/logstash/ssl/logstash.key"
    ssl_verify_mode => "force_peer"
  }
}

output {
  elasticsearch {
    hosts => ["https://es1:9200", "https://es2:9200"]
    ssl => true
    ssl_certificate_verification => true
    cacert => "/etc/logstash/ssl/ca.crt"
  }
}
```

### Auditing

Enable audit logging to track security events:

```yaml
# elasticsearch.yml
xpack.security.audit.enabled: true
xpack.security.audit.logfile.events.include: ["authentication_success", "authentication_failed", "access_denied", "connection_denied", "tampered_request"]
```

## High Availability

### Elasticsearch HA

Ensure Elasticsearch high availability:

- Deploy at least 3 master-eligible nodes
- Use proper discovery configuration
- Distribute nodes across availability zones
- Configure appropriate shard allocation awareness

```yaml
# elasticsearch.yml

# Shard allocation awareness
cluster.routing.allocation.awareness.attributes: zone
node.attr.zone: zone1  # Different for each node

# Fault detection
cluster.fault_detection.ping_retries: 3
cluster.fault_detection.ping_timeout: 30s
```

### Logstash HA

Ensure Logstash high availability:

- Deploy multiple Logstash instances
- Use load balancing for inputs
- Enable persistent queues
- Configure proper retry mechanisms

For Beats input:

```ruby
input {
  beats {
    port => 5044
    ssl => true
    ssl_certificate => "/etc/logstash/ssl/logstash.crt"
    ssl_key => "/etc/logstash/ssl/logstash.key"
  }
}
```

For Elasticsearch output:

```ruby
output {
  elasticsearch {
    hosts => ["es1:9200", "es2:9200", "es3:9200"]
    retry_initial_interval => 2
    retry_max_interval => 64
    retry_on_conflict => 5
  }
}
```

### Kibana HA

Ensure Kibana high availability:

- Deploy multiple Kibana instances
- Use load balancing with sticky sessions
- Configure proper session management
- Implement health checks

## Monitoring and Alerting

### Stack Monitoring

Enable monitoring of the ELK Stack:

```yaml
# elasticsearch.yml
xpack.monitoring.collection.enabled: true

# logstash.yml
xpack.monitoring.enabled: true
xpack.monitoring.elasticsearch.hosts: ["http://es1:9200", "http://es2:9200"]

# kibana.yml
xpack.monitoring.enabled: true
xpack.monitoring.ui.container.elasticsearch.enabled: true
```

### Metricbeat Monitoring

Use Metricbeat to collect metrics:

```yaml
metricbeat.modules:
- module: elasticsearch
  metricsets:
    - node
    - node_stats
    - index
    - index_recovery
  period: 10s
  hosts: ["http://es1:9200", "http://es2:9200"]
  username: "${ES_MONITOR_USER}"
  password: "${ES_MONITOR_PASSWORD}"

- module: logstash
  metricsets:
    - node
    - node_stats
  period: 10s
  hosts: ["http://logstash1:9600", "http://logstash2:9600"]

- module: kibana
  metricsets:
    - stats
  period: 10s
  hosts: ["http://kibana1:5601", "http://kibana2:5601"]
  username: "${KIBANA_MONITOR_USER}"
  password: "${KIBANA_MONITOR_PASSWORD}"
```

### Alerts and Watchs

Configure alerts for critical conditions:

```
PUT _watcher/watch/cluster_health_watch
{
  "trigger": {
    "schedule": {
      "interval": "1m"
    }
  },
  "input": {
    "http": {
      "request": {
        "url": "http://localhost:9200/_cluster/health",
        "auth": {
          "basic": {
            "username": "elastic",
            "password": "${ELASTIC_PASSWORD}"
          }
        }
      }
    }
  },
  "condition": {
    "compare": {
      "ctx.payload.status": {
        "eq": "red"
      }
    }
  },
  "actions": {
    "email_admin": {
      "email": {
        "to": "admin@example.com",
        "subject": "Cluster Health Alert",
        "body": {
          "text": "Cluster health is RED. Please investigate immediately."
        }
      }
    }
  }
}
```

## Backup and Disaster Recovery

### Snapshot Repository Configuration

Configure a snapshot repository:

```
PUT /_snapshot/backup_repo
{
  "type": "fs",
  "settings": {
    "location": "/mnt/elasticsearch-snapshots",
    "compress": true
  }
}
```

Or using S3:

```
PUT /_snapshot/s3_repo
{
  "type": "s3",
  "settings": {
    "bucket": "elk-backups",
    "region": "us-east-1",
    "compress": true
  }
}
```

### Automated Snapshots

Schedule regular snapshots:

```
PUT /_slm/policy/daily_snapshot
{
  "schedule": "0 30 1 * * ?", 
  "name": "<daily-snap-{now/d}>",
  "repository": "backup_repo",
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
```

### Disaster Recovery Planning

Develop a comprehensive disaster recovery plan:

1. **Regular Testing**: Test recovery processes regularly
2. **Recovery Time Objective (RTO)**: Define acceptable downtime
3. **Recovery Point Objective (RPO)**: Define acceptable data loss
4. **Documentation**: Document all recovery procedures
5. **Cross-Region Strategy**: Consider cross-region replication for critical data

Example cross-region configuration:

```
# Configure remote cluster
PUT /_cluster/settings
{
  "persistent": {
    "cluster.remote.dr_cluster.seeds": [
      "es-dr-1.example.com:9300",
      "es-dr-2.example.com:9300"
    ]
  }
}

# Set up CCR (Cross-Cluster Replication)
PUT /follower_index/_ccr/follow
{
  "remote_cluster": "dr_cluster",
  "leader_index": "leader_index"
}
```

## Maintenance Strategies

### Rolling Upgrades

Perform rolling upgrades to minimize downtime:

1. Disable shard allocation
   ```
   PUT /_cluster/settings
   {
     "persistent": {
       "cluster.routing.allocation.enable": "primaries"
     }
   }
   ```

2. Stop and upgrade one node at a time
3. Restart the node and confirm it joins the cluster
4. Re-enable shard allocation
   ```
   PUT /_cluster/settings
   {
     "persistent": {
       "cluster.routing.allocation.enable": null
     }
   }
   ```

5. Wait for the cluster to stabilize before moving to the next node

### Index Management

Implement proper index management:

- Use Index Lifecycle Management (ILM) for automated index management
- Implement index rollover for time-series data
- Set up index aliases for transparent transitions

```
PUT _ilm/policy/logs_policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_size": "50GB",
            "max_age": "1d"
          },
          "set_priority": {
            "priority": 100
          }
        }
      },
      "warm": {
        "min_age": "3d",
        "actions": {
          "allocate": {
            "include": {
              "data": "warm"
            }
          },
          "set_priority": {
            "priority": 50
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "allocate": {
            "include": {
              "data": "cold"
            }
          },
          "set_priority": {
            "priority": 0
          }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

### Performance Tuning

Regularly tune performance:

- Monitor cluster metrics for bottlenecks
- Optimize query patterns
- Adjust thread pools and queues based on workload
- Revisit shard allocation as data grows

## Containerization and Orchestration

### Docker Deployment

Deploy with Docker for greater flexibility:

```yaml
# docker-compose.yml
version: '3'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.8.0
    environment:
      - node.name=es01
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es02,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms8g -Xmx8g"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata01:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
    networks:
      - elastic

  logstash:
    image: docker.elastic.co/logstash/logstash:8.8.0
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml
    ports:
      - 5044:5044
    networks:
      - elastic
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.8.0
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - 5601:5601
    networks:
      - elastic
    depends_on:
      - elasticsearch

volumes:
  esdata01:

networks:
  elastic:
```

### Kubernetes Deployment

Deploy on Kubernetes for scalability and resilience:

```yaml
# elasticsearch-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
spec:
  serviceName: elasticsearch
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:8.8.0
        resources:
          limits:
            cpu: 2
            memory: 8Gi
          requests:
            cpu: 1
            memory: 4Gi
        env:
        - name: cluster.name
          value: k8s-logs
        - name: node.name
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: discovery.seed_hosts
          value: "elasticsearch-0.elasticsearch,elasticsearch-1.elasticsearch,elasticsearch-2.elasticsearch"
        - name: cluster.initial_master_nodes
          value: "elasticsearch-0,elasticsearch-1,elasticsearch-2"
        - name: ES_JAVA_OPTS
          value: "-Xms4g -Xmx4g"
        ports:
        - containerPort: 9200
          name: http
        - containerPort: 9300
          name: transport
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 200Gi
```

### Elastic Cloud Kubernetes (ECK)

Use ECK for Kubernetes-native ELK deployment:

```yaml
# elasticsearch-eck.yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch
spec:
  version: 8.8.0
  nodeSets:
  - name: master
    count: 3
    config:
      node.roles: ["master"]
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          resources:
            limits:
              memory: 4Gi
              cpu: 2
  - name: hot
    count: 3
    config:
      node.roles: ["data", "ingest"]
      node.attr.data: hot
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          resources:
            limits:
              memory: 16Gi
              cpu: 4
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 200Gi
```

## Cost Optimization

### Resource Efficiency

Optimize resource usage to minimize costs:

- Right-size your instances based on actual usage
- Implement auto-scaling where possible
- Use spot/preemptible instances for non-critical nodes
- Optimize shard count to reduce overhead

### Data Lifecycle Management

Implement effective data lifecycle management:

- Move older data to cheaper storage tiers
- Implement index rollover and ILM policies
- Set appropriate retention periods
- Use downsampling for metrics data

### Cloud Cost Management

Manage cloud costs effectively:

- Use reserved instances for stable workloads
- Implement auto-scaling based on usage patterns
- Leverage storage tiering (e.g., AWS S3 lifecycle policies)
- Monitor and alert on unexpected cost increases

## Conclusion

Deploying the ELK Stack in production requires careful planning across multiple dimensions including hardware, architecture, security, high availability, and maintenance. By following the guidelines and best practices outlined in this chapter, you can create a robust, scalable, and secure ELK Stack deployment that meets your organization's needs for log collection, analysis, and visualization.

Remember that a production ELK deployment is not a one-time task but an ongoing process that requires regular monitoring, tuning, and adaptation as your data volume and requirements evolve. Regular reviews of your architecture, performance metrics, and security posture will ensure your ELK Stack continues to deliver value while minimizing operational overhead and costs.

In the next chapter, we'll explore cloud-based deployments of the ELK Stack, including managed services and self-managed options across major cloud providers.