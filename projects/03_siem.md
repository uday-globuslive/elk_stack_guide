# Security Information and Event Management (SIEM) with ELK Stack

This chapter provides a comprehensive guide to implementing a Security Information and Event Management (SIEM) solution using the Elastic Stack. SIEM enables security teams to detect, investigate, and respond to security threats in real-time by collecting, analyzing, and correlating security events from across the organization.

## Table of Contents

- [SIEM Overview](#siem-overview)
- [Elastic Security Architecture](#elastic-security-architecture)
- [Data Collection for Security](#data-collection-for-security)
- [Detection Rules and Alerts](#detection-rules-and-alerts)
- [Incident Response Workflows](#incident-response-workflows)
- [Threat Intelligence Integration](#threat-intelligence-integration)
- [Security Analytics and Machine Learning](#security-analytics-and-machine-learning)
- [User and Entity Behavior Analytics (UEBA)](#user-and-entity-behavior-analytics-ueba)
- [Security Operations Center (SOC) Dashboard](#security-operations-center-soc-dashboard)
- [Compliance and Reporting](#compliance-and-reporting)
- [Advanced SIEM Deployments](#advanced-siem-deployments)
- [Best Practices](#best-practices)

## SIEM Overview

Elastic Security provides SIEM capabilities built on the Elastic Stack, allowing organizations to:

1. **Collect and normalize security data** from diverse sources
2. **Detect threats** using rules, machine learning, and anomaly detection
3. **Investigate incidents** with powerful search and visualization tools
4. **Respond to threats** using automated workflows and integrations
5. **Hunt for threats** proactively across collected data

Key benefits of implementing SIEM with Elastic include:

- Open architecture with flexible data ingestion
- Scalable data storage and search capabilities
- Built-in detection rules aligned with MITRE ATT&CK framework
- Integration with threat intelligence sources
- Machine learning for automated anomaly detection
- Unified approach to log management and security monitoring

## Elastic Security Architecture

![Elastic Security Architecture](https://www.elastic.co/guide/en/security/current/images/security-arch.png)

The Elastic Security architecture consists of:

1. **Data Collection Layer**: Beats, Agent, and integrations for security data
2. **Transport Layer**: Secure data transmission to Elasticsearch
3. **Processing Layer**: Elasticsearch for storage, indexing, and analysis
4. **Analytics Layer**: Detection engine, machine learning, and analytics
5. **Visualization Layer**: Kibana SIEM app, dashboards, and timelines
6. **Response Layer**: Integrations with security tools and workflows

## Data Collection for Security

### Configuring Beats for Security Data

#### Auditbeat for System Audit Data

Install and configure Auditbeat:

```bash
# On Debian/Ubuntu
sudo apt-get install auditbeat

# On RHEL/CentOS
sudo yum install auditbeat
```

Configure `/etc/auditbeat/auditbeat.yml`:

```yaml
auditbeat.modules:
- module: auditd
  audit_rules: |
    # Common rules for Linux systems
    -w /etc/passwd -p wa -k identity
    -w /etc/group -p wa -k identity
    -w /etc/shadow -p wa -k identity
    -w /etc/sudoers -p wa -k identity
    -w /var/log/secure -p r -k auth-log
    -w /var/log/audit -p r -k audit-log
    # Detect privilege escalation
    -a always,exit -F arch=b64 -S execve -C uid!=euid -F euid=0 -k setuid
    -a always,exit -F arch=b32 -S execve -C uid!=euid -F euid=0 -k setuid
    # Monitor command execution
    -a exit,always -F arch=b64 -S execve -k exec
    -a exit,always -F arch=b32 -S execve -k exec

- module: file_integrity
  paths:
  - /bin
  - /usr/bin
  - /sbin
  - /usr/sbin
  - /etc
  scan_rate_per_sec: 50000
  max_file_size: 100MiB
  hash_types: [sha1]
  recursive: true

- module: system
  datasets:
    - host
    - login
    - package
    - process
    - socket
    - user
  period: 10s
  state.period: 12h
  user.detect_password_changes: true

output.elasticsearch:
  hosts: ["https://elasticsearch:9200"]
  username: "elastic"
  password: "changeme"
  ssl.certificate_authorities: ["/etc/auditbeat/ca.crt"]
```

#### Packetbeat for Network Security Monitoring

Install and configure Packetbeat:

```bash
# On Debian/Ubuntu
sudo apt-get install packetbeat

# On RHEL/CentOS
sudo yum install packetbeat
```

Configure `/etc/packetbeat/packetbeat.yml`:

```yaml
packetbeat.interfaces.device: any

packetbeat.flows:
  timeout: 30s
  period: 10s

packetbeat.protocols:
- type: icmp
  enabled: true
- type: dns
  ports: [53]
  include_authorities: true
  include_additionals: true
- type: http
  ports: [80, 8080, 8000, 5000, 8002]
- type: tls
  ports: [443, 8443, 9443]
  send_certificates: true
- type: dhcpv4
  ports: [67, 68]

processors:
- add_host_metadata: ~
- add_cloud_metadata: ~

output.elasticsearch:
  hosts: ["https://elasticsearch:9200"]
  username: "elastic"
  password: "changeme"
  ssl.certificate_authorities: ["/etc/packetbeat/ca.crt"]
```

#### Filebeat with Security Modules

Install and configure Filebeat with security modules:

```bash
# On Debian/Ubuntu
sudo apt-get install filebeat

# On RHEL/CentOS
sudo yum install filebeat
```

Enable security modules:

```bash
sudo filebeat modules enable system
sudo filebeat modules enable suricata
sudo filebeat modules enable zeek
sudo filebeat modules enable iptables
sudo filebeat modules enable microsoft
sudo filebeat modules enable cisco
sudo filebeat modules enable aws
```

Configure `/etc/filebeat/filebeat.yml`:

```yaml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/auth.log
    - /var/log/secure
  fields:
    security_data_source: linux_auth
  fields_under_root: true

filebeat.modules:
- module: system
  syslog:
    enabled: true
  auth:
    enabled: true
- module: suricata
  eve:
    enabled: true
    var.paths: ["/var/log/suricata/eve.json"]
- module: zeek
  enabled: true
  var.paths:
    connection: ["/var/log/zeek/current/conn.log"]
    dns: ["/var/log/zeek/current/dns.log"]
    http: ["/var/log/zeek/current/http.log"]
    ssl: ["/var/log/zeek/current/ssl.log"]
    weird: ["/var/log/zeek/current/weird.log"]
    notice: ["/var/log/zeek/current/notice.log"]

output.elasticsearch:
  hosts: ["https://elasticsearch:9200"]
  username: "elastic"
  password: "changeme"
  ssl.certificate_authorities: ["/etc/filebeat/ca.crt"]
  index: "filebeat-%{[agent.version]}-%{+yyyy.MM.dd}"
```

### Elastic Agent for Unified Data Collection

For newer deployments, use Elastic Agent for unified data collection:

```bash
# Download and install
curl -L -O https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.8.0-linux-x86_64.tar.gz
tar xzvf elastic-agent-8.8.0-linux-x86_64.tar.gz
cd elastic-agent-8.8.0-linux-x86_64

# Install the agent as a managed service
sudo ./elastic-agent install --url=https://elasticsearch:8220 \
  --enrollment-token=<enrollment-token> \
  --ca-sha256=<ca-fingerprint>
```

Configure integrations in Kibana → Fleet → Agent policies → Add integration:

1. **System integration**: For host metrics, logs, and process information
2. **Endpoint Security**: For threat prevention and detection
3. **Network integrations**: For network traffic analysis
4. **Cloud integrations**: For AWS, Azure, GCP security logs
5. **Custom integrations**: For specific security tools

## Detection Rules and Alerts

### Implementing Detection Rules

Elastic Security comes with built-in detection rules. To create a custom rule:

1. Navigate to Kibana → Security → Rules → Create rule
2. Select a rule type:
   - Custom query
   - Machine learning
   - Threshold
   - Indicator match
   - EQL (Event Query Language)
   - Threat match
3. Define the rule parameters
4. Set up notifications and actions

Example custom rule to detect brute force attacks:

```json
{
  "rule_id": "custom-brute-force-detection",
  "name": "Multiple Failed SSH Authentication Attempts",
  "description": "Detects multiple failed SSH authentication attempts from the same source IP",
  "risk_score": 75,
  "severity": "high",
  "type": "threshold",
  "query": "event.module:system AND event.action:failed AND event.dataset:auth AND event.category:authentication",
  "threshold": {
    "field": "source.ip",
    "value": 5,
    "cardinality": [
      {
        "field": "user.name",
        "value": 3
      }
    ]
  },
  "from": "now-15m",
  "to": "now",
  "interval": "5m",
  "tags": ["authentication", "brute force", "ssh"],
  "references": ["https://attack.mitre.org/techniques/T1110/"],
  "threat": [
    {
      "framework": "MITRE ATT&CK",
      "tactic": {
        "id": "TA0006",
        "name": "Credential Access",
        "reference": "https://attack.mitre.org/tactics/TA0006/"
      },
      "technique": [
        {
          "id": "T1110",
          "name": "Brute Force",
          "reference": "https://attack.mitre.org/techniques/T1110/"
        }
      ]
    }
  ]
}
```

### Managing Rule Exceptions

Create exceptions to reduce false positives:

1. Go to a detection rule
2. Click "Manage detection rule exceptions"
3. Add exception items:
   - Field values to exclude from detection
   - OS-dependent exceptions
   - Time-based exceptions

### Setting Up Alert Notifications

Configure alert actions:

1. Go to Kibana → Stack Management → Rules and Connectors
2. Create connectors for:
   - Slack
   - Email
   - ServiceNow
   - JIRA
   - Webhook (custom integrations)
3. Use these connectors in rule actions

## Incident Response Workflows

### Timeline Investigation

The Timeline feature enables security analysts to investigate alerts:

1. Create a new timeline from an alert
2. Add relevant events to the timeline
3. Add notes and comments
4. Add investigation filters
5. Create cases for further tracking

### Case Management

Case management enables tracking security incidents:

1. Create a new case
2. Add relevant alerts and timelines
3. Set case status and severity
4. Assign case to team members
5. Add notes and comments
6. Integrate with external case management systems

Example case integration with ServiceNow:

1. Go to Kibana → Security → Cases → Configure → Connectors
2. Add ServiceNow connector with:
   - Instance URL
   - Username and password
   - Field mappings

## Threat Intelligence Integration

### Implementing Threat Intel Feeds

Load threat intelligence using Filebeat:

```yaml
# filebeat.yml
filebeat.inputs:
- type: threat
  dataset: threatintel
  enabled: true
  providers:
    - otx_pulse:
        api_key: "<API_KEY>"
        interval: 1h
    - anomali:
        api_key: "<API_KEY>"
        interval: 1h
    - abuse_ch:
        malware: true
        url: true
        interval: 1h
```

### Custom Threat Intelligence

Import custom threat intelligence:

1. Go to Kibana → Security → Threat Intelligence → Upload
2. Upload CSV or STIX files
3. Map fields to Elastic Common Schema (ECS)
4. Set expiration time for indicators

### Indicator Matching Rules

Create detection rules based on threat indicators:

1. Go to Kibana → Security → Rules → Create rule
2. Select "Indicator match" as rule type
3. Set indicator types (IP, domain, URL, file hash)
4. Configure matching criteria
5. Set up alert actions

## Security Analytics and Machine Learning

### Anomaly Detection

Set up machine learning jobs for anomaly detection:

1. Go to Kibana → Machine Learning → Anomaly Detection → Create job
2. Select security use cases:
   - Unusual process activity
   - Unusual network traffic
   - Unusual authentication patterns
   - Data exfiltration

Example anomaly detection job for unusual authentication patterns:

```json
{
  "job_id": "unusual_auth_patterns",
  "description": "Detect unusual authentication patterns",
  "groups": ["security", "authentication"],
  "analysis_config": {
    "bucket_span": "15m",
    "detectors": [
      {
        "detector_description": "rare by 'user.name', 'source.ip'",
        "function": "rare",
        "by_field_name": "user.name",
        "over_field_name": "source.ip"
      }
    ],
    "influencers": [
      "source.ip",
      "user.name",
      "host.name"
    ]
  },
  "data_description": {
    "time_field": "@timestamp"
  },
  "model_plot_config": {
    "enabled": true
  },
  "analysis_limits": {
    "model_memory_limit": "256mb"
  }
}
```

### Security Analytics Visualizations

Create visualizations for security analytics:

1. Auth success vs. failure by source IP
   - Visualization: Heat map
   - Split series: source.ip
   - Split chart: event.outcome
   - Metric: Count of events

2. Process execution frequency
   - Visualization: Bar chart
   - Split series: process.name
   - Metric: Count of events
   - Filters: event.category: process

3. Network traffic by destination
   - Visualization: Geographic map
   - Geospatial field: destination.geo.location
   - Metric: Sum of network.bytes

## User and Entity Behavior Analytics (UEBA)

### User Risk Scoring

Implement user risk scoring with machine learning:

1. Create baseline of normal user behavior
2. Monitor deviations from baseline
3. Calculate risk scores based on:
   - Authentication patterns
   - Process execution
   - File access
   - Network activity

Example user risk scoring model:

```json
{
  "job_id": "user_risk_scoring",
  "description": "Calculate user risk scores based on behavior",
  "groups": ["security", "ueba"],
  "analysis_config": {
    "bucket_span": "1h",
    "detectors": [
      {
        "detector_description": "high_mean('risk.score') by 'user.name'",
        "function": "high_mean",
        "field_name": "risk.score",
        "by_field_name": "user.name"
      }
    ],
    "influencers": [
      "user.name",
      "host.name",
      "event.category"
    ]
  },
  "data_description": {
    "time_field": "@timestamp"
  },
  "model_plot_config": {
    "enabled": true
  }
}
```

### Entity Analytics

Monitor entity behavior:

1. Create baselines for entities (servers, applications, IoT devices)
2. Track deviations from normal behavior
3. Correlate entity behavior with user activity
4. Identify compromised entities

## Security Operations Center (SOC) Dashboard

### SOC Overview Dashboard

Create a comprehensive SOC dashboard:

1. Go to Kibana → Dashboard → Create dashboard
2. Add the following visualizations:

**Security Alerts by Severity (Pie Chart)**:
- Metrics: Count of alerts
- Buckets: Split Slices = Terms of kibana.alert.severity
- Filter: event.kind: alert

**Recent High Severity Alerts (Data Table)**:
- Metrics: Count of alerts
- Buckets: Split Rows = Terms of kibana.alert.rule.name
- Buckets: Split Rows = Terms of source.ip
- Filter: event.kind: alert AND kibana.alert.severity: high

**Top Attack Vectors (Bar Chart)**:
- Metrics: Count of alerts
- Buckets: X-axis = Terms of threat.tactic.name
- Filter: event.kind: alert

**Geographic Attack Map (Map)**:
- Buckets: Geohash = source.geo.location
- Metrics: Count of events
- Filter: event.kind: alert OR event.category: intrusion_detection

**Authentication Failures (Line Chart)**:
- Metrics: Y-axis = Count
- Buckets: X-axis = Date Histogram of @timestamp
- Filter: event.action: authentication_failed

### Real-time Monitoring View

Create a real-time monitoring view:

1. Go to Kibana → Dashboard → Create dashboard
2. Configure auto-refresh for 10 seconds
3. Add visualizations focusing on real-time data
4. Use time range for "Last 5 minutes"

## Compliance and Reporting

### Compliance Dashboards

Create compliance-specific dashboards:

1. **PCI DSS Dashboard**:
   - Authentication monitoring
   - System change detection
   - Log monitoring
   - File integrity monitoring

2. **HIPAA Dashboard**:
   - Access control monitoring
   - Audit logging
   - Data access tracking
   - Encryption verification

3. **SOC 2 Dashboard**:
   - Access changes
   - Security configurations
   - System availability
   - Incident response metrics

### Automated Reporting

Set up automated reports:

1. Go to Kibana → Stack Management → Reporting
2. Create scheduled reports for:
   - Weekly security summary
   - Daily incident reports
   - Monthly compliance status
   - Custom executive reports

## Advanced SIEM Deployments

### Distributed SIEM Architecture

For large organizations, implement distributed SIEM:

1. **Collection layer**: Distributed Elastic Agents with Fleet management
2. **Processing layer**: Logstash for preprocessing and filtering
3. **Storage layer**: Hot-warm-cold Elasticsearch architecture
4. **Analytics layer**: Dedicated machine learning nodes
5. **Visualization layer**: Kibana with role-based access control

Example distributed deployment:

```yaml
# Collection Tier (Multiple locations)
# Elastic Agent with local buffering

# Processing Tier (Regional)
# Logstash configuration with preprocessing
input {
  elastic_agent {
    port => 5044
    ssl => true
    ssl_certificate_authorities => ["/etc/logstash/ca.crt"]
    ssl_certificate => "/etc/logstash/logstash.crt"
    ssl_key => "/etc/logstash/logstash.key"
  }
}

filter {
  # Security-specific preprocessing
  if [event][module] == "security" {
    # Normalize fields
    mutate {
      rename => { "[source][address]" => "[source][ip]" }
    }
    
    # Enrich with threat intelligence
    if [source][ip] {
      elasticsearch {
        hosts => ["https://elasticsearch:9200"]
        index => "threatintel-*"
        query => "source.ip:%{[source][ip]}"
        fields => { "threat.indicator.type" => "threat.indicator.type" }
        user => "elastic"
        password => "changeme"
      }
    }
  }
}

output {
  elasticsearch {
    hosts => ["https://elasticsearch:9200"]
    ssl => true
    ssl_certificate_authorities => ["/etc/logstash/ca.crt"]
    user => "elastic"
    password => "changeme"
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
  }
}
```

### Integration with Security Tools

Integrate with other security tools:

1. **SOAR Integration**:
   - Create Webhook connectors for bidirectional integration
   - Automate response actions via API

2. **SIEM-IDS Integration**:
   - Collect IDS alerts (Suricata, Snort) with Filebeat
   - Correlate IDS alerts with other security data
   - Trigger automated responses

3. **EDR Integration**:
   - Elastic Endpoint Security for EDR capabilities
   - Third-party EDR integration via API

## Best Practices

### SIEM Implementation Best Practices

1. **Start with critical data sources**:
   - Authentication logs
   - Network security monitoring
   - System audit logs
   - Critical application logs

2. **Implement progressive deployment**:
   - Begin with core security monitoring
   - Add threat intelligence integration
   - Implement advanced analytics
   - Enable automated response

3. **Optimize data collection**:
   - Use field filtering to reduce storage
   - Apply ECS mapping consistently
   - Implement index lifecycle management
   - Balance security visibility and storage costs

### SOC Operations Best Practices

1. **Establish clear workflows**:
   - Alert triage process
   - Incident investigation steps
   - Case management workflow
   - Escalation procedures

2. **Reduce alert fatigue**:
   - Tune detection rules regularly
   - Implement alert prioritization
   - Create rule exceptions for known false positives
   - Use correlation rules to combine related alerts

3. **Conduct regular reviews**:
   - Weekly review of detection coverage
   - Monthly review of false positives
   - Quarterly rule tuning
   - Regular threat hunting sessions

### Security Monitoring Best Practices

1. **Implement defense in depth**:
   - Monitor at multiple layers
   - Use diverse detection methods
   - Correlate events across systems
   - Don't rely on single detection techniques

2. **Continuously update defenses**:
   - Update threat intelligence regularly
   - Add detection rules for new threats
   - Tune machine learning models
   - Adapt to changing attack patterns

3. **Document everything**:
   - Create playbooks for common alerts
   - Document investigation steps
   - Maintain knowledge base of past incidents
   - Share lessons learned across the team

## Conclusion

Implementing a SIEM solution with the Elastic Stack provides organizations with powerful capabilities for security monitoring, threat detection, and incident response. By following the implementation guidelines and best practices outlined in this chapter, security teams can build a robust security monitoring system that scales with their organization's needs and adapts to evolving threats.

The open architecture of the Elastic Stack allows for flexible integration with existing security tools and customization to meet specific security requirements. Whether you're building a small security monitoring system or a comprehensive enterprise SOC, the Elastic Stack provides the foundation for effective security operations.