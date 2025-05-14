# Logstash Fundamentals

## Introduction to Logstash

Logstash is a powerful, open-source data processing pipeline that ingests, transforms, and sends data to various destinations. As the "L" in the ELK Stack, Logstash plays a critical role in collecting, normalizing, and enriching logs and events from diverse sources before they're indexed in Elasticsearch.

### Core Functions of Logstash

1. **Data Ingestion**: Collect data from multiple sources simultaneously
2. **Data Transformation**: Parse, modify, and enrich data in transit
3. **Data Output**: Send processed data to one or more destinations

### Key Features

- **Pluggable Pipeline**: Over 200 plugins for inputs, filters, and outputs
- **Extensibility**: Create custom plugins using Ruby
- **Scalability**: Scale horizontally to handle increased load
- **Reliability**: Buffer data to handle load spikes and outages
- **Security**: Support for encrypted communications and authentication

## Logstash Architecture

### Pipeline Architecture

Logstash processes data through a pipeline consisting of three stages:

![Logstash Pipeline](https://i.imgur.com/WzkFWqy.png)

1. **Inputs**: Collect data from sources
2. **Filters**: Transform and enrich data
3. **Outputs**: Send data to destinations

Data flows through this pipeline sequentially, with multiple inputs feeding into multiple filters, which then feed into multiple outputs.

### Event Processing

Logstash processes data as **events**. Each event:
- Represents a single unit of data (e.g., a log line, a metric reading)
- Is stored as a JSON-like object with fields and values
- Has metadata associated with it (timestamps, pipeline information)
- Can be modified at any stage of the pipeline

Example event:
```json
{
  "@timestamp": "2023-05-15T09:43:27.123Z",
  "@version": "1",
  "host": {
    "name": "webserver1"
  },
  "message": "GET /index.html 200",
  "client_ip": "192.168.1.10",
  "request": "/index.html",
  "status": 200
}
```

### Execution Model

Logstash uses a multi-threaded architecture:

- **Pipeline Workers**: Process events in parallel
- **Batch Processing**: Group events for efficient handling
- **Persistent Queues**: Store events on disk for reliability

## Installing Logstash

### System Requirements

Minimum requirements:
- Java 8 or later (Java 11 recommended)
- 2GB RAM (4GB+ recommended)
- 1 CPU core (2+ recommended)
- 1GB free disk space

### Installation Methods

#### Package Repositories

**Debian/Ubuntu**:
```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
sudo apt-get install apt-transport-https
echo "deb https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-8.x.list
sudo apt-get update && sudo apt-get install logstash
```

**RHEL/CentOS**:
```bash
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
cat << EOF | sudo tee /etc/yum.repos.d/elasticsearch.repo
[elasticsearch]
name=Elasticsearch repository for 8.x packages
baseurl=https://artifacts.elastic.co/packages/8.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF
sudo yum install logstash
```

#### Archive Installation

Download and extract the archive:
```bash
curl -L -O https://artifacts.elastic.co/downloads/logstash/logstash-8.8.0-linux-x86_64.tar.gz
tar -xzvf logstash-8.8.0-linux-x86_64.tar.gz
cd logstash-8.8.0
```

#### Docker

```bash
docker pull docker.elastic.co/logstash/logstash:8.8.0
docker run -it --rm -v $(pwd)/pipeline/:/usr/share/logstash/pipeline/ docker.elastic.co/logstash/logstash:8.8.0
```

### Directory Structure

After installation, you'll find these important directories:

- `bin/`: Executable files including `logstash` command
- `config/`: Configuration files
  - `logstash.yml`: Main configuration
  - `pipelines.yml`: Pipeline configuration
  - `jvm.options`: JVM settings
- `data/`: Data files including persistent queues
- `logs/`: Log files
- `plugins/`: Custom plugins

## Basic Configuration

### Configuration Files

Logstash uses several configuration files:

1. **logstash.yml**: System configuration (settings, paths, etc.)
2. **pipelines.yml**: Pipeline configuration
3. **jvm.options**: JVM settings
4. **log4j2.properties**: Logging configuration
5. **startup.options**: Startup parameters (for system installs)

### Pipeline Configuration

Pipeline configurations define how Logstash processes events. These can be specified in:

- A single configuration file
- Multiple configuration files in a directory
- The `pipelines.yml` file for multiple pipelines

Pipeline configuration follows this structure:

```
input {
  # Input plugins configuration
}

filter {
  # Filter plugins configuration
}

output {
  # Output plugins configuration
}
```

### Hello World Example

Let's create a simple pipeline that receives input from the console, adds a timestamp, and outputs back to the console:

```ruby
input {
  stdin {}
}

filter {
  mutate {
    add_field => { "greeting" => "Hello, %{message}!" }
  }
}

output {
  stdout {
    codec => rubydebug
  }
}
```

Save this to a file named `hello.conf` and run:

```bash
bin/logstash -f hello.conf
```

Type a name and press Enter. Logstash will respond with:

```
{
      "@version" => "1",
       "message" => "World",
    "@timestamp" => 2023-05-15T09:50:12.123Z,
          "host" => "your-hostname",
      "greeting" => "Hello, World!"
}
```

## Input Plugins

Input plugins enable Logstash to receive data from different sources. Some common input plugins include:

### File Input

Reads from files, similar to `tail -f`:

```ruby
input {
  file {
    path => ["/var/log/messages", "/var/log/*.log"]
    start_position => "beginning"
    sincedb_path => "/var/lib/logstash/sincedb"
    type => "syslog"
  }
}
```

### Beats Input

Receives data from Beats data shippers:

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

### HTTP Input

Receives data via HTTP(S) requests:

```ruby
input {
  http {
    host => "0.0.0.0"
    port => 8080
    codec => "json"
  }
}
```

### TCP/UDP Input

Listens for messages on TCP or UDP ports:

```ruby
input {
  tcp {
    port => 5000
    codec => json_lines
  }
  
  udp {
    port => 5001
    codec => json
  }
}
```

### Kafka Input

Consumes messages from Kafka topics:

```ruby
input {
  kafka {
    bootstrap_servers => "kafka1:9092,kafka2:9092"
    topics => ["logs"]
    consumer_threads => 3
    group_id => "logstash_group"
    codec => "json"
  }
}
```

### JDBC Input

Polls a database via JDBC:

```ruby
input {
  jdbc {
    jdbc_driver_library => "/path/to/mysql-connector-java-8.0.20.jar"
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://localhost:3306/mydatabase"
    jdbc_user => "mysql"
    jdbc_password => "password"
    schedule => "*/5 * * * *"
    statement => "SELECT * FROM users WHERE updated_at > :sql_last_value"
    use_column_value => true
    tracking_column => "updated_at"
    tracking_column_type => "timestamp"
  }
}
```

### S3 Input

Pulls data from Amazon S3 buckets:

```ruby
input {
  s3 {
    bucket => "my-bucket"
    region => "us-east-1"
    prefix => "logs/"
    interval => 60
    codec => "json_lines"
  }
}
```

## Filter Plugins

Filter plugins process and transform events as they pass through the Logstash pipeline.

### Grok Filter

Parses unstructured data into structured fields:

```ruby
filter {
  grok {
    match => { "message" => "%{IP:client} %{WORD:method} %{URIPATHPARAM:request} %{NUMBER:status}" }
  }
}
```

For a message like `192.168.0.1 GET /index.html 200`, this creates fields:
- `client`: "192.168.0.1"
- `method`: "GET"
- `request`: "/index.html"
- `status`: "200"

### Mutate Filter

Performs general transformations on fields:

```ruby
filter {
  mutate {
    convert => { "status" => "integer" }
    rename => { "client" => "client_ip" }
    uppercase => [ "method" ]
    strip => [ "request" ]
    add_field => { "processed_by" => "logstash" }
    remove_field => [ "unwanted_field" ]
  }
}
```

### Date Filter

Parses date strings and uses them for the timestamp:

```ruby
filter {
  date {
    match => [ "timestamp", "ISO8601", "MMM dd yyyy HH:mm:ss", "UNIX" ]
    target => "@timestamp"
    timezone => "UTC"
  }
}
```

### JSON Filter

Parses JSON strings into structured data:

```ruby
filter {
  json {
    source => "message"
    target => "parsed_message"
  }
}
```

### GeoIP Filter

Adds location information based on IP addresses:

```ruby
filter {
  geoip {
    source => "client_ip"
    target => "geoip"
    database => "/path/to/GeoLite2-City.mmdb"
    add_field => { "location" => "%{[geoip][city_name]}, %{[geoip][country_name]}" }
  }
}
```

### Key-Value Filter

Extracts key-value pairs from a field:

```ruby
filter {
  kv {
    source => "message"
    field_split => "&"
    value_split => "="
    target => "params"
  }
}
```

For a message like `user=john&action=login`, this creates:
- `params.user`: "john"
- `params.action`: "login"

### Ruby Filter

Executes custom Ruby code on events:

```ruby
filter {
  ruby {
    code => '
      event.set("environment", 
        case event.get("hostname")
        when /^dev/ then "development"
        when /^prod/ then "production"
        else "unknown"
        end
      )
    '
  }
}
```

## Output Plugins

Output plugins send processed events to various destinations.

### Elasticsearch Output

Sends events to Elasticsearch:

```ruby
output {
  elasticsearch {
    hosts => ["http://es1:9200", "http://es2:9200"]
    index => "logs-%{+YYYY.MM.dd}"
    document_id => "%{id}"
    user => "elastic"
    password => "changeme"
    ssl => true
    cacert => "/path/to/ca.crt"
  }
}
```

### File Output

Writes events to files:

```ruby
output {
  file {
    path => "/var/log/logstash/%{type}/%{+YYYY/MM/dd/HH}/%{host}.log"
    codec => line { format => "%{message}" }
    gzip => true
  }
}
```

### Kafka Output

Sends events to Kafka topics:

```ruby
output {
  kafka {
    bootstrap_servers => "kafka1:9092,kafka2:9092"
    topic_id => "processed_logs"
    codec => json
    acks => "all"
  }
}
```

### Email Output

Sends email alerts:

```ruby
output {
  if [loglevel] == "ERROR" {
    email {
      to => "admin@example.com"
      from => "logstash@example.com"
      subject => "Logstash Alert: %{message}"
      body => "Alert Details: %{message}"
      domain => "mail.example.com"
      port => 25
    }
  }
}
```

### HTTP Output

Sends events via HTTP requests:

```ruby
output {
  http {
    url => "http://api.example.com/endpoint"
    http_method => "post"
    format => "json"
    headers => {
      "Authorization" => "Bearer token"
      "Content-Type" => "application/json"
    }
  }
}
```

### S3 Output

Writes events to Amazon S3:

```ruby
output {
  s3 {
    bucket => "my-bucket"
    region => "us-east-1"
    prefix => "logs/%{+YYYY/MM/dd/}"
    time_file => 15
    codec => json_lines
  }
}
```

### Multiple Outputs

You can define multiple outputs with conditional logic:

```ruby
output {
  if [loglevel] == "ERROR" {
    elasticsearch {
      hosts => ["http://es1:9200"]
      index => "errors-%{+YYYY.MM.dd}"
    }
    email {
      to => "admin@example.com"
      subject => "Error Alert: %{message}"
    }
  } else {
    elasticsearch {
      hosts => ["http://es2:9200"]
      index => "logs-%{+YYYY.MM.dd}"
    }
  }
}
```

## Codec Plugins

Codec plugins encode and decode data formats. They can be used with both input and output plugins:

### JSON Codec

Decodes or encodes events as JSON objects:

```ruby
input {
  file {
    path => "/var/log/app.json"
    codec => json
  }
}
```

### Multiline Codec

Combines multiple lines into a single event:

```ruby
input {
  file {
    path => "/var/log/app.log"
    codec => multiline {
      pattern => "^%{TIMESTAMP_ISO8601}"
      negate => true
      what => "previous"
    }
  }
}
```

### Line Codec

Encodes events as text lines:

```ruby
output {
  file {
    path => "/var/log/output.log"
    codec => line { format => "%{host}: %{message}" }
  }
}
```

### Dots Codec

Outputs a dot for each event (useful for throughput monitoring):

```ruby
output {
  stdout {
    codec => dots
  }
}
```

## Advanced Configuration

### Multiple Pipelines

Define multiple pipelines in `pipelines.yml`:

```yaml
- pipeline.id: apache
  path.config: "/etc/logstash/conf.d/apache.conf"
  pipeline.workers: 2
- pipeline.id: syslog
  path.config: "/etc/logstash/conf.d/syslog.conf"
  pipeline.workers: 1
```

### Conditional Logic

Use conditionals to create branching logic:

```ruby
filter {
  if [source] == "apache" {
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
    }
  } else if [source] == "nginx" {
    grok {
      match => { "message" => "%{NGINXACCESS}" }
    }
  } else {
    mutate {
      add_tag => [ "unknown_source" ]
    }
  }
  
  if "_grokparsefailure" in [tags] {
    mutate {
      add_field => { "parse_error" => true }
    }
  }
}
```

### Pipeline-to-Pipeline Communication

Send events between pipelines:

```yaml
# In pipelines.yml
- pipeline.id: first_pipeline
  path.config: "/etc/logstash/first_pipeline.conf"
- pipeline.id: second_pipeline
  path.config: "/etc/logstash/second_pipeline.conf"
```

```ruby
# In first_pipeline.conf
output {
  pipeline {
    send_to => ["second_pipeline"]
  }
}

# In second_pipeline.conf
input {
  pipeline {
    address => "second_pipeline"
  }
}
```

### Dead Letter Queues

Capture and handle events that couldn't be processed:

```yaml
# In logstash.yml
dead_letter_queue.enable: true
path.dead_letter_queue: "/path/to/dlq"
```

Process dead letter queue events:
```ruby
input {
  dead_letter_queue {
    path => "/path/to/dlq"
    commit_offsets => true
    pipeline_id => "main"
  }
}
```

### Persistent Queues

Enable disk-based queuing for reliability:

```yaml
# In logstash.yml
queue.type: persisted
queue.max_bytes: 1gb
```

## Performance Tuning

### JVM Settings

Adjust JVM settings in `jvm.options`:

```
-Xms2g
-Xmx2g
-XX:+UseG1GC
-XX:G1ReservePercent=25
-XX:InitiatingHeapOccupancyPercent=75
```

### Pipeline Configuration

Optimize pipeline performance:

```yaml
# In logstash.yml
pipeline.workers: 4  # Number of threads (usually set to CPU cores)
pipeline.batch.size: 1000  # Number of events per batch
pipeline.batch.delay: 5  # Wait time for incomplete batches
```

### Filter Performance

Some tips for optimizing filter performance:

1. Order filters by execution frequency (most frequent first)
2. Use conditionals to avoid unnecessary processing
3. Use the `drop` filter to discard unneeded events early
4. Prefer simple patterns in grok filters
5. Avoid complex Ruby filters when possible

Example:
```ruby
filter {
  # Quick check first to potentially skip expensive filters
  if [message] =~ "DEBUG" and [loglevel] != "DEBUG" {
    drop { }
  }
  
  # Only apply expensive filters when needed
  if [source] == "application" {
    grok { ... }
    date { ... }
  }
}
```

## Monitoring Logstash

### Monitoring APIs

Logstash provides several HTTP APIs for monitoring:

```bash
# Basic info
curl -XGET 'localhost:9600/?pretty'

# Node info
curl -XGET 'localhost:9600/_node?pretty'

# Hot threads
curl -XGET 'localhost:9600/_node/hot_threads?pretty'

# Pipeline stats
curl -XGET 'localhost:9600/_node/stats/pipelines?pretty'
```

### Monitoring with Metricbeat

Use Metricbeat to collect Logstash metrics:

```yaml
metricbeat.modules:
- module: logstash
  metricsets:
    - node
    - node_stats
  period: 10s
  hosts: ["localhost:9600"]
```

### X-Pack Monitoring

Enable X-Pack monitoring in `logstash.yml`:

```yaml
xpack.monitoring.enabled: true
xpack.monitoring.elasticsearch.hosts: ["http://es1:9200", "http://es2:9200"]
xpack.monitoring.elasticsearch.username: "logstash_system"
xpack.monitoring.elasticsearch.password: "password"
```

## Conclusion

This chapter covered the fundamentals of Logstash, including its architecture, configuration, plugins, and performance tuning. Logstash is a powerful and flexible tool for data processing, capable of handling a wide range of sources, transformations, and destinations.

In the next chapters, we'll dive deeper into specific aspects of Logstash, including detailed coverage of input, filter, and output plugins, as well as advanced configuration scenarios and best practices for production deployments.