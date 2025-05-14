# High Availability

This chapter covers high availability (HA) architectures and strategies for the ELK Stack, ensuring your monitoring and logging infrastructure remains operational during failures.

## Table of Contents
- [Introduction to High Availability](#introduction-to-high-availability)
- [Elasticsearch High Availability](#elasticsearch-high-availability)
- [Logstash High Availability](#logstash-high-availability)
- [Kibana High Availability](#kibana-high-availability)
- [Beats High Availability](#beats-high-availability)
- [Cross-Cluster High Availability](#cross-cluster-high-availability)
- [Load Balancing](#load-balancing)
- [Monitoring HA Deployments](#monitoring-ha-deployments)
- [Failure Recovery Strategies](#failure-recovery-strategies)
- [Real-World HA Architectures](#real-world-ha-architectures)
- [HA Testing and Validation](#ha-testing-and-validation)

## Introduction to High Availability

High availability ensures that your ELK Stack remains operational and accessible even when components fail or require maintenance.

### Key Principles of High Availability

1. **Redundancy**: Duplicate critical components to eliminate single points of failure
2. **Monitoring**: Continuously monitor system health to detect and address issues
3. **Automatic Failover**: Redirect traffic away from failed components without manual intervention
4. **Load Distribution**: Spread workloads across multiple nodes to prevent overloading
5. **Geographic Distribution**: Deploy across multiple data centers or regions to survive localized failures
6. **Resiliency**: Design systems to degrade gracefully rather than fail completely
7. **Independent Failure Domains**: Ensure that the failure of one component doesn't cascade

### Availability Tiers

Common availability tiers and their characteristics:

| Tier | Uptime % | Downtime Per Year | Architecture                            |
|------|----------|-------------------|----------------------------------------|
| 99%  | 99%      | 3.65 days         | Basic redundancy                       |
| 99.9% | 99.9%    | 8.76 hours        | Multi-node, single zone                |
| 99.99% | 99.99%   | 52.6 minutes      | Multi-node, multi-zone, automated repair |
| 99.999% | 99.999%  | 5.26 minutes      | Multi-node, multi-region, minimal dependencies |

### Downtime Types

Understanding the nature of potential downtime:

1. **Planned Downtime**:
   - Software updates and upgrades
   - Configuration changes
   - Hardware maintenance
   - Scaling operations

2. **Unplanned Downtime**:
   - Hardware failures
   - Software crashes
   - Network issues
   - External dependencies failing
   - Performance degradation (effective downtime)

## Elasticsearch High Availability

Elasticsearch's distributed nature provides built-in high availability features that can be configured for various levels of resilience.

### Node Roles and Distribution

Configure specialized node roles for optimal HA:

```yaml
# Master eligible node
node.roles: ["master"]
cluster.name: elk-production
node.name: master-node-1
network.host: 192.168.1.10
discovery.seed_hosts: ["192.168.1.10", "192.168.1.11", "192.168.1.12"]
cluster.initial_master_nodes: ["master-node-1", "master-node-2", "master-node-3"]

# Data node
node.roles: ["data"]
cluster.name: elk-production
node.name: data-node-1
network.host: 192.168.1.20
discovery.seed_hosts: ["192.168.1.10", "192.168.1.11", "192.168.1.12"]

# Ingest node
node.roles: ["ingest"]
cluster.name: elk-production
node.name: ingest-node-1
network.host: 192.168.1.30
discovery.seed_hosts: ["192.168.1.10", "192.168.1.11", "192.168.1.12"]

# Coordinating node
node.roles: []
cluster.name: elk-production
node.name: coordinating-node-1
network.host: 192.168.1.40
discovery.seed_hosts: ["192.168.1.10", "192.168.1.11", "192.168.1.12"]
```

### Master Node Quorum

Ensure reliable master election with proper quorum settings:

```yaml
# Minimum master nodes for 3 master-eligible nodes
discovery.zen.minimum_master_nodes: 2  # For Elasticsearch 6.x and earlier

# For Elasticsearch 7.x+, quorum is automatically managed but you must set:
cluster.initial_master_nodes: ["master-node-1", "master-node-2", "master-node-3"]
```

### Shard Allocation and Replication

Configure shard allocation for data redundancy:

```yaml
# In elasticsearch.yml or via API
index.number_of_shards: 3
index.number_of_replicas: 1  # One replica means two copies total

# Prevent data loss during network partitions
cluster.routing.allocation.enable: all
cluster.routing.allocation.cluster_concurrent_rebalance: 2
cluster.routing.allocation.node_concurrent_recoveries: 2
cluster.routing.allocation.node_initial_primaries_recoveries: 4
```

Create an index template with HA-focused settings:

```json
PUT _template/logs_template
{
  "index_patterns": ["logs-*"],
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "routing.allocation.total_shards_per_node": 500,
    "routing.allocation.require._name": null,
    "routing.allocation.exclude._name": null,
    "routing.allocation.include._name": null
  }
}
```

### Awareness and Shard Distribution

Ensure shards are distributed across physical infrastructure:

```yaml
# Configure awareness attributes
node.attr.zone: zone1  # On nodes in first zone
node.attr.zone: zone2  # On nodes in second zone
node.attr.zone: zone3  # On nodes in third zone

# Configure shard allocation awareness
cluster.routing.allocation.awareness.attributes: zone
cluster.routing.allocation.awareness.force.zone.values: zone1,zone2,zone3
```

### Snapshotting for Recovery

Configure automated snapshots to protect against data loss:

```json
// Register repository
PUT _snapshot/s3_repository
{
  "type": "s3",
  "settings": {
    "bucket": "elasticsearch-snapshots",
    "region": "us-east-1",
    "role_arn": "arn:aws:iam::123456789012:role/elasticsearch-snapshot-role"
  }
}

// Create snapshot policy
PUT _slm/policy/daily-snapshots
{
  "schedule": "0 30 1 * * ?", 
  "name": "<logs-{now/d}>",
  "repository": "s3_repository",
  "config": {
    "indices": ["logs-*"],
    "ignore_unavailable": true,
    "include_global_state": false
  },
  "retention": {
    "expire_after": "30d",
    "min_count": 5,
    "max_count": 50
  }
}
```

### Cluster-Level Settings

Optimize cluster-level settings for high availability:

```yaml
# Improve recovery speed
cluster.routing.allocation.node_concurrent_recoveries: 4
cluster.routing.allocation.node_initial_primaries_recoveries: 8
indices.recovery.max_bytes_per_sec: 100mb

# Prevent split-brain scenarios
discovery.zen.minimum_master_nodes: 2  # For 3 master nodes (ES 6.x)
gateway.recover_after_nodes: 2
gateway.expected_nodes: 3
gateway.recover_after_time: 5m

# Fault detection
cluster.fault_detection.leader_check.timeout: 10s
cluster.fault_detection.follower_check.timeout: 10s
```

## Logstash High Availability

Design Logstash deployments for resilience and throughput.

### Input Redundancy

Configure redundant inputs to prevent data loss:

```ruby
# Redundant Beats input
input {
  beats {
    port => 5044
    ssl => true
    ssl_certificate => "/etc/logstash/certs/logstash.crt"
    ssl_key => "/etc/logstash/certs/logstash.key"
    ssl_certificate_authorities => ["/etc/logstash/certs/ca.crt"]
  }
}

# Kafka input for better resilience
input {
  kafka {
    bootstrap_servers => "kafka1:9092,kafka2:9092,kafka3:9092"
    topics => ["logs"]
    group_id => "logstash_group"
    consumer_threads => 5
    decorate_events => true
    codec => "json"
  }
}

# Redis input as a buffer
input {
  redis {
    host => "redis-master"
    port => 6379
    data_type => "list"
    key => "logstash"
    password => "${REDIS_PASSWORD}"
  }
}
```

### Persistent Queues

Enable persistent queues to prevent data loss during outages:

```ruby
# In logstash.yml
queue.type: persisted
queue.max_bytes: 8gb
path.queue: /var/lib/logstash/queue
queue.checkpoint.writes: 1024
```

### Multiple Pipeline Configuration

Configure multiple pipelines for isolation and resilience:

```yaml
# pipelines.yml
- pipeline.id: apache
  path.config: "/etc/logstash/conf.d/apache.conf"
  pipeline.workers: 2
  queue.type: persisted
  queue.max_bytes: 1gb
  
- pipeline.id: mysql
  path.config: "/etc/logstash/conf.d/mysql.conf"
  pipeline.workers: 2
  queue.type: persisted
  queue.max_bytes: 1gb
  
- pipeline.id: system
  path.config: "/etc/logstash/conf.d/system.conf"
  pipeline.workers: 2
  queue.type: persisted
  queue.max_bytes: 2gb
```

### Output Redundancy and Failure Handling

Configure outputs with retry logic and failover capabilities:

```ruby
output {
  elasticsearch {
    hosts => ["es1:9200", "es2:9200", "es3:9200"]
    sniffing => true
    manage_template => false
    index => "logs-%{+YYYY.MM.dd}"
    
    # Retry configuration
    retry_initial_interval => 2
    retry_max_interval => 64
    retry_on_conflict => 5
    max_retries => 10
    
    # Load balancing
    loadbalance => true
  }
  
  # Dead letter queue for failed events
  if "_elasticsearch_failure" in [tags] {
    file {
      path => "/var/log/logstash/failed_events_%{+YYYY-MM-dd}.log"
      codec => json
    }
  }
}
```

Enable the dead letter queue:

```yaml
# In logstash.yml
dead_letter_queue.enable: true
path.dead_letter_queue: /var/lib/logstash/dead_letter_queue
```

### Deployment Architecture

Deploy Logstash in a highly available configuration:

1. **Load-balanced Tier**:
   - Multiple Logstash instances behind a load balancer
   - Each with identical configuration
   - Health checks to remove unhealthy instances

2. **Specialized Tier**:
   - Dedicated Logstash instances for different data sources
   - Custom scaling based on workload characteristics
   - Isolated resource allocation

3. **Buffer Tier**:
   - Kafka or Redis as a central buffer
   - Decouples data collection from processing
   - Provides backpressure management

### Monitoring and Auto-healing

Configure monitoring and auto-healing:

```yaml
# In logstash.yml
xpack.monitoring.enabled: true
xpack.monitoring.elasticsearch.hosts: ["https://monitoring-es:9200"]
xpack.monitoring.elasticsearch.username: "${MONITORING_USERNAME}"
xpack.monitoring.elasticsearch.password: "${MONITORING_PASSWORD}"
```

Kubernetes-based auto-healing:

```yaml
# Kubernetes deployment with probes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash
spec:
  replicas: 3
  selector:
    matchLabels:
      app: logstash
  template:
    metadata:
      labels:
        app: logstash
    spec:
      containers:
      - name: logstash
        image: docker.elastic.co/logstash/logstash:8.1.0
        ports:
        - containerPort: 5044
        readinessProbe:
          httpGet:
            path: /
            port: 9600
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /
            port: 9600
          initialDelaySeconds: 60
          periodSeconds: 30
        resources:
          requests:
            cpu: "1"
            memory: "2Gi"
          limits:
            cpu: "2"
            memory: "4Gi"
```

## Kibana High Availability

Configure Kibana for high availability and load distribution.

### Multiple Kibana Instances

Deploy multiple Kibana instances with shared state:

```yaml
# kibana.yml
server.name: "kibana-1"
server.host: "0.0.0.0"
elasticsearch.hosts: ["https://es1:9200", "https://es2:9200", "https://es3:9200"]
elasticsearch.username: "${ELASTICSEARCH_USERNAME}"
elasticsearch.password: "${ELASTICSEARCH_PASSWORD}"
elasticsearch.ssl.verificationMode: certificate
elasticsearch.ssl.certificateAuthorities: ["/etc/kibana/certs/ca.crt"]

# Secure settings
xpack.security.encryptionKey: "${ENCRYPTION_KEY}"
xpack.reporting.encryptionKey: "${REPORTING_ENCRYPTION_KEY}"
xpack.encryptedSavedObjects.encryptionKey: "${SAVED_OBJECTS_ENCRYPTION_KEY}"

# High availability settings
elasticsearch.pingTimeout: 3000
elasticsearch.requestTimeout: 30000
elasticsearch.shardTimeout: 30000
```

### Session Management

Configure session management for HA Kibana deployments:

1. **Elasticsearch Session Store**:
   ```yaml
   # For versions 7.10+
   xpack.security.session.idleTimeout: "1h"
   xpack.security.session.lifespan: "24h"
   ```

2. **External Session Store** (Redis):
   - Use a custom Kibana plugin for Redis session storage
   - Configure Redis with replication for HA

### Load Balancing

Configure load balancing for Kibana:

```
                      +--------------+
                      | Load Balancer|
                      +------+-------+
                             |
                 +-----------+-----------+
                 |           |           |
          +------+----+ +----+------+ +--+--------+
          | Kibana 1   | | Kibana 2  | | Kibana 3  |
          +-----+-----+ +-----+-----+ +-----+-----+
                |             |             |
                +-------------+-------------+
                              |
                     +--------+---------+
                     | Elasticsearch   |
                     | Cluster         |
                     +-----------------+
```

NGINX load balancer configuration:

```nginx
upstream kibana {
    server kibana1:5601 max_fails=3 fail_timeout=30s;
    server kibana2:5601 max_fails=3 fail_timeout=30s;
    server kibana3:5601 max_fails=3 fail_timeout=30s;
    
    keepalive 32;
}

server {
    listen 443 ssl;
    server_name kibana.example.com;
    
    ssl_certificate /etc/nginx/certs/kibana.crt;
    ssl_certificate_key /etc/nginx/certs/kibana.key;
    
    location / {
        proxy_pass http://kibana;
        proxy_http_version 1.1;
        proxy_set_header Connection "Keep-Alive";
        proxy_set_header Proxy-Connection "Keep-Alive";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $http_host;
        
        # Websocket support
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # Health check
        health_check interval=10 fails=3 passes=2;
    }
}
```

### Specialized Kibana Instances

Deploy specialized Kibana instances for different user groups:

```yaml
# Data analyst Kibana
server.name: "kibana-analysts"
server.host: "0.0.0.0"
elasticsearch.hosts: ["https://es1:9200", "https://es2:9200", "https://es3:9200"]
server.defaultRoute: "/app/dashboards"
elasticsearch.requestTimeout: 90000
xpack.reporting.queue.timeout: 180000

# Operations Kibana
server.name: "kibana-ops"
server.host: "0.0.0.0"
elasticsearch.hosts: ["https://es1:9200", "https://es2:9200", "https://es3:9200"]
server.defaultRoute: "/app/monitoring"
elasticsearch.requestTimeout: 30000
xpack.monitoring.ui.container.elasticsearch.enabled: true
```

### Kibana HA in Kubernetes

Deploy Kibana in Kubernetes for automated recovery:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: logging
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:8.1.0
        env:
        - name: ELASTICSEARCH_HOSTS
          value: "https://elasticsearch-client.logging.svc.cluster.local:9200"
        - name: ELASTICSEARCH_USERNAME
          valueFrom:
            secretKeyRef:
              name: elasticsearch-credentials
              key: username
        - name: ELASTICSEARCH_PASSWORD
          valueFrom:
            secretKeyRef:
              name: elasticsearch-credentials
              key: password
        - name: ENCRYPTION_KEY
          valueFrom:
            secretKeyRef:
              name: kibana-encryption
              key: encryptionKey
        ports:
        - containerPort: 5601
        readinessProbe:
          httpGet:
            path: /api/status
            port: 5601
          initialDelaySeconds: 60
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /api/status
            port: 5601
          initialDelaySeconds: 120
          periodSeconds: 30
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "1"
            memory: "2Gi"
---
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: logging
spec:
  type: ClusterIP
  ports:
  - port: 5601
    targetPort: 5601
    protocol: TCP
  selector:
    app: kibana
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kibana
  namespace: logging
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  rules:
  - host: kibana.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kibana
            port:
              number: 5601
  tls:
  - hosts:
    - kibana.example.com
    secretName: kibana-tls
```

## Beats High Availability

Deploy Beats agents for resilient data collection.

### Filebeat High Availability

Configure Filebeat for reliable log shipping:

```yaml
# filebeat.yml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/*.log
  fields:
    source_node: "node1"
  scan_frequency: 10s
  ignore_older: 24h
  clean_inactive: 72h
  close_inactive: 5m
  harvester_buffer_size: 16384
  max_bytes: 10485760

# Multiple outputs for redundancy
output.elasticsearch:
  enabled: true
  hosts: ["es1:9200", "es2:9200", "es3:9200"]
  loadbalance: true
  bulk_max_size: 50

# Alternative Logstash output
output.logstash:
  enabled: false  # Disabled by default, enable if Elasticsearch is unavailable
  hosts: ["logstash1:5044", "logstash2:5044"]
  loadbalance: true
  
# Use persistent queue for reliability
queue.mem.events: 4096
queue.mem.flush.min_events: 512
queue.mem.flush.timeout: 5s
```

### Metricbeat High Availability

Configure Metricbeat for reliable metrics collection:

```yaml
# metricbeat.yml
metricbeat.modules:
- module: system
  metricsets: ["cpu", "load", "memory", "network", "process", "process_summary", "filesystem", "fsstat"]
  enabled: true
  period: 10s
  processes: ['.*']
  process.include_top_n:
    by_cpu: 5
    by_memory: 5

# Store unshipped metrics to disk
queue.spool:
  file:
    path: "${path.data}/spool.dat"
    size: 100MiB
    page_size: 16KiB
  write:
    buffer_size: 10MiB
    flush.timeout: 5s
    flush.events: 1000

# Configure multiple outputs
output.elasticsearch:
  hosts: ["es1:9200", "es2:9200", "es3:9200"]
  loadbalance: true
  retry.max_retries: 5
  retry.backoff.init: 1s
  retry.backoff.max: 60s
```

### Heartbeat High Availability

Deploy multiple Heartbeat instances in different locations:

```yaml
# US East Region heartbeat.yml
heartbeat.monitors:
- type: http
  id: website-us-east
  name: "Website Monitoring from US East"
  urls: ["https://www.example.com"]
  schedule: '@every 30s'
  check.response.status: [200]
  tags: ["production", "us-east"]
  
# EU West Region heartbeat.yml
heartbeat.monitors:
- type: http
  id: website-eu-west
  name: "Website Monitoring from EU West"
  urls: ["https://www.example.com"]
  schedule: '@every 30s'
  check.response.status: [200]
  tags: ["production", "eu-west"]

# Asia Pacific Region heartbeat.yml
heartbeat.monitors:
- type: http
  id: website-ap-southeast
  name: "Website Monitoring from AP Southeast"
  urls: ["https://www.example.com"]
  schedule: '@every 30s'
  check.response.status: [200]
  tags: ["production", "ap-southeast"]
```

### Beat Deployment Strategies

Deploy Beats using different strategies for HA:

1. **Sidecar Pattern**:
   - Deploy Beats as sidecars in containerized environments
   - Dedicated Beat per application container
   - Direct access to application logs

2. **Node Agent Pattern**:
   - Deploy one set of Beats per node
   - Collect all data from the node
   - Requires appropriate permissions

3. **Centralized Collection Pattern**:
   - Dedicated collection nodes
   - Receives logs/metrics from multiple sources
   - Higher resource allocation

Kubernetes DaemonSet example:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: logging
spec:
  selector:
    matchLabels:
      app: filebeat
  template:
    metadata:
      labels:
        app: filebeat
    spec:
      serviceAccount: filebeat
      terminationGracePeriodSeconds: 30
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:8.1.0
        args: ["-c", "/etc/filebeat.yml", "-e"]
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        securityContext:
          runAsUser: 0
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - name: config
          mountPath: /etc/filebeat.yml
          subPath: filebeat.yml
        - name: data
          mountPath: /usr/share/filebeat/data
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: varlog
          mountPath: /var/log
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: filebeat-config
          defaultMode: 0600
      - name: data
        hostPath:
          path: /var/lib/filebeat-data
          type: DirectoryOrCreate
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: varlog
        hostPath:
          path: /var/log
```

## Cross-Cluster High Availability

Connect multiple Elasticsearch clusters for higher availability.

### Cross-Cluster Replication (CCR)

Configure cross-cluster replication for disaster recovery:

```json
// Set up remote cluster connection
PUT _cluster/settings
{
  "persistent": {
    "cluster.remote.dr-cluster.seeds": [
      "es1-dr.example.com:9300",
      "es2-dr.example.com:9300",
      "es3-dr.example.com:9300"
    ]
  }
}

// Create a follower index
PUT logs-2023.05.01-follow
{
  "remote_cluster": "dr-cluster",
  "leader_index": "logs-2023.05.01"
}

// Check replication status
GET /_ccr/stats
```

Auto-follow patterns for automatic replication:

```json
PUT /_ccr/auto_follow/logs-pattern
{
  "remote_cluster": "dr-cluster",
  "leader_index_patterns": ["logs-*"],
  "follow_index_pattern": "{{leader_index}}-follow",
  "settings": {
    "index.number_of_replicas": 1
  }
}
```

### Cross-Cluster Search

Configure cross-cluster search for seamless fail over:

```json
// Set up remote cluster connection
PUT _cluster/settings
{
  "persistent": {
    "cluster.remote.cluster2.seeds": [
      "es1-c2.example.com:9300",
      "es2-c2.example.com:9300",
      "es3-c2.example.com:9300"
    ]
  }
}

// Search across clusters
GET /local-logs-*,cluster2:logs-*/_search
{
  "query": {
    "match": {
      "message": "error"
    }
  }
}
```

Configure Kibana for cross-cluster search:

```yaml
# kibana.yml
elasticsearch.hosts: ["https://es-local:9200"]
elasticsearch.requestHeadersWhitelist: [ "authorization", "x-proxy-remote-user", "x-forwarded-for", "x-cluster" ]

server.xsrf.allowlist: ["cluster-remote.cross-cluster-form"]
server.cors: true
server.cors.allowlist: ["https://kibana1.example.com", "https://kibana2.example.com"]
```

### Multi-Cluster Architecture

Design a multi-cluster, multi-region architecture:

```
Region A (Primary)                      Region B (DR)
+------------------------+              +------------------------+
|                        |              |                        |
|  +------------------+  |              |  +------------------+  |
|  | Elasticsearch    |  |              |  | Elasticsearch    |  |
|  | Cluster A        |<----------------->| Cluster B        |  |
|  +------------------+  |  Replication  |  +------------------+  |
|          Λ             |              |          Λ             |
|          |             |              |          |             |
|          V             |              |          V             |
|  +------------------+  |              |  +------------------+  |
|  | Kibana Cluster A |  |              |  | Kibana Cluster B |  |
|  +------------------+  |              |  +------------------+  |
|          Λ             |              |          Λ             |
|          |             |              |          |             |
+----------+-------------+              +----------+-------------+
           |                                       |
           |                                       |
           V                                       V
  +--------------------+                  +--------------------+
  | Global Load        |                  | Logstash Cluster B |
  | Balancer           |                  +----------Λ---------+
  +--------+-----------+                             |
           |                                         |
           V                                         |
  +--------------------+                             |
  | Logstash Cluster A |-----------------------------+
  +----------Λ---------+        Cross-region
           |                     replication
           |
           V
  +--------------------+
  | Beat Agents        |
  +--------------------+
```

## Load Balancing

Implement load balancing for ELK components.

### Load Balancer Technologies

Choose appropriate load balancing technology:

1. **Hardware Load Balancers**:
   - F5, Citrix NetScaler, Kemp
   - High performance
   - Advanced features
   - Higher cost

2. **Software Load Balancers**:
   - HAProxy, NGINX, Traefik
   - Flexible deployment
   - Scriptable configuration
   - Lower cost

3. **Cloud Load Balancers**:
   - AWS ALB/NLB, Google Cloud Load Balancing, Azure Load Balancer
   - Managed service
   - Tight cloud integration
   - Usage-based pricing

4. **Service Mesh**:
   - Istio, Linkerd, Consul
   - Advanced traffic management
   - Built-in observability
   - Suitable for microservices

### HAProxy Configuration

Example HAProxy configuration for Elasticsearch:

```
global
    log /dev/log local0
    log /dev/log local1 notice
    user haproxy
    group haproxy
    daemon
    maxconn 4096
    ssl-default-bind-options no-sslv3 no-tlsv10 no-tlsv11

defaults
    log global
    mode tcp
    option tcplog
    option dontlognull
    timeout connect 5s
    timeout client 30s
    timeout server 30s

frontend elasticsearch_front
    bind *:9200
    default_backend elasticsearch_back
    mode tcp

backend elasticsearch_back
    mode tcp
    balance roundrobin
    option httpchk GET /_cluster/health
    http-check expect string green
    default-server inter 3s fall 3 rise 2
    server es1 es1.example.com:9200 check
    server es2 es2.example.com:9200 check
    server es3 es3.example.com:9200 check

frontend kibana_front
    bind *:5601
    default_backend kibana_back
    mode http
    option httplog

backend kibana_back
    mode http
    balance roundrobin
    option httpchk GET /api/status
    http-check expect status 200
    default-server inter 3s fall 3 rise 2
    server kibana1 kibana1.example.com:5601 check
    server kibana2 kibana2.example.com:5601 check
    server kibana3 kibana3.example.com:5601 check

frontend logstash_front
    bind *:5044
    default_backend logstash_back
    mode tcp

backend logstash_back
    mode tcp
    balance leastconn
    default-server inter 3s fall 3 rise 2
    server logstash1 logstash1.example.com:5044 check
    server logstash2 logstash2.example.com:5044 check
    server logstash3 logstash3.example.com:5044 check
```

### NGINX Configuration

Example NGINX configuration for Elasticsearch and Kibana:

```nginx
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 4096;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                     '$status $body_bytes_sent "$http_referer" '
                     '"$http_user_agent" "$http_x_forwarded_for"';
    
    access_log /var/log/nginx/access.log main;
    sendfile on;
    keepalive_timeout 65;
    
    # Elasticsearch HTTP
    upstream elasticsearch {
        zone elasticsearch 64k;
        server es1.example.com:9200 max_fails=3 fail_timeout=30s;
        server es2.example.com:9200 max_fails=3 fail_timeout=30s;
        server es3.example.com:9200 max_fails=3 fail_timeout=30s;
        keepalive 16;
    }
    
    # Kibana upstream
    upstream kibana {
        zone kibana 64k;
        server kibana1.example.com:5601 max_fails=3 fail_timeout=30s;
        server kibana2.example.com:5601 max_fails=3 fail_timeout=30s;
        server kibana3.example.com:5601 max_fails=3 fail_timeout=30s;
        keepalive 16;
    }
    
    # Elasticsearch proxy
    server {
        listen 9200 ssl;
        server_name elasticsearch.example.com;
        
        ssl_certificate /etc/nginx/certs/elasticsearch.crt;
        ssl_certificate_key /etc/nginx/certs/elasticsearch.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        
        location / {
            proxy_pass http://elasticsearch;
            proxy_http_version 1.1;
            proxy_set_header Connection "Keep-Alive";
            proxy_set_header Proxy-Connection "Keep-Alive";
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Host $http_host;
            
            health_check interval=10 fails=3 passes=2;
            
            proxy_connect_timeout 30s;
            proxy_send_timeout 90s;
            proxy_read_timeout 90s;
        }
    }
    
    # Kibana proxy
    server {
        listen 5601 ssl;
        server_name kibana.example.com;
        
        ssl_certificate /etc/nginx/certs/kibana.crt;
        ssl_certificate_key /etc/nginx/certs/kibana.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        
        location / {
            proxy_pass http://kibana;
            proxy_http_version 1.1;
            proxy_set_header Connection "Keep-Alive";
            proxy_set_header Proxy-Connection "Keep-Alive";
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Host $http_host;
            
            # Websocket support
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            
            health_check interval=10 fails=3 passes=2;
            
            proxy_connect_timeout 30s;
            proxy_send_timeout 90s;
            proxy_read_timeout 90s;
        }
    }
}

stream {
    # Logstash TCP
    upstream logstash {
        zone logstash 64k;
        least_conn;
        server logstash1.example.com:5044 max_fails=3 fail_timeout=30s;
        server logstash2.example.com:5044 max_fails=3 fail_timeout=30s;
        server logstash3.example.com:5044 max_fails=3 fail_timeout=30s;
    }
    
    # Logstash proxy
    server {
        listen 5044;
        proxy_pass logstash;
        proxy_connect_timeout 1s;
        proxy_timeout 3s;
    }
    
    # Elasticsearch transport
    upstream elasticsearch_transport {
        zone elasticsearch_transport 64k;
        server es1.example.com:9300 max_fails=3 fail_timeout=30s;
        server es2.example.com:9300 max_fails=3 fail_timeout=30s;
        server es3.example.com:9300 max_fails=3 fail_timeout=30s;
    }
    
    # Elasticsearch transport proxy
    server {
        listen 9300;
        proxy_pass elasticsearch_transport;
        proxy_connect_timeout 1s;
        proxy_timeout 600s;
    }
}
```

### Session Persistence

Configure session persistence for Kibana:

```nginx
# NGINX sticky session configuration
upstream kibana {
    ip_hash;  # Use client IP for session persistence
    server kibana1.example.com:5601;
    server kibana2.example.com:5601;
    server kibana3.example.com:5601;
}
```

HAProxy sticky session:

```
backend kibana_back
    mode http
    balance roundrobin
    cookie SERVERID insert indirect nocache
    option httpchk GET /api/status
    http-check expect status 200
    default-server inter 3s fall 3 rise 2
    server kibana1 kibana1.example.com:5601 check cookie k1
    server kibana2 kibana2.example.com:5601 check cookie k2
    server kibana3 kibana3.example.com:5601 check cookie k3
```

### Health Checks

Configure health checks for ELK components:

1. **Elasticsearch Health Checks**:
   - HTTP endpoint: `/_cluster/health`
   - Expected response: Status code 200, status "green" or "yellow"
   - Check frequency: Every 10 seconds
   - Failure threshold: 3 consecutive failures

2. **Kibana Health Checks**:
   - HTTP endpoint: `/api/status`
   - Expected response: Status code 200
   - Check frequency: Every 10 seconds
   - Failure threshold: 3 consecutive failures

3. **Logstash Health Checks**:
   - HTTP endpoint: `/`
   - Expected response: Status code 200
   - Check frequency: Every 10 seconds
   - Failure threshold: 3 consecutive failures

## Monitoring HA Deployments

Set up monitoring for your HA ELK deployment.

### Internal Monitoring

Configure internal monitoring using X-Pack:

```yaml
# elasticsearch.yml
xpack.monitoring.collection.enabled: true

# logstash.yml
xpack.monitoring.enabled: true
xpack.monitoring.elasticsearch.hosts: ["https://es1:9200", "https://es2:9200", "https://es3:9200"]
xpack.monitoring.elasticsearch.username: "${MONITORING_USER}"
xpack.monitoring.elasticsearch.password: "${MONITORING_PASSWORD}"

# kibana.yml
xpack.monitoring.enabled: true
xpack.monitoring.ui.container.elasticsearch.enabled: true
```

### External Monitoring

Set up external monitoring with Metricbeat:

```yaml
# metricbeat.yml for monitoring Elasticsearch
metricbeat.modules:
- module: elasticsearch
  metricsets:
    - node
    - node_stats
    - cluster_stats
    - index
    - index_recovery
    - ccr
    - enrich
    - index_summary
    - ml_job
    - shard
  period: 10s
  hosts: ["https://es1:9200", "https://es2:9200", "https://es3:9200"]
  username: "${MONITORING_USER}"
  password: "${MONITORING_PASSWORD}"
  xpack.enabled: true

# For monitoring Logstash
- module: logstash
  metricsets:
    - node
    - node_stats
  period: 10s
  hosts: ["http://logstash1:9600", "http://logstash2:9600", "http://logstash3:9600"]
  xpack.enabled: true

# For monitoring Kibana
- module: kibana
  metricsets:
    - stats
  period: 10s
  hosts: ["http://kibana1:5601", "http://kibana2:5601", "http://kibana3:5601"]
  xpack.enabled: true

output.elasticsearch:
  hosts: ["https://monitoring-es1:9200", "https://monitoring-es2:9200"]
  username: "${MONITORING_OUTPUT_USER}"
  password: "${MONITORING_OUTPUT_PASSWORD}"
```

### Alerting on HA Issues

Set up alerting for high availability issues:

```json
// Elasticsearch cluster health alert
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
        "url": "http://elasticsearch:9200/_cluster/health",
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
    "notify_email": {
      "email": {
        "to": "ops@example.com",
        "subject": "Elasticsearch Cluster Status Red Alert",
        "body": {
          "html": "The cluster status is currently <strong>RED</strong>. Please investigate."
        }
      }
    },
    "notify_slack": {
      "slack": {
        "message": {
          "from": "Elasticsearch Watcher",
          "to": ["#es-alerts"],
          "text": "The cluster status is currently RED. Please investigate."
        }
      }
    }
  }
}

// Node disconnection alert
PUT _watcher/watch/node_disconnect_watch
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
              "must": [
                {
                  "range": {
                    "timestamp": {
                      "gte": "now-2m",
                      "lte": "now"
                    }
                  }
                }
              ]
            }
          },
          "aggs": {
            "nodes": {
              "terms": {
                "field": "elasticsearch.node.name",
                "size": 50
              }
            }
          }
        }
      }
    }
  },
  "condition": {
    "script": {
      "source": "return ctx.payload.aggregations.nodes.buckets.size() < ctx.metadata.expected_nodes",
      "params": {
        "expected_nodes": 3
      }
    }
  },
  "actions": {
    "notify_email": {
      "email": {
        "to": "ops@example.com",
        "subject": "Elasticsearch Node Disconnection Alert",
        "body": {
          "html": "Expected 3 nodes but only found {{ctx.payload.aggregations.nodes.buckets.size()}} nodes reporting metrics."
        }
      }
    }
  },
  "metadata": {
    "expected_nodes": 3
  }
}
```

### Monitoring Dashboard

Create a monitoring dashboard for HA status:

```json
// Sample Kibana dashboard for monitoring HA
{
  "dashboard": {
    "title": "ELK Stack HA Status",
    "hits": 0,
    "description": "Overview of high availability status for the ELK Stack",
    "panelsJSON": "[...]",  // Actual panel JSON is quite long
    "optionsJSON": "{\"darkTheme\":false,\"hidePanelTitles\":false,\"useMargins\":true}",
    "version": 1,
    "timeRestore": true,
    "timeTo": "now",
    "timeFrom": "now-24h",
    "refreshInterval": {
      "pause": false,
      "value": 30000
    }
  },
  "visualizations": [
    {
      "title": "Cluster Health Status",
      "visState": "{\"title\":\"Cluster Health Status\",\"type\":\"metrics\",\"params\":{...}}",
      "uiStateJSON": "{}",
      "description": "",
      "version": 1,
      "kibanaSavedObjectMeta": {
        "searchSourceJSON": "{\"query\":{\"query\":\"\",\"language\":\"kuery\"},\"filter\":[]}"
      }
    },
    {
      "title": "Node Status",
      "visState": "{\"title\":\"Node Status\",\"type\":\"table\",\"params\":{...}}",
      "uiStateJSON": "{}",
      "description": "",
      "version": 1,
      "kibanaSavedObjectMeta": {
        "searchSourceJSON": "{\"query\":{\"query\":\"\",\"language\":\"kuery\"},\"filter\":[]}"
      }
    }
    // Additional visualizations...
  ]
}
```

## Failure Recovery Strategies

Implement strategies for recovering from failures.

### Automated Failover

Configure automated failover procedures:

1. **Master Node Failover**:
   - Automatic with proper discovery and minimum_master_nodes settings
   - Quorum-based election of new master

2. **Data Node Failover**:
   - Shard reallocation to remaining nodes
   - Recovery from replicas

3. **Coordinating Node Failover**:
   - Load balancer health checks
   - Remove unhealthy nodes from rotation

4. **Ingest Node Failover**:
   - Multiple ingest nodes
   - Round-robin or least-connection load balancing

### Manual Recovery Procedures

Document manual recovery procedures:

1. **Elasticsearch Cluster Recovery**:
   ```bash
   # Check cluster health
   curl -XGET "http://localhost:9200/_cluster/health?pretty"
   
   # Check shard allocation
   curl -XGET "http://localhost:9200/_cat/shards?v"
   
   # Check unassigned shards
   curl -XGET "http://localhost:9200/_cat/shards?v&h=index,shard,prirep,state,unassigned.reason"
   
   # Force shard allocation if needed
   curl -XPOST "http://localhost:9200/_cluster/reroute" -H 'Content-Type: application/json' -d'
   {
     "commands": [
       {
         "allocate_stale_primary": {
           "index": "logs-2023.05.20",
           "shard": 3,
           "node": "es-data-2",
           "accept_data_loss": true
         }
       }
     ]
   }'
   ```

2. **Logstash Recovery**:
   ```bash
   # Check Logstash status
   curl -XGET "http://localhost:9600/_node/stats?pretty"
   
   # Restart Logstash if needed
   sudo systemctl restart logstash
   
   # Check pipeline status
   curl -XGET "http://localhost:9600/_node/pipelines?pretty"
   
   # Process dead letter queue if needed
   /usr/share/logstash/bin/logstash -e '
   input { 
     dead_letter_queue { 
       path => "/var/lib/logstash/dead_letter_queue" 
       commit_offsets => true 
     } 
   }
   output {
     elasticsearch {
       hosts => ["localhost:9200"]
       index => "recovered-%{[@metadata][dead_letter_queue][original_index]}"
     }
   }'
   ```

3. **Kibana Recovery**:
   ```bash
   # Check Kibana status
   curl -XGET "http://localhost:5601/api/status"
   
   # Restart Kibana if needed
   sudo systemctl restart kibana
   
   # Clear Kibana cache if needed
   rm -rf /var/lib/kibana/optimize/*
   ```

### Disaster Recovery

Implement disaster recovery procedures:

1. **Snapshot Recovery**:
   ```bash
   # List available snapshots
   curl -XGET "http://localhost:9200/_snapshot/s3_repository/_all?pretty"
   
   # Restore from snapshot
   curl -XPOST "http://localhost:9200/_snapshot/s3_repository/snapshot_2023_05_20/_restore" -H 'Content-Type: application/json' -d'
   {
     "indices": "logs-2023.05.*",
     "ignore_unavailable": true,
     "include_global_state": false,
     "rename_pattern": "logs-(.+)",
     "rename_replacement": "restored-logs-$1",
     "include_aliases": false
   }'
   ```

2. **Cross-Cluster Recovery**:
   ```bash
   # If primary cluster is down, update Kibana to point to DR cluster
   # kibana.yml
   elasticsearch.hosts: ["https://es-dr1:9200", "https://es-dr2:9200", "https://es-dr3:9200"]
   
   # Promote followers to leaders if needed
   curl -XPOST "http://localhost:9200/_ccr/auto_follow/logs-pattern/pause"
   
   # Stop following specific indices
   curl -XPOST "http://localhost:9200/logs-2023.05.20-follow/_ccr/pause_follow"
   
   # Convert follower index to regular index
   curl -XPOST "http://localhost:9200/logs-2023.05.20-follow/_ccr/unfollow"
   ```

3. **Complete Site Failover**:
   - Update DNS or global load balancer to point to DR site
   - Promote DR cluster to primary
   - Verify service availability
   - Update monitoring and alerting configurations

## Real-World HA Architectures

Examples of high availability architectures for different scale deployments.

### Small-Scale HA (3-5 servers)

Compact deployment with minimal hardware:

```
   +----------------+    +----------------+    +----------------+
   | Server 1       |    | Server 2       |    | Server 3       |
   |                |    |                |    |                |
   | Elasticsearch  |    | Elasticsearch  |    | Elasticsearch  |
   | Master+Data    |    | Master+Data    |    | Master+Data    |
   |                |    |                |    |                |
   | Kibana         |    | Kibana         |    | Kibana         |
   |                |    |                |    |                |
   | Logstash       |    | Logstash       |    | Logstash       |
   |                |    |                |    |                |
   +-------+--------+    +-------+--------+    +-------+--------+
           |                     |                     |
           +-----------------+   |   +-----------------+
                             |   |   |
                          +--+---+---+--+
                          |  Load       |
                          |  Balancer   |
                          +------+------+
                                 |
                                 v
                         +--------------+
                         | Beats Agents |
                         +--------------+
```

Configuration highlights:
- All nodes are master-eligible, data, and ingest nodes
- 3 shards, 1 replica for all indices
- Nginx or HAProxy load balancer
- Multiple Kibana instances for HA
- Same Logstash configuration on all nodes

### Medium-Scale HA (6-12 servers)

Dedicated roles for better performance:

```
   +-----------+   +-----------+   +-----------+
   | Master 1  |   | Master 2  |   | Master 3  |
   +-----------+   +-----------+   +-----------+
         |               |               |
         +---------------+---------------+
                         |
         +---------------+---------------+
         |               |               |
   +-----+-----+   +-----+-----+   +-----+-----+
   | Data 1    |   | Data 2    |   | Data 3    |
   +-----------+   +-----------+   +-----------+
         |               |               |
         +---------------+---------------+
                         |
                         v
   +-----------+   +-----------+   +-----------+
   | Ingest 1  |   | Ingest 2  |   | Ingest 3  |
   +-----------+   +-----------+   +-----------+
         |               |               |
         +---------------+---------------+
                         |
                         v
   +-----------+   +-----------+   +-----------+
   | Logstash 1|   | Logstash 2|   | Logstash 3|
   +-----------+   +-----------+   +-----------+
         |               |               |
         +---------------+---------------+
                         |
                         v
   +-----------+   +-----------+   +-----------+
   | Kibana 1  |   | Kibana 2  |   | Kibana 3  |
   +-----------+   +-----------+   +-----------+
         |               |               |
         +---------------+---------------+
                         |
                         v
                 +--------------+
                 | Load         |
                 | Balancer     |
                 +--------------+
                         |
                         v
                 +--------------+
                 | Beats Agents |
                 +--------------+
```

Configuration highlights:
- Dedicated master, data, and ingest nodes
- 5 shards, 1 replica for most indices
- Specialized Logstash instances with different pipelines
- Load balancing between Logstash instances
- Kibana behind load balancer

### Large-Scale HA (12+ servers)

Fully distributed deployment with specialized nodes:

```
Zone A                  Zone B                  Zone C
+------------------+    +------------------+    +------------------+
| Master 1         |    | Master 2         |    | Master 3         |
+------------------+    +------------------+    +------------------+
| Coord 1          |    | Coord 2          |    | Coord 3          |
+------------------+    +------------------+    +------------------+
| Hot Data 1-3     |    | Hot Data 4-6     |    | Hot Data 7-9     |
+------------------+    +------------------+    +------------------+
| Warm Data 1-2    |    | Warm Data 3-4    |    | Warm Data 5-6    |
+------------------+    +------------------+    +------------------+
| Cold Data 1      |    | Cold Data 2      |    | Cold Data 3      |
+------------------+    +------------------+    +------------------+
| Ingest 1-2       |    | Ingest 3-4       |    | Ingest 5-6       |
+------------------+    +------------------+    +------------------+
| ML 1             |    | ML 2             |    | ML 3             |
+------------------+    +------------------+    +------------------+
| Logstash 1-3     |    | Logstash 4-6     |    | Logstash 7-9     |
+------------------+    +------------------+    +------------------+
| Kibana 1-2       |    | Kibana 3-4       |    | Kibana 5-6       |
+------------------+    +------------------+    +------------------+
         |                       |                       |
         +-----------------------+-----------------------+
                                 |
                        +------------------+
                        | Global Load      |
                        | Balancer         |
                        +------------------+
                                 |
                        +------------------+
                        | Beats Agents     |
                        +------------------+
```

Configuration highlights:
- Multiple data tiers (hot, warm, cold)
- Zone-aware shard allocation
- Dedicated ML nodes for machine learning
- Specialized Logstash instances for different data sources
- High-performance storage for hot data nodes
- Multiple coordinating nodes for search and ingest
- Global load balancer with geographic routing

### Multi-Region HA

Cross-region deployment for maximum resilience:

```
Region A (Primary)                            Region B (DR)
+---------------------------+                +---------------------------+
| ES Cluster A              |                | ES Cluster B              |
| - 3 Master Nodes          | Replication   | - 3 Master Nodes          |
| - 6+ Data Nodes           +--------------->| - 6+ Data Nodes           |
| - 3 Coordinating Nodes    |                | - 3 Coordinating Nodes    |
| - 3 Ingest Nodes          |                | - 3 Ingest Nodes          |
+-------------+-------------+                +-------------+-------------+
              |                                            |
+-------------+-------------+                +-------------+-------------+
| Kibana Cluster A          |                | Kibana Cluster B          |
| - 3+ Kibana Nodes         |                | - 3+ Kibana Nodes         |
+-------------+-------------+                +-------------+-------------+
              |                                            |
+-------------+-------------+                +-------------+-------------+
| Logstash Cluster A        |                | Logstash Cluster B        |
| - 6+ Logstash Nodes       |                | - 6+ Logstash Nodes       |
+-------------+-------------+                +-------------+-------------+
              |                                            |
              |              +------------+                |
              +-------------->            |                |
                             | Global LB  |<---------------+
                             |            |
                             +-----+------+
                                   |
                             +-----+------+
                             | Beat Agents |
                             +------------+
```

Configuration highlights:
- Complete replica cluster in secondary region
- Cross-cluster replication between regions
- Global load balancer with region failover
- Data shipping to both regions or through cross-region replication
- Automated DR procedures
- Regular DR testing

## HA Testing and Validation

Test your high availability setup to ensure it functions as expected.

### Failure Simulation Tests

Simulate various failure scenarios:

1. **Node Failure Tests**:
   ```bash
   # Test master node failure
   sudo systemctl stop elasticsearch.service  # On a master node
   
   # Test data node failure
   sudo systemctl stop elasticsearch.service  # On a data node
   
   # Test multiple node failures
   sudo systemctl stop elasticsearch.service  # On multiple nodes
   ```

2. **Network Partition Tests**:
   ```bash
   # Create network partition using iptables
   sudo iptables -A INPUT -s 192.168.1.10 -j DROP
   sudo iptables -A OUTPUT -d 192.168.1.10 -j DROP
   
   # Or using tc for temporary partitions
   sudo tc qdisc add dev eth0 root handle 1: prio
   sudo tc qdisc add dev eth0 parent 1:3 handle 30: netem loss 100%
   sudo tc filter add dev eth0 protocol ip parent 1:0 prio 3 u32 match ip dst 192.168.1.10/32 flowid 1:3
   ```

3. **Service Failure Tests**:
   ```bash
   # Test Logstash failure
   sudo systemctl stop logstash.service
   
   # Test Kibana failure
   sudo systemctl stop kibana.service
   
   # Test load balancer failure
   sudo systemctl stop nginx.service  # Or haproxy, etc.
   ```

### Performance Under Failure

Test performance during failure conditions:

```bash
# Run benchmark during normal operation
./benchmark-tool.sh --duration=10m --threads=4 --output=normal.json

# Simulate failure
sudo systemctl stop elasticsearch.service  # On one node

# Run benchmark during failure
./benchmark-tool.sh --duration=10m --threads=4 --output=failure.json

# Compare results
./compare-benchmarks.sh normal.json failure.json
```

### Recovery Testing

Test recovery procedures:

```bash
# Test automatic recovery
sudo systemctl stop elasticsearch.service  # On one node
# Wait for cluster to stabilize
sleep 120
sudo systemctl start elasticsearch.service
# Verify shards reallocate properly

# Test snapshot recovery
curl -XPUT "http://localhost:9200/_snapshot/backup/snapshot_1?wait_for_completion=true"
# Delete an index
curl -XDELETE "http://localhost:9200/test-index"
# Restore from snapshot
curl -XPOST "http://localhost:9200/_snapshot/backup/snapshot_1/_restore" -H 'Content-Type: application/json' -d'
{
  "indices": "test-index",
  "ignore_unavailable": true,
  "include_global_state": false
}'
```

### Chaos Testing

Implement chaos testing for continuous HA validation:

1. **Using Chaos Monkey for Kubernetes**:
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: kube-monkey
     namespace: kube-system
   spec:
     replicas: 1
     template:
       metadata:
         labels:
           app: kube-monkey
       spec:
         containers:
         - name: kube-monkey
           image: asobti/kube-monkey:v0.3.0
           env:
             - name: KUBERNETES
               value: "true"
           volumeMounts:
             - name: config
               mountPath: /etc/kube-monkey
         volumes:
         - name: config
           configMap:
             name: kube-monkey-config
   ```

2. **Chaos Mesh for Advanced Testing**:
   ```bash
   # Install Chaos Mesh
   curl -sSL https://mirrors.chaos-mesh.org/v2.0.0/install.sh | bash
   
   # Create pod failure experiment
   kubectl apply -f - <<EOF
   apiVersion: chaos-mesh.org/v1alpha1
   kind: PodChaos
   metadata:
     name: elasticsearch-pod-failure
     namespace: elastic-system
   spec:
     action: pod-failure
     mode: one
     selector:
       namespaces:
         - elastic-system
       labelSelectors:
         "app.kubernetes.io/name": "elasticsearch"
     duration: "30s"
     scheduler:
       cron: "@every 10m"
   EOF
   ```

## Conclusion

A well-designed high availability architecture is essential for ensuring the reliability and resilience of your ELK Stack deployment. By implementing redundancy at every layer, configuring appropriate failover mechanisms, and regularly testing your HA setup, you can build an ELK Stack that remains operational even during component failures.

Remember that high availability is not just about technology but also about processes and people. Document your architecture, maintain runbooks for recovery procedures, and ensure your team is trained to handle failure scenarios. Regular testing and validation will help identify gaps and improve your HA strategy over time.

In the next chapter, we'll explore disaster recovery strategies for the ELK Stack, focusing on backup, recovery, and cross-region resilience.