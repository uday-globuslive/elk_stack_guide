# Output Plugins

## Introduction to Output Plugins

Output plugins are the final stage of the Logstash pipeline, responsible for sending processed events to various destinations. These plugins allow you to send data to storage systems, messaging queues, monitoring tools, notification services, and more.

This chapter covers common output plugins, their configuration options, and best practices to effectively deliver processed data to your target destinations.

## Elasticsearch Output

The Elasticsearch output plugin sends events to Elasticsearch indices, forming the crucial connection between Logstash and Elasticsearch in the ELK Stack.

### Basic Configuration

```ruby
output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "logstash-%{+YYYY.MM.dd}"
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `hosts` | Elasticsearch host(s) | ["localhost:9200"] | `["es1:9200", "es2:9200"]` |
| `index` | Index name or pattern | "logstash-%{+YYYY.MM.dd}" | `"logs-%{type}-%{+YYYY.MM}"` |
| `document_id` | Event document ID | None | `"%{id}"` |
| `user` | Username for auth | None | `"elastic"` |
| `password` | Password for auth | None | `"changeme"` |
| `api_key` | API key for auth | None | `"id:api_key"` |
| `ssl` | Use SSL | false | `true` |
| `cacert` | CA certificate path | None | `"/path/to/ca.crt"` |
| `manage_template` | Use index templates | true | `false` |
| `template_name` | Template name | "logstash" | `"custom_template"` |
| `template` | Path to template file | None | `"/path/to/template.json"` |
| `template_overwrite` | Overwrite templates | false | `true` |
| `ilm_enabled` | Use ILM | auto | `true` |
| `ilm_policy` | ILM policy | "logstash-policy" | `"custom-policy"` |
| `ilm_pattern` | ILM index pattern | "{now/d}-000001" | `"{now/M}-000001"` |
| `ilm_rollover_alias` | ILM rollover alias | "logstash" | `"logs"` |
| `action` | Document action | "index" | `"update"` |
| `retry_on_conflict` | Retries for conflicts | 1 | `5` |
| `doc_as_upsert` | Use doc as upsert | false | `true` |
| `pipeline` | Ingest pipeline | None | `"log_pipeline"` |
| `bulk_size` | Events per bulk request | 1000 | `500` |
| `timeout` | Request timeout (s) | 60 | `120` |
| `sniffing` | Auto-discover nodes | false | `true` |

### Secure Connection Configuration

Configure TLS/SSL and authentication:

```ruby
output {
  elasticsearch {
    hosts => ["https://elasticsearch:9200"]
    user => "${ES_USER}"
    password => "${ES_PASSWORD}"
    ssl => true
    ssl_certificate_verification => true
    cacert => "/path/to/ca.crt"
  }
}
```

### Dynamic Index Names

Create dynamic indices based on event fields:

```ruby
output {
  elasticsearch {
    hosts => ["https://elasticsearch:9200"]
    
    # Index by type and date
    index => "%{[@metadata][type]}-%{+YYYY.MM.dd}"
    
    # Handle missing type
    index => "%{[type]:%{[@metadata][type]:logs}}-%{+YYYY.MM.dd}"
  }
}
```

### Multiple Indices Based on Conditions

Route events to different indices:

```ruby
output {
  if [type] == "apache" {
    elasticsearch {
      hosts => ["https://elasticsearch:9200"]
      index => "apache-%{+YYYY.MM.dd}"
    }
  } else if [type] == "nginx" {
    elasticsearch {
      hosts => ["https://elasticsearch:9200"]
      index => "nginx-%{+YYYY.MM.dd}"
    }
  } else {
    elasticsearch {
      hosts => ["https://elasticsearch:9200"]
      index => "other-%{+YYYY.MM.dd}"
    }
  }
}
```

### Document ID Assignment

Specify document IDs for updates or control:

```ruby
output {
  elasticsearch {
    hosts => ["https://elasticsearch:9200"]
    document_id => "%{unique_id}"  # Use an existing field
    document_id => "%{[customer][id]}_%{+YYYY.MM.dd}"  # Use a composite value
  }
}
```

### Ingest Pipeline Integration

Use Elasticsearch ingest pipelines for processing:

```ruby
output {
  elasticsearch {
    hosts => ["https://elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
    pipeline => "log-enrichment"
  }
}
```

### Index Lifecycle Management (ILM)

Configure ILM for automated index lifecycle:

```ruby
output {
  elasticsearch {
    hosts => ["https://elasticsearch:9200"]
    
    # Enable ILM
    ilm_enabled => true
    ilm_rollover_alias => "logs"
    ilm_pattern => "{now/d}-000001"
    ilm_policy => "logs-policy"
    
    # Use the rollover alias as the index name
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
  }
}
```

### Bulk Indexing Optimization

Optimize for high-throughput indexing:

```ruby
output {
  elasticsearch {
    hosts => ["https://elasticsearch:9200"]
    
    # Tune bulk requests
    bulk_size => 5000
    flush_size => 10000  # Flush threshold
    
    # Concurrent connections
    worker_count => 4
    
    # Manage indexing pressure
    retry_max_interval => 64
    retry_initial_interval => 1
    timeout => 120
  }
}
```

### Template Management

Control index templates:

```ruby
output {
  elasticsearch {
    hosts => ["https://elasticsearch:9200"]
    
    # Template management
    template_name => "logs-template"
    template => "/etc/logstash/templates/logs-template.json"
    template_overwrite => true
    
    # Disable automatic template management
    manage_template => false
  }
}
```

### Best Practices for Elasticsearch Output

- Use multiple hosts for high availability
- Store credentials securely (e.g., environment variables, keystore)
- Configure appropriate error handling and retries
- Optimize bulk size for your workload
- Use indices appropriate for your volume and retention needs
- Implement appropriate document ID strategy
- Consider using Index Lifecycle Management for long-term index management
- Monitor indexing performance and adjust parameters as needed

## File Output

The File output plugin writes events to files on disk.

### Basic Configuration

```ruby
output {
  file {
    path => "/var/log/logstash/output.log"
    codec => "line"
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `path` | Output file path | Required | `"/path/to/file.log"` |
| `codec` | Output format codec | "line" | `"json_lines"` |
| `flush_interval` | Write flush interval (s) | 2 | `5` |
| `gzip` | Compress output files | false | `true` |
| `create_if_deleted` | Create file if deleted | true | `false` |
| `write_behavior` | Write behavior | "append" | `"overwrite"` |
| `dir_mode` | Directory permissions | 0750 | `0755` |
| `file_mode` | File permissions | 0644 | `0664` |

### Dynamic File Paths

Generate dynamic file paths based on event data:

```ruby
output {
  file {
    path => "/var/log/%{type}/%{+YYYY/MM/dd}/%{host}.log"
    codec => line { format => "%{message}" }
  }
}
```

### JSON File Output

Output structured JSON data:

```ruby
output {
  file {
    path => "/var/log/json/%{+YYYY-MM-dd}-events.json"
    codec => json_lines
  }
}
```

### Rotated and Compressed Logs

Create compressed and rotated logs:

```ruby
output {
  file {
    path => "/var/log/logstash/%{+YYYY-MM-dd-HH}.log"
    gzip => true
    flush_interval => 10
    file_mode => "0640"
  }
}
```

### CSV Output

Generate CSV files:

```ruby
output {
  file {
    path => "/var/log/data/%{+YYYY-MM-dd}-export.csv"
    codec => line { 
      format => "%{timestamp},%{client},%{method},%{status},%{response_time}" 
    }
  }
}
```

### Best Practices for File Output

- Use dynamic path elements to organize files
- Set appropriate file permissions
- Configure flush interval based on data criticality
- Use compression for large files
- Implement log rotation for long-running outputs
- Consider filesystem performance and capacity
- Ensure the logstash user has write permissions to target directories

## Kafka Output

The Kafka output plugin sends events to Apache Kafka topics.

### Basic Configuration

```ruby
output {
  kafka {
    bootstrap_servers => "kafka1:9092,kafka2:9092,kafka3:9092"
    topic_id => "logs"
    key => "%{id}"
    codec => json
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `bootstrap_servers` | Kafka brokers | Required | `"kafka1:9092,kafka2:9092"` |
| `topic_id` | Target topic | Required | `"logs"` |
| `key` | Message key | None | `"%{id}"` |
| `partitioner` | Partitioning strategy | "random" | `"round_robin"` |
| `compression_type` | Compression | "none" | `"gzip"` |
| `batch_size` | Messages per batch | 16384 | `32768` |
| `linger_ms` | Batch delay (ms) | 0 | `10` |
| `buffer_memory` | Producer buffer (bytes) | 33554432 | `67108864` |
| `acks` | Acknowledgment level | 1 | `"all"` |
| `retries` | Retry count | 0 | `5` |
| `retry_backoff_ms` | Retry delay (ms) | 100 | `200` |
| `security_protocol` | Security protocol | "PLAINTEXT" | `"SASL_SSL"` |
| `sasl_mechanism` | SASL mechanism | None | `"PLAIN"` |
| `jaas_path` | JAAS config path | None | `"/path/to/jaas.conf"` |
| `ssl_keystore_location` | Keystore path | None | `"/path/to/keystore.jks"` |
| `ssl_keystore_password` | Keystore password | None | `"password"` |
| `ssl_truststore_location` | Truststore path | None | `"/path/to/truststore.jks"` |
| `ssl_truststore_password` | Truststore password | None | `"password"` |

### Dynamic Topic Routing

Route events to different topics:

```ruby
output {
  kafka {
    bootstrap_servers => "kafka1:9092,kafka2:9092"
    topic_id => "logs-%{type}"  # Dynamic topic based on event type
    key => "%{id}"  # Consistent routing key for related events
    codec => json
  }
}
```

### Secure Kafka Connection

Connect to secure Kafka clusters:

```ruby
output {
  kafka {
    bootstrap_servers => "kafka1:9092,kafka2:9092"
    topic_id => "secure-logs"
    
    # Security settings
    security_protocol => "SASL_SSL"
    sasl_mechanism => "PLAIN"
    sasl_jaas_config => "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"user\" password=\"pass\";"
    
    # SSL settings
    ssl_truststore_location => "/etc/logstash/kafka.truststore.jks"
    ssl_truststore_password => "${TRUSTSTORE_PASSWORD}"
    
    # Producer settings
    codec => json
    compression_type => "gzip"
    acks => "all"
  }
}
```

### Performance Optimization

Optimize for throughput and reliability:

```ruby
output {
  kafka {
    bootstrap_servers => "kafka1:9092,kafka2:9092"
    topic_id => "high-volume-logs"
    
    # Performance tuning
    batch_size => 32768
    linger_ms => 50
    compression_type => "lz4"
    max_request_size => 5242880
    buffer_memory => 134217728
    
    # Reliability settings
    acks => "all"
    retries => 5
    retry_backoff_ms => 200
    max_in_flight_requests_per_connection => 5
  }
}
```

### Message Headers

Add custom headers to Kafka messages:

```ruby
output {
  kafka {
    bootstrap_servers => "kafka1:9092,kafka2:9092"
    topic_id => "logs"
    
    # Add headers
    message_key => "%{id}"
    header_keys => ["app_name", "environment", "trace_id"]
  }
}
```

### Best Practices for Kafka Output

- Use multiple bootstrap servers for failover
- Implement appropriate security for production
- Set message keys for ordered processing of related events
- Choose compression type based on your bandwidth/CPU trade-offs
- Configure batching for your throughput requirements
- Set appropriate acknowledgment level for your reliability needs
- Monitor producer performance and adjust parameters as needed
- Store sensitive configuration in secure variables

## HTTP Output

The HTTP output plugin sends events to an HTTP endpoint.

### Basic Configuration

```ruby
output {
  http {
    url => "https://api.example.com/logs"
    http_method => "post"
    format => "json"
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `url` | Destination URL | Required | `"https://api.example.com/logs"` |
| `http_method` | HTTP method | "post" | `"put"` |
| `format` | Payload format | "json" | `"form"` |
| `content_type` | Content-Type header | "application/json" | `"application/x-ndjson"` |
| `headers` | HTTP headers | None | `{"Authorization" => "Bearer token"}` |
| `mapping` | Field mapping | None | `{"event" => "%{message}"}` |
| `pool_max` | Connection pool size | 50 | `100` |
| `automatic_retries` | Retry count | 1 | `3` |
| `retry_non_idempotent` | Retry non-idempotent | false | `true` |
| `connect_timeout` | Connect timeout (s) | 10 | `5` |
| `socket_timeout` | Socket timeout (s) | 10 | `5` |
| `validate_after_inactivity` | Validate after idle (ms) | 1000 | `2000` |
| `cacert` | CA certificate path | None | `"/path/to/ca.crt"` |
| `client_cert` | Client certificate path | None | `"/path/to/client.crt"` |
| `client_key` | Client key path | None | `"/path/to/client.key"` |
| `truststore` | Java truststore path | None | `"/path/to/truststore.jks"` |
| `truststore_password` | Truststore password | None | `"password"` |
| `keystore` | Java keystore path | None | `"/path/to/keystore.jks"` |
| `keystore_password` | Keystore password | None | `"password"` |

### JSON Payload Formatting

Send JSON-formatted payloads:

```ruby
output {
  http {
    url => "https://api.example.com/ingest"
    http_method => "post"
    format => "json"
    content_type => "application/json"
    
    # Include only specific fields
    mapping => {
      "timestamp" => "%{@timestamp}"
      "log_level" => "%{level}"
      "message" => "%{message}"
      "source" => "%{type}"
    }
  }
}
```

### Authentication

Configure various authentication methods:

```ruby
output {
  http {
    url => "https://api.example.com/logs"
    
    # Basic auth
    user => "username"
    password => "${HTTP_PASSWORD}"
    
    # OR OAuth token
    headers => {
      "Authorization" => "Bearer ${AUTH_TOKEN}"
    }
    
    # OR API key
    headers => {
      "X-API-Key" => "${API_KEY}"
    }
  }
}
```

### HTTPS with Client Certificates

Configure mutual TLS authentication:

```ruby
output {
  http {
    url => "https://secure-api.example.com/logs"
    http_method => "post"
    format => "json"
    
    # TLS configuration
    ssl_certificate_validation => true
    cacert => "/etc/logstash/ca.crt"
    client_cert => "/etc/logstash/client.crt"
    client_key => "/etc/logstash/client.key"
  }
}
```

### Handling Retries and Failures

Configure retry behavior:

```ruby
output {
  http {
    url => "https://api.example.com/logs"
    http_method => "post"
    
    # Retry configuration
    automatic_retries => 5
    retry_non_idempotent => true
    backoff_time => 3
    
    # Timeouts
    connect_timeout => 5
    socket_timeout => 10
    request_timeout => 60
  }
}
```

### Webhooks and Alerting

Send alerts to chat systems or incident management:

```ruby
output {
  # Only send critical errors
  if [log_level] == "ERROR" {
    http {
      url => "https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX"
      format => "json"
      content_type => "application/json"
      
      # Slack message format
      mapping => {
        "text" => "Error detected: %{message}"
        "attachments" => [
          {
            "color" => "danger",
            "fields" => [
              {
                "title" => "Application",
                "value" => "%{application}",
                "short" => true
              },
              {
                "title" => "Environment",
                "value" => "%{environment}",
                "short" => true
              },
              {
                "title" => "Error",
                "value" => "%{error_message}"
              }
            ]
          }
        ]
      }
    }
  }
}
```

### Best Practices for HTTP Output

- Set appropriate timeouts for your API endpoint
- Implement proper authentication
- Configure retries based on endpoint reliability
- Use mapping to control payload format
- Set up error handling for HTTP failures
- Monitor response times and adjust settings as needed
- Secure credentials using environment variables or keystore
- Consider rate limiting to avoid overloading endpoints

## S3 Output

The S3 output plugin writes events to Amazon S3 buckets.

### Basic Configuration

```ruby
output {
  s3 {
    bucket => "my-log-bucket"
    region => "us-east-1"
    prefix => "logs/%{+YYYY/MM/dd}"
    time_file => 15
    codec => "json_lines"
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `bucket` | S3 bucket name | Required | `"log-bucket"` |
| `region` | AWS region | "us-east-1" | `"eu-west-1"` |
| `prefix` | S3 key prefix | None | `"logs/"` |
| `time_file` | Periodic upload minutes | 15 | `5` |
| `size_file` | Max file size (MB) | 0 | `1024` |
| `codec` | Output format | "line" | `"json_lines"` |
| `canned_acl` | S3 ACL | "private" | `"bucket-owner-full-control"` |
| `storage_class` | S3 storage class | "STANDARD" | `"STANDARD_IA"` |
| `access_key_id` | AWS access key | None | `"AKIAIOSFODNN7EXAMPLE"` |
| `secret_access_key` | AWS secret key | None | `"wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"` |
| `session_token` | AWS session token | None | `"token"` |
| `role_arn` | AWS IAM role | None | `"arn:aws:iam::123456789012:role/LogstashS3"` |
| `temporary_directory` | Temp dir for files | System temp | `"/tmp/logstash-s3"` |
| `rotation_strategy` | Rotation strategy | "size_and_time" | `"time"` |
| `server_side_encryption` | S3 encryption | false | `true` |
| `ssekms_key_id` | KMS key ID | None | `"arn:aws:kms:region:account:key/key-id"` |
| `signature_version` | S3 signature version | None | `"v4"` |
| `upload_queue_size` | Upload queue size | 4 | `8` |
| `upload_workers_count` | Upload worker threads | 1 | `4` |

### Dynamic S3 Path Structure

Create organized S3 paths:

```ruby
output {
  s3 {
    bucket => "log-storage-bucket"
    region => "us-west-2"
    
    # Dynamic path based on event data
    prefix => "logs/%{type}/%{+YYYY/MM/dd/HH}"
    
    # Filename pattern
    additional_settings => {
      "key_format" => "%{host}.%{+YYYY-MM-dd-HH-mm}.log"
    }
    
    # File management
    time_file => 60
    size_file => 1024
    codec => json_lines
  }
}
```

### AWS Authentication Options

Configure different authentication methods:

```ruby
output {
  s3 {
    bucket => "log-storage-bucket"
    region => "us-west-2"
    
    # Method 1: Access keys (use environment variables instead when possible)
    access_key_id => "${AWS_ACCESS_KEY_ID}"
    secret_access_key => "${AWS_SECRET_ACCESS_KEY}"
    
    # Method 2: IAM role
    role_arn => "arn:aws:iam::123456789012:role/LogstashS3Role"
    role_session_name => "logstash-s3-upload"
    
    # Method 3: Default credential provider chain
    # (No explicit credentials - use instance profile, environment variables, etc.)
  }
}
```

### S3 Storage Options

Configure S3 storage features:

```ruby
output {
  s3 {
    bucket => "log-storage-bucket"
    region => "us-west-2"
    
    # File rotation
    time_file => 15
    size_file => 5120
    rotation_strategy => "size_and_time"
    
    # Storage settings
    storage_class => "STANDARD_IA"
    canned_acl => "bucket-owner-full-control"
    
    # Encryption
    server_side_encryption => true
    server_side_encryption_algorithm => "aws:kms"
    ssekms_key_id => "arn:aws:kms:us-west-2:123456789012:key/key-id"
    
    # Performance
    upload_queue_size => 8
    upload_workers_count => 4
  }
}
```

### Compression and Format

Configure file compression and format:

```ruby
output {
  s3 {
    bucket => "log-storage-bucket"
    region => "us-west-2"
    
    # Compression
    gzip => true
    
    # File format
    codec => json_lines
    
    # Or for CSV
    codec => line {
      format => "%{timestamp},%{client_ip},%{method},%{status},%{response_time}"
    }
  }
}
```

### Best Practices for S3 Output

- Create a logical path hierarchy using prefix
- Use time-based rotation for regular uploads
- Configure size-based rotation to handle volume spikes
- Use appropriate IAM roles instead of access keys when possible
- Choose storage class based on access patterns and retention
- Configure compression for large log volumes
- Monitor temporary storage usage on busy systems
- Set up appropriate S3 lifecycle policies for retention and storage tier transitions

## Redis Output

The Redis output plugin sends events to Redis lists, channels, or as key-value pairs.

### Basic Configuration

```ruby
output {
  redis {
    host => "redis.example.com"
    port => 6379
    data_type => "list"
    key => "logstash:events"
    codec => json
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `host` | Redis host | "127.0.0.1" | `"redis.example.com"` |
| `port` | Redis port | 6379 | `6380` |
| `db` | Redis database | 0 | `1` |
| `password` | Redis password | None | `"password"` |
| `data_type` | Redis data type | "list" | `"channel"` |
| `key` | Redis key | "logstash" | `"logstash:events"` |
| `timeout` | Connection timeout (s) | 5 | `10` |
| `congestion_interval` | Congestion retry (s) | 1 | `5` |
| `congestion_threshold` | Queue threshold | 0 | `5000` |
| `batch` | Enable batching | false | `true` |
| `batch_events` | Events per batch | 50 | `100` |
| `batch_timeout` | Batch timeout (s) | 5 | `1` |
| `ssl` | Enable SSL | false | `true` |
| `ssl_cert` | SSL certificate path | None | `"/path/to/cert.pem"` |
| `ssl_key` | SSL key path | None | `"/path/to/key.pem"` |
| `ssl_ca_file` | SSL CA certificate | None | `"/path/to/ca.pem"` |

### Redis List Producer

Send events to a Redis list:

```ruby
output {
  redis {
    host => "redis.example.com"
    port => 6379
    password => "${REDIS_PASSWORD}"
    data_type => "list"
    key => "logs:apache"
    codec => json
    
    # Performance settings
    batch => true
    batch_events => 100
    batch_timeout => 1
  }
}
```

### Redis Pub/Sub Channel

Publish events to a Redis channel:

```ruby
output {
  redis {
    host => "redis.example.com"
    port => 6379
    data_type => "channel"
    key => "logs-channel"
    codec => json
  }
}
```

### Redis Sentinel Configuration

Connect through Redis Sentinel:

```ruby
output {
  redis {
    host => ["sentinel1", "sentinel2", "sentinel3"]
    port => 26379
    sentinel => true
    sentinel_master => "mymaster"
    data_type => "list"
    key => "logs:apache"
  }
}
```

### Redis with SSL/TLS

Configure secure Redis connection:

```ruby
output {
  redis {
    host => "redis.example.com"
    port => 6380
    password => "${REDIS_PASSWORD}"
    data_type => "list"
    key => "secure-logs"
    
    # SSL configuration
    ssl => true
    ssl_cert => "/path/to/client.crt"
    ssl_key => "/path/to/client.key"
    ssl_ca_file => "/path/to/ca.crt"
    ssl_verify => true
  }
}
```

### Dynamic Redis Keys

Use dynamic key names based on event data:

```ruby
output {
  redis {
    host => "redis.example.com"
    data_type => "list"
    key => "logs:%{type}:%{+YYYY.MM.dd}"
    codec => json
  }
}
```

### Best Practices for Redis Output

- Use batching for high-throughput scenarios
- Monitor Redis memory usage
- Set up Redis persistence for reliability
- Configure timeouts based on network stability
- Use appropriate congestion handling for variable loads
- Consider Redis Sentinel or Redis Cluster for high availability
- Use SSL for connections across untrusted networks
- Store credentials securely using environment variables or keystore

## Email Output

The Email output plugin sends events as emails.

### Basic Configuration

```ruby
output {
  email {
    to => "admin@example.com"
    from => "logstash@example.com"
    subject => "Logstash Alert"
    body => "Alert condition detected: %{message}"
    domain => "mail.example.com"
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `to` | Recipient email(s) | Required | `"admin@example.com"` |
| `from` | Sender email | "logstash@example.com" | `"alerts@example.com"` |
| `cc` | CC recipients | None | `"manager@example.com"` |
| `bcc` | BCC recipients | None | `"archive@example.com"` |
| `subject` | Email subject | None | `"Alert: %{host}"` |
| `subject_template` | Subject template path | None | `"/path/to/subject.erb"` |
| `body` | Email body | None | `"Error: %{message}"` |
| `body_template` | Body template path | None | `"/path/to/body.erb"` |
| `htmlbody` | HTML body | None | `"<b>Error:</b> %{message}"` |
| `attachments` | File attachments | None | `["/path/to/file.log"]` |
| `contenttype` | Content type | "text/plain" | `"text/html"` |
| `domain` | SMTP domain | "localhost" | `"mail.example.com"` |
| `address` | SMTP server | "localhost" | `"smtp.example.com"` |
| `port` | SMTP port | 25 | `587` |
| `username` | SMTP username | None | `"user"` |
| `password` | SMTP password | None | `"password"` |
| `authentication` | Auth type | "plain" | `"login"` |
| `use_tls` | Use STARTTLS | false | `true` |
| `use_ssl` | Use SSL/TLS | false | `true` |

### HTML Email Formatting

Send formatted HTML emails:

```ruby
output {
  if [log_level] == "ERROR" {
    email {
      to => "alerts@example.com"
      from => "logstash@example.com"
      subject => "Error Alert: %{application} - %{environment}"
      contenttype => "text/html"
      htmlbody => "
        <html>
          <body>
            <h2>Error Alert</h2>
            <p><strong>Application:</strong> %{application}</p>
            <p><strong>Environment:</strong> %{environment}</p>
            <p><strong>Error:</strong> %{message}</p>
            <p><strong>Timestamp:</strong> %{@timestamp}</p>
            <p><strong>Host:</strong> %{host}</p>
          </body>
        </html>
      "
      
      # SMTP settings
      domain => "example.com"
      address => "smtp.example.com"
      port => 587
      username => "${SMTP_USER}"
      password => "${SMTP_PASSWORD}"
      authentication => "plain"
      use_tls => true
    }
  }
}
```

### Email Templates

Use external templates for emails:

```ruby
output {
  if [log_level] == "ERROR" {
    email {
      to => "alerts@example.com"
      from => "logstash@example.com"
      subject => "Error Alert: %{application}"
      
      # Use ERB templates
      body_template => "/etc/logstash/templates/alert_email.erb"
      
      # SMTP settings
      domain => "example.com"
      address => "smtp.example.com"
      port => 587
      username => "${SMTP_USER}"
      password => "${SMTP_PASSWORD}"
      authentication => "plain"
      use_tls => true
    }
  }
}
```

Example ERB template (`/etc/logstash/templates/alert_email.erb`):
```erb
Error Alert Details:

Application: <%= event.get('application') %>
Environment: <%= event.get('environment') %>
Error: <%= event.get('message') %>
Timestamp: <%= event.get('@timestamp') %>
Host: <%= event.get('host') %>

Additional Context:
<% if event.get('stack_trace') %>
Stack Trace:
<%= event.get('stack_trace') %>
<% end %>
```

### Conditional Emails with Throttling

Send alerts conditionally with throttling:

```ruby
output {
  if [log_level] == "ERROR" {
    # First aggregate errors to deduplicate
    aggregate {
      task_id => "%{error_code}"
      code => "
        map['count'] ||= 0
        map['count'] += 1
        map['first_seen'] ||= event.get('@timestamp')
        map['examples'] ||= []
        if map['examples'].length < 5
          map['examples'] << event.get('message')
        end
        event.set('error_count', map['count'])
        event.set('first_seen', map['first_seen'])
        event.set('error_examples', map['examples'])
      "
      push_map_as_event_on_timeout => true
      timeout => 300  # 5 minute aggregation window
      timeout_code => "
        event.set('aggregate', true)
      "
    }
    
    # Only send email for aggregated events or first occurrence
    if [aggregate] or [error_count] == 1 {
      email {
        to => "alerts@example.com"
        from => "logstash@example.com"
        subject => "Error Alert: %{error_count} errors with code %{error_code}"
        contenttype => "text/html"
        htmlbody => "
          <html>
            <body>
              <h2>Error Alert</h2>
              <p><strong>Error Code:</strong> %{error_code}</p>
              <p><strong>Count:</strong> %{error_count} errors</p>
              <p><strong>First Seen:</strong> %{first_seen}</p>
              <p><strong>Latest:</strong> %{@timestamp}</p>
              <h3>Examples:</h3>
              <ul>
                <% @error_examples.each do |example| %>
                  <li><%= example %></li>
                <% end %>
              </ul>
            </body>
          </html>
        "
        # SMTP settings...
      }
    }
  }
}
```

### Best Practices for Email Output

- Use conditionals to send alerts only for important events
- Implement throttling to prevent email floods
- Use templates for consistent formatting
- Include relevant context in email content
- Set up proper authentication for reliable delivery
- Use TLS for secure transmission
- Store SMTP credentials securely
- Consider using an email delivery service for high-volume alerts

## Stdout Output

The Stdout output plugin writes events to the standard output, useful for debugging and development.

### Basic Configuration

```ruby
output {
  stdout {
    codec => rubydebug
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `codec` | Output format | "rubydebug" | `"json"` |

### Different Output Formats

Configure various output formats:

```ruby
output {
  # Ruby debug format (pretty-printed)
  stdout { codec => rubydebug }
  
  # JSON format
  stdout { codec => json }
  
  # Line format
  stdout { codec => line { format => "%{host}: %{message}" } }
  
  # Dots (for throughput monitoring)
  stdout { codec => dots }
}
```

### Debugging with Field Selection

Show only specific fields for clarity:

```ruby
output {
  stdout {
    codec => rubydebug {
      metadata => true
      fields => ["timestamp", "host", "message", "level", "trace_id"]
    }
  }
}
```

### Conditional Debug Output

Use for selective debugging:

```ruby
output {
  # Primary output to Elasticsearch
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }
  
  # Debug output only for specific conditions
  if [log_level] == "DEBUG" or [debug] == true {
    stdout {
      codec => rubydebug
    }
  }
}
```

### Best Practices for Stdout Output

- Use primarily for development and troubleshooting
- Remove or disable in production pipelines
- Apply conditionals to filter what's shown
- Select appropriate codec for your needs
- Be aware of performance impact for high-volume pipelines

## Multiple Outputs

You can use multiple outputs simultaneously to send events to different destinations.

### Basic Multiple Outputs

Send events to multiple destinations:

```ruby
output {
  # Primary storage in Elasticsearch
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }
  
  # Backup to S3
  s3 {
    bucket => "log-backup"
    region => "us-east-1"
    prefix => "logs/%{+YYYY/MM/dd}"
    codec => json_lines
  }
  
  # Real-time streaming to Kafka
  kafka {
    bootstrap_servers => "kafka:9092"
    topic_id => "logs-stream"
    codec => json
  }
}
```

### Conditional Output Routing

Route different events to different destinations:

```ruby
output {
  # All logs go to Elasticsearch
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }
  
  # Critical errors also go to alerting systems
  if [log_level] == "ERROR" {
    # Send to PagerDuty
    http {
      url => "https://events.pagerduty.com/v2/enqueue"
      http_method => "post"
      format => "json"
      content_type => "application/json"
      mapping => {
        "routing_key" => "${PAGERDUTY_KEY}"
        "event_action" => "trigger"
        "payload" => {
          "summary" => "Error: %{message}"
          "source" => "%{host}"
          "severity" => "critical"
        }
      }
    }
    
    # Send email alerts
    email {
      to => "ops@example.com"
      subject => "Critical Error: %{application}"
      body => "Error: %{message}\nHost: %{host}\nTime: %{@timestamp}"
    }
  }
  
  # Archive access logs only
  if [type] == "access_log" {
    s3 {
      bucket => "access-logs"
      region => "us-east-1"
      prefix => "logs/%{+YYYY/MM/dd}"
      codec => json_lines
    }
  }
  
  # Send metrics to monitoring system
  if [type] == "metric" {
    http {
      url => "https://metrics.example.com/ingest"
      http_method => "post"
      format => "json"
    }
  }
}
```

### Cloning and Multiple Paths

Process the same event differently for different destinations:

```ruby
output {
  # Clone the event for different processing paths
  if [type] == "access_log" {
    clone {
      clones => ["archive", "metrics"]
    }
  }
  
  # Original events go to Elasticsearch
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }
  
  # Archive clone with minimal processing
  if "archive" in [tags] {
    s3 {
      bucket => "log-archive"
      region => "us-east-1"
      prefix => "raw/%{+YYYY/MM/dd}"
      codec => json_lines
    }
  }
  
  # Metrics clone with aggregation
  if "metrics" in [tags] {
    # Process for metrics
    aggregate {
      task_id => "%{host}-%{+YYYY.MM.dd.HH}"
      code => "
        map['requests'] ||= 0
        map['requests'] += 1
        
        map['bytes'] ||= 0
        map['bytes'] += event.get('bytes').to_i
        
        status = event.get('status').to_i
        map['errors'] ||= 0
        map['errors'] += 1 if status >= 500
        
        # Only emit every 5 minutes
        if map['requests'] % 100 == 0 || event.get('@timestamp') - (map['last_emit'] || 0) > 300
          map['last_emit'] = event.get('@timestamp')
          event.set('total_requests', map['requests'])
          event.set('total_bytes', map['bytes'])
          event.set('error_count', map['errors'])
          event.set('metric', true)
        else
          event.cancel()
        end
      "
    }
    
    # Only send if it's a metric event
    if [metric] {
      http {
        url => "https://metrics.example.com/ingest"
        format => "json"
        mapping => {
          "host" => "%{host}"
          "timestamp" => "%{@timestamp}"
          "requests" => "%{total_requests}"
          "bytes" => "%{total_bytes}"
          "errors" => "%{error_count}"
        }
      }
    }
  }
}
```

### Load Balancing and Failover

Configure outputs for high availability:

```ruby
output {
  # Load balanced Elasticsearch cluster
  elasticsearch {
    hosts => ["es1:9200", "es2:9200", "es3:9200"]
    loadbalance => true
    index => "logs-%{+YYYY.MM.dd}"
  }
  
  # Failover outputs
  if "_elasticsearch_output_failure" in [tags] {
    # Primary backup: Kafka
    kafka {
      bootstrap_servers => "kafka1:9092,kafka2:9092"
      topic_id => "elasticsearch_failures"
      codec => json
      max_request_size => 10485760
    }
    
    # Secondary backup: File
    file {
      path => "/var/log/logstash/elasticsearch_failures_%{+YYYY-MM-dd}.log"
      codec => json_lines
    }
  }
}
```

### Best Practices for Multiple Outputs

- Consider the performance impact of multiple outputs
- Use conditionals for efficient event routing
- Implement proper error handling for each output
- Monitor resources when using resource-intensive outputs
- Balance real-time needs with batch processing
- Design for fault tolerance with failover outputs
- Be cautious with complex processing that may impact throughput

## Conclusion

Output plugins are the final stage in the Logstash pipeline, delivering processed events to their destinations. By understanding the capabilities and configuration options of these plugins, you can effectively route your data to various storage systems, services, and applications.

When choosing output plugins for your use case, consider:

1. **Data destination requirements**: Storage, messaging, notification, or monitoring
2. **Performance needs**: Throughput, latency, and resource usage
3. **Reliability requirements**: Guarantees, retries, and buffering capabilities
4. **Security considerations**: Authentication, encryption, and access control
5. **Operational complexity**: Maintenance, monitoring, and troubleshooting

The right combination of output plugins will ensure your processed data reaches its intended destinations reliably and efficiently, completing the data processing pipeline.

In the next chapter, we'll explore pipeline configuration, covering advanced topics like multiple pipelines, conditional logic, and pipeline performance tuning.