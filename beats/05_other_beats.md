# Heartbeat and Other Beats

This chapter covers Heartbeat for uptime monitoring and introduces other specialized Beats in the Elastic Stack ecosystem.

## Table of Contents
- [Heartbeat Overview](#heartbeat-overview)
- [Heartbeat Installation and Setup](#heartbeat-installation-and-setup)
- [Heartbeat Configuration](#heartbeat-configuration)
- [Heartbeat Monitors](#heartbeat-monitors)
- [Heartbeat Visualizations](#heartbeat-visualizations)
- [Auditbeat](#auditbeat)
- [Functionbeat](#functionbeat)
- [Winlogbeat](#winlogbeat)
- [Journalbeat](#journalbeat)
- [Community Beats](#community-beats)
- [Combining Multiple Beats](#combining-multiple-beats)
- [Best Practices](#best-practices)

## Heartbeat Overview

Heartbeat is a lightweight shipper for uptime monitoring. It periodically checks the status of services to determine whether they're available.

### Key Features

- **Lightweight Monitoring**: Small footprint with minimal impact on resources
- **Multiple Protocols**: Support for HTTP, TCP, and ICMP protocols
- **SSL/TLS Verification**: Check certificate validity and expiration
- **Authentication Support**: Monitor endpoints requiring authentication
- **Proxy Support**: Route checks through proxy servers
- **Flexible Scheduling**: Configure check frequency and timeout settings
- **Geographic Distribution**: Deploy across multiple locations for regional monitoring
- **Content Verification**: Validate response content against expected patterns

### Common Use Cases

1. **Service Uptime Monitoring**: Track availability of critical services
2. **Website Monitoring**: Monitor website availability and response times
3. **API Endpoint Checks**: Verify API endpoints are responding correctly
4. **SSL Certificate Monitoring**: Monitor certificate validity and expiration
5. **Internal Service Health**: Track internal microservices health
6. **SLA Compliance**: Collect data for service level agreement reporting
7. **Synthetic Monitoring**: Perform synthetic transactions to verify functionality
8. **Multi-region Availability**: Check service availability from different geographic locations

## Heartbeat Installation and Setup

### System Requirements

- **Memory**: 50MB minimum
- **CPU**: Minimal (less than 1% in most cases)
- **Disk**: 200MB free space for installation
- **Operating Systems**:
  - Linux (most distributions)
  - Windows Server 2012 R2 or later
  - macOS 10.13 or later
  - Docker containers

### Installation Methods

1. **Package Repositories (Linux)**:
   ```bash
   # Debian/Ubuntu
   curl -L -O https://artifacts.elastic.co/downloads/beats/heartbeat/heartbeat-8.1.0-amd64.deb
   sudo dpkg -i heartbeat-8.1.0-amd64.deb
   
   # RHEL/CentOS
   curl -L -O https://artifacts.elastic.co/downloads/beats/heartbeat/heartbeat-8.1.0-x86_64.rpm
   sudo rpm -vi heartbeat-8.1.0-x86_64.rpm
   ```

2. **Windows**:
   - Download installer from Elastic website
   - Install using PowerShell as administrator:
     ```powershell
     # Install as a Windows service
     PS> .\heartbeat.exe install
     ```

3. **Docker**:
   ```bash
   docker pull docker.elastic.co/beats/heartbeat:8.1.0
   docker run docker.elastic.co/beats/heartbeat:8.1.0
   ```

4. **Kubernetes**:
   - Use Helm chart or Elastic Cloud on Kubernetes operator
   - Deploy Heartbeat pods in multiple regions or clusters

### Initial Setup

1. **Configuration File**: Edit `heartbeat.yml` in the configuration directory
2. **Configure Monitors**: Specify which services to monitor
3. **Set Up Assets**: Load index templates and dashboards:
   ```bash
   heartbeat setup -e
   ```
4. **Start Heartbeat**:
   ```bash
   # Systemd (Linux)
   sudo systemctl start heartbeat-elastic
   
   # Windows
   Start-Service heartbeat
   ```

### Verification

Confirm Heartbeat is running correctly:

```bash
# Check service status (Linux)
sudo systemctl status heartbeat-elastic

# View logs
sudo journalctl -u heartbeat-elastic

# Test configuration
heartbeat test config -c /etc/heartbeat/heartbeat.yml

# Test output connectivity
heartbeat test output -c /etc/heartbeat/heartbeat.yml
```

## Heartbeat Configuration

The `heartbeat.yml` file is the main configuration file, typically located in `/etc/heartbeat/` on Linux or `C:\Program Files\Heartbeat\` on Windows.

### Configuration Structure

```yaml
heartbeat.monitors:             # Monitor definitions
heartbeat.scheduler:            # Scheduler settings
processors:                     # Event processors
output.elasticsearch:           # Output configuration
setup:                          # Setup configurations
logging:                        # Logging configurations
```

### Minimal Configuration Example

```yaml
heartbeat.monitors:
- type: http
  id: my-website
  name: My Website
  urls: ["https://mywebsite.com"]
  schedule: '@every 1m'

output.elasticsearch:
  hosts: ["localhost:9200"]
  username: "elastic"
  password: "changeme"
```

### Common Settings

1. **General Settings**:
   ```yaml
   name: "my-heartbeat"          # Custom name for the Beat
   tags: ["production", "web"]   # Tags for all events
   fields:                       # Custom fields for all events
     env: production
     location: us-east
   fields_under_root: true       # Places custom fields at root level
   ```

2. **Scheduler Settings**:
   ```yaml
   heartbeat.scheduler:
     limit: 10                   # Max concurrent tasks
     location: Europe/Berlin     # Timezone for cron schedules
   ```

3. **Logging Settings**:
   ```yaml
   logging.level: info
   logging.to_files: true
   logging.files:
     path: /var/log/heartbeat
     name: heartbeat
     keepfiles: 7
     permissions: 0644
   ```

## Heartbeat Monitors

Monitors are the core configuration elements in Heartbeat, defining what services to check and how to check them.

### HTTP Monitors

Monitor web services and APIs:

```yaml
heartbeat.monitors:
- type: http
  id: website-check
  name: Website Monitoring
  
  # URLs to check
  urls: ["https://example.com", "https://api.example.com/status"]
  
  # Check frequency
  schedule: '@every 1m'
  
  # Request configuration
  check.request:
    method: GET
    headers:
      X-API-Key: "my-api-key"
    body: |
      {"test": "data"}
  
  # Response validation
  check.response:
    status: [200, 201, 204]
    headers:
      Content-Type: "application/json"
    body:
      contains: '"status":"ok"'
  
  # Timeouts
  timeout: 30s
  
  # Authentication
  username: "user"
  password: "pass"
  
  # Proxy
  proxy_url: "http://proxy.example.com:3128"
  
  # TLS settings
  ssl:
    verification_mode: full
    certificate_authorities: ["/etc/pki/root/ca.pem"]
    supported_protocols: ["TLSv1.2", "TLSv1.3"]
```

### TCP Monitors

Check TCP service availability:

```yaml
heartbeat.monitors:
- type: tcp
  id: database-check
  name: Database Connection Check
  
  # Hosts to check
  hosts: ["db.example.com:5432", "db-replica.example.com:5432"]
  
  # Check frequency
  schedule: '@every 5m'
  
  # Send data (optional)
  check.send: "PING\n"
  
  # Expect response (optional)
  check.receive: "PONG"
  
  # Timeouts
  timeout: 5s
  
  # TLS settings
  ssl:
    enabled: true
    verification_mode: none
```

### ICMP Monitors

Basic ping checks:

```yaml
heartbeat.monitors:
- type: icmp
  id: gateway-ping
  name: Gateway Ping
  
  # Hosts to ping
  hosts: ["192.168.1.1", "10.0.0.1"]
  
  # Check frequency
  schedule: '@every 30s'
  
  # Timeouts and count
  timeout: 3s
  wait: 1s
  
  # ICMP settings (Linux/macOS only, requires root privileges)
  ipv4: true
  ipv6: false
```

### Schedule Options

Configure check frequency using various schedule formats:

```yaml
# Common scheduling patterns
schedule: '@every 30s'          # Every 30 seconds
schedule: '@every 5m'           # Every 5 minutes
schedule: '@every 1h'           # Every hour
schedule: '@hourly'             # At the start of every hour
schedule: '@daily'              # Every day at midnight
schedule: '@midnight'           # Every day at midnight

# Cron-like schedules
schedule: '*/5 * * * * * *'     # Every 5 seconds
schedule: '0 */5 * * * * *'     # Every 5 minutes
schedule: '0 0 * * * * *'       # Every hour
schedule: '0 0 12 * * * *'      # Every day at noon
```

### Multiple Monitor Configuration Example

```yaml
heartbeat.monitors:
# Website monitoring
- type: http
  id: website-prod
  name: Production Website
  urls: ["https://example.com"]
  schedule: '@every 1m'
  timeout: 10s
  check.response.status: [200]
  tags: ["production", "website"]

# API monitoring
- type: http
  id: api-health
  name: API Health Endpoint
  urls: ["https://api.example.com/health"]
  schedule: '@every 2m'
  timeout: 15s
  check.response:
    status: [200]
    body:
      contains: '"status":"ok"'
  tags: ["production", "api"]

# Database monitoring
- type: tcp
  id: database-check
  name: Database Connectivity
  hosts: ["db.example.com:5432"]
  schedule: '@every 5m'
  timeout: 5s
  tags: ["production", "database"]

# Network infrastructure monitoring
- type: icmp
  id: network-gateways
  name: Network Gateways
  hosts: ["192.168.1.1", "10.0.0.1"]
  schedule: '@every 30s'
  timeout: 3s
  tags: ["production", "network"]
```

### Dynamic Monitor Loading

Load monitors from separate configuration files:

```yaml
# In heartbeat.yml
heartbeat.config.monitors:
  # Path to monitors
  path: ${path.config}/monitors.d/*.yml
  
  # Reload period
  reload.enabled: true
  reload.period: 5s
```

Then create individual monitor files:

```yaml
# monitors.d/websites.yml
- type: http
  id: website-prod
  name: Production Website
  urls: ["https://example.com"]
  schedule: '@every 1m'

# monitors.d/apis.yml
- type: http
  id: api-endpoints
  name: API Endpoints
  urls: ["https://api.example.com/health", "https://api.example.com/status"]
  schedule: '@every 2m'
```

## Heartbeat Visualizations

Heartbeat provides pre-built visualizations and dashboards for monitoring service uptime and performance.

### Uptime App

The Uptime app in Kibana provides a dedicated interface for monitoring service availability:

1. **Features**:
   - Overview of service status
   - Uptime percentage calculations
   - Response time trends
   - SSL certificate monitoring
   - Geographical monitoring distribution
   - Alert integration

2. **Accessing the Uptime App**:
   - Navigate to Kibana
   - Select "Uptime" from the main menu
   - View the dashboard of monitored services

3. **Installation**:
   ```bash
   # Load Uptime dashboards
   heartbeat setup --dashboards
   ```

### Enriching Uptime Data

Add context to uptime monitoring data:

```yaml
heartbeat.monitors:
- type: http
  id: website-prod
  name: Production Website
  urls: ["https://example.com"]
  schedule: '@every 1m'
  
  # Add metadata for better context
  tags: ["production", "website", "customer-facing"]
  fields:
    service.name: "main-website"
    team: "web-platform"
    criticality: "high"
    sla.uptime: "99.9%"
    expected_response_time: "500ms"

processors:
  # Add geographic location of the monitor
  - add_fields:
      target: ''
      fields:
        monitor.geo.name: "us-east"
        monitor.geo.location:
          lat: 40.7128
          lon: -74.0060
```

### Alert Setup

Configure alerts for downtime and performance issues:

1. **In Kibana**:
   - Navigate to Uptime
   - Select "Alerts" 
   - Configure downtime alerts
   - Set thresholds for response time
   - Configure TLS certificate expiration alerts

2. **Alert Types**:
   - **Status**: Service is down
   - **Response Time**: Service response time exceeds threshold
   - **TLS Certificate**: Certificate is expiring soon
   - **Anomaly**: Unusual response time patterns

## Auditbeat

Auditbeat is a lightweight shipper that collects audit data from your systems and sends it to the Elastic Stack.

### Key Features

- **File Integrity Monitoring**: Track changes to critical files
- **System Audit Events**: Collect system call data (with auditd)
- **User and Process Activity**: Monitor user logins and process execution
- **Socket Information**: Track network connections
- **Package Information**: Monitor installed packages and versions
- **Host Information**: Collect system metadata
- **Low Overhead**: Minimal impact on system performance

### Basic Configuration

```yaml
auditbeat.modules:
# File integrity monitoring
- module: file_integrity
  paths:
    - /bin
    - /usr/bin
    - /etc
    - /usr/local/bin
    - /home/admin/.ssh
  exclude_files:
    - (?i)\.bak$
    - (?i)\.tmp$
  scan_rate_per_sec: 50
  max_file_size: 100MiB
  hash_types: [sha256]
  recursive: true

# Auditd-based system audit
- module: auditd
  audit_rules: |
    # Audit changes to system-wide configuration
    -w /etc/passwd -p wa -k identity
    -w /etc/group -p wa -k identity
    -w /etc/shadow -p wa -k identity
    -w /etc/sudoers -p wa -k sudoers
    
    # Audit user logins
    -w /var/log/faillog -p wa -k logins
    -w /var/log/lastlog -p wa -k logins
    
    # Audit process and command execution
    -a always,exit -F arch=b64 -S execve,execveat -k exec
    -a always,exit -F arch=b32 -S execve,execveat -k exec
    
    # Audit privileged command execution
    -a always,exit -F path=/usr/bin/sudo -F perm=x -k sudo
    -a always,exit -F path=/bin/su -F perm=x -k su

# System module for baseline information
- module: system
  datasets:
    - host    # General host information
    - login   # User login attempts
    - package # Installed packages
    - process # Process information
    - socket  # Socket information
    - user    # User information
  period: 10m
  state.period: 12h
  user.detect_password_changes: true
  process.include_top_n:
    enabled: true
    by_cpu: 5
    by_memory: 5

output.elasticsearch:
  hosts: ["localhost:9200"]
  username: "elastic"
  password: "changeme"
```

### Use Cases

1. **Compliance Monitoring**: Track changes to meet compliance requirements (PCI DSS, HIPAA, SOC2)
2. **Security Monitoring**: Detect unauthorized changes to critical files
3. **User Activity Tracking**: Monitor user logins and actions
4. **Privileged Access Monitoring**: Track administrative activities
5. **System Change Detection**: Identify configuration changes

## Functionbeat

Functionbeat is a serverless shipper for collecting and shipping logs from cloud services. It deploys as a function in your Function-as-a-Service (FaaS) platform.

### Key Features

- **Serverless Architecture**: Deploy as cloud functions
- **Native Cloud Integration**: Direct integration with cloud services
- **Event-Driven**: Only runs when triggered by events
- **Auto-Scaling**: Scales automatically with event volume
- **Low Operational Overhead**: No servers to maintain
- **Cost-Efficient**: Pay only for actual event processing

### AWS Configuration Example

```yaml
functionbeat.provider.aws:
  enabled: true
  access_key_id: "${AWS_ACCESS_KEY_ID}"
  secret_access_key: "${AWS_SECRET_ACCESS_KEY}"
  region: "us-east-1"

functionbeat.functions:
  # CloudWatch Logs
  - name: cloudwatch
    enabled: true
    type: cloudwatch_logs
    description: "Lambda function for CloudWatch Logs"
    triggers:
      - log_group_name: /aws/lambda/my-lambda-function
      - log_group_name: /aws/api-gateway/my-api
    
  # Kinesis
  - name: kinesis
    enabled: true
    type: kinesis
    description: "Lambda function for Kinesis"
    triggers:
      - event_source_arn: arn:aws:kinesis:us-east-1:123456789012:stream/my-stream
    
  # SQS
  - name: sqs
    enabled: true
    type: sqs
    description: "Lambda function for SQS"
    triggers:
      - event_source_arn: arn:aws:sqs:us-east-1:123456789012:my-queue
    
  # CloudWatch Logs
  - name: cloudwatch_vpc
    enabled: true
    type: cloudwatch_logs
    description: "Lambda function for CloudWatch Logs in VPC"
    triggers:
      - log_group_name: /aws/vpc/flow-logs
    virtual_private_cloud:
      security_group_ids: ["sg-123456789"]
      subnet_ids: ["subnet-123456789"]

processors:
  - add_fields:
      target: ''
      fields:
        environment: production
        region: us-east-1

output.elasticsearch:
  hosts: ["${ES_HOSTS}"]
  username: "${ES_USERNAME}"
  password: "${ES_PASSWORD}"
```

### Deployment

Deploy Functionbeat to the cloud provider:

```bash
# Deploy to AWS
functionbeat deploy cloudwatch
functionbeat deploy kinesis
functionbeat deploy sqs

# Check deployment status
functionbeat list

# Remove a function
functionbeat remove cloudwatch
```

### Use Cases

1. **CloudWatch Logs Collection**: Collect and process AWS CloudWatch logs
2. **Kinesis Data Stream Processing**: Process records from Kinesis data streams
3. **SQS Message Processing**: Process messages from SQS queues
4. **VPC Flow Logs Analysis**: Analyze network traffic within VPCs
5. **CloudTrail Audit Processing**: Process AWS CloudTrail events

## Winlogbeat

Winlogbeat is a lightweight shipper for Windows event logs, designed to efficiently stream Windows event logs to the Elastic Stack.

### Key Features

- **Windows Event Collection**: Gather events from Windows event logs
- **Efficient Processing**: Lightweight, efficient log collection
- **Multiple Channel Support**: Monitor application, security, system, and custom logs
- **Filtering Capabilities**: Include/exclude events based on criteria
- **Field Extraction**: Extract relevant fields from event data
- **Windows-Specific Metadata**: Include Windows-specific context

### Basic Configuration

```yaml
winlogbeat.event_logs:
  # System events
  - name: System
    ignore_older: 72h
    level: critical, error, warning
    processors:
      - script:
          lang: javascript
          id: service_filters
          source: >
            function process(event) {
              var keywords = ["service", "driver", "device"];
              var description = event.Get("message");
              if (description) {
                for (var i = 0; i < keywords.length; i++) {
                  if (description.toLowerCase().indexOf(keywords[i]) >= 0) {
                    return event;
                  }
                }
              }
              event.Cancel();
              return event;
            }

  # Security events
  - name: Security
    ignore_older: 72h
    event_id: 4624, 4625, 4634, 4648, 4672, 4720, 4722, 4724, 4728, 4738, 5024
    processors:
      - drop_event:
          when:
            equals:
              winlog.event_data.LogonType: "5"

  # Application events
  - name: Application
    ignore_older: 72h
    level: critical, error
    providers:
      - Application Error
      - Application Hang
      - Windows Error Reporting
      - MSSQLSERVER
      - MSSQL$SQLEXPRESS

  # Windows PowerShell events
  - name: "Microsoft-Windows-PowerShell/Operational"
    event_id: 4103, 4104, 4105, 4106

  # Windows Defender events
  - name: "Microsoft-Windows-Windows Defender/Operational"
    event_id: 1116, 1117, 1118, 1119

  # Windows Firewall events
  - name: "Microsoft-Windows-Windows Firewall With Advanced Security/Firewall"

  # Microsoft Sysmon events (if installed)
  - name: "Microsoft-Windows-Sysmon/Operational"

processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~
  - add_fields:
      target: ''
      fields:
        environment: production
        site: headquarters

output.elasticsearch:
  hosts: ["localhost:9200"]
  username: "elastic"
  password: "changeme"
  index: "winlogbeat-%{[agent.version]}-%{+yyyy.MM.dd}"
```

### Use Cases

1. **Security Monitoring**: Track logon events, privilege changes, and security policy changes
2. **System Troubleshooting**: Monitor system errors and warnings
3. **Application Monitoring**: Track application crashes and errors
4. **Compliance Requirements**: Collect audit logs for compliance purposes
5. **PowerShell Activity**: Monitor PowerShell command execution
6. **Service Monitoring**: Track service starts, stops, and failures

## Journalbeat

Journalbeat is a lightweight shipper for systemd journal logs, designed to efficiently send journal entries to the Elastic Stack.

### Key Features

- **Systemd Journal Integration**: Collect logs from systemd journal
- **Field Mapping**: Map journal fields to ECS fields
- **Efficient Persistence**: Track journal reading position
- **Filtering**: Include/exclude based on units or fields
- **Metadata Enrichment**: Add host and process information

### Basic Configuration

```yaml
journalbeat.inputs:
- paths: ["/var/log/journal"]
  include_matches:
    - _SYSTEMD_UNIT=ssh.service
    - _SYSTEMD_UNIT=docker.service
    - _SYSTEMD_UNIT=kubelet.service
    - _SYSTEMD_UNIT=nginx.service
    - _SYSTEMD_UNIT=apache2.service
    - _TRANSPORT=kernel
  seek: cursor
  cursor_seek_fallback: tail

  # Filter based on systemd unit or syslog identifier
  units: ["ssh.service", "docker.service", "nginx.service"]
  
  # Syslog identifiers to collect
  syslog_identifiers: ["sshd", "systemd", "kernel"]
  
  # Collect messages only with selected priorities
  # 0-Emergency, 1-Alert, 2-Critical, 3-Error, 4-Warning
  # 5-Notice, 6-Informational, 7-Debug
  max_priority: 5
  
  # Collapse similar entries in the same timeframe
  coalesce: true
  
  # Configure parameters for coalescing
  coalesce_timeout: 1s
  coalesce_interval: 5s
  coalesce_max_events: 20
  
  # Set additional fields from journal entries
  additional_fields:
    - "_HOSTNAME"
    - "_UID"
    - "_GID"
    - "_COMM"
    - "_EXE"
    - "_CMDLINE"
    - "_SYSTEMD_SLICE"

processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~
  - add_field:
      target: ''
      fields:
        environment: production
        site: datacenter-east

output.elasticsearch:
  hosts: ["localhost:9200"]
  username: "elastic"
  password: "changeme"
  index: "journalbeat-%{[agent.version]}-%{+yyyy.MM.dd}"
```

### Use Cases

1. **System Log Collection**: Efficiently collect systemd journal logs
2. **Container Log Monitoring**: Gather logs from containerized applications
3. **Service Monitoring**: Track service starts, stops, and failures
4. **Kernel Log Analysis**: Collect and analyze kernel logs
5. **Centralized Logging**: Forward journal logs to a central Elasticsearch cluster

## Community Beats

The Elastic Beats platform is extensible, allowing developers to create custom Beats for specific use cases. Here's a selection of notable community Beats:

### Selected Community Beats

1. **Logstashbeat**: Monitors Logstash metrics and pipeline statistics
   ```yaml
   logstashbeat.hosts: ["http://localhost:9600"]
   logstashbeat.period: 10s
   ```

2. **Cloudflarebeat**: Collects logs from Cloudflare's Enterprise Log Share (ELS)
   ```yaml
   cloudflarebeat.api_key: "${CLOUDFLARE_API_KEY}"
   cloudflarebeat.zone_id: "${CLOUDFLARE_ZONE_ID}"
   cloudflarebeat.period: 5m
   ```

3. **Githubbeat**: Collects metrics and information from GitHub repositories
   ```yaml
   githubbeat.period: 10m
   githubbeat.repos:
     - owner: elastic
       name: beats
     - owner: elastic
       name: elasticsearch
   githubbeat.token: "${GITHUB_TOKEN}"
   ```

4. **Apachebeat**: Collects and parses the Apache httpd server status page
   ```yaml
   apachebeat.urls: ["http://localhost/server-status?auto"]
   apachebeat.period: 10s
   ```

5. **Kubernetesbeat**: Collects Kubernetes events and metrics beyond what Metricbeat provides
   ```yaml
   kubernetesbeat.period: 10s
   kubernetesbeat.kubernetes.node: ${NODE_NAME}
   kubernetesbeat.kubernetes.in_cluster: true
   ```

6. **Awsbeat**: Collects AWS billing details and cost allocation data
   ```yaml
   awsbeat.period: 24h
   awsbeat.aws.access_key_id: "${AWS_ACCESS_KEY_ID}"
   awsbeat.aws.secret_access_key: "${AWS_SECRET_ACCESS_KEY}"
   awsbeat.aws.region: "us-east-1"
   ```

### Developing Custom Beats

Creating a custom Beat involves several steps:

1. **Install Requirements**:
   - Go language environment
   - Git
   - Mage build tool

2. **Generate Beat Template**:
   ```bash
   git clone https://github.com/elastic/beats.git
   cd beats
   make setup
   make update
   cd dev-tools/generator
   go run main.go -path=~/mybeat -project=yourorg/mybeat -module=mymodule
   ```

3. **Implement Beat Logic**:
   - Define data collection logic
   - Implement event creation
   - Configure persistence if needed

4. **Build and Package**:
   ```bash
   cd ~/mybeat
   mage build
   mage package
   ```

## Combining Multiple Beats

Deploying multiple Beats provides comprehensive monitoring across your infrastructure.

### Integration Strategy

Combine Beats for complete visibility:

```
Filebeat → Logs and Document Data
Metricbeat → System and Service Metrics
Packetbeat → Network Traffic
Heartbeat → Uptime and Availability
Auditbeat → Security and Compliance Data
```

### Shared Configuration Elements

Common configuration for all Beats:

```yaml
# Common Output Configuration
output.elasticsearch:
  hosts: ["https://elasticsearch.example.com:9200"]
  username: "${ES_USERNAME}"
  password: "${ES_PASSWORD}"
  ssl.certificate_authorities: ["/etc/pki/root/ca.pem"]
  pipelines:
    - pipeline: "geoip-info"
      when.contains:
        source: "filebeat"
    - pipeline: "metrics-transform"
      when.contains:
        source: "metricbeat"

# Common Processors
processors:
  - add_host_metadata:
      netinfo.enabled: true
  - add_cloud_metadata: ~
  - add_fields:
      target: ''
      fields:
        environment: production
        datacenter: us-east
        team: operations
```

### Central Management with Elastic Agent

Manage all Beats centrally with Elastic Agent and Fleet:

1. **Install Elastic Agent**:
   ```bash
   # Download and install
   curl -L -O https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.1.0-linux-x86_64.tar.gz
   tar xzvf elastic-agent-8.1.0-linux-x86_64.tar.gz
   cd elastic-agent-8.1.0-linux-x86_64
   sudo ./elastic-agent install \
     --url=https://fleet.example.com:8220 \
     --enrollment-token=<enrollment-token>
   ```

2. **Configure in Kibana Fleet**:
   - Navigate to Fleet → Agents
   - Add integrations:
     - System integration (logs and metrics)
     - Nginx integration
     - MySQL integration
     - Uptime monitoring
     - Network monitoring
   - Configure settings
   - Deploy to agents

### Resource Considerations

When running multiple Beats on the same host:

1. **CPU and Memory Allocation**:
   - Metricbeat: 100-200MB memory
   - Filebeat: 100-300MB memory (depends on volume)
   - Packetbeat: 100-300MB memory (depends on traffic)
   - Heartbeat: 50-100MB memory
   - Auditbeat: 100-200MB memory

2. **Staggered Collection**:
   - Offset collection times to distribute load
   - Prioritize critical monitoring
   - Adjust collection frequencies based on importance

3. **Output Batching**:
   - Use appropriate batch sizes
   - Implement output compression
   - Consider queue settings

## Best Practices

### Deployment Strategies

1. **Basic Monitoring Stack**:
   - Filebeat for logs
   - Metricbeat for system metrics
   - Heartbeat for key service uptime
   
2. **Enhanced Monitoring Stack**:
   - Add Packetbeat for network visibility
   - Include Auditbeat for security monitoring
   - Deploy specialized Beats for specific services
   
3. **Comprehensive Monitoring**:
   - Deploy all relevant Beats
   - Implement central management
   - Enable cross-beat correlation
   - Set up coordinated alerting

### Performance Considerations

Optimize Beat performance:

1. **Resource Allocation**:
   - Adjust configuration based on resource availability
   - Monitor Beat resource usage
   - Scale horizontally when needed
   
2. **Collection Frequency**:
   - Adjust based on importance and volatility
   - Critical: 30s-1m
   - Important: 1m-5m
   - Standard: 5m-10m
   - Low priority: 15m+
   
3. **Data Volume Management**:
   - Implement appropriate filtering
   - Use selective collection
   - Apply efficient processors
   - Consider sampling for high-volume data

### Security Recommendations

Secure your Beat deployments:

1. **Authentication**:
   - Use API keys or credentials
   - Implement principle of least privilege
   - Rotate credentials regularly
   - Use environment variables for sensitive values
   
2. **Encryption**:
   - Enable TLS for all communications
   - Verify certificates properly
   - Use secure key storage
   - Implement proper certificate management
   
3. **Network Security**:
   - Run Beats behind firewalls
   - Limit connectivity to required services
   - Implement proper network segmentation
   - Use secure communication channels

### Operational Guidelines

1. **Monitoring the Monitors**:
   - Set up monitoring for Beats themselves
   - Create alerts for Beat failures
   - Track Beat performance metrics
   - Implement health checks
   
2. **Update Strategy**:
   - Keep Beats updated to latest minor version
   - Test upgrades in non-production
   - Implement rolling updates
   - Monitor for post-upgrade issues
   
3. **Documentation**:
   - Document all custom configurations
   - Maintain deployment architecture diagrams
   - Keep runbooks for common issues
   - Document monitor sources and meanings

## Conclusion

Elastic Beats provide a comprehensive suite of lightweight data shippers, each specialized for specific types of data. By deploying the appropriate combination of Beats, you can achieve complete visibility into your infrastructure, applications, and services.

Heartbeat offers essential uptime monitoring to track service availability, while specialized Beats like Auditbeat, Functionbeat, Winlogbeat, and Journalbeat extend monitoring capabilities to specific platforms and use cases. Community Beats further expand the ecosystem to address niche requirements.

The ability to combine multiple Beats, especially when managed through Fleet and Elastic Agent, creates a powerful, cohesive monitoring system that can scale from small environments to large distributed infrastructures.

In the next chapter, we'll explore how to develop custom Beats for specialized use cases not covered by existing Beats.