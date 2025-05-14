# Metricbeat

This chapter covers Metricbeat, the lightweight shipper for metrics that collects and reports system and service statistics.

## Table of Contents
- [Metricbeat Overview](#metricbeat-overview)
- [Architecture and Components](#architecture-and-components)
- [Installation and Setup](#installation-and-setup)
- [Basic Configuration](#basic-configuration)
- [Modules and Metricsets](#modules-and-metricsets)
- [System Metrics](#system-metrics)
- [Service Metrics](#service-metrics)
- [Cloud Provider Metrics](#cloud-provider-metrics)
- [Kubernetes Metrics](#kubernetes-metrics)
- [Processors and Enrichment](#processors-and-enrichment)
- [Output Configuration](#output-configuration)
- [Advanced Features](#advanced-features)
- [Performance Tuning](#performance-tuning)
- [Monitoring Metricbeat](#monitoring-metricbeat)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)

## Metricbeat Overview

Metricbeat is a lightweight shipper for metrics that collects data from your systems and services, providing insights into resource usage, performance, and health states.

### Key Features

- **Lightweight**: Small footprint with minimal system impact
- **Extensive Coverage**: Monitor systems, services, and applications
- **Modularity**: Enable only the modules you need
- **Auto-discovery**: Detect and monitor services automatically
- **Secure**: Support for TLS, authentication, and secure outputs
- **Extensible**: Add custom metrics and modules
- **Integrations**: Pre-built dashboards and visualizations
- **Resilient**: Reliable metrics collection and delivery

### Common Use Cases

1. **Infrastructure Monitoring**: Track CPU, memory, disk, and network usage
2. **Application Performance**: Monitor application-specific metrics
3. **Database Monitoring**: Collect metrics from databases like MySQL, PostgreSQL, MongoDB
4. **Cloud Monitoring**: Monitor cloud services and infrastructure
5. **Container Orchestration**: Monitor Kubernetes, Docker, and container services
6. **Service Health**: Track service availability and performance
7. **Capacity Planning**: Collect data for capacity forecasting
8. **Alerting**: Feed data to alerting systems for proactive notifications

### Advantages of Metricbeat

- **Consistent Data Format**: All metrics follow the Elastic Common Schema (ECS)
- **Low Overhead**: Minimal resource usage compared to agent-based alternatives
- **Easy Deployment**: Simple installation and configuration
- **Central Management**: Configure through Kibana when using Fleet
- **Elastic Stack Integration**: Seamless integration with Elasticsearch and Kibana

## Architecture and Components

Metricbeat's architecture consists of several components working together to collect and ship metrics.

### Core Components

1. **Beat Framework**: Foundational code shared by all Elastic Beats
2. **Modules**: Collections of metricsets for specific services/systems
3. **Metricsets**: Individual groups of related metrics
4. **Period**: Regular interval for metrics collection
5. **Host**: Target system or service to monitor
6. **Processors**: Transform metrics before sending to output
7. **Output**: Destination for collected metrics

### Data Flow

```
Source Systems/Services → Modules → Metricsets → Processors → Output
```

1. **Collection**: Metricsets collect data at defined intervals
2. **Processing**: Raw metrics are processed and transformed
3. **Aggregation**: Metrics are batched for efficient transport
4. **Output**: Processed metrics are sent to configured outputs

### Module Architecture

Each module follows a similar structure:

- **Module Definition**: Configuration and metadata
- **Metricsets**: Individual metric collectors
- **Default Configuration**: Reasonable defaults for quick setup
- **Dashboards**: Pre-built visualizations for Kibana
- **Docs**: Documentation on available metrics

## Installation and Setup

### System Requirements

- **Memory**: 100MB minimum (varies with enabled modules)
- **CPU**: 1 core minimum
- **Disk**: 200MB free space for installation
- **Operating Systems**:
  - Linux (most distributions)
  - Windows Server 2012 R2 or later
  - macOS 10.13 or later
  - FreeBSD
  - Containerized environments

### Installation Methods

1. **Package Repositories (Linux)**:
   ```bash
   # Debian/Ubuntu
   curl -L -O https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-8.1.0-amd64.deb
   sudo dpkg -i metricbeat-8.1.0-amd64.deb
   
   # RHEL/CentOS
   curl -L -O https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-8.1.0-x86_64.rpm
   sudo rpm -vi metricbeat-8.1.0-x86_64.rpm
   ```

2. **Windows**:
   - Download installer from Elastic website
   - Install using PowerShell as administrator:
     ```powershell
     # Install as a Windows service
     PS> .\metricbeat.exe install
     ```

3. **Docker**:
   ```bash
   docker pull docker.elastic.co/beats/metricbeat:8.1.0
   docker run docker.elastic.co/beats/metricbeat:8.1.0
   ```

4. **Kubernetes**:
   - Use Helm chart or Elastic Cloud on Kubernetes operator
   - Apply Metricbeat DaemonSet for node metrics
   - Deploy Metricbeat Deployment for cluster-level metrics

### Initial Setup

1. **Configuration File**: Edit `metricbeat.yml` in the configuration directory
2. **Enable Modules**: Turn on modules for specific systems and services
3. **Set Up Assets**: Load index templates and dashboards:
   ```bash
   metricbeat setup -e
   ```
4. **Start Metricbeat**:
   ```bash
   # Systemd (Linux)
   sudo systemctl start metricbeat
   
   # Windows
   Start-Service metricbeat
   ```

### Verification

Confirm Metricbeat is running correctly:

```bash
# Check service status (Linux)
sudo systemctl status metricbeat

# View logs
sudo journalctl -u metricbeat

# Test configuration
metricbeat test config -c /etc/metricbeat/metricbeat.yml

# Test output connectivity
metricbeat test output -c /etc/metricbeat/metricbeat.yml
```

## Basic Configuration

The `metricbeat.yml` file is the main configuration file, typically located in `/etc/metricbeat/` on Linux or `C:\Program Files\Metricbeat\` on Windows.

### Configuration Structure

```yaml
metricbeat.config.modules:      # Module configuration loading
metricbeat.modules:             # Module definitions
output.elasticsearch:           # Output configuration
processors:                     # Processors for data transformation
setup:                          # Setup configurations
logging:                        # Logging configurations
```

### Minimal Configuration Example

```yaml
metricbeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false

metricbeat.modules:
- module: system
  metricsets: ["cpu", "memory", "network", "filesystem"]
  period: 10s
  enabled: true

output.elasticsearch:
  hosts: ["localhost:9200"]
  username: "elastic"
  password: "changeme"
```

### Common Settings

1. **General Settings**:
   ```yaml
   name: "my-metricbeat"        # Custom name for the Beat
   tags: ["production", "web"]  # Tags for all events
   fields:                      # Custom fields for all events
     env: production
     dc: us-east-1
   fields_under_root: true      # Places custom fields at root level
   max_procs: 4                 # Limit CPU usage
   ```

2. **Logging Settings**:
   ```yaml
   logging.level: info
   logging.to_files: true
   logging.files:
     path: /var/log/metricbeat
     name: metricbeat
     keepfiles: 7
     permissions: 0644
   ```

3. **Module Loading**:
   ```yaml
   metricbeat.config.modules:
     path: ${path.config}/modules.d/*.yml
     reload.enabled: true
     reload.period: 10s
   ```

## Modules and Metricsets

Metricbeat's functionality is organized into modules (systems/services to monitor) and metricsets (groups of related metrics within a module).

### Available Modules

Metricbeat includes modules for various systems and services:

1. **Operating Systems**:
   - System: Core OS metrics
   - Linux: Linux-specific metrics
   - Windows: Windows-specific metrics

2. **Web Servers**:
   - Apache
   - Nginx
   - IIS

3. **Databases**:
   - MySQL
   - PostgreSQL
   - MongoDB
   - Redis
   - Elasticsearch
   - SQL Server

4. **Message Queues**:
   - Kafka
   - RabbitMQ
   - ActiveMQ

5. **Containers & Orchestration**:
   - Docker
   - Kubernetes
   - Containerd

6. **Cloud Providers**:
   - AWS
   - GCP
   - Azure

7. **Other Services**:
   - Prometheus
   - ZooKeeper
   - HAProxy
   - Consul
   - etcd
   - Aerospike
   - Ceph

### Module Activation

Enable modules via configuration:

```yaml
metricbeat.modules:
- module: system
  metricsets: ["cpu", "memory", "load", "network", "process"]
  period: 10s
  enabled: true
```

Or using the command line:

```bash
# List available modules
metricbeat modules list

# Enable specific modules
metricbeat modules enable system nginx mysql

# Disable modules
metricbeat modules disable redis
```

### Module Configuration

Configure modules via YAML files in `modules.d/` directory:

```yaml
# modules.d/system.yml
- module: system
  period: 10s
  metricsets:
    - cpu
    - load
    - memory
    - network
    - process
    - process_summary
    - socket_summary
    - filesystem
    - fsstat
  process.include_top_n:
    by_cpu: 5
    by_memory: 5
  filesystem.ignore_types:
    - nfs
    - smbfs
    - tmpfs
```

### Metricsets Configuration

Each module contains multiple metricsets that can be individually configured:

```yaml
- module: elasticsearch
  metricsets:
    - node
    - node_stats
    - cluster_stats
    - index
  period: 10s
  hosts: ["https://localhost:9200"]
  username: "elastic"
  password: "changeme"
  ssl.verification_mode: "certificate"
  xpack.enabled: true
```

## System Metrics

The system module collects metrics from the operating system and is enabled by default.

### CPU Metrics

```yaml
- module: system
  metricsets: ["cpu"]
  period: 10s
  cpu.metrics: ["percentages", "normalized_percentages"]
```

Collected metrics include:
- `system.cpu.user.pct`: CPU time spent in user space
- `system.cpu.system.pct`: CPU time spent in kernel space
- `system.cpu.idle.pct`: CPU idle time
- `system.cpu.iowait.pct`: CPU time waiting for I/O operations
- `system.cpu.irq.pct`: CPU time servicing hardware interrupts
- `system.cpu.softirq.pct`: CPU time servicing software interrupts
- `system.cpu.steal.pct`: CPU time spent in other operating systems (virtualization)
- `system.cpu.total.pct`: Total CPU utilization

### Memory Metrics

```yaml
- module: system
  metricsets: ["memory"]
  period: 10s
```

Collected metrics include:
- `system.memory.total`: Total physical memory
- `system.memory.used.bytes`: Used memory
- `system.memory.free`: Available memory
- `system.memory.used.pct`: Percentage of memory used
- `system.memory.actual.used.bytes`: Actual memory used
- `system.memory.actual.free`: Actual free memory
- `system.memory.swap.total`: Total swap memory
- `system.memory.swap.used.bytes`: Used swap memory
- `system.memory.swap.free`: Free swap memory
- `system.memory.swap.used.pct`: Percentage of swap used

### Disk Metrics

```yaml
- module: system
  metricsets: ["filesystem", "fsstat"]
  period: 10s
  filesystem.ignore_types: ["tmpfs", "devtmpfs", "devfs", "overlay", "autofs", "squashfs"]
```

Collected metrics include:
- `system.filesystem.total`: Total disk space
- `system.filesystem.used.bytes`: Used disk space
- `system.filesystem.used.pct`: Percentage of disk space used
- `system.filesystem.available`: Available disk space
- `system.filesystem.device_name`: Device name
- `system.filesystem.mount_point`: Mount point
- `system.filesystem.type`: File system type
- `system.fsstat.count`: Number of file systems
- `system.fsstat.total_size.total`: Total size across all file systems
- `system.fsstat.total_size.used`: Total used space across all file systems
- `system.fsstat.total_size.free`: Total free space across all file systems

### Network Metrics

```yaml
- module: system
  metricsets: ["network"]
  period: 10s
```

Collected metrics include:
- `system.network.in.bytes`: Incoming bytes
- `system.network.in.packets`: Incoming packets
- `system.network.in.errors`: Incoming errors
- `system.network.in.dropped`: Incoming dropped packets
- `system.network.out.bytes`: Outgoing bytes
- `system.network.out.packets`: Outgoing packets
- `system.network.out.errors`: Outgoing errors
- `system.network.out.dropped`: Outgoing dropped packets
- `system.network.in.bytes_sec`: Incoming bytes per second
- `system.network.out.bytes_sec`: Outgoing bytes per second

### Process Metrics

```yaml
- module: system
  metricsets: ["process"]
  period: 10s
  process.include_top_n:
    by_cpu: 5
    by_memory: 5
```

Collected metrics include:
- `system.process.name`: Process name
- `system.process.pid`: Process ID
- `system.process.state`: Process state
- `system.process.cpu.total.pct`: Total CPU used by the process
- `system.process.cpu.system.pct`: System CPU used by the process
- `system.process.cpu.user.pct`: User CPU used by the process
- `system.process.memory.rss.bytes`: Resident Set Size
- `system.process.memory.rss.pct`: RSS as percentage of total memory
- `system.process.memory.share`: Shared memory
- `system.process.memory.size`: Virtual memory size

### Load Metrics

```yaml
- module: system
  metricsets: ["load"]
  period: 10s
```

Collected metrics include:
- `system.load.1`: Load average for the last 1 minute
- `system.load.5`: Load average for the last 5 minutes
- `system.load.15`: Load average for the last 15 minutes
- `system.load.norm.1`: Normalized load average for the last 1 minute (divided by CPU count)
- `system.load.norm.5`: Normalized load average for the last 5 minutes
- `system.load.norm.15`: Normalized load average for the last 15 minutes

## Service Metrics

Metricbeat can collect metrics from various services and applications. Below are examples of common service modules.

### Elasticsearch Metrics

```yaml
- module: elasticsearch
  metricsets:
    - node
    - node_stats
    - cluster_stats
    - index
  period: 10s
  hosts: ["https://localhost:9200"]
  username: "elastic"
  password: "changeme"
  xpack.enabled: true
```

Key metrics collected:
- Node stats (JVM, thread pools, transport, HTTP)
- Cluster health and stats
- Index performance and storage
- Cache usage and evictions
- Search and indexing performance

### MySQL Metrics

```yaml
- module: mysql
  metricsets:
    - status
    - performance
    - query
  period: 10s
  hosts: ["root:password@tcp(localhost:3306)/"]
```

Key metrics collected:
- Server status variables
- Connection statistics
- Buffer pool utilization
- Query throughput and latency
- InnoDB metrics
- Table statistics

### Nginx Metrics

```yaml
- module: nginx
  metricsets: ["stubstatus"]
  period: 10s
  hosts: ["http://localhost/nginx_status"]
```

Key metrics collected:
- Active connections
- Connection states
- Requests per second
- Reading/writing/waiting connections

### Redis Metrics

```yaml
- module: redis
  metricsets: ["info", "keyspace"]
  period: 10s
  hosts: ["redis://localhost:6379"]
  password: "changeme"
```

Key metrics collected:
- Memory usage
- Clients connected
- Commands processed
- Hit/miss ratio
- Keyspace statistics
- Replication status

### Apache Metrics

```yaml
- module: apache
  metricsets: ["status"]
  period: 10s
  hosts: ["http://localhost/server-status?auto"]
```

Key metrics collected:
- Requests per second
- Bytes per second
- Workers (busy, idle)
- Scoreboard statistics
- CPU load

### MongoDB Metrics

```yaml
- module: mongodb
  metricsets: ["status", "dbstats"]
  period: 10s
  hosts: ["mongodb://localhost:27017"]
```

Key metrics collected:
- Operations counts (inserts, queries, updates, deletes)
- Connection statistics
- Memory usage
- Database statistics
- Replication status
- Lock information

## Cloud Provider Metrics

Metricbeat can collect metrics from major cloud providers.

### AWS Metrics

```yaml
- module: aws
  period: 300s
  metricsets:
    - cloudwatch
    - sqs
    - s3_daily_storage
    - s3_request
  regions:
    - us-east-1
  access_key_id: '${AWS_ACCESS_KEY_ID}'
  secret_access_key: '${AWS_SECRET_ACCESS_KEY}'
  cloudwatch:
    metrics:
      - namespace: AWS/EC2
        name: ["CPUUtilization", "DiskReadOps"]
        resource_type: ec2:instance
        dimensions:
          - name: InstanceId
            value: i-0123456789
      - namespace: AWS/EBS
        resource_type: ebs
```

Key metrics collected:
- EC2 instance metrics (CPU, network, disk)
- EBS volume metrics
- S3 bucket metrics
- SQS queue metrics
- RDS database metrics
- CloudWatch custom metrics

### Azure Metrics

```yaml
- module: azure
  period: 300s
  metricsets:
    - monitor
  client_id: '${AZURE_CLIENT_ID}'
  client_secret: '${AZURE_CLIENT_SECRET}'
  tenant_id: '${AZURE_TENANT_ID}'
  subscription_id: '${AZURE_SUBSCRIPTION_ID}'
  monitor:
    resources:
      - resource_query: "resourceType eq 'Microsoft.Compute/virtualMachines'"
        metrics:
          - name: ["Percentage CPU", "Network In", "Network Out"]
      - resource_query: "resourceType eq 'Microsoft.Storage/storageAccounts'"
        metrics:
          - name: ["UsedCapacity", "Transactions"]
```

Key metrics collected:
- Virtual machine metrics
- Storage account metrics
- App Service metrics
- SQL database metrics
- Networking metrics

### Google Cloud Metrics

```yaml
- module: gcp
  period: 300s
  metricsets:
    - compute
    - pubsub
    - storage
  project_id: "${GCP_PROJECT_ID}"
  credentials_file_path: "/etc/metricbeat/gcp-credentials.json"
  compute:
    regions:
      - us-central1
    metrics:
      - name: "instance/cpu/utilization"
      - name: "instance/disk/read_bytes_count"
      - name: "instance/disk/write_bytes_count"
      - name: "instance/network/received_bytes_count"
      - name: "instance/network/sent_bytes_count"
```

Key metrics collected:
- Compute Engine metrics
- Cloud Storage metrics
- Cloud Pub/Sub metrics
- Cloud SQL metrics
- Billing metrics

## Kubernetes Metrics

Metricbeat provides comprehensive monitoring for Kubernetes environments.

### Kubernetes Node Metrics

```yaml
- module: kubernetes
  metricsets:
    - node
    - system
    - pod
    - container
    - volume
  period: 10s
  host: "https://${NODE_NAME}:10250"
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  ssl.verification_mode: "none"
  add_metadata: true
```

Collects metrics from kubelet on each node:
- Node resource usage
- Pod resource usage
- Container metrics
- Volume statistics

### Kubernetes State Metrics

```yaml
- module: kubernetes
  metricsets:
    - state_node
    - state_deployment
    - state_replicaset
    - state_pod
    - state_container
  period: 10s
  hosts: ["kube-state-metrics:8080"]
  add_metadata: true
```

Collects state metrics from kube-state-metrics:
- Deployment status and availability
- Pod status and phase
- ReplicaSet desired/current counts
- Node conditions and capacity
- Resource limits and requests

### Kubernetes API Server Metrics

```yaml
- module: kubernetes
  metricsets:
    - apiserver
  hosts: ["https://kubernetes.default.svc.cluster.local:443"]
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  ssl.verification_mode: "none"
  period: 30s
```

Collects metrics from the Kubernetes API server:
- API request rates
- API latency
- Etcd operations
- Authentication attempts
- Request duration histograms

### Kubernetes Events

```yaml
- module: kubernetes
  metricsets:
    - event
  period: 10s
```

Collects Kubernetes events:
- Pod lifecycle events
- Node events
- Deployment events
- Warning events
- Error events

### Full Kubernetes Monitoring Example

```yaml
- module: kubernetes
  metricsets:
    # Node metrics
    - node
    - system
    - pod
    - container
    - volume
    
    # State metrics
    - state_node
    - state_deployment
    - state_replicaset
    - state_pod
    - state_container
    - state_statefulset
    - state_cronjob
    
    # Control plane
    - apiserver
    - controllermanager
    - scheduler
    
    # Other
    - proxy
    - event
  period: 10s
  hosts:
    - "https://${NODE_NAME}:10250"  # For kubelet
    - "kube-state-metrics:8080"     # For kube-state-metrics
    - "https://kubernetes.default"   # For API server
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  ssl.verification_mode: "none"
  add_metadata: true
  node.names: ["${NODE_NAME}"]
```

## Processors and Enrichment

Processors transform events before they are sent to the output, allowing for data enrichment, filtering, and transformation.

### Common Processors

1. **Add Fields**:
   ```yaml
   processors:
     - add_fields:
         target: ''
         fields:
           environment: production
           datacenter: us-east-1
   ```

2. **Drop Fields**:
   ```yaml
   processors:
     - drop_fields:
         fields: ["agent.ephemeral_id", "ecs.version"]
         ignore_missing: true
   ```

3. **Rename Fields**:
   ```yaml
   processors:
     - rename:
         fields:
           - from: "host.name"
             to: "server.name"
         ignore_missing: true
         fail_on_error: false
   ```

4. **Add Host Metadata**:
   ```yaml
   processors:
     - add_host_metadata:
         netinfo.enabled: true
         geo:
           name: nyc-dc-1
           location: 40.7128, -74.0060
           continent_name: North America
           country_iso_code: US
   ```

5. **Add Cloud Metadata**:
   ```yaml
   processors:
     - add_cloud_metadata:
         providers: [aws, azure, gcp]
         timeout: 3s
   ```

### Conditional Processing

Apply processors based on conditions:

```yaml
processors:
  - if:
      contains:
        system.filesystem.mount_point: "/var"
      equals:
        system.memory.used.pct: 0.9
    then:
      - add_tag:
          tags: ["high_utilization"]
      - add_fields:
          target: ''
          fields:
            alert_level: warning
    else:
      - drop_event: {}
```

### Metric-Specific Processors

1. **Unit Conversion**:
   ```yaml
   processors:
     - convert:
         fields:
           - from: "system.memory.total"
             to: "system.memory.total_gb"
             type: float
             scale: 0.000000001
         ignore_missing: true
         fail_on_error: false
   ```

2. **Rate Calculation**:
   ```yaml
   processors:
     - rate:
         fields:
           - from: "system.network.in.bytes"
             to: "system.network.in.bytes_per_sec"
             unit: second
         ignore_missing: true
   ```

3. **Histogram Calculation**:
   ```yaml
   processors:
     - histogram:
         field: system.cpu.total.pct
         interval: 0.1
   ```

## Output Configuration

Metricbeat can send data to various outputs, with Elasticsearch and Logstash being the most common.

### Elasticsearch Output

Send data directly to Elasticsearch:

```yaml
output.elasticsearch:
  hosts: ["https://elasticsearch:9200"]
  index: "metricbeat-%{[agent.version]}-%{+yyyy.MM.dd}"
  username: "elastic"
  password: "changeme"
  ssl:
    certificate_authorities: ["/etc/pki/root/ca.pem"]
    certificate: "/etc/pki/client/cert.pem"
    key: "/etc/pki/client/cert.key"
    verification_mode: full
  worker: 3
  bulk_max_size: 50
  compression_level: 3
  loadbalance: true
  max_retries: 5
  backoff.init: 1s
  backoff.max: 60s
```

### Logstash Output

Send data to Logstash for additional processing:

```yaml
output.logstash:
  hosts: ["logstash:5044"]
  loadbalance: true
  ttl: 30s
  index: "metricbeat"
  ssl:
    enabled: true
    verification_mode: none
    certificate_authorities: ["/etc/pki/root/ca.pem"]
    certificate: "/etc/pki/client/cert.pem"
    key: "/etc/pki/client/cert.key"
  worker: 2
```

### Kafka Output

Send data to Apache Kafka:

```yaml
output.kafka:
  hosts: ["kafka1:9092", "kafka2:9092"]
  topic: "metricbeat"
  partition.round_robin:
    reachable_only: true
  compression: gzip
  required_acks: 1
  max_message_bytes: 1000000
  client_id: metricbeat
```

### Multiple Outputs

Send data to multiple destinations:

```yaml
output.elasticsearch:
  enabled: true
  hosts: ["elasticsearch:9200"]
  index: "metricbeat-%{[agent.version]}-%{+yyyy.MM.dd}"

output.logstash:
  enabled: true
  hosts: ["logstash:5044"]
```

## Advanced Features

### Auto-discovery

Automatically discover and monitor services in dynamic environments:

```yaml
metricbeat.autodiscover:
  providers:
    - type: docker
      templates:
        - condition:
            contains:
              docker.container.image: nginx
          config:
            - module: nginx
              metricsets: ["stubstatus"]
              period: 10s
              hosts: ["${data.host}:80"]
```

### Kubernetes Auto-discovery

Discover and monitor services in Kubernetes:

```yaml
metricbeat.autodiscover:
  providers:
    - type: kubernetes
      node: ${NODE_NAME}
      hints.enabled: true
      templates:
        - condition:
            equals:
              kubernetes.namespace: default
          config:
            - module: redis
              metricsets: ["info", "keyspace"]
              period: 10s
              hosts: ["${data.host}:6379"]
```

### Central Configuration with Fleet

Configure Metricbeat through Kibana Fleet:

1. Install Elastic Agent:
   ```bash
   curl -L -O https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.1.0-linux-x86_64.tar.gz
   tar xzvf elastic-agent-8.1.0-linux-x86_64.tar.gz
   cd elastic-agent-8.1.0-linux-x86_64
   sudo ./elastic-agent install --url=https://fleet-server:8220 --enrollment-token=<token>
   ```

2. Configure integrations in Kibana:
   - Navigate to Fleet > Integrations
   - Add System integration for system metrics
   - Add service-specific integrations as needed
   - Configure settings and deploy

### HTTP Endpoint for Scraping

Configure Metricbeat to expose metrics for scraping:

```yaml
http.enabled: true
http.host: 0.0.0.0
http.port: 5066
```

Metrics are exposed at `http://localhost:5066/stats` and include:
- Libbeat metrics (events, outputs, pipeline)
- System metrics
- Per-module metrics

### Metrics API

Access internal metrics via the API:

```bash
# Get all stats
curl -XGET http://localhost:5066/stats | jq

# Get specific stats
curl -XGET http://localhost:5066/stats?metrics=beat | jq
```

## Performance Tuning

### System Configuration

Optimize system settings for Metricbeat:

1. **File Descriptor Limits**:
   ```bash
   # /etc/security/limits.conf
   metricbeat soft nofile 65536
   metricbeat hard nofile 65536
   ```

2. **Memory and CPU**:
   - Allocate sufficient resources based on enabled modules
   - Monitor memory usage and adjust settings if needed

### Metricbeat Configuration Tuning

Optimize Metricbeat configuration for performance:

1. **Period Settings**:
   - Adjust collection intervals based on need vs. overhead
   - Use longer periods for stable metrics
   - Use shorter periods for critical monitoring
   ```yaml
   period: 30s  # Default
   period: 10s  # For more responsive monitoring
   period: 60s  # For stable metrics with lower overhead
   ```

2. **Module Selection**:
   - Enable only necessary modules
   - Enable only required metricsets within modules
   - Disable unnecessary metrics
   ```yaml
   metricsets: ["cpu", "memory", "network"]  # Instead of all
   ```

3. **Output Optimization**:
   ```yaml
   output.elasticsearch:
     worker: 4
     bulk_max_size: 50
     compression_level: 3
   ```

### Resource Considerations

Understanding resource impact of different features:

1. **High CPU Impact**:
   - Too many modules enabled
   - Short collection periods
   - Complex processors
   - Docker/Kubernetes monitoring with many containers

2. **High Memory Impact**:
   - Large number of metricsets
   - Long-running internal queues
   - Unreachable outputs causing backpressure
   - Data buffering during output unavailability

## Monitoring Metricbeat

### Self Monitoring

Enable Metricbeat to report its own metrics:

```yaml
monitoring.enabled: true
monitoring.elasticsearch:
  hosts: ["https://monitoring-es:9200"]
  username: "beats_system"
  password: "changeme"
```

Or use Metricbeat to monitor itself:

```yaml
- module: beat
  metricsets: ["stats", "state"]
  period: 10s
  hosts: ["http://localhost:5066"]
  xpack.enabled: true
```

### Health Metrics

Key metrics to monitor:

1. **System Metrics**:
   - CPU usage
   - Memory usage
   - File descriptors
   - Disk I/O

2. **Metricbeat Metrics**:
   - Events rate (collected, published)
   - Memory usage
   - Output errors/retries
   - Module-specific metrics

### Logging Configuration

Configure Metricbeat's own logs:

```yaml
logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/metricbeat
  name: metricbeat
  keepfiles: 7
  permissions: 0644
logging.metrics.enabled: true
logging.metrics.period: 30s
```

## Troubleshooting

### Common Issues

1. **No Metrics Being Collected**:
   - Incorrect module configuration
   - Service not accessible
   - Authentication issues
   - Insufficient permissions

2. **High CPU/Memory Usage**:
   - Too many modules enabled
   - Collection period too short
   - Complex processing
   - Many discovered services

3. **Output Issues**:
   - Network connectivity problems
   - Authentication failures
   - Index mapping errors
   - Elasticsearch cluster health

### Diagnostic Techniques

1. **Test Configuration**:
   ```bash
   metricbeat test config -c /etc/metricbeat/metricbeat.yml
   ```

2. **Test Output Connectivity**:
   ```bash
   metricbeat test output -c /etc/metricbeat/metricbeat.yml
   ```

3. **Test Modules**:
   ```bash
   metricbeat test modules system
   ```

4. **Debug Mode**:
   ```bash
   metricbeat -e -c /etc/metricbeat/metricbeat.yml -d "*"
   ```

5. **Check System Logs**:
   ```bash
   journalctl -u metricbeat
   ```

### Solving Common Problems

1. **Permission Issues**:
   ```bash
   # Check if Metricbeat can access required resources
   sudo -u metricbeat ls -la /var/run/docker.sock
   
   # Fix permissions if needed
   sudo usermod -aG docker metricbeat
   ```

2. **Service Discovery Issues**:
   ```bash
   # Check if services are accessible
   curl -v http://nginx-server/nginx_status
   
   # Verify Kubernetes API access
   curl -v --insecure --header "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" https://kubernetes.default.svc.cluster.local/api/v1/nodes
   ```

3. **Output Issues**:
   ```bash
   # Test Elasticsearch connectivity
   curl -v http://elasticsearch:9200
   
   # Check Elasticsearch cluster health
   curl -v http://elasticsearch:9200/_cluster/health
   ```

## Best Practices

### Deployment Strategies

1. **Installation**:
   - Use official packages when possible
   - Keep Metricbeat updated to latest minor version
   - Automate deployment with configuration management
   - Consider containerization for consistent environments

2. **Configuration Management**:
   - Use version control for configurations
   - Implement CI/CD for config validation
   - Use templates with environment-specific variables
   - Document all custom configurations

3. **Scaling**:
   - One Metricbeat per host for system metrics
   - Dedicated instances for cluster-level metrics
   - Consider resource limits in containerized environments

### Security Recommendations

1. **Authentication**:
   - Use dedicated user accounts with minimal privileges
   - Rotate credentials regularly
   - Use environment variables for sensitive information
   - Store credentials in secure locations

2. **Encryption**:
   - Enable TLS for all communications
   - Verify certificates properly
   - Use secure key storage
   - Implement proper certificate management

3. **Access Control**:
   - Limit metrics collection to necessary services
   - Use network segmentation
   - Implement service-level authentication
   - Follow least privilege principle

### Operational Guidelines

1. **Monitoring**:
   - Set up alerts for Metricbeat failures
   - Monitor event throughput
   - Track CPU and memory usage
   - Set up log loss detection

2. **Upgrades**:
   - Test upgrades in non-production first
   - Plan for breaking changes
   - Consider rolling upgrades for high-availability
   - Archive old metric data if index templates change

3. **Documentation**:
   - Document all custom configurations
   - Maintain deployment architecture diagrams
   - Keep runbooks for common issues
   - Document metrics sources and meanings

### Configuration Tips

1. **General Best Practices**:
   - Keep configurations simple and modular
   - Use appropriate collection periods
   - Implement proper logging
   - Validate configurations before deployment

2. **Performance Optimization**:
   - Only collect necessary metrics
   - Use conditional collection where possible
   - Balance collection frequency and overhead
   - Monitor and adjust resource usage

3. **Resilience**:
   - Configure proper retry behavior
   - Implement output load balancing
   - Consider secondary outputs for critical metrics
   - Monitor for missed collections

## Conclusion

Metricbeat provides a robust, lightweight solution for collecting metrics from systems, services, and cloud providers. By properly configuring and optimizing Metricbeat, you can achieve comprehensive visibility into your infrastructure with minimal overhead. The modular architecture and extensive integrations make it adaptable to virtually any environment, from simple on-premises deployments to complex multi-cloud infrastructures.

In the next chapter, we'll explore Packetbeat, which captures network traffic to provide visibility into your application communication.