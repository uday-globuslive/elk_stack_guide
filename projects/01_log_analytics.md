# Log Analytics Pipeline Project

This chapter walks through building a complete log analytics pipeline using the ELK Stack. We'll create a production-ready solution for collecting, processing, storing, analyzing, and visualizing logs from multiple applications and systems.

## Project Overview

### Objectives

- Centralize logs from multiple sources
- Structure and normalize log data
- Enable real-time log analysis and visualization
- Set up alerts for critical events
- Implement log retention policies
- Secure the entire pipeline

### Architecture

Our log analytics pipeline will consist of:

1. **Collection Layer**: Filebeat, Metricbeat, and other Beats
2. **Processing Layer**: Logstash for enrichment and transformation
3. **Storage Layer**: Elasticsearch for indexed storage
4. **Analysis Layer**: Kibana for visualization and analysis
5. **Alerting Layer**: Kibana Alerting and Elasticsearch Watcher

![Log Analytics Architecture](https://i.imgur.com/9AYOecT.png)

## Part 1: Setting Up the Collection Layer

### Installing and Configuring Filebeat

Filebeat will be our primary log collector, deployed on all servers to collect application and system logs.

#### Filebeat Installation

```bash
# On Debian/Ubuntu
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.8.0-amd64.deb
sudo dpkg -i filebeat-8.8.0-amd64.deb

# On RHEL/CentOS
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.8.0-x86_64.rpm
sudo rpm -vi filebeat-8.8.0-x86_64.rpm
```

#### Basic Filebeat Configuration

Create a configuration file at `/etc/filebeat/filebeat.yml`:

```yaml
# ============================== Filebeat inputs ===============================
filebeat.inputs:
- type: filestream
  id: system-logs
  enabled: true
  paths:
    - /var/log/syslog
    - /var/log/auth.log
  fields:
    log_type: system
  fields_under_root: true

- type: filestream
  id: apache-logs
  enabled: true
  paths:
    - /var/log/apache2/access.log
  fields:
    log_type: apache
    service: web
  fields_under_root: true

- type: filestream
  id: application-logs
  enabled: true
  paths:
    - /var/log/app/*.log
  fields:
    log_type: application
    service: backend
  fields_under_root: true

# ============================== Processors ===================================
processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
  - add_kubernetes_metadata: ~

# ------------------------------ Output: Logstash -----------------------------
output.logstash:
  hosts: ["logstash.example.com:5044"]
  ssl.enabled: true
  ssl.certificate_authorities: ["/etc/filebeat/ca.crt"]
```

### Deploying Metricbeat for System Metrics

Metricbeat will collect system and service metrics.

#### Metricbeat Installation

```bash
# On Debian/Ubuntu
curl -L -O https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-8.8.0-amd64.deb
sudo dpkg -i metricbeat-8.8.0-amd64.deb

# On RHEL/CentOS
curl -L -O https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-8.8.0-x86_64.rpm
sudo rpm -vi metricbeat-8.8.0-x86_64.rpm
```

#### Basic Metricbeat Configuration

Create a configuration file at `/etc/metricbeat/metricbeat.yml`:

```yaml
# ============================ Modules configuration ============================
metricbeat.modules:
- module: system
  metricsets:
    - cpu
    - load
    - memory
    - network
    - process
    - process_summary
    - uptime
    - socket_summary
    - filesystem
    - fsstat
  enabled: true
  period: 10s
  processes: ['.*']
  process.include_top_n:
    by_cpu: 5
    by_memory: 5

- module: apache
  metricsets: ["status"]
  enabled: true
  period: 30s
  hosts: ["http://localhost:80/server-status?auto"]

# ------------------------------ Output: Logstash ------------------------------
output.logstash:
  hosts: ["logstash.example.com:5044"]
  ssl.enabled: true
  ssl.certificate_authorities: ["/etc/filebeat/ca.crt"]
```

### Specialized Log Collection

#### Java Application Logs with Log4j/Logback

For Java applications using Log4j2 or Logback, configure them to output in JSON format:

**Log4j2 Configuration (log4j2.xml)**:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration>
  <Appenders>
    <Console name="Console" target="SYSTEM_OUT">
      <JsonLayout compact="true" eventEol="true" properties="true" stacktraceAsString="true" />
    </Console>
    <File name="File" fileName="/var/log/app/application.log">
      <JsonLayout compact="true" eventEol="true" properties="true" stacktraceAsString="true" />
    </File>
  </Appenders>
  <Loggers>
    <Root level="info">
      <AppenderRef ref="Console" />
      <AppenderRef ref="File" />
    </Root>
  </Loggers>
</Configuration>
```

#### Container Logs with Filebeat Docker Module

For containerized applications, enable the Docker module in Filebeat:

```bash
sudo filebeat modules enable docker
```

Update the module configuration:

```yaml
- module: docker
  container:
    enabled: true
    paths: 
      - /var/lib/docker/containers/*/*.log
  hints.enabled: true
```

#### Kubernetes Logs with Filebeat

For Kubernetes environments, deploy Filebeat as a DaemonSet:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: logging
  labels:
    app: filebeat
spec:
  selector:
    matchLabels:
      app: filebeat
  template:
    metadata:
      labels:
        app: filebeat
    spec:
      serviceAccountName: filebeat
      terminationGracePeriodSeconds: 30
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:8.8.0
        args: [
          "-c", "/etc/filebeat.yml",
          "-e",
        ]
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        securityContext:
          runAsUser: 0
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

With a ConfigMap for configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: logging
data:
  filebeat.yml: |-
    filebeat.autodiscover:
      providers:
        - type: kubernetes
          node: ${NODE_NAME}
          hints.enabled: true
          hints.default_config:
            type: container
            paths:
              - /var/log/containers/*${data.kubernetes.container.id}.log
    
    processors:
      - add_cloud_metadata: ~
      - add_host_metadata: ~
      - add_kubernetes_metadata:
          host: ${NODE_NAME}
          matchers:
          - logs_path:
              logs_path: "/var/log/containers/"
    
    output.elasticsearch:
      hosts: ['${ELASTICSEARCH_HOST:elasticsearch}:${ELASTICSEARCH_PORT:9200}']
      username: ${ELASTICSEARCH_USERNAME}
      password: ${ELASTICSEARCH_PASSWORD}
```

## Part 2: Building the Processing Layer

Logstash will process and enrich logs before sending them to Elasticsearch.

### Logstash Installation

```bash
# On Debian/Ubuntu
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
sudo apt-get install apt-transport-https
echo "deb https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-8.x.list
sudo apt-get update && sudo apt-get install logstash

# On RHEL/CentOS
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
sudo tee /etc/yum.repos.d/elasticsearch.repo << EOF
[elasticsearch]
name=Elasticsearch repository for 8.x packages
baseurl=https://artifacts.elastic.co/packages/8.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=0
autorefresh=1
type=rpm-md
EOF
sudo yum install --enablerepo=elasticsearch logstash
```

### Configuring Logstash Pipelines

Create a pipeline configuration directory:

```bash
sudo mkdir -p /etc/logstash/conf.d
```

#### Input Configuration

Create `/etc/logstash/conf.d/01-beats-input.conf`:

```
input {
  beats {
    port => 5044
    ssl => true
    ssl_certificate => "/etc/logstash/certs/logstash.crt"
    ssl_key => "/etc/logstash/certs/logstash.key"
    ssl_certificate_authorities => ["/etc/logstash/certs/ca.crt"]
    ssl_verify_mode => "force_peer"
  }
}
```

#### Processing Configuration

Create `/etc/logstash/conf.d/30-processing.conf`:

```
filter {
  # Add timestamp for when the event was processed
  ruby {
    code => "event.set('processing_timestamp', Time.now.utc)"
  }
  
  # Process system logs
  if [log_type] == "system" {
    if [path] == "/var/log/syslog" or [path] == "/var/log/messages" {
      grok {
        match => { "message" => "%{SYSLOGTIMESTAMP:system.syslog.timestamp} %{SYSLOGHOST:system.syslog.hostname} %{DATA:system.syslog.program}(?:\[%{POSINT:system.syslog.pid}\])?: %{GREEDYDATA:system.syslog.message}" }
      }
      date {
        match => [ "system.syslog.timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
        target => "@timestamp"
      }
    }
    if [path] == "/var/log/auth.log" {
      grok {
        match => { "message" => "%{SYSLOGTIMESTAMP:system.auth.timestamp} %{SYSLOGHOST:system.auth.hostname} %{DATA:system.auth.program}(?:\[%{POSINT:system.auth.pid}\])?: %{GREEDYDATA:system.auth.message}" }
      }
      date {
        match => [ "system.auth.timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
        target => "@timestamp"
      }
      
      # Extract failed login attempts
      if [system.auth.message] =~ "Failed password" {
        grok {
          match => { "system.auth.message" => "Failed password for %{DATA:system.auth.user} from %{IP:system.auth.ip} port %{NUMBER:system.auth.port}" }
          tag_on_failure => ["_grokparsefailure_auth_failed_password"]
        }
      }
    }
  }
  
  # Process Apache logs
  if [log_type] == "apache" {
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
    }
    date {
      match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
      target => "@timestamp"
    }
    
    # GeoIP enrichment for client IP
    geoip {
      source => "clientip"
      target => "geoip"
    }
    
    # User agent parsing
    useragent {
      source => "agent"
      target => "user_agent"
    }
  }
  
  # Process application logs
  if [log_type] == "application" {
    if [message] =~ /^\{.*\}$/ {
      # Handle JSON formatted logs
      json {
        source => "message"
        target => "application"
      }
      if [application][timestamp] {
        date {
          match => [ "[application][timestamp]", "ISO8601" ]
          target => "@timestamp"
        }
      }
    } else {
      # Handle plain text application logs
      grok {
        match => { "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:log.level} %{GREEDYDATA:application.message}" }
      }
      date {
        match => [ "timestamp", "ISO8601" ]
        target => "@timestamp"
      }
    }
    
    # Extract and normalize log levels
    if [application][level] {
      mutate {
        rename => { "[application][level]" => "log.level" }
      }
    }
    
    # Normalize log levels
    if [log.level] {
      mutate {
        gsub => [ "log.level", "(?i)warning|warn", "WARN" ]
        gsub => [ "log.level", "(?i)error|err|fatal", "ERROR" ]
        gsub => [ "log.level", "(?i)info|information", "INFO" ]
        gsub => [ "log.level", "(?i)debug|dbg", "DEBUG" ]
      }
    }
    
    # Extract exceptions/stack traces
    if [application.message] =~ /Exception|Error:|stack trace|Caused by:/ {
      grok {
        match => { "application.message" => "(?m)%{JAVACLASS:error.type}:? %{GREEDYDATA:error.message}(?:\n%{SPACE}+at %{JAVACLASS:error.stack.method}\(%{GREEDYDATA:error.stack.location}\))+" }
        tag_on_failure => ["_grokparsefailure_stacktrace"]
      }
    }
  }
  
  # Add standard metadata
  mutate {
    add_field => {
      "environment" => "${ENV:production}"
      "datacenter" => "${DC:dc1}"
    }
  }
}
```

#### Output Configuration

Create `/etc/logstash/conf.d/90-output.conf`:

```
output {
  # Split output based on log type
  if [log_type] == "system" {
    elasticsearch {
      hosts => ["https://es1.example.com:9200", "https://es2.example.com:9200", "https://es3.example.com:9200"]
      user => "${ES_USER}"
      password => "${ES_PASSWORD}"
      index => "logs-system-%{+YYYY.MM.dd}"
      ssl => true
      ssl_certificate_verification => true
      cacert => "/etc/logstash/certs/ca.crt"
      ilm_enabled => true
      ilm_rollover_alias => "logs-system"
      ilm_pattern => "{now/d}-000001"
      ilm_policy => "logs-policy"
    }
  } else if [log_type] == "apache" {
    elasticsearch {
      hosts => ["https://es1.example.com:9200", "https://es2.example.com:9200", "https://es3.example.com:9200"]
      user => "${ES_USER}"
      password => "${ES_PASSWORD}"
      index => "logs-apache-%{+YYYY.MM.dd}"
      ssl => true
      ssl_certificate_verification => true
      cacert => "/etc/logstash/certs/ca.crt"
      ilm_enabled => true
      ilm_rollover_alias => "logs-apache"
      ilm_pattern => "{now/d}-000001"
      ilm_policy => "logs-policy"
    }
  } else if [log_type] == "application" {
    elasticsearch {
      hosts => ["https://es1.example.com:9200", "https://es2.example.com:9200", "https://es3.example.com:9200"]
      user => "${ES_USER}"
      password => "${ES_PASSWORD}"
      index => "logs-app-%{+YYYY.MM.dd}"
      ssl => true
      ssl_certificate_verification => true
      cacert => "/etc/logstash/certs/ca.crt"
      ilm_enabled => true
      ilm_rollover_alias => "logs-app"
      ilm_pattern => "{now/d}-000001"
      ilm_policy => "logs-policy"
    }
  } else {
    elasticsearch {
      hosts => ["https://es1.example.com:9200", "https://es2.example.com:9200", "https://es3.example.com:9200"]
      user => "${ES_USER}"
      password => "${ES_PASSWORD}"
      index => "logs-other-%{+YYYY.MM.dd}"
      ssl => true
      ssl_certificate_verification => true
      cacert => "/etc/logstash/certs/ca.crt"
      ilm_enabled => true
      ilm_rollover_alias => "logs-other"
      ilm_pattern => "{now/d}-000001"
      ilm_policy => "logs-policy"
    }
  }
}
```

### Logstash Pipeline Configuration

Create `/etc/logstash/pipelines.yml`:

```yaml
- pipeline.id: main
  path.config: "/etc/logstash/conf.d/*.conf"
  pipeline.workers: 4
  pipeline.batch.size: 1000
  queue.type: persisted
  queue.max_bytes: 1gb
```

### Starting Logstash

```bash
sudo systemctl start logstash
sudo systemctl enable logstash
```

## Part 3: Configuring the Storage Layer

### Elasticsearch Index Templates and ILM Policies

Connect to Elasticsearch and create an Index Lifecycle Management policy:

```
PUT _ilm/policy/logs-policy
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
          "shrink": {
            "number_of_shards": 1
          },
          "forcemerge": {
            "max_num_segments": 1
          },
          "set_priority": {
            "priority": 50
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "freeze": {},
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

Create a component template for log mappings:

```
PUT _component_template/logs-mappings
{
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "processing_timestamp": { "type": "date" },
        "message": { "type": "text" },
        "log_type": { "type": "keyword" },
        "service": { "type": "keyword" },
        "host": {
          "properties": {
            "name": { "type": "keyword" },
            "ip": { "type": "ip" }
          }
        },
        "log": {
          "properties": {
            "level": { "type": "keyword" },
            "logger": { "type": "keyword" }
          }
        },
        "error": {
          "properties": {
            "type": { "type": "keyword" },
            "message": { "type": "text" },
            "stack": { "type": "text" }
          }
        },
        "user": {
          "properties": {
            "name": { "type": "keyword" },
            "id": { "type": "keyword" }
          }
        },
        "geoip": {
          "properties": {
            "city_name": { "type": "keyword" },
            "continent_name": { "type": "keyword" },
            "country_iso_code": { "type": "keyword" },
            "location": { "type": "geo_point" },
            "region_name": { "type": "keyword" }
          }
        }
      }
    }
  }
}
```

Create a component template for index settings:

```
PUT _component_template/logs-settings
{
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1,
      "index.lifecycle.name": "logs-policy",
      "index.lifecycle.rollover_alias": "logs"
    }
  }
}
```

Create index templates for each log type:

```
PUT _index_template/logs-system
{
  "index_patterns": ["logs-system-*"],
  "template": {
    "settings": {
      "index.lifecycle.rollover_alias": "logs-system"
    }
  },
  "composed_of": ["logs-mappings", "logs-settings"],
  "priority": 100
}

PUT _index_template/logs-apache
{
  "index_patterns": ["logs-apache-*"],
  "template": {
    "settings": {
      "index.lifecycle.rollover_alias": "logs-apache"
    }
  },
  "composed_of": ["logs-mappings", "logs-settings"],
  "priority": 100
}

PUT _index_template/logs-app
{
  "index_patterns": ["logs-app-*"],
  "template": {
    "settings": {
      "index.lifecycle.rollover_alias": "logs-app"
    }
  },
  "composed_of": ["logs-mappings", "logs-settings"],
  "priority": 100
}

PUT _index_template/logs-other
{
  "index_patterns": ["logs-other-*"],
  "template": {
    "settings": {
      "index.lifecycle.rollover_alias": "logs-other"
    }
  },
  "composed_of": ["logs-mappings", "logs-settings"],
  "priority": 100
}
```

Create initial indices with aliases:

```
PUT logs-system-000001
{
  "aliases": {
    "logs-system": {
      "is_write_index": true
    }
  }
}

PUT logs-apache-000001
{
  "aliases": {
    "logs-apache": {
      "is_write_index": true
    }
  }
}

PUT logs-app-000001
{
  "aliases": {
    "logs-app": {
      "is_write_index": true
    }
  }
}

PUT logs-other-000001
{
  "aliases": {
    "logs-other": {
      "is_write_index": true
    }
  }
}
```

## Part 4: Building the Analysis Layer

### Creating Kibana Index Patterns

1. Navigate to Kibana → Stack Management → Index Patterns → Create index pattern
2. Enter "logs-*" as the index pattern
3. Select "@timestamp" as the time field
4. Click "Create index pattern"

Repeat this process for specific log types if needed:
- logs-system-*
- logs-apache-*
- logs-app-*

### Building Dashboards

#### System Logs Dashboard

1. Go to Kibana → Dashboard → Create dashboard
2. Add the following visualizations:

**System Log Levels Over Time (Line Chart)**:
- Metrics: Y-axis = Count
- Buckets: X-axis = Date Histogram of @timestamp
- Buckets: Split Series = Terms of log.level
- Filter: log_type: system

**Top Hosts by Log Volume (Pie Chart)**:
- Metrics: Slice Size = Count
- Buckets: Split Slices = Terms of host.name
- Filter: log_type: system

**Failed SSH Authentication (Data Table)**:
- Metrics: Count
- Buckets: Split Rows = Terms of system.auth.ip
- Buckets: Split Rows = Terms of system.auth.user
- Filter: log_type: system AND system.auth.message: "Failed password"

#### Web Server Dashboard

1. Go to Kibana → Dashboard → Create dashboard
2. Add the following visualizations:

**HTTP Status Codes Over Time (Line Chart)**:
- Metrics: Y-axis = Count
- Buckets: X-axis = Date Histogram of @timestamp
- Buckets: Split Series = Terms of response
- Filter: log_type: apache

**Top Client IPs (Bar Chart)**:
- Metrics: Y-axis = Count
- Buckets: X-axis = Terms of clientip
- Filter: log_type: apache

**Geographic Distribution (Map)**:
- Buckets: Geohash = geoip.location
- Metrics: Count
- Filter: log_type: apache

**Top URLs (Data Table)**:
- Metrics: Count
- Buckets: Split Rows = Terms of request
- Filter: log_type: apache

#### Application Logs Dashboard

1. Go to Kibana → Dashboard → Create dashboard
2. Add the following visualizations:

**Log Levels Over Time (Area Chart)**:
- Metrics: Y-axis = Count
- Buckets: X-axis = Date Histogram of @timestamp
- Buckets: Split Series = Terms of log.level
- Filter: log_type: application

**Error Occurrences (Line Chart)**:
- Metrics: Y-axis = Count
- Buckets: X-axis = Date Histogram of @timestamp
- Filter: log_type: application AND log.level: ERROR

**Top Error Types (Pie Chart)**:
- Metrics: Slice Size = Count
- Buckets: Split Slices = Terms of error.type
- Filter: log_type: application AND error.type: *

**Recent Errors (Data Table)**:
- Buckets: Split Rows = Date Histogram of @timestamp (1m interval)
- Metrics: Count
- Filter: log_type: application AND log.level: ERROR

### Creating Saved Searches

#### Critical Error Search

1. Go to Kibana → Discover
2. Set up the search:
   - Index pattern: logs-*
   - Time range: Last 24 hours
   - Query: log.level:ERROR OR log.level:FATAL
3. Save the search as "Critical Errors"

#### Failed Authentication Search

1. Go to Kibana → Discover
2. Set up the search:
   - Index pattern: logs-system-*
   - Time range: Last 24 hours
   - Query: system.auth.message:"Failed password"
3. Save the search as "Failed Authentication Attempts"

## Part 5: Setting Up Alerting

### Configuring Email Alerts

First, configure Elasticsearch to send emails:

```
PUT _cluster/settings
{
  "persistent": {
    "xpack.notification.email.account": {
      "gmail_account": {
        "profile": "gmail",
        "email_defaults": {
          "from": "alerts@example.com"
        },
        "smtp": {
          "auth": true,
          "starttls.enable": true,
          "host": "smtp.gmail.com",
          "port": 587,
          "user": "alerts@example.com",
          "password": "your-app-password-here"
        }
      }
    }
  }
}
```

### Creating Watcher Alerts

#### High Error Rate Alert

```
PUT _watcher/watch/high_error_rate
{
  "trigger": {
    "schedule": {
      "interval": "5m"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": ["logs-app-*"],
        "body": {
          "size": 0,
          "query": {
            "bool": {
              "must": [
                {
                  "term": {
                    "log.level": "ERROR"
                  }
                },
                {
                  "range": {
                    "@timestamp": {
                      "gte": "now-15m"
                    }
                  }
                }
              ]
            }
          },
          "aggs": {
            "error_count": {
              "value_count": {
                "field": "_id"
              }
            }
          }
        }
      }
    }
  },
  "condition": {
    "compare": {
      "ctx.payload.aggregations.error_count.value": {
        "gt": 50
      }
    }
  },
  "actions": {
    "email_admin": {
      "email": {
        "to": "admin@example.com",
        "subject": "High Error Rate Alert",
        "body": {
          "html": "<p>There have been {{ctx.payload.aggregations.error_count.value}} errors in the last 15 minutes.</p><p>Please check the <a href='https://kibana.example.com/app/kibana#/dashboard/app-errors'>error dashboard</a>.</p>"
        }
      }
    }
  }
}
```

#### Failed Login Alert

```
PUT _watcher/watch/failed_login_alert
{
  "trigger": {
    "schedule": {
      "interval": "5m"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": ["logs-system-*"],
        "body": {
          "size": 0,
          "query": {
            "bool": {
              "must": [
                {
                  "match_phrase": {
                    "system.auth.message": "Failed password"
                  }
                },
                {
                  "range": {
                    "@timestamp": {
                      "gte": "now-15m"
                    }
                  }
                }
              ]
            }
          },
          "aggs": {
            "ips": {
              "terms": {
                "field": "system.auth.ip",
                "size": 10
              },
              "aggs": {
                "attempts": {
                  "value_count": {
                    "field": "_id"
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
      "source": "for (ip in ctx.payload.aggregations.ips.buckets) { if (ip.attempts.value >= 5) { return true; } } return false;"
    }
  },
  "actions": {
    "email_security": {
      "email": {
        "to": "security@example.com",
        "subject": "Multiple Failed Login Attempts",
        "body": {
          "html": "<p>Multiple failed login attempts detected:</p><ul>{{#ctx.payload.aggregations.ips.buckets}}<li>IP: {{key}} - {{attempts.value}} attempts</li>{{/ctx.payload.aggregations.ips.buckets}}</ul>"
        }
      }
    }
  }
}
```

### Kibana Alerting

For newer versions of Kibana with built-in alerting:

1. Go to Kibana → Stack Management → Rules and Connectors → Connectors
2. Create a new Email connector
3. Go to Rules → Create rule
4. Select "Threshold" rule type
5. Configure:
   - Indices: logs-app-*
   - Time field: @timestamp
   - Aggregation field: Count documents
   - Group by: log.level
   - Condition: is above 100 over 15m
   - Filter: log.level: ERROR
6. Set actions to send email via the created connector

## Part 6: Security Implementation

### Securing the Pipeline

#### SSL/TLS Configuration

1. Generate certificates for all components:

```bash
# Create a CA
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 365 -key ca.key -out ca.crt

# Create server certificates
for component in filebeat logstash elasticsearch kibana; do
  openssl genrsa -out ${component}.key 2048
  openssl req -new -key ${component}.key -out ${component}.csr
  openssl x509 -req -days 365 -in ${component}.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out ${component}.crt
done
```

2. Distribute certificates to appropriate servers and update configurations

#### Authentication and Authorization

1. Enable security in Elasticsearch:

```yaml
# elasticsearch.yml
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: http.p12
xpack.security.http.ssl.truststore.path: http.p12
```

2. Create users and roles:

```
# Create logstash_writer role
POST /_security/role/logstash_writer
{
  "cluster": ["manage_index_templates", "manage_ilm", "monitor"],
  "indices": [
    {
      "names": ["logs-*"],
      "privileges": ["write", "create_index", "manage"]
    }
  ]
}

# Create kibana_user role
POST /_security/role/kibana_user
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
POST /_security/user/logstash_internal
{
  "password" : "secure_password_123",
  "roles" : ["logstash_writer"],
  "full_name" : "Logstash Internal User"
}

POST /_security/user/kibana_reader
{
  "password" : "secure_password_456",
  "roles" : ["kibana_user"],
  "full_name" : "Kibana Reader"
}
```

3. Update Logstash and Beats configurations to use authentication

#### Network Security

1. Configure firewall rules to restrict access:

```bash
# Allow Filebeat → Logstash
sudo ufw allow from 10.0.0.0/24 to any port 5044 proto tcp

# Allow Logstash → Elasticsearch
sudo ufw allow from 10.1.0.0/24 to any port 9200 proto tcp

# Allow Kibana → Elasticsearch
sudo ufw allow from 10.2.0.0/24 to any port 9200 proto tcp

# Allow admin access to Kibana
sudo ufw allow from 10.3.0.0/24 to any port 5601 proto tcp
```

2. Implement network segmentation between components

## Part 7: Monitoring the Pipeline

### Monitoring Filebeat

Enable monitoring in Filebeat:

```yaml
# filebeat.yml
monitoring.enabled: true
monitoring.elasticsearch:
  hosts: ["https://es1.example.com:9200", "https://es2.example.com:9200"]
  username: "${ES_MONITOR_USER}"
  password: "${ES_MONITOR_PASSWORD}"
  ssl.certificate_authorities: ["/etc/filebeat/ca.crt"]
```

### Monitoring Logstash

Enable monitoring in Logstash:

```yaml
# logstash.yml
xpack.monitoring.enabled: true
xpack.monitoring.elasticsearch.hosts: ["https://es1.example.com:9200", "https://es2.example.com:9200"]
xpack.monitoring.elasticsearch.username: "${ES_MONITOR_USER}"
xpack.monitoring.elasticsearch.password: "${ES_MONITOR_PASSWORD}"
xpack.monitoring.elasticsearch.ssl.certificate_authority: "/etc/logstash/certs/ca.crt"
```

### Monitoring Elasticsearch and Kibana

Enable monitoring in Elasticsearch:

```yaml
# elasticsearch.yml
xpack.monitoring.collection.enabled: true
```

Enable monitoring in Kibana:

```yaml
# kibana.yml
xpack.monitoring.enabled: true
xpack.monitoring.elasticsearch.hosts: ["https://es1.example.com:9200", "https://es2.example.com:9200"]
```

### Creating Stack Monitoring Dashboard

1. Go to Kibana → Stack Monitoring
2. Click on "Elasticsearch" to view cluster health, node metrics, and index statistics
3. Click on "Logstash" to view pipeline throughput and node metrics
4. Click on "Beats" to view Filebeat status and metrics

## Conclusion

This project has walked through the complete implementation of a production-ready log analytics pipeline using the ELK Stack. The solution provides:

- Centralized log collection from multiple sources
- Structured log processing and enrichment
- Efficient storage with lifecycle management
- Powerful visualizations and dashboards
- Proactive alerting for critical events
- End-to-end security
- Comprehensive monitoring

This foundation can be extended with additional data sources, custom dashboards, and more sophisticated analysis as your organization's needs evolve. The modular nature of the ELK Stack makes it easy to add capabilities like machine learning for anomaly detection or APM for application performance monitoring.

In the next project chapter, we'll explore building an Application Performance Monitoring (APM) solution using the Elastic Stack.