# Performance Optimization

## Introduction to Elasticsearch Performance

Elasticsearch is designed to handle large volumes of data and provide fast search capabilities. However, achieving optimal performance requires careful planning, configuration, and ongoing monitoring. This chapter covers comprehensive strategies for optimizing Elasticsearch performance across various dimensions.

Performance optimization in Elasticsearch involves several key areas:

1. Hardware and infrastructure configuration
2. Cluster sizing and topology
3. Index design and shard management
4. Query optimization
5. Caching and memory management
6. Indexing performance tuning
7. Monitoring and troubleshooting

By understanding and applying these optimization techniques, you can ensure your Elasticsearch deployment performs efficiently under your specific workloads.

## Hardware and Infrastructure Optimization

### Memory Configuration

Memory is often the most critical resource for Elasticsearch performance.

#### JVM Heap Sizing

Configure the JVM heap size appropriately:

```
-Xms16g -Xmx16g
```

Best practices for heap sizing:
- Set minimum and maximum heap to the same value to prevent heap resizing
- Allocate 50% of available RAM to the JVM heap, but not more than 31GB
- Leave remaining memory for OS file cache
- Use the G1GC garbage collector for heaps larger than 4GB

```yaml
# jvm.options
-Xms16g
-Xmx16g
-XX:+UseG1GC
-XX:G1ReservePercent=25
-XX:InitiatingHeapOccupancyPercent=30
```

#### Memory Lock

Prevent swapping by enabling memory lock:

```yaml
# elasticsearch.yml
bootstrap.memory_lock: true
```

Verify it's working:
```
GET _nodes?filter_path=**.mlockall
```

### CPU Optimization

CPU affects search throughput and indexing speed:

- Provide sufficient CPU cores for concurrent operations
- Modern CPUs with higher clock speeds generally perform better
- Monitor CPU usage to identify bottlenecks

Elasticsearch thread pools are sized relative to available CPU:
```yaml
# elasticsearch.yml
thread_pool.search.size: 12  # Adjust based on available cores
thread_pool.write.size: 8
```

### Storage Configuration

Storage performance directly impacts Elasticsearch operations:

#### SSD vs. HDD

SSDs provide significantly better performance for:
- Random reads and writes during searches
- Index merges and segment operations
- Faster recovery and shard allocation

Recommended storage setup:
- Use SSDs for all production deployments
- For hot-warm-cold architectures:
  - Hot nodes: NVMe SSD
  - Warm nodes: SATA SSD
  - Cold nodes: HDD (acceptable for rarely accessed data)

#### RAID Configuration

For Elasticsearch, RAID considerations include:
- RAID 0 provides best performance (but no redundancy)
- RAID 10 offers good performance with redundancy
- Avoid RAID 5/6 due to poor write performance
- Use Elasticsearch replication instead of RAID for redundancy

#### File System

Optimize file system settings:
```bash
# Mount with optimal options
mount -o noatime,data=writeback,barrier=0,nobh /dev/sda1 /elasticsearch

# Increase read-ahead buffer for HDDs
blockdev --setra 1024 /dev/sda
```

Recommended file system settings:
- ext4 or XFS for Linux
- Disable access time updates (noatime)
- For SSDs, enable TRIM/discard

### Network Configuration

Network performance is critical for distributed operations:

#### Network Bandwidth

Ensure sufficient bandwidth:
- 1 Gbps minimum for production
- 10 Gbps recommended for large clusters
- Monitor network utilization to identify bottlenecks

#### Network Latency

Minimize latency:
- Co-locate nodes in the same data center when possible
- For geo-distributed clusters, use cross-cluster replication
- Optimize TCP settings for performance

```bash
# Optimize TCP for Elasticsearch
sysctl -w net.ipv4.tcp_retries2=5
sysctl -w net.ipv4.tcp_keepalive_time=60
sysctl -w net.ipv4.tcp_keepalive_intvl=10
sysctl -w net.ipv4.tcp_keepalive_probes=5
```

### Operating System Settings

Configure OS parameters for optimal performance:

#### File Descriptors

Increase file descriptor limits:
```bash
# /etc/security/limits.conf
elasticsearch soft nofile 65536
elasticsearch hard nofile 65536
```

#### Virtual Memory

Set vm.max_map_count for adequate memory mapping:
```bash
sysctl -w vm.max_map_count=262144
```

#### Swapping

Disable or minimize swapping:
```bash
# Disable swap completely
swapoff -a

# Or reduce swappiness
sysctl -w vm.swappiness=1
```

## Cluster Design for Performance

### Node Sizing and Specialization

Distribute workloads with specialized nodes:

#### Master Nodes

Dedicated master nodes for cluster management:
```yaml
# elasticsearch.yml for master node
node.roles: [ master ]
node.master: true
node.data: false
node.ingest: false
```

Recommended resources:
- 4-8 CPU cores
- 8-16GB RAM
- Fast, reliable storage

#### Data Nodes

Optimize data nodes for storage and search:
```yaml
# elasticsearch.yml for data node
node.roles: [ data ]
node.master: false
node.data: true
node.ingest: false
```

Recommended resources:
- 8-16 CPU cores
- 32-64GB RAM
- Fast SSD storage

#### Coordinating Nodes

Dedicated nodes for search coordination:
```yaml
# elasticsearch.yml for coordinating node
node.roles: []
node.master: false
node.data: false
node.ingest: false
```

Recommended resources:
- 8-16 CPU cores
- 16-32GB RAM
- Minimal storage requirements

#### Ingest Nodes

Specialized nodes for document pre-processing:
```yaml
# elasticsearch.yml for ingest node
node.roles: [ ingest ]
node.master: false
node.data: false
node.ingest: true
```

Recommended resources:
- 4-8 CPU cores
- 8-16GB RAM
- Moderate storage

### Shard Size and Count

Proper shard sizing is critical for performance:

#### Shard Sizing Guidelines

- **Target shard size**: 20-50GB per shard
- **Avoid small shards**: Shards smaller than 1GB create overhead
- **Avoid very large shards**: Shards larger than 100GB can slow recovery
- **Total shard count**: Keep below 20 shards per GB of heap memory

```yaml
# elasticsearch.yml - control shard count
index.number_of_shards: 3
index.number_of_replicas: 1
```

#### Shard Allocation

Control shard placement for balanced performance:

```yaml
# elasticsearch.yml
cluster.routing.allocation.awareness.attributes: zone,rack
cluster.routing.allocation.awareness.force.zone.values: zone1,zone2
cluster.routing.allocation.cluster_concurrent_rebalance: 2
cluster.routing.allocation.node_concurrent_recoveries: 2
cluster.routing.allocation.node_initial_primaries_recoveries: 4
```

### Hot-Warm-Cold Architecture

Implement data tiering for cost-effective performance:

```yaml
# Hot node configuration
node.attr.data: hot

# Warm node configuration
node.attr.data: warm

# Cold node configuration
node.attr.data: cold
```

Index lifecycle policy for tiering:
```json
PUT _ilm/policy/logs_policy
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
          "allocate": {
            "include": {
              "data": "warm"
            }
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
          "allocate": {
            "include": {
              "data": "cold"
            }
          },
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

## Index Design Optimization

### Mapping Optimization

Optimize mappings for better performance:

#### Field Type Selection

Choose appropriate field types:
```json
{
  "mappings": {
    "properties": {
      "id": { "type": "keyword" },
      "name": { "type": "text" },
      "status": { "type": "keyword" },
      "created_at": { "type": "date" },
      "price": { "type": "double" },
      "count": { "type": "integer" },
      "small_count": { "type": "short" }
    }
  }
}
```

Use scaled_float for decimal values when precision can be predetermined:
```json
"price": {
  "type": "scaled_float",
  "scaling_factor": 100
}
```

#### Index Options

Reduce index size with targeted options:
```json
"description": {
  "type": "text",
  "index_options": "freqs",
  "norms": false
}
```

Index options for text fields:
- `docs`: Only document numbers (minimal)
- `freqs`: Document numbers and term frequencies
- `positions`: Document numbers, frequencies, and term positions
- `offsets`: All of the above plus start/end character offsets

#### Dynamic Field Mapping

Control dynamic mapping behavior:
```json
"mappings": {
  "dynamic": "strict",
  "properties": {
    "known_field": { "type": "keyword" }
  }
}
```

Options:
- `true`: Automatically add fields
- `false`: Ignore new fields
- `strict`: Reject documents with unknown fields

### Indexing Performance

Optimize for fast indexing:

#### Bulk Indexing

Use the Bulk API for efficient indexing:
```json
POST /_bulk
{"index":{"_index":"products","_id":"1"}}
{"name":"Product 1","price":100}
{"index":{"_index":"products","_id":"2"}}
{"name":"Product 2","price":200}
```

Bulk indexing recommendations:
- Optimal request size: 5-15MB
- Batch sizes: 1,000-5,000 documents per request
- Monitor performance to find the sweet spot

#### Refresh Interval

Adjust refresh interval during bulk indexing:
```json
PUT /my_index/_settings
{
  "index": {
    "refresh_interval": "30s"
  }
}
```

For maximum indexing throughput:
```json
PUT /my_index/_settings
{
  "index": {
    "refresh_interval": "-1"
  }
}
```

Remember to restore the refresh interval after bulk indexing:
```json
PUT /my_index/_settings
{
  "index": {
    "refresh_interval": "1s"
  }
}
```

#### Translog Settings

Tune translog for better indexing performance:
```json
PUT /my_index/_settings
{
  "index": {
    "translog.durability": "async",
    "translog.sync_interval": "5s",
    "translog.flush_threshold_size": "1gb"
  }
}
```

#### Replica Configuration

Temporarily reduce replicas during initial indexing:
```json
PUT /my_index/_settings
{
  "index": {
    "number_of_replicas": 0
  }
}
```

Restore replicas after indexing:
```json
PUT /my_index/_settings
{
  "index": {
    "number_of_replicas": 1
  }
}
```

### Segment Management

Optimize segment handling for better performance:

#### Force Merge

Perform force merge on read-only indices:
```json
POST /my_index/_forcemerge?max_num_segments=1
```

Force merge recommendations:
- Only use on read-only or rarely-updated indices
- Schedule during off-peak hours
- Merge to a single segment for optimal read performance

#### Segment Merging

Configure merge policy:
```json
PUT /my_index/_settings
{
  "index": {
    "merge.policy.expunge_deletes_allowed": 20,
    "merge.policy.floor_segment": "2mb",
    "merge.policy.max_merge_at_once": 10,
    "merge.policy.max_merged_segment": "5gb"
  }
}
```

## Query Optimization

### Search Performance

Optimize search queries for better performance:

#### Query vs. Filter Context

Use filter context for criteria that don't affect relevance:

```json
# Less efficient
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "laptop" } },
        { "term": { "status": "active" } },
        { "range": { "price": { "gte": 500, "lte": 1000 } } }
      ]
    }
  }
}

# More efficient
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "laptop" } }
      ],
      "filter": [
        { "term": { "status": "active" } },
        { "range": { "price": { "gte": 500, "lte": 1000 } } }
      ]
    }
  }
}
```

Benefits of filter context:
- Filters are cached
- No relevance score calculation
- Often faster execution

#### Field Selection

Retrieve only needed fields:
```json
GET /products/_search
{
  "query": { "match": { "name": "laptop" } },
  "_source": ["name", "price", "category"]
}
```

Or exclude large fields:
```json
GET /products/_search
{
  "query": { "match": { "name": "laptop" } },
  "_source": {
    "excludes": ["description", "specifications"]
  }
}
```

#### Search After for Deep Pagination

Use search_after instead of from/size for deep pagination:
```json
GET /products/_search
{
  "size": 100,
  "query": { "match_all": {} },
  "sort": [
    { "price": "asc" },
    { "_id": "asc" }
  ],
  "search_after": [500, "product_1000"]
}
```

#### Query Rewriting

Use more specific queries when possible:
```json
# Less efficient
GET /products/_search
{
  "query": {
    "match": {
      "status": "active"
    }
  }
}

# More efficient for exact matches
GET /products/_search
{
  "query": {
    "term": {
      "status.keyword": "active"
    }
  }
}
```

### Aggregation Performance

Optimize aggregations for large datasets:

#### Sampling

Use sampling for approximate aggregations:
```json
GET /products/_search
{
  "size": 0,
  "query": {
    "function_score": {
      "query": { "match_all": {} },
      "random_score": {},
      "boost_mode": "replace"
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

#### Field Selection

For range aggregations on text fields, create a numeric field:
```json
# Mapping with dedicated numeric field
"price_text": { "type": "text" },
"price": { "type": "double" }

# Use the numeric field for range aggregations
GET /products/_search
{
  "aggs": {
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          { "to": 100 },
          { "from": 100, "to": 200 },
          { "from": 200 }
        ]
      }
    }
  }
}
```

#### Limit Cardinality

Be cautious with high-cardinality fields in aggregations:
```json
# May be expensive on high-cardinality fields
GET /logs/_search
{
  "aggs": {
    "unique_ips": {
      "cardinality": {
        "field": "ip",
        "precision_threshold": 100
      }
    }
  }
}
```

#### Runtime Fields

Use runtime fields for occasional complex aggregations:
```json
GET /orders/_search
{
  "runtime_mappings": {
    "total_with_tax": {
      "type": "double",
      "script": {
        "source": "emit(doc['price'].value * 1.2)"
      }
    }
  },
  "aggs": {
    "average_with_tax": {
      "avg": {
        "field": "total_with_tax"
      }
    }
  }
}
```

### Script Performance

Optimize scripting for better performance:

#### Painless Scripts

Use Painless for best performance:
```json
GET /products/_search
{
  "query": {
    "script_score": {
      "query": { "match_all": {} },
      "script": {
        "source": "doc['price'].value / 100",
        "lang": "painless"
      }
    }
  }
}
```

#### Script Caching

Store and reuse scripts:
```json
PUT _scripts/price_discount
{
  "script": {
    "lang": "painless",
    "source": "doc['price'].value * params.discount_factor"
  }
}

GET /products/_search
{
  "query": {
    "script_score": {
      "query": { "match_all": {} },
      "script": {
        "id": "price_discount",
        "params": {
          "discount_factor": 0.9
        }
      }
    }
  }
}
```

#### Script Optimization

Optimize script logic:
- Access document fields directly with `doc['field'].value`
- Avoid accessing `_source` when possible
- Use simple operations and avoid complex loops
- Pre-compute values and store them in fields when possible

## Memory Management and Caching

### Cache Configuration

Configure caching for optimal performance:

#### Field Data Cache

Control field data loading:
```yaml
# elasticsearch.yml
indices.fielddata.cache.size: 30%
```

For specific indices:
```json
PUT /my_index/_settings
{
  "index": {
    "fielddata.cache.size": "10%"
  }
}
```

#### Query Cache

Configure the query cache size:
```yaml
# elasticsearch.yml
indices.queries.cache.size: 5%
```

Enable/disable for specific indices:
```json
PUT /my_index/_settings
{
  "index": {
    "queries.cache.enabled": true
  }
}
```

#### Request Cache

Configure the request cache:
```yaml
# elasticsearch.yml
indices.requests.cache.size: 2%
```

Use in searches:
```json
GET /my_index/_search?request_cache=true
{
  "size": 0,
  "aggs": {
    "popular_colors": {
      "terms": {
        "field": "color"
      }
    }
  }
}
```

### Circuit Breakers

Configure circuit breakers to prevent OOM errors:

```yaml
# elasticsearch.yml
indices.breaker.total.limit: 70%
indices.breaker.fielddata.limit: 40%
indices.breaker.request.limit: 60%
```

### Search Throttling

Control search resource usage:

```yaml
# elasticsearch.yml
indices.search.throttled: true
thread_pool.search.queue_size: 1000
```

For specific indices:
```json
PUT /my_index/_settings
{
  "index": {
    "search.throttled": true
  }
}
```

## Advanced Performance Techniques

### Index Sorting

Sort indices for faster range queries:

```json
PUT /sorted_index
{
  "settings": {
    "index": {
      "sort.field": "timestamp",
      "sort.order": "desc"
    }
  },
  "mappings": {
    "properties": {
      "timestamp": { "type": "date" }
    }
  }
}
```

Benefits:
- Faster range queries on the sort field
- More efficient segment merging
- Improved compression

### Sequence IDs

Use sequence IDs for efficient updates:

```json
GET /products/_doc/1?_source=false&_seq_no=true&_primary_term=true

PUT /products/_doc/1?if_seq_no=10&if_primary_term=2
{
  "updated_field": "new value"
}
```

Benefits:
- Prevents update conflicts
- More efficient than versioning

### Adaptive Replica Selection

Enable adaptive replica selection:

```yaml
# elasticsearch.yml
cluster.routing.use_adaptive_replica_selection: true
```

This selects the best-performing replica based on:
- Response time
- Queue size
- Service time

### Search Profiling

Use the Profile API to identify slow queries:

```json
GET /products/_search
{
  "profile": true,
  "query": {
    "match": {
      "description": "smartphone"
    }
  }
}
```

The response includes detailed timing information for each query component.

### Index Analysis

Analyze index performance:

```json
GET /products/_stats?level=shards

GET /_cat/indices?v&h=index,docs.count,store.size,pri.store.size,pri,rep

GET /_cat/shards?v
```

## Monitoring and Benchmarking

### Key Performance Metrics

Monitor these key metrics:

#### Cluster Health

```
GET /_cluster/health

GET /_cat/health?v
```

Key health indicators:
- Status (green, yellow, red)
- Number of nodes
- Unassigned shards
- Pending tasks

#### Node Statistics

```
GET /_nodes/stats

GET /_cat/nodes?v&h=id,ip,name,heap.percent,cpu,load_1m,disk.total,disk.used_percent,node.role
```

Key node metrics:
- CPU usage
- JVM heap usage
- Disk space
- Query load
- Indexing throughput

#### Indexing Performance

```
GET /_nodes/stats/indices/indexing

GET /_cat/indices?v&h=index,docs.count,docs.deleted,pri.store.size,pri.search.query_current,pri.indexing.index_total
```

Key indexing metrics:
- Indexing rate
- Indexing latency
- Merge activity
- Refresh times

#### Search Performance

```
GET /_nodes/stats/indices/search

GET /_cat/thread_pool?v&h=node_name,name,active,queue,rejected
```

Key search metrics:
- Query rate
- Query latency
- Fetch phase time
- Thread pool rejections

### Monitoring Tools

Use these tools for comprehensive monitoring:

#### X-Pack Monitoring

Enable X-Pack monitoring:
```yaml
# elasticsearch.yml
xpack.monitoring.collection.enabled: true
```

View in Kibana under Stack Monitoring.

#### Metricbeat

Configure Metricbeat for Elasticsearch monitoring:
```yaml
metricbeat.modules:
- module: elasticsearch
  metricsets:
    - node
    - node_stats
    - index
    - index_recovery
    - shard
  period: 10s
  hosts: ["http://localhost:9200"]
```

#### Prometheus and Grafana

Use the Prometheus exporter:
```yaml
# elasticsearch.yml
prometheus.metric.enabled: true
```

### Performance Testing

Benchmark tools for Elasticsearch:

#### Rally

Elastic's official benchmarking tool:
```bash
esrally --distribution-version=8.6.0 --track=geonames
```

#### JMeter

Create custom load tests with JMeter for specific workloads.

#### Custom Benchmarking

Create benchmark scripts that simulate your workload:
```python
import time
import elasticsearch
from elasticsearch import helpers

es = elasticsearch.Elasticsearch(["localhost:9200"])

start_time = time.time()
docs = [{"_index": "test", "_source": {"field": "value"}} for _ in range(100000)]
helpers.bulk(es, docs)
end_time = time.time()

print(f"Indexed 100,000 docs in {end_time - start_time} seconds")
```

## Performance Troubleshooting

### Common Issues and Solutions

#### Slow Queries

Identify slow queries:
```
GET /_nodes/stats/indices/search?human

PUT /my_index/_settings
{
  "index.search.slowlog.threshold": {
    "query.warn": "5s",
    "query.info": "2s"
  }
}
```

Solutions:
- Use filters instead of queries for exact matches
- Add appropriate field-level optimizations
- Consider using more specific queries
- Review and optimize complex aggregations

#### High CPU Usage

Identify CPU-intensive operations:
```
GET /_nodes/hot_threads
```

Solutions:
- Scale horizontally by adding more nodes
- Review and optimize expensive queries
- Check for inefficient scripts or aggregations
- Adjust thread pool settings

#### Memory Pressure

Check memory usage:
```
GET /_cat/nodes?v&h=name,id,heap.percent,heap.current,heap.max
```

Solutions:
- Adjust JVM heap size
- Check field data usage and limit expensive aggregations
- Configure circuit breakers appropriately
- Consider adding more nodes

#### Shard Issues

Identify problem shards:
```
GET /_cat/shards?v
```

Solutions:
- Adjust shard allocation settings
- Rebalance shards with allocation filtering
- For oversized shards, consider reindexing with more shards
- For too many small shards, consider force merging or reindexing

#### Disk I/O Bottlenecks

Check disk I/O:
```
GET /_nodes/stats/fs
```

Solutions:
- Use SSDs for better performance
- Consider a hot-warm architecture
- Adjust merge policy settings
- Use index sorting for more efficient merges

### Diagnostic Procedures

#### Thread Pool Analysis

Check thread pool status:
```
GET /_cat/thread_pool?v

GET /_nodes/stats/thread_pool
```

Look for:
- High queue size
- Rejected requests
- Exhausted pools

#### Shard Analysis

Analyze shard health:
```
GET /_cat/shards?v

GET /_cluster/allocation/explain
```

#### Hot Threads

Identify CPU-intensive threads:
```
GET /_nodes/hot_threads?threads=10&interval=1s
```

#### Profile API

Profile specific queries:
```json
GET /my_index/_search
{
  "profile": true,
  "query": {
    "match": {
      "field": "value"
    }
  }
}
```

## Conclusion

Optimizing Elasticsearch performance requires a multi-faceted approach, addressing hardware, cluster design, indexing strategies, and query patterns. By applying the techniques covered in this chapter, you can significantly improve your Elasticsearch deployment's speed, efficiency, and reliability.

Remember that performance optimization is an ongoing process:
1. Establish performance baselines
2. Monitor key metrics
3. Identify bottlenecks
4. Apply targeted optimizations
5. Measure improvements
6. Repeat as your data and query patterns evolve

In the next chapter, we'll explore Elasticsearch security features to ensure your optimized cluster is also properly protected.