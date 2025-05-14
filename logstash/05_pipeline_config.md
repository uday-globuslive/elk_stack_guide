# Logstash Pipeline Configuration

This chapter covers how to configure and manage Logstash pipelines effectively.

## Table of Contents
- [Pipeline Architecture](#pipeline-architecture)
- [Pipeline Configuration Files](#pipeline-configuration-files)
- [Multiple Pipelines](#multiple-pipelines)
- [Pipeline-to-Pipeline Communication](#pipeline-to-pipeline-communication)
- [Conditional Processing](#conditional-processing)
- [Error Handling](#error-handling)
- [Configuration Management and Deployment](#configuration-management-and-deployment)

## Pipeline Architecture

A Logstash pipeline consists of three main stages:

1. **Inputs**: Collect data from various sources
2. **Filters**: Process and transform the data
3. **Outputs**: Send the processed data to destinations

```
Input → Filter → Output
```

Pipelines operate on events, which are individual units of data flowing through Logstash. Each event contains:
- Original message data
- Metadata about the source
- Any fields added during processing

### Event Flow

Events flow through a pipeline following these steps:
1. Input plugins collect data and create events
2. Codec plugins decode/encode data formats
3. Filter plugins transform events
4. Output plugins send events to destinations

## Pipeline Configuration Files

Logstash pipeline configurations are defined in plain text files using a DSL (Domain Specific Language) based on the Ruby programming language.

### Basic Structure

```ruby
# Input section
input {
  # Input plugin configurations
}

# Filter section
filter {
  # Filter plugin configurations
}

# Output section
output {
  # Output plugin configurations
}
```

### File Organization

Best practices for organizing configuration files:

- **Single Pipeline**: Use one file (`logstash.conf`) for simple setups
- **Multiple Files**: For complex setups, split configuration by function:
  - `inputs.conf`
  - `filters.conf`
  - `outputs.conf`
- **Module-Based**: Organize by data source or application:
  - `apache.conf`
  - `mysql.conf`
  - `system.conf`

### Configuration Path

By default, Logstash looks for pipeline configuration files in:
- `/etc/logstash/conf.d/*.conf` (Linux/macOS)
- `{Logstash_HOME}\config\*.conf` (Windows)

Start Logstash with specific configuration:

```bash
bin/logstash -f path/to/logstash.conf
```

Or with a configuration directory:

```bash
bin/logstash -f path/to/config/dir/
```

## Multiple Pipelines

Logstash supports running multiple pipelines within a single Logstash instance, enabling isolation of different data flows.

### pipelines.yml

Define multiple pipelines in `config/pipelines.yml`:

```yaml
- pipeline.id: main
  path.config: "/etc/logstash/conf.d/*.conf"
  pipeline.workers: 4
  
- pipeline.id: monitoring
  path.config: "/etc/logstash/monitoring/*.conf"
  pipeline.batch.size: 125
  queue.type: persisted
```

### Pipeline Settings

Common pipeline settings:

- `pipeline.id`: Unique identifier for the pipeline
- `path.config`: Path to configuration files
- `pipeline.workers`: Number of worker threads (default: CPU cores)
- `pipeline.batch.size`: Number of events per batch (default: 125)
- `pipeline.batch.delay`: Batch delay in milliseconds (default: 50)
- `queue.type`: Queue type (`memory` or `persisted`)
- `queue.max_bytes`: Maximum size of the queue (default: 1GB)
- `pipeline.ordered`: Whether to preserve event order (default: auto)

## Pipeline-to-Pipeline Communication

Logstash 6.0+ supports pipeline-to-pipeline communication, allowing events to flow between pipelines.

### Configuration Example

Producer pipeline (`first_pipeline.conf`):
```ruby
input { ... }
filter { ... }
output {
  pipeline { send_to => "second_pipeline" }
}
```

Consumer pipeline (`second_pipeline.conf`):
```ruby
input { 
  pipeline { address => "second_pipeline" }
}
filter { ... }
output { ... }
```

This pattern is useful for:
- Creating modular, reusable pipeline components
- Implementing complex routing logic
- Isolating resource-intensive operations

## Conditional Processing

Logstash pipelines support conditional processing to route events based on their contents.

### If/Else Conditions

```ruby
filter {
  if [status] == 404 {
    mutate { add_tag => "not_found" }
  } else if [status] >= 500 {
    mutate { add_tag => "server_error" }
  } else {
    mutate { add_tag => "ok" }
  }
}
```

### Complex Conditions

```ruby
filter {
  if [source] =~ /^\/var\/log\/apache\// and [type] == "apache" {
    grok { ... }
  } else if [message] =~ "ERROR" or [message] =~ "FATAL" {
    mutate { add_tag => "error_event" }
  }
}
```

### Conditional Outputs

```ruby
output {
  if "error" in [tags] {
    elasticsearch {
      index => "errors-%{+YYYY.MM.dd}"
    }
    email { ... }
  } else {
    elasticsearch {
      index => "events-%{+YYYY.MM.dd}"
    }
  }
}
```

## Error Handling

Managing errors in Logstash pipelines is crucial for reliability.

### Dead Letter Queues

Configure a Dead Letter Queue (DLQ) for handling problematic events:

```ruby
# In logstash.yml
dead_letter_queue.enable: true
path.dead_letter_queue: "/path/to/dlq"
```

Processing DLQ events:
```ruby
input {
  dead_letter_queue {
    path => "/path/to/dlq"
    commit_offsets => true
  }
}
```

### Retry Logic

Implement retry logic for outputs:

```ruby
output {
  elasticsearch {
    hosts => ["es-host:9200"]
    retry_initial_interval => 2
    retry_max_interval => 64
    retry_on_conflict => 5
    max_retries => 10
  }
}
```

### Circuit Breakers

Use circuit breaker patterns to prevent pipeline failures:

```ruby
output {
  if [retry_count] and [retry_count] > 3 {
    file {
      path => "/path/to/failed_events/%{+YYYY-MM-dd}.log"
    }
  } else {
    elasticsearch { ... }
    
    if "_elasticsearch_failure" in [tags] {
      ruby {
        code => "
          event.set('[retry_count]', event.get('[retry_count]').to_i + 1)
        "
      }
      # Send back to beginning of pipeline
    }
  }
}
```

## Configuration Management and Deployment

Best practices for managing Logstash configurations in production.

### Version Control

Store configurations in a version control system (Git):
- Track changes over time
- Collaborate with team members
- Deploy configurations using CI/CD

### Testing Configurations

Validate configurations before deployment:

```bash
# Check for syntax errors
bin/logstash -f pipeline.conf --config.test_and_exit

# Debug configuration
bin/logstash -f pipeline.conf --config.debug
```

### Configuration as Code

Treat Logstash configurations as code:
- Use templates for common patterns
- Parameterize environments (dev, test, prod)
- Automate deployment with tools like Ansible, Chef, or Puppet

Example Ansible deployment:
```yaml
- name: Deploy Logstash configurations
  template:
    src: templates/logstash/{{ item }}.conf.j2
    dest: /etc/logstash/conf.d/{{ item }}.conf
    owner: logstash
    group: logstash
    mode: 0644
  with_items:
    - inputs
    - filters
    - outputs
  notify: restart logstash
```

### Secrets Management

Avoid hardcoding sensitive information in pipeline configurations:

```ruby
input {
  jdbc {
    jdbc_connection_string => "${JDBC_CONNECTION_STRING}"
    jdbc_user => "${JDBC_USER}"
    jdbc_password => "${JDBC_PASSWORD}"
  }
}
```

Use environment variables or a secure secret management solution like HashiCorp Vault.

### Centralized Configuration

For large deployments, use centralized configuration management:
- Elastic Stack's Central Pipeline Management
- Configuration management tools
- Custom configuration distribution systems

## Conclusion

Effective pipeline configuration is key to building reliable and maintainable Logstash deployments. By understanding pipeline architecture, implementing proper error handling, and following configuration management best practices, you can ensure your Logstash pipelines process data efficiently and reliably.

In the next chapter, we will explore Logstash performance tuning to optimize your pipelines for maximum throughput and minimal resource usage.