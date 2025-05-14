# Filebeat

This chapter provides a comprehensive guide to Filebeat, the lightweight log shipper for forwarding and centralizing log data from files.

## Table of Contents
- [Filebeat Overview](#filebeat-overview)
- [Architecture and Components](#architecture-and-components)
- [Installation and Setup](#installation-and-setup)
- [Basic Configuration](#basic-configuration)
- [Input Types](#input-types)
- [Modules](#modules)
- [Processors](#processors)
- [Output Configuration](#output-configuration)
- [Advanced Features](#advanced-features)
- [Performance Tuning](#performance-tuning)
- [Monitoring Filebeat](#monitoring-filebeat)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)

## Filebeat Overview

Filebeat is a lightweight log shipper designed to collect and forward log data from files with minimal resource overhead. It's part of the Beats family of lightweight, single-purpose data shippers.

### Key Features

- **Lightweight**: Small memory footprint and minimal CPU usage
- **Reliable Delivery**: Ensures log data reaches its destination
- **Backpressure-Sensitive**: Adjusts collection speed based on output capacity
- **Secure**: Supports TLS/SSL encryption and authentication
- **Centralized Configuration**: When used with Elastic Stack, configuration can be centrally managed
- **Built-in Modules**: Pre-configured settings for common log types
- **Data Enrichment**: Add metadata and transform logs during collection
- **Auto-Discovery**: Detect and collect logs from containerized environments

### Common Use Cases

1. **Centralized Logging**: Forward logs from multiple sources to a central location
2. **Application Monitoring**: Collect application logs for performance and error analysis
3. **Security Analytics**: Ship security logs for threat detection and compliance
4. **Audit Logging**: Collect and centralize audit logs for compliance and security
5. **Container Monitoring**: Collect logs from Docker, Kubernetes, and other containerized environments
6. **Cloud Integration**: Ship logs from cloud services and applications

## Architecture and Components

Filebeat's architecture consists of several key components that work together to reliably collect and ship logs.

### Core Components

1. **Input**: Configures the log sources (files, containers, etc.)
2. **Harvester**: Reads each file and sends content to the spooler
3. **Spooler**: Aggregates events before publishing to the output
4. **Registry**: Tracks file state to ensure reliable delivery
5. **Processors**: Transform events before sending to the output
6. **Output**: Sends events to the configured destination

### Data Flow

```
  Files/Sources → Harvester → Spooler → Processors → Output
        ↑                        ↓
        |                        |
        └──────── Registry ──────┘
```

1. **Harvester** reads a file line by line
2. **Registry** keeps track of harvesting state
3. **Spooler** buffers events and sends to output
4. **Processors** enrich and transform events
5. **Output** sends events to the destination

### Registry Files

Filebeat maintains a registry to track the state of files it's monitoring:

- **Purpose**: Ensure each log line is sent exactly once
- **Tracked Information**:
  - File identity (inode, device)
  - Last read position
  - Source information
  - Harvester status

- **Location**:
  - Default: `data/registry/filebeat` directory
  - Configurable via `filebeat.registry.path`

## Installation and Setup

### System Requirements

- **Memory**: 100MB minimum (varies with configuration)
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
   curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.1.0-amd64.deb
   sudo dpkg -i filebeat-8.1.0-amd64.deb
   
   # RHEL/CentOS
   curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.1.0-x86_64.rpm
   sudo rpm -vi filebeat-8.1.0-x86_64.rpm
   ```

2. **Windows**:
   - Download installer from Elastic website
   - Install using PowerShell as administrator:
     ```powershell
     # Install as a Windows service
     PS> .\filebeat.exe install
     ```

3. **Docker**:
   ```bash
   docker pull docker.elastic.co/beats/filebeat:8.1.0
   docker run docker.elastic.co/beats/filebeat:8.1.0
   ```

4. **Kubernetes**:
   - Use Helm chart or Elastic Cloud on Kubernetes operator
   - Apply Filebeat DaemonSet to collect logs from all nodes:
     ```bash
     kubectl apply -f filebeat-kubernetes.yaml
     ```

### Initial Setup

1. **Configuration File**: Edit `filebeat.yml` in the configuration directory
2. **Configure Outputs**: Set up Elasticsearch or Logstash output
3. **Enable Modules**: Turn on modules for specific log types
4. **Set Up Assets**: Run setup command to load index templates:
   ```bash
   # Load Elasticsearch index template
   filebeat setup --index-management
   
   # Load dashboards to Kibana
   filebeat setup --dashboards
   ```
5. **Start Filebeat**:
   ```bash
   # Systemd (Linux)
   sudo systemctl start filebeat
   
   # Windows
   Start-Service filebeat
   ```

## Basic Configuration

The `filebeat.yml` file is the main configuration file for Filebeat, typically located in `/etc/filebeat/` on Linux or `C:\Program Files\Filebeat\` on Windows.

### Configuration Structure

```yaml
filebeat.inputs:                # Input configuration
filebeat.modules:               # Modules configuration
output.elasticsearch:           # Output configuration
processors:                     # Processors for data transformation
setup:                          # Setup configurations
logging:                        # Logging configurations
```

### Minimal Configuration Example

```yaml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/*.log
    - /var/log/messages

output.elasticsearch:
  hosts: ["localhost:9200"]
  username: "elastic"
  password: "changeme"
```

### Common Settings

1. **General Settings**:
   ```yaml
   name: "my-filebeat"          # Custom name for the Beat
   tags: ["production", "web"]  # Tags for all events
   fields:                      # Custom fields for all events
     env: production
     dc: us-east-1
   fields_under_root: true      # Places custom fields at root level
   ```

2. **Logging Settings**:
   ```yaml
   logging.level: info
   logging.to_files: true
   logging.files:
     path: /var/log/filebeat
     name: filebeat
     keepfiles: 7
     permissions: 0644
   ```

3. **Filebeat-Specific Settings**:
   ```yaml
   filebeat.registry.path: /var/lib/filebeat/registry
   filebeat.registry.file_permissions: 0600
   filebeat.shutdown_timeout: 5s
   filebeat.config.modules.path: ${path.config}/modules.d/*.yml
   ```

## Input Types

Filebeat supports various input types to collect logs from different sources.

### Log Input

Collects logs from text files:

```yaml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/*.log
    - /var/log/apache2/*log
  exclude_files: ['\.gz$']
  include_lines: ['^ERR', '^WARN']
  exclude_lines: ['^DEBUG']
  tags: ["apache"]
  fields:
    source_type: apache
  fields_under_root: true
  multiline:
    pattern: '^[[:space:]]'
    negate: false
    match: after
  tail_files: true
  scan_frequency: 10s
  close_inactive: 5m
  harvester_buffer_size: 16384
  max_bytes: 10485760
```

### Container Input

Collects logs from containers:

```yaml
filebeat.inputs:
- type: container
  enabled: true
  paths:
    - /var/lib/docker/containers/*/*.log
  json.message_key: log
  json.keys_under_root: true
  processors:
    - add_docker_metadata:
        host: "unix:///var/run/docker.sock"
```

### Syslog Input

Listens for syslog messages:

```yaml
filebeat.inputs:
- type: syslog
  enabled: true
  protocol.tcp:
    host: "localhost:9000"
  format: auto
```

### UDP Input

Receives logs over UDP:

```yaml
filebeat.inputs:
- type: udp
  enabled: true
  host: "localhost:9000"
  max_message_size: 10KiB
```

### TCP Input

Receives logs over TCP:

```yaml
filebeat.inputs:
- type: tcp
  enabled: true
  host: "localhost:9000"
  ssl.enabled: true
  ssl.certificate: "/etc/pki/tls/certs/server.crt"
  ssl.key: "/etc/pki/tls/private/server.key"
```

### HTTP JSON Input

Receives JSON events over HTTP:

```yaml
filebeat.inputs:
- type: http_endpoint
  enabled: true
  host: "localhost:8080"
  response_code: 200
  response_body: '{"message": "success"}'
```

### AWS S3 Input (Requires AWS plugin)

Collects logs from S3 buckets:

```yaml
filebeat.inputs:
- type: s3
  enabled: true
  queue_url: https://sqs.us-east-1.amazonaws.com/123456/my-queue
  credential_profile_name: elastic-beats
```

### Azure Event Hub Input

Collects logs from Azure Event Hub:

```yaml
filebeat.inputs:
- type: azure-eventhub
  enabled: true
  connection_string: "${EVENTHUB_CONNECTION_STRING}"
  eventhub: "insights-operational-logs"
  consumer_group: "$Default"
  storage_account: "${STORAGE_ACCOUNT}"
  storage_account_key: "${STORAGE_KEY}"
  storage_account_container: "filebeat-eh-container"
```

## Modules

Filebeat modules provide pre-built configurations for collecting, parsing, and visualizing common log types.

### Available Modules

- **System**: Linux and Windows system logs
- **Apache**: HTTP server logs
- **Nginx**: HTTP server logs
- **MySQL**: Database logs
- **AWS**: Various AWS service logs
- **Azure**: Azure platform logs
- **Cisco**: Network device logs
- **Elasticsearch**: Elasticsearch logs
- **Google Cloud**: GCP service logs
- **IIS**: Microsoft IIS web server logs
- **Kafka**: Apache Kafka logs
- **Logstash**: Logstash plain and JSON logs
- **MongoDB**: Database logs
- **Netflow**: Network flow logs
- **Osquery**: OS state monitoring logs
- **PostgreSQL**: Database logs
- **Redis**: In-memory data structure store logs
- **Suricata**: Network security monitoring logs

### Module Activation

Enable modules via configuration:

```yaml
filebeat.modules:
- module: system
  syslog:
    enabled: true
    var.paths: ["/var/log/syslog"]
  auth:
    enabled: true
    var.paths: ["/var/log/auth.log"]
```

Or using the command line:

```bash
# List available modules
filebeat modules list

# Enable specific modules
filebeat modules enable system nginx mysql

# Configure module settings
filebeat modules enable system -E 'filebeat.modules.0.syslog.var.paths=["/var/log/syslog*"]'
```

### Module Configuration

Configure modules via YAML files in `modules.d/` directory:

```yaml
# modules.d/apache.yml
- module: apache
  access:
    enabled: true
    var.paths: ["/var/log/apache2/access.log*"]
  error:
    enabled: true
    var.paths: ["/var/log/apache2/error.log*"]
```

### Module Variables

Each module has variables that can be customized:

```yaml
- module: nginx
  access:
    enabled: true
    var.paths: ["/var/log/nginx/access.log*"]
    var.pipeline: "nginx-access-pipeline"
  error:
    enabled: true
    var.paths: ["/var/log/nginx/error.log*"]
    var.format: ""
```

### Custom Modules

Create custom modules for specialized log formats:

1. Create module directory structure:
   ```
   my-module/
   ├── module.yml
   ├── manifest.yml
   ├── ingest/
   │   └── pipeline.yml
   ├── config/
   │   └── my-module.yml
   └── kibana/
       ├── dashboard/
       └── visualization/
   ```

2. Define module configuration:
   ```yaml
   # module.yml
   module_name: my-module
   var:
     - name: paths
       default:
         - /var/log/my-app/*.log
   ```

3. Create ingest pipeline:
   ```yaml
   # pipeline.yml
   processors:
     - grok:
         field: message
         patterns:
           - '%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:log.level} %{GREEDYDATA:message}'
   ```

## Processors

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
           - from: "source"
             to: "source_ip"
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

5. **Add Process Metadata**:
   ```yaml
   processors:
     - add_process_metadata:
         match_pids: [process.pid]
         target: process
         include_fields: ["process.name", "process.executable"]
   ```

6. **Community ID for Network Data**:
   ```yaml
   processors:
     - community_id:
         fields:
           source_ip: source.ip
           destination_ip: destination.ip
           source_port: source.port
           destination_port: destination.port
           transport: network.transport
   ```

### Conditional Processing

Apply processors based on conditions:

```yaml
processors:
  - if:
      contains:
        message: "ERROR"
      regexp:
        message: "^\[ERROR\]"
      equals:
        source: "/var/log/app.log"
    then:
      - add_tag:
          tags: ["error"]
      - add_fields:
          target: ''
          fields:
            error_found: true
    else:
      - drop_event: {}
```

### Processing Order

Processors run in the order defined in the configuration:

```yaml
processors:
  # First convert JSON
  - decode_json_fields:
      fields: ["message"]
      target: "json"
  
  # Then extract specific fields
  - extract_field:
      field: "json.user_id"
      target: "user.id"
  
  # Drop sensitive data
  - drop_fields:
      fields: ["json.password", "json.credit_card"]
  
  # Add contextual information
  - add_host_metadata: {}
```

### Advanced Processing Examples

1. **Parse User-Agent Strings**:
   ```yaml
   processors:
     - user_agent:
         field: user_agent
         target_field: user_agent
         ignore_missing: true
   ```

2. **Geo-IP Enrichment**:
   ```yaml
   processors:
     - geoip:
         field: client.ip
         target_field: client.geo
         database_file: /path/to/GeoLite2-City.mmdb
         ignore_missing: true
   ```

3. **URL Parsing**:
   ```yaml
   processors:
     - urldecode:
         field: http.request.uri
         target_field: url.decoded
     - dissect:
         field: "url.decoded"
         pattern: "%{url.path}?%{url.query}"
   ```

4. **Timestamp Conversion**:
   ```yaml
   processors:
     - timestamp:
         field: "log_timestamp"
         layouts:
           - "2006-01-02T15:04:05Z"
           - "2006/01/02 15:04:05"
         timezone: "UTC"
         target_field: "@timestamp"
   ```

## Output Configuration

Filebeat can send data to various outputs, with Elasticsearch and Logstash being the most common.

### Elasticsearch Output

Send data directly to Elasticsearch:

```yaml
output.elasticsearch:
  hosts: ["https://elasticsearch:9200"]
  index: "filebeat-%{[agent.version]}-%{+yyyy.MM.dd}"
  username: "elastic"
  password: "changeme"
  ssl:
    certificate_authorities: ["/etc/pki/root/ca.pem"]
    certificate: "/etc/pki/client/cert.pem"
    key: "/etc/pki/client/cert.key"
    verification_mode: full
  bulk_max_size: 50
  worker: 3
  compression_level: 3
  pipeline: "filebeat-pipeline"
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
  proxy_url: http://proxy:3128
  index: "filebeat"
  slow_start: true
  ssl:
    enabled: true
    verification_mode: none
    certificate_authorities: ["/etc/pki/root/ca.pem"]
    certificate: "/etc/pki/client/cert.pem"
    key: "/etc/pki/client/cert.key"
  timeout: 15
  pipelining: 2
  worker: 2
```

### Kafka Output

Send data to Apache Kafka:

```yaml
output.kafka:
  hosts: ["kafka1:9092", "kafka2:9092"]
  topic: "filebeat"
  key: "%{[fields.id]}"
  partition.round_robin:
    reachable_only: true
  compression: gzip
  required_acks: 1
  max_message_bytes: 1000000
  client_id: filebeat
```

### Redis Output

Send data to Redis:

```yaml
output.redis:
  hosts: ["redis:6379"]
  password: "password"
  key: "filebeat"
  db: 0
  timeout: 5s
  datatype: list
```

### File Output

Write to file (useful for debugging):

```yaml
output.file:
  path: "/tmp/filebeat"
  filename: filebeat
  rotate_every_kb: 10000
  number_of_files: 7
  permissions: 0600
  codec.format:
    string: '%{[message]}'
```

### Console Output

Print to standard output (useful for testing):

```yaml
output.console:
  pretty: true
  codec.format:
    string: '%{[message]}'
```

### Multiple Outputs

Send data to multiple destinations:

```yaml
output.elasticsearch:
  enabled: true
  hosts: ["elasticsearch:9200"]
  index: "filebeat-%{[agent.version]}-%{+yyyy.MM.dd}"

output.logstash:
  enabled: true
  hosts: ["logstash:5044"]

output.file:
  enabled: true
  path: "/tmp/filebeat-copy"
  filename: "filebeat.json"
```

## Advanced Features

### Multiline Log Handling

Configure Filebeat to handle multiline log entries, such as stack traces:

```yaml
filebeat.inputs:
- type: log
  paths:
    - /var/log/java-app.log
  multiline:
    pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2}'
    negate: true
    match: after
    max_lines: 500
    timeout: 5s
```

Multiline examples for common patterns:

1. **Java Stack Traces**:
   ```yaml
   multiline:
     pattern: '^[\\t ]+(at|\.{3})[\\t ]'
     negate: false
     match: after
   ```

2. **Python Tracebacks**:
   ```yaml
   multiline:
     pattern: '^[\\t ]+File \"[^\"]+\", line [0-9]+'
     negate: false
     match: after
   ```

3. **Ruby Exceptions**:
   ```yaml
   multiline:
     pattern: '^\s+(from|[.][.][.])'
     negate: false
     match: after
   ```

### Log Rotation Handling

Filebeat automatically handles common log rotation scenarios:

```yaml
filebeat.inputs:
- type: log
  paths:
    - /var/log/syslog*
  scan_frequency: 10s
  clean_inactive: 24h
  ignore_older: 12h
  close_inactive: 5m
  close_renamed: true
  close_removed: true
  close_eof: false
```

Configuration explanations:
- `scan_frequency`: How often Filebeat checks for new files
- `clean_inactive`: Remove state for files not seen for this duration
- `ignore_older`: Ignore files modified before this duration
- `close_inactive`: Close file handle after inactivity
- `close_renamed`: Close file when renamed
- `close_removed`: Close file when removed
- `close_eof`: Close file when EOF is reached

### Docker Integration

Collect and enrich logs from Docker containers:

```yaml
filebeat.inputs:
- type: container
  paths:
    - /var/lib/docker/containers/*/*.log
  json.message_key: log
  json.keys_under_root: true
  json.add_error_key: true
  json.overwrite_keys: false
  
processors:
  - add_docker_metadata:
      host: "unix:///var/run/docker.sock"
  - add_labels:
      labels.dedot: true
  - add_kubernetes_metadata:
      in_cluster: true
```

### Kubernetes Integration

Collect logs from Kubernetes pods:

```yaml
filebeat.autodiscover:
  providers:
    - type: kubernetes
      node: ${NODE_NAME}
      hints.enabled: true
      hints.default_config:
        type: container
        paths:
          - /var/log/containers/*${data.kubernetes.container.id}.log
      templates:
        - config:
            - type: container
              paths:
                - /var/log/containers/*${data.kubernetes.container.id}.log
              exclude_lines: ["^\\s+[\\-`('.|_]"]

processors:
  - add_kubernetes_metadata:
      host: ${NODE_NAME}
      matchers:
        - logs_path:
            logs_path: "/var/log/containers/"
```

### Auto-discovery

Automatically discover and monitor logs from dynamic environments:

```yaml
filebeat.autodiscover:
  providers:
    - type: docker
      hints.enabled: true
      templates:
        - condition:
            contains:
              docker.container.image: nginx
          config:
            - type: container
              paths:
                - /var/lib/docker/containers/${data.docker.container.id}/*.log
              exclude_lines: ["^\\s+[\\-`('.|_]"]
              processors:
                - add_docker_metadata: ~
                - add_fields:
                    target: ''
                    fields:
                      service: nginx
```

### Using Elastic Common Schema (ECS)

Ensure logs conform to the Elastic Common Schema:

```yaml
filebeat.inputs:
- type: log
  paths: ["/var/log/syslog"]
  fields_under_root: true
  fields:
    ecs.version: "1.8.0"

processors:
  - convert:
      fields:
        - {from: "source", to: "event.original"}
      ignore_missing: true
      fail_on_error: false
  - rename:
      fields:
        - {from: "host", to: "host.name"}
      ignore_missing: true
```

## Performance Tuning

### System Configuration

Optimize system settings for Filebeat:

1. **File Descriptor Limits**:
   ```bash
   # /etc/security/limits.conf
   filebeat soft nofile 65536
   filebeat hard nofile 65536
   ```

2. **Memory and CPU**:
   - Run Filebeat on a separate host if possible
   - Dedicate CPU cores for Filebeat in high-volume scenarios
   - Monitor memory usage and adjust settings if needed

### Filebeat Configuration Tuning

Optimize Filebeat configuration for performance:

1. **Harvester Settings**:
   ```yaml
   filebeat.inputs:
   - type: log
     harvester_buffer_size: 16384
     max_bytes: 10485760
     backoff.init: 1s
     backoff.max: 20s
   ```

2. **Spooler and Queue Settings**:
   ```yaml
   filebeat.spool_size: 2048
   filebeat.idle_timeout: 5s
   queue.mem.events: 4096
   queue.mem.flush.min_events: 512
   queue.mem.flush.timeout: 5s
   ```

3. **Output Optimization**:
   ```yaml
   output.elasticsearch:
     worker: 4
     bulk_max_size: 50
     compression_level: 5
   ```

### Resource Usage Considerations

Understand resource impacts of different features:

1. **High CPU Usage Causes**:
   - Too many harvesters running simultaneously
   - Complex processors (regex, grok)
   - High log volume
   - Frequent backoffs due to output issues

2. **High Memory Usage Causes**:
   - Large registry files
   - Large harvester buffers
   - Large queue sizes
   - Memory leaks in older versions

3. **High Disk I/O Causes**:
   - Frequent registry updates
   - Reading many small files
   - Slow disk subsystem
   - File output with frequent rotations

### Scaling Strategies

Approaches for scaling Filebeat for high-volume environments:

1. **Horizontal Scaling**:
   - Deploy multiple Filebeat instances
   - Split log collection responsibility
   - Use load balancing for outputs

2. **Filtering at Source**:
   - Use `include_lines` and `exclude_lines` to filter early
   - Apply multiline processing efficiently
   - Drop unnecessary events using processors

3. **Output Configuration**:
   - Increase worker count for parallel processing
   - Use compression to reduce network bandwidth
   - Adjust bulk sizes based on document size

## Monitoring Filebeat

### Self Monitoring

Enable Filebeat to report its own metrics:

```yaml
monitoring.enabled: true
monitoring.elasticsearch:
  hosts: ["https://monitoring-es:9200"]
  username: "beats_system"
  password: "changeme"
```

Or send to Metricbeat:

```yaml
monitoring.enabled: true
monitoring.cluster_uuid: "${ES_CLUSTER_ID}"
monitoring.elasticsearch: null
```

### Health Metrics

Key metrics to monitor:

1. **System Metrics**:
   - CPU usage
   - Memory usage
   - File descriptors
   - Disk I/O

2. **Filebeat Metrics**:
   - Events rate (input, output)
   - Harvester activity
   - Failed harvests
   - Registry size
   - Output errors/retries
   - Pipeline processing times

3. **Log Metrics**:
   - Log volume by source
   - Log types distribution
   - Processing errors
   - Parsing failures

### Logging Configuration

Configure Filebeat's own logs:

```yaml
logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
  permissions: 0644
logging.metrics.enabled: true
logging.metrics.period: 30s
```

### Tracing and Debugging

Enable detailed logging for troubleshooting:

```yaml
logging.level: debug
logging.selectors: ["harvester", "publisher", "spooler"]
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat-debug
```

## Troubleshooting

### Common Issues

1. **No Logs Being Collected**:
   - Check file permissions
   - Verify file paths exist
   - Check if files are readable by Filebeat user
   - Ensure harvesters can keep up with log generation

2. **Duplicate Events**:
   - Registry file corruption
   - Multiple Filebeat instances monitoring same files
   - Registry mismatches after file rotation

3. **High Resource Usage**:
   - Too many files monitored
   - Complex processing
   - Output bottlenecks causing backpressure
   - Memory leaks in certain versions

4. **Connectivity Issues**:
   - Network connectivity to outputs
   - TLS/SSL certificate problems
   - Authentication failures
   - Firewall restrictions

### Diagnostic Techniques

1. **Test Configuration**:
   ```bash
   filebeat test config -c /etc/filebeat/filebeat.yml
   ```

2. **Test Output Connectivity**:
   ```bash
   filebeat test output -c /etc/filebeat/filebeat.yml
   ```

3. **Run in Debug Mode**:
   ```bash
   filebeat -e -c /etc/filebeat/filebeat.yml -d "publish,beat"
   ```

4. **Check Registry Files**:
   ```bash
   # For Filebeat 7.0+
   ls -la /var/lib/filebeat/registry/filebeat/
   ```

5. **Inspect Events**:
   ```bash
   # Temporarily change output to console
   filebeat -e -c /etc/filebeat/filebeat.yml -d "publish" -output.elasticsearch.enabled=false -output.console.enabled=true
   ```

### Solving Common Problems

1. **Registry Issues**:
   ```bash
   # Stop Filebeat
   sudo systemctl stop filebeat
   
   # Back up registry
   sudo cp -r /var/lib/filebeat/registry /var/lib/filebeat/registry.bak
   
   # Remove registry
   sudo rm -rf /var/lib/filebeat/registry
   
   # Start Filebeat (will create new registry)
   sudo systemctl start filebeat
   ```

2. **Permission Problems**:
   ```bash
   # Check Filebeat user access to log files
   sudo -u filebeat ls -la /var/log/application/
   
   # Fix permissions if needed
   sudo chmod 644 /var/log/application/*.log
   sudo setfacl -m u:filebeat:r /var/log/application/
   ```

3. **Output Connectivity**:
   ```bash
   # Test network connectivity
   curl -v http://elasticsearch:9200
   
   # Check TLS certificate
   openssl s_client -connect elasticsearch:9200
   ```

## Best Practices

### Deployment Strategies

1. **Installation**:
   - Use official packages when possible
   - Keep Filebeat updated to latest minor version
   - Automate deployment with configuration management
   - Consider containerization for consistent environments

2. **Configuration Management**:
   - Use version control for configurations
   - Implement CI/CD for config validation
   - Use templates with environment-specific variables
   - Document all custom configurations

3. **Scaling**:
   - Deploy close to log sources
   - One Filebeat per host when possible
   - Use separate instances for high-volume log sources
   - Consider load balancing for outputs

### Security Recommendations

1. **Encryption**:
   - Enable TLS for all communications
   - Use strong cipher suites
   - Verify certificates properly
   - Rotate certificates according to policy

2. **Authentication**:
   - Use dedicated user accounts
   - Implement least privilege principle
   - Use API keys where supported
   - Regularly rotate credentials

3. **Input Security**:
   - Validate log sources
   - Be cautious with arbitrary file inputs
   - Implement proper file permissions
   - Consider log source authentication when possible

### Operational Guidelines

1. **Monitoring**:
   - Set up alerts for Filebeat failures
   - Monitor event throughput
   - Track CPU and memory usage
   - Set up log loss detection

2. **Backup and Recovery**:
   - Regularly back up configurations
   - Consider registry backups for critical deployments
   - Document recovery procedures
   - Test recovery processes

3. **Upgrades**:
   - Test upgrades in non-production first
   - Plan for registry format changes
   - Check breaking changes in release notes
   - Consider rolling upgrades for high-availability

4. **Documentation**:
   - Document all custom configurations
   - Maintain deployment architecture diagrams
   - Keep runbooks for common issues
   - Document log sources and formats

### Configuration Tips

1. **General Best Practices**:
   - Keep configurations simple
   - Use modules when available
   - Implement proper logging
   - Validate configurations before deployment

2. **Performance Optimization**:
   - Filter logs at source when possible
   - Limit the use of expensive processors
   - Monitor and adjust harvester settings
   - Use appropriate scan frequencies

3. **Resilience**:
   - Configure proper retry behavior
   - Implement output load balancing
   - Use persistent queues for critical data
   - Consider secondary outputs for redundancy

## Conclusion

Filebeat is a powerful, lightweight log shipper that plays a crucial role in centralized logging infrastructures. By properly configuring and optimizing Filebeat, you can ensure reliable log collection with minimal resource overhead. The various input types, modules, processors, and output options make it adaptable to a wide range of use cases, from simple logging setups to complex, multi-cloud environments.

In the next chapter, we'll explore Metricbeat, which collects and ships metrics from your systems and services.