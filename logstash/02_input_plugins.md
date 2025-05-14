# Input Plugins

## Introduction to Input Plugins

Input plugins are the starting point of the Logstash pipeline, responsible for collecting data from various sources. They enable Logstash to ingest data from files, messaging systems, databases, APIs, and more.

This chapter covers common input plugins, their configuration options, and best practices to effectively collect data from different sources.

## File Input

The file input plugin reads data from files on the filesystem, similar to the `tail -f` command.

### Basic Configuration

```ruby
input {
  file {
    path => ["/var/log/apache/access.log", "/var/log/messages"]
    start_position => "beginning"
    type => "log"
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `path` | File path(s) to read | Required | `["/var/log/*.log"]` |
| `exclude` | Files to exclude | None | `"*.gz"` |
| `start_position` | Where to start reading: "beginning" or "end" | "end" | `"beginning"` |
| `sincedb_path` | File to track read positions | `$HOME/.sincedb_*` | `"/path/to/sincedb"` |
| `sincedb_write_interval` | How often to write position data | "15s" | `"5s"` |
| `stat_interval` | How often to check for file changes | "1s" | `"0.5s"` |
| `discover_interval` | How often to discover new files | "15s" | `"10s"` |
| `ignore_older` | Ignore files older than a given age | None | `"86400"` (1 day) |
| `close_older` | Close file handles older than a given age | "1h" | `"3600"` (1 hour) |
| `file_completed_action` | What to do after reading a file completely | "delete" | `"log"` |
| `mode` | Read mode: "tail" or "read" | "tail" | `"read"` |

### Handling Log Rotation

To handle log rotation properly:

```ruby
input {
  file {
    path => "/var/log/apache/access.log"
    start_position => "end"
    stat_interval => "1"
    discover_interval => "5"
    sincedb_write_interval => "1"
    close_older => "10m"
    ignore_older => "1d"
  }
}
```

### File Input Use Cases

- Server logs (Apache, Nginx, system logs)
- Application logs
- Batch processing of log archives
- Monitoring text-based data files

### Best Practices for File Input

- Use wildcards sparingly to avoid excessive file system scans
- Set `ignore_older` to avoid processing old files
- Use `close_older` to free file handles
- Maintain the sincedb file across Logstash restarts
- Add file metadata using the `add_field` option

## Beats Input

The Beats input plugin receives events from the Elastic Beats framework, which includes Filebeat, Metricbeat, Packetbeat, and others.

### Basic Configuration

```ruby
input {
  beats {
    port => 5044
    host => "0.0.0.0"
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `port` | Port to listen on | Required | `5044` |
| `host` | Host to bind to | "0.0.0.0" | `"10.0.0.1"` |
| `client_inactivity_timeout` | Client timeout in seconds | 60 | `3600` |
| `include_codec_tag` | Include codec details in tags | true | `false` |
| `ssl` | Enable SSL/TLS | false | `true` |
| `ssl_certificate` | SSL certificate path | None | `"/path/to/ssl.crt"` |
| `ssl_key` | SSL key path | None | `"/path/to/ssl.key"` |
| `ssl_certificate_authorities` | SSL CA certificate paths | None | `["/path/to/ca.crt"]` |
| `ssl_verify_mode` | SSL verification mode | "none" | `"force_peer"` |
| `ssl_client_authentication` | Require client auth | "optional" | `"required"` |

### TLS/SSL Configuration

For secure communication:

```ruby
input {
  beats {
    port => 5044
    ssl => true
    ssl_certificate => "/etc/logstash/certificates/logstash.crt"
    ssl_key => "/etc/logstash/certificates/logstash.key"
    ssl_certificate_authorities => ["/etc/logstash/certificates/ca.crt"]
    ssl_verify_mode => "force_peer"
    ssl_client_authentication => "required"
  }
}
```

### Scaling Beats Input

For handling many Beats clients:

```ruby
input {
  beats {
    port => 5044
    client_inactivity_timeout => 3600
    congestion_threshold => 40
    target_field_for_codec => "message"
    add_field => {
      "[@metadata][input]" => "beats"
    }
  }
}
```

### Best Practices for Beats Input

- Enable TLS for production deployments
- Set a reasonable `client_inactivity_timeout` for long-running connections
- Consider running multiple Logstash instances for load balancing
- Use metadata to track the source of events
- Implement monitoring for the Beats input

## HTTP Input

The HTTP input plugin receives events over HTTP(S), creating a web service endpoint.

### Basic Configuration

```ruby
input {
  http {
    host => "0.0.0.0"
    port => 8080
    codec => "json"
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `port` | Port to listen on | 8080 | `8888` |
| `host` | Host to bind to | "0.0.0.0" | `"127.0.0.1"` |
| `response_code` | HTTP response code | 200 | `202` |
| `response_headers` | Custom response headers | None | `{"Content-Type" => "text/plain"}` |
| `codec` | Codec to use | "plain" | `"json"` |
| `ssl` | Enable SSL/TLS | false | `true` |
| `ssl_certificate` | SSL certificate path | None | `"/path/to/ssl.crt"` |
| `ssl_key` | SSL key path | None | `"/path/to/ssl.key"` |
| `ssl_verify_mode` | SSL verification mode | "none" | `"peer"` |
| `user` | Basic auth username | None | `"logstash"` |
| `password` | Basic auth password | None | `"password"` |

### Secure HTTP Endpoint

For secure HTTP input:

```ruby
input {
  http {
    host => "0.0.0.0"
    port => 8443
    codec => "json"
    ssl => true
    ssl_certificate => "/etc/logstash/certificates/logstash.crt" 
    ssl_key => "/etc/logstash/certificates/logstash.key"
    ssl_verify_mode => "none"
    user => "webhook"
    password => "complex_password"
  }
}
```

### HTTP Input with Custom Response

Customize the HTTP response:

```ruby
input {
  http {
    port => 8080
    response_code => 202
    response_headers => {
      "Content-Type" => "application/json"
      "X-Processed-By" => "Logstash"
    }
    additional_codecs => {
      "application/json" => "json"
      "application/xml" => "xml"
      "text/plain" => "plain"
    }
  }
}
```

### HTTP Input Use Cases

- Webhook endpoints (GitHub, GitLab, JIRA, etc.)
- Custom application logs via HTTP
- Simple API endpoints
- Integration with services that support HTTP output
- Health checks and monitoring endpoints

### Best Practices for HTTP Input

- Always secure production endpoints with HTTPS
- Implement authentication for sensitive endpoints
- Use JSON codec for structured data
- Set up a proxy in front of Logstash for production deployments
- Monitor endpoint availability and performance

## TCP/UDP Input

The TCP and UDP input plugins receive events over TCP or UDP sockets.

### TCP Input Configuration

```ruby
input {
  tcp {
    port => 5000
    host => "0.0.0.0"
    codec => "json_lines"
  }
}
```

### UDP Input Configuration

```ruby
input {
  udp {
    port => 5000
    host => "0.0.0.0"
    codec => "json"
    buffer_size => 65536
  }
}
```

### Key Options for TCP Input

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `port` | Port to listen on | Required | `5000` |
| `host` | Host to bind to | "0.0.0.0" | `"10.0.0.1"` |
| `codec` | Codec to use | "line" | `"json_lines"` |
| `ssl_enable` | Enable SSL/TLS | false | `true` |
| `ssl_cert` | SSL certificate path | None | `"/path/to/ssl.crt"` |
| `ssl_key` | SSL key path | None | `"/path/to/ssl.key"` |
| `ssl_verify` | Verify SSL | true | `false` |
| `tcp_keep_alive` | Enable TCP keepalive | false | `true` |
| `mode` | Mode (server or client) | "server" | `"client"` |

### Key Options for UDP Input

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `port` | Port to listen on | Required | `5000` |
| `host` | Host to bind to | "0.0.0.0" | `"10.0.0.1"` |
| `codec` | Codec to use | "plain" | `"json"` |
| `buffer_size` | UDP buffer size | 65536 | `131072` |
| `workers` | Number of UDP receiver threads | 2 | `4` |
| `queue_size` | Queue size for worker threads | 2000 | `4000` |
| `receive_buffer_bytes` | Socket receive buffer size | 65536 | `1048576` |

### Secure TCP Input with TLS

```ruby
input {
  tcp {
    port => 5000
    ssl_enable => true
    ssl_cert => "/etc/logstash/certificates/logstash.crt"
    ssl_key => "/etc/logstash/certificates/logstash.key"
    ssl_verify => false
  }
}
```

### TCP/UDP Input Use Cases

- Syslog collection
- Legacy application logs
- Network device logs
- Custom application integrations
- High-throughput logging scenarios

### Best Practices for TCP/UDP Input

- Use TCP for reliable delivery, UDP for high-volume logs
- Choose appropriate codecs based on data format
- Enable keepalive for long-lived TCP connections
- Set proper buffer sizes for high-throughput scenarios
- Enable TLS for TCP connections in production
- Consider implementing load balancing for high-volume deployments

## Syslog Input

The Syslog input plugin specifically handles syslog messages.

### Basic Configuration

```ruby
input {
  syslog {
    port => 514
    type => "syslog"
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `port` | Port to listen on | 514 | `5514` |
| `host` | Host to bind to | "0.0.0.0" | `"10.0.0.1"` |
| `protocol` | Protocol (tcp or udp) | "udp" | `"tcp"` |
| `mode` | Mode (server or client) | "server" | `"client"` |
| `timezone` | Timezone for parsing timestamps | "UTC" | `"America/Los_Angeles"` |
| `locale` | Locale for parsing timestamps | "en" | `"fr"` |
| `syslog_field` | Field to parse as syslog | "message" | `"syslog_message"` |
| `use_labels` | Use RFC5424 labels for facility/severity | false | `true` |
| `ssl_enable` | Enable SSL/TLS | false | `true` |

### RFC3164 Syslog Configuration

For traditional BSD syslog format:

```ruby
input {
  syslog {
    port => 5514
    protocol => "tcp"
    type => "syslog-rfc3164"
    timezone => "America/New_York"
    grok_pattern => "<%{POSINT:priority}>%{SYSLOGTIMESTAMP:timestamp} %{SYSLOGHOST:hostname} %{DATA:program}(?:\\[%{POSINT:pid}\\])?: %{GREEDYDATA:message}"
  }
}
```

### RFC5424 Syslog Configuration

For modern structured syslog format:

```ruby
input {
  syslog {
    port => 5614
    protocol => "tcp"
    type => "syslog-rfc5424"
    syslog_field => "message"
    use_labels => true
    proxy_protocol => true
  }
}
```

### Best Practices for Syslog Input

- Set the correct timezone for accurate timestamp parsing
- Use TCP for reliable delivery when possible
- Implement load balancing for high-volume environments
- Consider using the Grok filter for custom syslog formats
- Map facility and severity numbers to human-readable values
- Enable TLS for secure transport in production

## Kafka Input

The Kafka input plugin reads events from Apache Kafka topics.

### Basic Configuration

```ruby
input {
  kafka {
    bootstrap_servers => "kafka1:9092,kafka2:9092,kafka3:9092"
    topics => ["logs", "metrics"]
    group_id => "logstash"
    codec => "json"
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `bootstrap_servers` | Kafka brokers | Required | `"kafka1:9092,kafka2:9092"` |
| `topics` | Kafka topics to consume | Required | `["logs", "metrics"]` |
| `group_id` | Consumer group ID | "logstash" | `"logstash_prod"` |
| `client_id` | Client ID | "logstash" | `"logstash_kafka_client"` |
| `consumer_threads` | Number of consumer threads | 1 | `4` |
| `auto_offset_reset` | Offset reset policy | "latest" | `"earliest"` |
| `max_poll_records` | Max records per poll | 500 | `1000` |
| `session_timeout_ms` | Consumer session timeout | 30000 | `60000` |
| `fetch_max_wait_ms` | Max wait time for fetch | 500 | `1000` |
| `ssl_truststore_location` | SSL truststore path | None | `"/path/to/truststore.jks"` |
| `ssl_truststore_password` | SSL truststore password | None | `"password"` |
| `sasl_mechanism` | SASL mechanism | None | `"PLAIN"` |
| `sasl_jaas_config` | JAAS configuration | None | `"org.apache.kafka.common.security.plain.PlainLoginModule required username=\"user\" password=\"pass\";"` |
| `security_protocol` | Security protocol | "PLAINTEXT" | `"SASL_SSL"` |
| `key_deserializer_class` | Key deserializer | "org.apache.kafka.common.serialization.StringDeserializer" | `"..."` |
| `value_deserializer_class` | Value deserializer | "org.apache.kafka.common.serialization.StringDeserializer" | `"..."` |

### Kafka Input with Security

For secure Kafka connections:

```ruby
input {
  kafka {
    bootstrap_servers => "kafka1:9092,kafka2:9092"
    topics => ["secure-logs"]
    security_protocol => "SASL_SSL"
    sasl_mechanism => "PLAIN"
    sasl_jaas_config => "org.apache.kafka.common.security.plain.PlainLoginModule required username=\"logstash\" password=\"${KAFKA_PASSWORD}\";"
    ssl_truststore_location => "/etc/logstash/kafka.truststore.jks"
    ssl_truststore_password => "${TRUSTSTORE_PASSWORD}"
  }
}
```

### Kafka Input with Dynamic Topics

To consume from dynamically matching topics:

```ruby
input {
  kafka {
    bootstrap_servers => "kafka1:9092,kafka2:9092"
    topics_pattern => "logs-.*"
    decorate_events => true
    codec => "json"
    consumer_threads => 4
    auto_offset_reset => "earliest"
  }
}
```

### Best Practices for Kafka Input

- Use multiple consumer threads for parallel processing
- Set appropriate batch sizes and timeouts for your workload
- Configure a dedicated consumer group for each Logstash pipeline
- Enable event decoration to retain Kafka metadata
- Set up monitoring for consumer lag
- Use proper security settings for production

## JDBC Input

The JDBC input plugin allows Logstash to retrieve data from JDBC-compatible databases.

### Basic Configuration

```ruby
input {
  jdbc {
    jdbc_driver_library => "/path/to/mysql-connector-java-8.0.23.jar"
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://localhost:3306/mydb"
    jdbc_user => "mysql"
    jdbc_password => "password"
    statement => "SELECT * FROM users WHERE updated_at > :sql_last_value"
    use_column_value => true
    tracking_column => "updated_at"
    tracking_column_type => "timestamp"
    schedule => "*/5 * * * *"
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `jdbc_driver_library` | Path to JDBC driver | Required | `"/path/to/driver.jar"` |
| `jdbc_driver_class` | JDBC driver class | Required | `"org.postgresql.Driver"` |
| `jdbc_connection_string` | JDBC connection URL | Required | `"jdbc:postgresql://localhost:5432/db"` |
| `jdbc_user` | Database username | None | `"postgres"` |
| `jdbc_password` | Database password | None | `"password"` |
| `statement` | SQL query | Required | `"SELECT * FROM table"` |
| `schedule` | Schedule in cron format | None | `"0 */2 * * *"` |
| `tracking_column` | Column to track for incremental imports | None | `"id"` |
| `tracking_column_type` | Tracking column type | "numeric" | `"timestamp"` |
| `use_column_value` | Use column value for tracking | false | `true` |
| `record_last_run` | Record last run time/value | true | `false` |
| `clean_run` | Start with clean state | false | `true` |
| `last_run_metadata_path` | Path to store last run metadata | "$HOME/.logstash_jdbc_last_run" | `"/path/to/metadata"` |
| `jdbc_paging_enabled` | Enable result paging | false | `true` |
| `jdbc_page_size` | Page size when paging | 100000 | `10000` |
| `jdbc_validate_connection` | Validate connection on checkout | false | `true` |
| `jdbc_validation_timeout` | Validation timeout | 3600 | `300` |

### Incremental Imports

For incremental data loading:

```ruby
input {
  jdbc {
    jdbc_driver_library => "/path/to/postgresql-42.2.19.jar"
    jdbc_driver_class => "org.postgresql.Driver"
    jdbc_connection_string => "jdbc:postgresql://localhost:5432/app_db"
    jdbc_user => "logstash"
    jdbc_password => "${DB_PASSWORD}"
    jdbc_paging_enabled => true
    jdbc_page_size => 50000
    statement => "SELECT id, name, email, created_at, updated_at FROM users WHERE updated_at > :sql_last_value ORDER BY updated_at ASC"
    use_column_value => true
    tracking_column => "updated_at"
    tracking_column_type => "timestamp"
    schedule => "*/10 * * * *"
    last_run_metadata_path => "/var/logstash/jdbc_last_run_users"
  }
}
```

### JDBC with Parameters

To use parameters in your SQL query:

```ruby
input {
  jdbc {
    jdbc_driver_library => "/path/to/driver.jar"
    jdbc_driver_class => "oracle.jdbc.OracleDriver"
    jdbc_connection_string => "jdbc:oracle:thin:@localhost:1521:orcl"
    jdbc_user => "system"
    jdbc_password => "password"
    parameters => { "department_id" => 42 }
    statement => "SELECT * FROM employees WHERE department_id = :department_id AND hire_date > :sql_last_value"
    schedule => "0 * * * *"
  }
}
```

### JDBC Input Use Cases

- Database data ingestion into Elasticsearch
- Synchronizing database data with Elasticsearch
- Combining database data with logs for enhanced analytics
- ETL workflows with Logstash
- Alerting based on database data

### Best Practices for JDBC Input

- Use incremental loading for large tables
- Create optimized SQL queries with proper indexes
- Set a reasonable schedule to avoid database load
- Use connection pooling for efficiency
- Implement column tracking for incremental imports
- Secure database credentials using Logstash keystore
- Enable paging for large result sets

## Redis Input

The Redis input plugin reads events from Redis lists, channels, or patterns.

### Basic Configuration

```ruby
input {
  redis {
    host => "redis.example.com"
    port => 6379
    data_type => "list"
    key => "logstash:events"
    codec => "json"
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `host` | Redis host | "localhost" | `"redis.example.com"` |
| `port` | Redis port | 6379 | `6380` |
| `db` | Redis database number | 0 | `2` |
| `password` | Redis password | None | `"redis_pass"` |
| `data_type` | Redis data type (list, channel, pattern_channel) | "list" | `"channel"` |
| `key` | Redis key for list or channel | "logstash" | `"logstash:events"` |
| `batch_count` | Number of events to fetch | 125 | `250` |
| `ssl` | Use SSL | false | `true` |
| `threads` | Number of threads | 1 | `2` |
| `timeout` | Redis connection timeout | 5 | `10` |

### Redis List Consumer

To consume from a Redis list:

```ruby
input {
  redis {
    host => "redis.example.com"
    port => 6379
    password => "${REDIS_PASSWORD}"
    data_type => "list"
    key => "app:logs"
    batch_count => 100
    threads => 2
  }
}
```

### Redis Pub/Sub Consumer

To consume from a Redis pub/sub channel:

```ruby
input {
  redis {
    host => "redis.example.com"
    port => 6379
    data_type => "channel"
    key => "logs-channel"
  }
}
```

### Redis Pattern Channel

To consume from multiple channels matching a pattern:

```ruby
input {
  redis {
    host => "redis.example.com"
    port => 6379
    data_type => "pattern_channel"
    key => "logs-*"
  }
}
```

### Redis Input with Sentinel

For Redis Sentinel configuration:

```ruby
input {
  redis {
    host => ["sentinel1", "sentinel2", "sentinel3"]
    port => 26379
    sentinel => true
    sentinel_master => "mymaster"
    data_type => "list"
    key => "logstash:events"
  }
}
```

### Best Practices for Redis Input

- Choose the right data type for your use case
- Set appropriate batch sizes for optimal throughput
- Enable persistence on the Redis server
- Use a dedicated Redis database for Logstash
- Monitor Redis memory usage and performance
- Implement high availability with Redis Sentinel or Redis Cluster
- Use SSL for secure communication in production

## Amazon S3 Input

The S3 input plugin retrieves logs from Amazon S3 buckets.

### Basic Configuration

```ruby
input {
  s3 {
    bucket => "my-logs-bucket"
    region => "us-east-1"
    prefix => "logs/"
    codec => "json_lines"
    interval => 60
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `bucket` | S3 bucket name | Required | `"logs-bucket"` |
| `region` | AWS region | "us-east-1" | `"eu-west-1"` |
| `prefix` | Key prefix for filtering | None | `"logs/2023/"` |
| `additional_settings` | Additional AWS settings | {} | `{"force_path_style" => true}` |
| `interval` | Polling interval in seconds | 60 | `300` |
| `sincedb_path` | Path to track processed files | None | `"/path/to/sincedb"` |
| `delete` | Delete objects after processing | false | `true` |
| `access_key_id` | AWS access key | None | `"AKIAIOSFODNN7EXAMPLE"` |
| `secret_access_key` | AWS secret key | None | `"wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"` |
| `session_token` | AWS session token | None | `"TOKEN"` |
| `role_arn` | AWS IAM role to assume | None | `"arn:aws:iam::123456789012:role/LogstashRole"` |
| `proxy_uri` | HTTP proxy URI | None | `"http://proxy:8080"` |
| `backup_to_bucket` | Backup bucket name | None | `"backup-bucket"` |
| `backup_add_prefix` | Prefix to add to backup objects | None | `"processed/"` |

### S3 Input with AWS Authentication

For secure S3 access:

```ruby
input {
  s3 {
    bucket => "application-logs"
    region => "us-west-2"
    prefix => "app1/logs/"
    codec => "json_lines"
    
    # Authentication method 1: Access keys
    access_key_id => "${AWS_ACCESS_KEY_ID}"
    secret_access_key => "${AWS_SECRET_ACCESS_KEY}"
    
    # OR Authentication method 2: IAM role
    # role_arn => "arn:aws:iam::123456789012:role/LogstashS3Role"
    
    # Processing settings
    interval => 300
    sincedb_path => "/var/lib/logstash/s3_sincedb"
    delete => false
    backup_to_bucket => "processed-logs"
    backup_add_prefix => "processed/"
  }
}
```

### S3 with SQS Notification

To process S3 objects based on SQS notifications:

```ruby
input {
  sqs {
    queue => "s3-notification-queue"
    region => "us-east-1"
    codec => "json"
    polling_frequency => 20
    attribute_names => ["All"]
  }
}

filter {
  ruby {
    code => '
      if event.get("[Records][0][eventSource]") == "aws:s3"
        bucket = event.get("[Records][0][s3][bucket][name]")
        key = event.get("[Records][0][s3][object][key]")
        event.set("[@metadata][s3_bucket]", bucket)
        event.set("[@metadata][s3_key]", key)
      end
    '
  }
}

filter {
  if [@metadata][s3_bucket] and [@metadata][s3_key] {
    s3 {
      bucket => "%{[@metadata][s3_bucket]}"
      key => "%{[@metadata][s3_key]}"
      region => "us-east-1"
      codec => "json_lines"
    }
  }
}
```

### Best Practices for S3 Input

- Use a dedicated IAM role with least privilege
- Store credentials in the Logstash keystore
- Set appropriate polling intervals to reduce API calls
- Use SQS notifications for near-real-time processing
- Configure sincedb properly to avoid reprocessing files
- Apply a proper prefix to limit the scope of processing
- Consider using compression for large log files
- Monitor AWS costs related to S3 operations

## Azure Event Hub Input

The Azure Event Hub input plugin consumes events from Azure Event Hubs.

### Basic Configuration

```ruby
input {
  azure_event_hubs {
    event_hub_connections => [
      "Endpoint=sb://namespace.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=key;EntityPath=event_hub_name"
    ]
    threads => 4
    decorate_events => true
    consumer_group => "$Default"
    storage_connection => "DefaultEndpointsProtocol=https;AccountName=example;AccountKey=key;EndpointSuffix=core.windows.net"
    storage_container => "logstash"
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `event_hub_connections` | Event Hub connection strings | Required | `["Endpoint=..."]` |
| `threads` | Number of threads | 4 | `8` |
| `decorate_events` | Add Event Hub metadata | false | `true` |
| `consumer_group` | Event Hub consumer group | "$Default" | `"logstash"` |
| `storage_connection` | Azure Storage connection string | Required | `"DefaultEndpoints..."` |
| `storage_container` | Storage container for checkpoints | Required | `"eventhub-checkpoints"` |
| `max_batch_size` | Maximum batch size | 10 | `50` |
| `prefetch_count` | Number of events to prefetch | 300 | `500` |
| `initial_position` | Initial position (beginning or end) | "end" | `"beginning"` |

### Event Hub Input with Managed Identity

Using Azure Managed Identity for authentication:

```ruby
input {
  azure_event_hubs {
    event_hub_connections => [
      "Endpoint=sb://namespace.servicebus.windows.net/;EntityPath=event_hub_name"
    ]
    use_managed_identity => true
    managed_identity_client_id => "client-id"  # Optional, only for user-assigned identity
    storage_connection => "DefaultEndpointsProtocol=https;AccountName=example;EndpointSuffix=core.windows.net"
    use_storage_managed_identity => true
    storage_container => "logstash"
    consumer_group => "logstash"
    initial_position => "beginning"
    decorate_events => true
  }
}
```

### Multiple Event Hubs

To consume from multiple Event Hubs:

```ruby
input {
  azure_event_hubs {
    event_hub_connections => [
      "Endpoint=sb://namespace1.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=key1;EntityPath=event_hub1",
      "Endpoint=sb://namespace2.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=key2;EntityPath=event_hub2"
    ]
    threads => 8
    decorate_events => true
    consumer_group => "$Default"
    storage_connection => "DefaultEndpointsProtocol=https;AccountName=example;AccountKey=key;EndpointSuffix=core.windows.net"
    storage_container => "logstash"
    add_field => { "environment" => "production" }
  }
}
```

### Best Practices for Azure Event Hub Input

- Use a dedicated consumer group for Logstash
- Set appropriate thread count based on Event Hub partitions
- Use managed identities in production when possible
- Configure proper checkpoint storage for reliable processing
- Add Event Hub metadata with decorate_events
- Monitor throughput and adjust configuration accordingly
- Consider using compression for large messages

## Google Cloud Pub/Sub Input

The Google Cloud Pub/Sub input plugin consumes messages from Google Cloud Pub/Sub topics.

### Basic Configuration

```ruby
input {
  google_pubsub {
    project_id => "my-project"
    topic => "logstash-input-topic"
    subscription => "logstash-subscription"
    json_key_file => "/path/to/key.json"
    codec => "json"
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `project_id` | GCP project ID | Required | `"my-gcp-project"` |
| `topic` | Pub/Sub topic name | Required | `"logs-topic"` |
| `subscription` | Pub/Sub subscription name | Required | `"logstash-sub"` |
| `json_key_file` | GCP service account key file | None | `"/path/to/key.json"` |
| `max_messages` | Maximum messages per request | 5 | `10` |
| `auto_create_resources` | Create topic and subscription | false | `true` |
| `create_subscription` | Create subscription only | false | `true` |
| `delay_threshold_secs` | Deadline extension threshold | None | `60` |
| `message_count_threshold` | Maximum messages for batching | None | `1000` |
| `request_byte_threshold` | Maximum bytes for batching | None | `10485760` |
| `include_metadata` | Include Pub/Sub metadata | false | `true` |

### Google Pub/Sub with Authentication

Using a service account for authentication:

```ruby
input {
  google_pubsub {
    project_id => "my-logging-project"
    topic => "application-logs"
    subscription => "logstash-consumer"
    json_key_file => "/etc/logstash/gcp-credentials.json"
    codec => "json"
    max_messages => 100
    include_metadata => true
    add_field => {
      "[@metadata][input]" => "google_pubsub"
      "source" => "application-logs"
    }
  }
}
```

### Auto-Creating Resources

To automatically create the subscription:

```ruby
input {
  google_pubsub {
    project_id => "my-project"
    topic => "logs-topic"
    subscription => "logstash-subscription"
    create_subscription => true
    subscription_type => "pull"
    include_metadata => true
    max_messages => 100
  }
}
```

### Best Practices for Google Pub/Sub Input

- Use a dedicated service account with minimal permissions
- Store credentials securely using the Logstash keystore
- Set appropriate batch sizes for optimal throughput
- Create subscriptions with proper retention and ack deadlines
- Include metadata for debugging and tracking
- Monitor subscription lag to ensure real-time processing
- Configure proper error handling and dead-letter topics

## RabbitMQ Input

The RabbitMQ input plugin consumes messages from RabbitMQ exchanges.

### Basic Configuration

```ruby
input {
  rabbitmq {
    host => "rabbitmq.example.com"
    port => 5672
    vhost => "/"
    exchange => "logs"
    queue => "logstash_queue"
    durable => true
    codec => "json"
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `host` | RabbitMQ host | "localhost" | `"rabbitmq.example.com"` |
| `port` | RabbitMQ port | 5672 | `5671` |
| `vhost` | RabbitMQ virtual host | "/" | `"/logs"` |
| `user` | RabbitMQ username | "guest" | `"logstash"` |
| `password` | RabbitMQ password | "guest" | `"password"` |
| `exchange` | RabbitMQ exchange name | None | `"logs_exchange"` |
| `exchange_type` | Exchange type | "fanout" | `"topic"` |
| `queue` | Queue name | None | `"logstash_queue"` |
| `key` | Routing key | None | `"logs.#"` |
| `durable` | Durable queue | true | `false` |
| `ack` | Enable acknowledgements | true | `false` |
| `auto_delete` | Auto-delete queue | false | `true` |
| `exclusive` | Exclusive queue | false | `true` |
| `prefetch_count` | Prefetch count | 256 | `100` |
| `threads` | Number of consumer threads | 1 | `4` |
| `ssl` | Enable SSL | false | `true` |
| `ssl_certificate` | SSL certificate path | None | `"/path/to/cert.pem"` |
| `ssl_key` | SSL key path | None | `"/path/to/key.pem"` |
| `ssl_certificate_authorities` | SSL CA certificates | None | `["/path/to/ca.pem"]` |

### RabbitMQ with SSL/TLS

For secure RabbitMQ connections:

```ruby
input {
  rabbitmq {
    host => "rabbitmq.example.com"
    port => 5671
    ssl => true
    ssl_version => "TLSv1.2"
    ssl_certificate => "/etc/logstash/certificates/client.pem"
    ssl_key => "/etc/logstash/certificates/client.key"
    ssl_certificate_authorities => ["/etc/logstash/certificates/ca.pem"]
    verify_ssl => true
    
    vhost => "/logs"
    user => "logstash"
    password => "${RABBITMQ_PASSWORD}"
    exchange => "logs"
    queue => "logstash_queue"
    exchange_type => "topic"
    key => "application.logs.#"
    durable => true
    prefetch_count => 100
    threads => 2
  }
}
```

### Multiple Consumers

To set up multiple consumers:

```ruby
input {
  rabbitmq {
    host => "rabbitmq.example.com"
    vhost => "/logs"
    exchange => "logs"
    queue => "logstash_queue"
    threads => 4
    prefetch_count => 250
    ack => true
    durable => true
    heartbeat => 30
    metadata_enabled => true
    add_field => {
      "[@metadata][input]" => "rabbitmq"
    }
  }
}
```

### Best Practices for RabbitMQ Input

- Use durable exchanges and queues for reliability
- Enable message acknowledgements
- Set appropriate prefetch counts to balance throughput and memory usage
- Use SSL/TLS for secure connections
- Implement proper queue naming conventions
- Set up proper dead-letter exchanges for failed messages
- Monitor queue depths and consumer lag
- Use multiple consumer threads for parallel processing

## Conclusion

Input plugins are the gateway for data into your Logstash pipeline. By understanding the capabilities and configuration options of these plugins, you can effectively collect data from a wide variety of sources.

When choosing input plugins for your use case, consider:

1. **Data source characteristics**: File-based logs, network protocols, messaging systems, etc.
2. **Volume and velocity**: How much data and how quickly it arrives
3. **Reliability requirements**: Whether data loss is acceptable
4. **Security considerations**: Authentication, encryption, and access control
5. **Performance impact**: Resources required to collect and process the data

The next chapter will cover filter plugins, which allow you to transform, enrich, and structure your data after it has been collected.