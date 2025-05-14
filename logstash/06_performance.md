# Logstash Performance Tuning

This chapter covers techniques and best practices for optimizing Logstash performance to handle high-volume data processing efficiently.

## Table of Contents
- [Understanding Logstash Performance](#understanding-logstash-performance)
- [Hardware Considerations](#hardware-considerations)
- [JVM Tuning](#jvm-tuning)
- [Pipeline Configuration](#pipeline-configuration)
- [Worker and Batch Settings](#worker-and-batch-settings)
- [Queue Management](#queue-management)
- [Plugin Optimization](#plugin-optimization)
- [Monitoring Performance](#monitoring-performance)
- [Scaling Strategies](#scaling-strategies)
- [Real-World Tuning Examples](#real-world-tuning-examples)

## Understanding Logstash Performance

Logstash performance is influenced by several factors:

1. **Data volume**: The amount of data flowing through pipelines
2. **Event complexity**: The structure and size of individual events
3. **Processing complexity**: The number and type of transformations applied
4. **Hardware resources**: CPU, memory, disk I/O, and network bandwidth
5. **Configuration settings**: Pipeline workers, batch sizes, queue settings

Performance bottlenecks typically occur in one of these areas:
- Input stages (data ingestion)
- Filter stages (data processing)
- Output stages (data delivery)
- Resource constraints (CPU, memory, I/O)

## Hardware Considerations

### CPU

Logstash is CPU-intensive, especially for complex filter operations:
- **Recommendation**: 8+ CPU cores for production workloads
- **Scaling**: CPU usage scales linearly with worker threads (up to a point)

### Memory

Memory requirements depend on:
- Pipeline configuration
- Number of concurrent events
- JVM heap size
- Persistent queue usage

**Recommendations**:
- Minimum: 4GB RAM
- Production: 8GB to 64GB depending on workload
- Reserve 50% of system memory for OS file cache

### Disk

Disk I/O becomes important when:
- Using persistent queues
- Writing to file outputs
- Processing large files via file inputs

**Recommendations**:
- SSD storage for persistent queues
- Adequate disk space for queue storage (10GB minimum)
- Separate disks for OS/application and queue storage

### Network

Network bandwidth may become a bottleneck for:
- Distributed deployments
- Remote input/output sources
- High-volume data transfers

**Recommendations**:
- 1Gbps+ network interfaces
- Low-latency connections between Logstash and Elasticsearch
- Network traffic monitoring to identify bottlenecks

## JVM Tuning

Logstash runs on the Java Virtual Machine (JVM), making JVM tuning critical for performance.

### Heap Size

Configure the JVM heap size in `jvm.options`:

```
-Xms2g
-Xmx2g
```

General recommendations:
- Set min and max heap to the same value to prevent heap resizing
- Maximum heap size: 8GB (Avoid larger sizes due to garbage collection issues)
- Minimum recommendation: 4GB for production
- Maximum recommendation: 8GB

### Garbage Collection

Tune garbage collection for better performance:

```
## GC configuration
-XX:+UseG1GC
-XX:G1ReservePercent=25
-XX:InitiatingHeapOccupancyPercent=75
```

For high-throughput scenarios with large heaps:
```
-XX:+UseG1GC
-XX:MaxGCPauseMillis=100
```

### Other JVM Options

Additional JVM options for performance:

```
# Disable compressed oops for heaps > 32GB
-XX:-UseCompressedOops

# Improve string performance
-XX:+OptimizeStringConcat

# Enable aggressive optimizations
-XX:+AggressiveOpts
```

## Pipeline Configuration

### Workers and Parallelism

Configure pipeline workers in `logstash.yml` or per pipeline:

```yaml
# Global setting
pipeline.workers: 8

# Per-pipeline setting in pipelines.yml
- pipeline.id: main
  pipeline.workers: 8
```

Guidelines:
- Start with workers equal to available CPU cores
- Increase gradually and monitor performance
- For IO-bound pipelines, set workers higher than CPU cores
- For CPU-bound pipelines, set workers equal to or slightly less than CPU cores

### Batch Settings

Optimize batch processing:

```yaml
# Global settings in logstash.yml
pipeline.batch.size: 500
pipeline.batch.delay: 50

# Per-pipeline settings in pipelines.yml
- pipeline.id: high_volume
  pipeline.batch.size: 1000
  pipeline.batch.delay: 25
```

Guidelines:
- Increase batch size for higher throughput (125-3000 events)
- Decrease batch delay for lower latency (10-100ms)
- High throughput scenario: large batch size (1000+), low delay (25ms)
- Low latency scenario: smaller batch size (125-250), very low delay (5-10ms)

## Queue Management

### Memory Queue

Default in-memory queue settings:

```yaml
queue.type: memory
queue.max_events: 0 # unlimited
```

Benefits:
- Fastest performance
- No disk I/O overhead

Drawbacks:
- Risk of data loss if Logstash crashes
- Limited by available memory

### Persistent Queue

Enable persistent queues for durability:

```yaml
queue.type: persisted
queue.max_bytes: 8gb
path.queue: /path/to/queue
```

Benefits:
- Data durability across restarts
- Protection against data loss
- Buffer for traffic spikes

Drawbacks:
- Slower than memory queues
- Disk I/O requirements
- More complex management

Optimization tips:
- Use SSDs for queue storage
- Monitor queue growth/usage
- Adjust `queue.max_bytes` based on expected traffic patterns
- Set `queue.checkpoint.writes` to balance durability and performance

## Plugin Optimization

### Input Plugins

Optimize input plugins:

```ruby
input {
  beats {
    port => 5044
    client_inactivity_timeout => 60
    # Increase for high-volume environments
    receive_buffer_bytes => 32768
  }
  
  kafka {
    topics => ["logs"]
    consumer_threads => 8
    decorate_events => false # Disable if not needed
  }
}
```

Input optimization tips:
- Use efficient input plugins (Beats, Kafka, Redis)
- Adjust thread counts for network inputs
- Disable unnecessary event decoration
- Use appropriate batching settings for the input type

### Filter Plugins

Optimize filter plugins:

```ruby
filter {
  # Use dissect instead of grok when possible
  dissect {
    mapping => { "message" => "%{timestamp} %{+timestamp} %{level} %{message}" }
  }
  
  # Efficient date parsing
  date {
    match => [ "timestamp", "UNIX" ]
    remove_field => [ "timestamp" ]
  }
  
  # Only apply expensive filters when needed
  if [type] == "apache" {
    grok { ... }
  }
}
```

Filter optimization tips:
- Use lightweight alternatives when possible (dissect vs. grok)
- Apply conditional processing to heavy filters
- Minimize the number of filter operations
- Process events in parallel (non-ordered pipelines)
- Remove unnecessary fields early in the pipeline

### Output Plugins

Optimize output plugins:

```ruby
output {
  elasticsearch {
    hosts => ["es-node1:9200", "es-node2:9200"]
    workers => 4
    template_overwrite => false
    manage_template => false
    # Increase for better throughput
    bulk_size => 5000
    # Reduce for better throughput with reliability
    retry_initial_interval => 2
  }
}
```

Output optimization tips:
- Use bulk operations for Elasticsearch and HTTP outputs
- Increase worker count for outputs
- Tune retry logic to balance reliability and performance
- Disable template management if not needed
- Use sniffing for Elasticsearch but be aware of network implications

## Monitoring Performance

### Metrics API

Enable and use the Logstash Metrics API:

```yaml
# In logstash.yml
api.enabled: true
```

Query metrics:
```bash
curl -XGET 'localhost:9600/_node/stats/pipelines?pretty'
```

Key metrics to monitor:
- Events per second in/out
- Filter processing time
- Queue utilization
- Plugin processing times
- Hot threads

### Performance Testing

Benchmark Logstash performance:
- Generate sample datasets of varying sizes
- Use the `--debug` flag for detailed performance information
- Monitor resource usage using tools like `top`, `htop`, `iostat`
- Gradually increase load until bottlenecks appear

Benchmarking command:
```bash
bin/logstash -f pipeline.conf --config.debug --log.level=warn
```

### X-Pack Monitoring

Use X-Pack monitoring for real-time performance visibility:

```yaml
# In logstash.yml
xpack.monitoring.enabled: true
xpack.monitoring.elasticsearch.hosts: ["http://es-host:9200"]
```

Benefits:
- Visual dashboards for performance metrics
- Historical performance data
- Alerting on performance issues
- Integrated with Elasticsearch and Kibana monitoring

## Scaling Strategies

### Vertical Scaling

Optimize a single Logstash instance:
- Increase hardware resources (CPU, memory)
- Tune JVM settings
- Optimize pipeline configurations
- Enable persistent queues

Limitations:
- JVM heap size constraints
- Single point of failure
- Hardware limits

### Horizontal Scaling

Deploy multiple Logstash instances:
- Load balancing inputs
- Functional separation of pipelines
- Pipeline-specific optimization

Common patterns:
1. **Clone pattern**: Identical pipelines across instances with load balancing
2. **Worker pattern**: Split processing across specialized instances
3. **Filter farm pattern**: Centralized input/output with distributed filtering

Example deployment architecture:
```
Input Tier → Processing Tier → Output Tier
(Collection)  (Transformation)  (Delivery)
```

## Real-World Tuning Examples

### High-Volume Log Processing

Scenario: 20,000+ events per second from Filebeat

Configuration:
```ruby
# logstash.yml
pipeline.workers: 12
pipeline.batch.size: 2000
pipeline.batch.delay: 20

# JVM
-Xms6g
-Xmx6g

# Pipeline
input {
  beats {
    port => 5044
    receive_buffer_bytes => 65536
  }
}

filter {
  # Lightweight parsing - use dissect when possible
  # Group similar filters to improve compiler optimization
}

output {
  elasticsearch {
    hosts => ["es1:9200", "es2:9200", "es3:9200"]
    workers => 6
    bulk_size => 4000
  }
}
```

Hardware: 16 CPU cores, 32GB RAM, SSD storage

### Low-Latency Event Processing

Scenario: Security events requiring sub-second processing

Configuration:
```ruby
# logstash.yml
pipeline.workers: 8
pipeline.batch.size: 100
pipeline.batch.delay: 5
pipeline.ordered: true

# JVM
-Xms4g
-Xmx4g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=50

# Pipeline
input {
  kafka {
    topics => ["security_events"]
    consumer_threads => 4
    decorate_events => false
  }
}

filter {
  # Efficient, focused processing
  # Use conditionals to apply specific processing
}

output {
  elasticsearch {
    hosts => ["es-host:9200"]
    workers => 4
    bulk_size => 250
    routing => "%{event_type}"
  }
}
```

Hardware: 8 CPU cores, 16GB RAM, NVMe storage

### Distributed Processing

Scenario: Mixed log types with complex processing

Architecture:
1. **Input Tier**: 2 instances handling data collection
   ```ruby
   output {
     kafka {
       topic => "%{log_type}"
     }
   }
   ```

2. **Processing Tier**: 4 instances each specializing in different log types
   ```ruby
   input {
     kafka {
       topics => ["apache"]
       consumer_threads => 4
     }
   }
   # Specialized processing for Apache logs
   ```

3. **Output Tier**: 2 instances handling Elasticsearch output
   ```ruby
   input {
     kafka {
       topics => ["processed_logs"]
       consumer_threads => 6
     }
   }
   output {
     elasticsearch { ... }
   }
   ```

Benefits:
- Specialized optimization per tier
- Independent scaling
- Fault isolation

## Conclusion

Performance tuning Logstash requires understanding the interaction between hardware resources, JVM settings, pipeline configurations, and plugin optimizations. By monitoring performance metrics and applying the appropriate scaling strategies, you can build Logstash deployments capable of handling even the most demanding data processing requirements.

In the next chapters, we'll explore Kibana, the visualization layer of the ELK stack, starting with the fundamentals of Kibana's interface and capabilities.