# Performance Best Practices

This chapter covers performance best practices for the ELK Stack, providing actionable recommendations to optimize each component for speed, efficiency, and scalability.

## Elasticsearch Performance Optimization

### Hardware Considerations

#### Memory Configuration

- **RAM Allocation**: Allocate 50% of available system memory to the JVM heap, but not more than 31GB
  ```
  # Example for a 64GB machine
  -Xms31g -Xmx31g
  ```

- **Memory Locking**: Enable bootstrap.memory_lock to prevent swapping
  ```yaml
  bootstrap.memory_lock: true
  ```

- **Heap Dump Path**: Configure properly to avoid disk space issues
  ```
  -XX:HeapDumpPath=/var/lib/elasticsearch
  ```

#### Storage Configuration

- **Disk Type**: Use SSDs for all Elasticsearch data for optimal performance
- **RAID Configuration**: Use RAID 0 for maximum performance (Elasticsearch handles replication)
- **File System**: Use ext4 or XFS with discard option enabled for SSDs
- **Multiple Paths**: Spread data across multiple disks using the path.data setting
  ```yaml
  path.data:
    - /mnt/elasticsearch_1
    - /mnt/elasticsearch_2
  ```

#### CPU Considerations

- **CPU Cores**: More cores help with concurrent operations
- **CPU/Memory Ratio**: Aim for 1:8 ratio (1 CPU core for every 8GB RAM)
- **Processor Features**: Enable processor features like AVX2, AVX-512 for better performance

### Indexing Optimization

#### Bulk Indexing

- **Batch Size**: Optimize bulk request size (5-15MB is typically optimal)
- **Concurrent Requests**: Keep enough in-flight requests to fully utilize resources (typically 2-3× number of cores)
- **Client-Side Queuing**: Implement client-side queuing for consistent throughput

#### Index Settings

- **Refresh Interval**: Increase refresh interval during bulk indexing
  ```
  PUT /my_index/_settings
  {
    "index": {
      "refresh_interval": "30s"
    }
  }
  ```

- **Translog Settings**: Tune translog durability and flush for bulk performance
  ```
  PUT /my_index/_settings
  {
    "index": {
      "translog.durability": "async",
      "translog.sync_interval": "5s",
      "translog.flush_threshold_size": "1gb"
    }
  }
  ```

- **Auto-throttling**: Enable automatic index throttling
  ```
  PUT /_cluster/settings
  {
    "persistent": {
      "indices.store.throttle.type": "merge"
    }
  }
  ```

#### Mapping Optimization

- **Field Selection**: Only index fields you need to search or aggregate on
- **Dynamic Mapping**: Disable dynamic mapping in production
  ```
  PUT /my_index
  {
    "mappings": {
      "dynamic": "strict"
    }
  }
  ```

- **Field Types**: Use appropriate field types (keyword vs text, scaled_float vs double)
- **Doc Values**: Disable doc_values for fields not used in aggregations or sorting
  ```
  "mappings": {
    "properties": {
      "text_field": {
        "type": "text",
        "doc_values": false
      }
    }
  }
  ```

### Search Optimization

#### Query Design

- **Filter vs Query**: Use filter context for exact matches (faster and cached)
  ```
  GET /my_index/_search
  {
    "query": {
      "bool": {
        "filter": [
          {"term": {"status": "active"}}
        ],
        "must": [
          {"match": {"description": "important document"}}
        ]
      }
    }
  }
  ```

- **Query Rewriting**: Use more specific queries when possible (term vs match)
- **Scroll vs Search After**: Use search_after for deep pagination instead of from/size
  ```
  GET /my_index/_search
  {
    "size": 100,
    "sort": [
      {"date": "asc"},
      {"_id": "asc"}
    ],
    "search_after": [1586215507, "doc_id"]
  }
  ```

#### Aggregation Optimization

- **Sampling**: Use sampling for approximate aggregations when possible
  ```
  GET /my_index/_search
  {
    "size": 0,
    "query": {
      "bool": {
        "filter": {
          "random_score": {
            "seed": 10,
            "field": "_seq_no"
          }
        }
      }
    },
    "aggs": {
      "sampled_terms": {
        "terms": {
          "field": "category",
          "size": 10
        }
      }
    }
  }
  ```

- **Cardinality Limitations**: Be cautious with high-cardinality fields in aggregations
- **Composite Aggregations**: Use composite aggregations for paginating through buckets
  ```
  GET /my_index/_search
  {
    "size": 0,
    "aggs": {
      "my_buckets": {
        "composite": {
          "size": 100,
          "sources": [
            { "date": { "date_histogram": { "field": "timestamp", "calendar_interval": "1d" } } },
            { "category": { "terms": { "field": "category" } } }
          ]
        }
      }
    }
  }
  ```

### Shard Optimization

#### Shard Sizing

- **Size Range**: Aim for shards between 20-40GB (max 50GB)
- **Count Formula**: `(Daily Data Volume × Retention Days) / Target Shard Size`
- **Overshard Prevention**: Avoid excessive shards (causes CPU overhead)

#### Shard Allocation

- **Allocation Filtering**: Use allocation filters to control shard placement
  ```
  PUT /_cluster/settings
  {
    "persistent": {
      "cluster.routing.allocation.disk.threshold_enabled": true,
      "cluster.routing.allocation.disk.watermark.low": "85%",
      "cluster.routing.allocation.disk.watermark.high": "90%",
      "cluster.routing.allocation.disk.watermark.flood_stage": "95%"
    }
  }
  ```

- **Rebalancing**: Tune rebalancing settings to prevent excessive shard movement
  ```
  PUT /_cluster/settings
  {
    "persistent": {
      "cluster.routing.allocation.cluster_concurrent_rebalance": 2
    }
  }
  ```

- **Recovery Settings**: Configure recovery settings for faster restart and rebalancing
  ```
  PUT /_cluster/settings
  {
    "persistent": {
      "indices.recovery.max_bytes_per_sec": "100mb",
      "indices.recovery.concurrent_streams": 4
    }
  }
  ```

### Caching and Memory

#### Field Data Cache

- **Circuit Breaker**: Configure fielddata circuit breaker to prevent OOM issues
  ```
  PUT /_cluster/settings
  {
    "persistent": {
      "indices.breaker.fielddata.limit": "40%"
    }
  }
  ```

- **Field Data Settings**: Properly size fielddata cache based on available memory
  ```
  PUT /_cluster/settings
  {
    "persistent": {
      "indices.fielddata.cache.size": "30%"
    }
  }
  ```

#### Query Cache

- **Query Cache Size**: Set appropriate query cache size
  ```
  PUT /_cluster/settings
  {
    "persistent": {
      "indices.queries.cache.size": "5%"
    }
  }
  ```

- **Request Cache**: Enable request cache for aggregation-heavy workloads
  ```
  PUT /my_index/_settings
  {
    "index.requests.cache.enable": true
  }
  ```

#### Segment Merging

- **Merge Scheduler**: Tune merge scheduler based on available resources
  ```
  PUT /my_index/_settings
  {
    "index": {
      "merge.scheduler.max_thread_count": 1,
      "merge.scheduler.max_merge_count": 2
    }
  }
  ```

- **Force Merge**: Periodically force merge read-only indices
  ```
  POST /my_index/_forcemerge?max_num_segments=1
  ```

## Logstash Performance Optimization

### Pipeline Configuration

#### Input Optimization

- **Batch Size**: Optimize input batch size for performance
  ```ruby
  input {
    beats {
      port => 5044
      client_inactivity_timeout => 3600
      ssl => true
      batch_size => 4096  # Increased batch size
    }
  }
  ```

- **Input Workers**: Configure appropriate number of input threads
  ```ruby
  input {
    beats {
      port => 5044
      workers => 4  # Adjust based on CPU cores
    }
  }
  ```

#### Filter Optimization

- **Multithreading**: Use pipeline workers for multithreading
  ```
  pipelines.yml:
    - pipeline.id: main
      pipeline.workers: 8  # Typically 1 per CPU core
  ```

- **Batch Processing**: Increase filter batch size
  ```
  pipelines.yml:
    - pipeline.id: main
      pipeline.batch.size: 1000  # Default is 125
  ```

- **Efficient Pattern Matching**: Optimize grok patterns and use oniguruma where possible
  ```ruby
  filter {
    # More efficient pattern with anchors
    grok {
      match => { "message" => "^%{IP:client} %{WORD:method} %{URIPATHPARAM:request}" }
      timeout_millis => 500  # Timeout for complex patterns
    }
  }
  ```

- **Filter Ordering**: Place conditionals and common filters first
  ```ruby
  filter {
    # Fast check first to potentially skip expensive filters
    if [type] != "apache_log" {
      drop { }
    }
    
    # Now apply expensive filters only to relevant events
    grok { ... }
    date { ... }
  }
  ```

#### Output Optimization

- **Elasticsearch Bulk Size**: Optimize bulk indexing parameters
  ```ruby
  output {
    elasticsearch {
      hosts => ["es1:9200", "es2:9200", "es3:9200"]
      index => "logs-%{+YYYY.MM.dd}"
      manage_template => false
      bulk_size => 5000  # Default is 1000
      flush_size => 10000  # Events per bulk request
      timeout => 180  # Longer timeout for bulk operations
    }
  }
  ```

- **HTTP Compression**: Enable compression for reduced network usage
  ```ruby
  output {
    elasticsearch {
      hosts => ["es1:9200", "es2:9200", "es3:9200"]
      compression_level => 3  # 1-9, higher is more compression
    }
  }
  ```

### Memory Management

- **JVM Heap Size**: Set appropriate JVM heap size (1-8GB is typical)
  ```
  -Xms4g -Xmx4g
  ```

- **Persistent Queues**: Enable persistent queues for durability and performance
  ```
  pipelines.yml:
    - pipeline.id: main
      queue.type: persisted
      queue.max_bytes: 1gb
  ```

- **Pipeline Tuning**: Adjust pipeline configurations based on event volume
  ```
  pipelines.yml:
    - pipeline.id: high-volume
      path.config: "/etc/logstash/conf.d/high-volume.conf"
      pipeline.workers: 8 
      pipeline.batch.size: 2000
      queue.type: persisted
      queue.max_bytes: 4gb
    
    - pipeline.id: low-volume
      path.config: "/etc/logstash/conf.d/low-volume.conf"
      pipeline.workers: 2
      pipeline.batch.size: 125
  ```

### Multi-Pipeline Architecture

- **Separate Processing Pipelines**: Split by data source or processing needs
  ```
  - pipeline.id: apache
    path.config: "/etc/logstash/conf.d/apache.conf"
    pipeline.workers: 4
  
  - pipeline.id: mysql
    path.config: "/etc/logstash/conf.d/mysql.conf"
    pipeline.workers: 2
  ```

- **Event Routing**: Use pipeline-to-pipeline communication for complex workflows
  ```ruby
  # First pipeline
  output {
    pipeline {
      send_to => ["enrichment_pipeline"]
    }
  }
  
  # Second pipeline
  input {
    pipeline {
      address => "enrichment_pipeline"
    }
  }
  ```

## Kibana Performance Optimization

### Browser Performance

- **Dashboard Complexity**: Limit the number of visualizations per dashboard (8-12 maximum)
- **Time Range Selection**: Use shorter time ranges for complex dashboards
- **Browser Caching**: Enable browser caching for static assets

### Visualization Optimization

- **Data Table Size**: Limit data table results to necessary rows
- **Aggregation Optimizations**:
  - Use date histograms with appropriate intervals
  - Avoid high-cardinality field aggregations
  - Use terms aggregations with reasonable size limits

- **Visual Settings**: Disable auto-refresh for complex dashboards
- **Caching**: Use query result caching in Elasticsearch

### Server-Side Optimization

- **Memory Allocation**: Increase Node.js memory limit for complex instances
  ```
  NODE_OPTIONS="--max-old-space-size=4096"
  ```

- **Concurrent Users**: Monitor and scale based on concurrent user sessions
- **Timelion and Vega**: Be cautious with complex Timelion or Vega visualizations

### Production Settings

- **Saved Object Optimization**: Regularly clean up unused saved objects
- **Index Pattern Management**: Create specific index patterns instead of using wildcards where possible
- **Reporting Queue**: Tune reporting queue size based on usage

## Beats Performance Optimization

### Filebeat Optimization

- **Harvester Configuration**:
  ```yaml
  filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/*.log
    harvester_buffer_size: 16384
    max_bytes: 10485760
  ```

- **Registry File**: Configure appropriate registry file size
  ```yaml
  filebeat.registry.flush: 5s
  filebeat.registry.file: /var/lib/filebeat/registry
  filebeat.registry.migrate_file: /var/lib/filebeat/registry
  ```

- **Scan Frequency**: Adjust scan frequency based on log volume
  ```yaml
  filebeat.inputs:
  - type: log
    scan_frequency: 5s
  ```

### Metricbeat Optimization

- **Collection Period**: Adjust collection period based on needs
  ```yaml
  metricbeat.modules:
  - module: system
    period: 30s
    metricsets:
      - cpu
      - memory
    cpu.metrics: ["percentages"]
  ```

- **Module Selection**: Only enable necessary modules
- **Process Monitoring**: Limit process monitoring scope
  ```yaml
  - module: system
    period: 30s
    processes: ['.*']
    process.include_top_n:
      enabled: true
      by_cpu: 5
      by_memory: 5
  ```

### Output Buffering

- **Output Batch Size**: Tune output batching settings
  ```yaml
  output.elasticsearch:
    hosts: ["elasticsearch:9200"]
    bulk_max_size: 1000
    backoff.max: 60s
  ```

- **Queue Settings**: Configure internal queue settings
  ```yaml
  queue.mem:
    events: 4096
    flush.min_events: 1024
    flush.timeout: 5s
  ```

## System-Level Optimization

### Linux Kernel Settings

- **File Descriptors**: Increase open file limits
  ```
  # /etc/security/limits.conf
  elasticsearch soft nofile 65536
  elasticsearch hard nofile 131072
  ```

- **Memory Maps**: Increase vm.max_map_count for Elasticsearch
  ```
  # /etc/sysctl.conf
  vm.max_map_count = 262144
  ```

- **Swappiness**: Reduce swappiness for better performance
  ```
  # /etc/sysctl.conf
  vm.swappiness = 1
  ```

### Network Optimization

- **TCP Settings**: Optimize TCP for high-throughput connections
  ```
  # /etc/sysctl.conf
  net.core.somaxconn = 65536
  net.ipv4.tcp_max_syn_backlog = 40000
  net.ipv4.tcp_fin_timeout = 20
  net.ipv4.tcp_tw_reuse = 1
  ```

- **Keepalive Settings**: Configure proper keepalive settings
  ```
  # /etc/sysctl.conf
  net.ipv4.tcp_keepalive_time = 600
  net.ipv4.tcp_keepalive_intvl = 30
  net.ipv4.tcp_keepalive_probes = 10
  ```

### Disk I/O Optimization

- **I/O Scheduler**: Use appropriate I/O scheduler for SSDs (deadline or noop)
  ```
  echo noop > /sys/block/sda/queue/scheduler
  ```

- **Readahead**: Optimize readahead settings
  ```
  blockdev --setra 0 /dev/sda
  ```

- **Disk Mount Options**: Use appropriate mount options
  ```
  # /etc/fstab
  /dev/sda1 /var/lib/elasticsearch ext4 noatime,data=writeback,discard 0 0
  ```

## Load Testing and Benchmarking

### Tools and Methodologies

- **Rally**: Elasticsearch's official benchmarking tool
  ```bash
  esrally --track=geonames --target-hosts=localhost:9200
  ```

- **JMeter**: General purpose load testing tool
- **Locust**: Python-based load testing tool

### Performance Metrics to Monitor

- **Query Latency**: 95th and 99th percentile response times
- **Indexing Throughput**: Documents per second
- **Search Throughput**: Queries per second
- **System Metrics**: CPU, memory, disk I/O, network I/O
- **JVM Metrics**: Heap usage, GC activity, thread pool utilization

### Capacity Planning

- **Resource Estimation**:
  - Elasticsearch: ~1GB heap per 10M documents (varies by mapping)
  - Disk space: Raw data × 1.1 × (1 + number of replicas)
  - CPU: 1 core per 5-10GB heap (approx)

- **Growth Planning**: Plan for 1.5-2× expected growth over the next year

## Performance Troubleshooting

### Common Elasticsearch Issues

1. **Slow Queries**
   - Check query patterns with slow log
   ```
   PUT /my_index/_settings
   {
     "index.search.slowlog.threshold.query.warn": "2s",
     "index.search.slowlog.threshold.fetch.warn": "500ms",
     "index.search.slowlog.level": "info"
   }
   ```
   - Use Profile API to analyze query execution
   ```
   GET /my_index/_search
   {
     "profile": true,
     "query": { ... }
   }
   ```

2. **Indexing Bottlenecks**
   - Check indexing slowlog
   ```
   PUT /my_index/_settings
   {
     "index.indexing.slowlog.threshold.index.warn": "500ms",
     "index.indexing.slowlog.level": "info"
   }
   ```
   - Monitor thread pool stats
   ```
   GET /_cat/thread_pool?v
   ```

3. **Memory Pressure**
   - Monitor field data circuit breaker trips
   - Check JVM heap usage pattern
   ```
   GET /_nodes/stats?filter_path=**.jvm.mem
   ```

### Common Logstash Issues

1. **Pipeline Congestion**
   - Monitor pipeline throughput with monitoring API
   ```
   GET /_node/stats/pipelines?pretty
   ```
   - Check for filter plugins creating bottlenecks

2. **Memory Issues**
   - Monitor heap usage with JVM stats
   - Check persistent queue usage

### Common Kibana Issues

1. **Slow Dashboard Loading**
   - Check browser console for slow requests
   - Monitor Elasticsearch query performance
   - Review visualization complexity

2. **Server Performance**
   - Monitor Node.js process memory
   - Check concurrent user sessions

## Conclusion

Optimizing the ELK Stack for performance requires a systematic approach to hardware selection, configuration tuning, and monitoring. The best practices outlined in this chapter should help you identify and resolve performance bottlenecks in your ELK deployment, resulting in faster indexing, quicker searches, and a more responsive user experience.

Remember that performance optimization is an iterative process—start with the highest-impact changes, measure the results, and then move on to the next opportunity for improvement.