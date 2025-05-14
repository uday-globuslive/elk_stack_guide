# Scaling Elasticsearch

## Introduction to Elasticsearch Scaling

Scaling is a critical aspect of Elasticsearch architecture, ensuring your deployment can handle increasing data volumes, query loads, and user concurrency while maintaining performance and reliability. This chapter explores comprehensive strategies for scaling Elasticsearch effectively, along with practical implementations and considerations.

Elasticsearch offers several approaches to scaling:

- **Vertical Scaling**: Adding more resources to existing nodes
- **Horizontal Scaling**: Adding more nodes to the cluster
- **Functional Scaling**: Separating different workloads and node roles
- **Data Distribution**: Using sharding and routing strategies
- **Cross-Cluster Scaling**: Federated search and replication across clusters

Understanding these approaches will help you design an Elasticsearch architecture that meets your performance goals and can grow with your needs.

## Vertical Scaling

Vertical scaling involves adding more resources (CPU, memory, disk, or network bandwidth) to existing Elasticsearch nodes.

### When to Consider Vertical Scaling

- Early stages of deployment with limited data
- Simpler operational overhead (fewer nodes to manage)
- When horizontal scaling introduces too much coordination overhead
- When a specific resource is the bottleneck (e.g., memory for aggregations)

### Memory Considerations

Memory is often the most critical resource for Elasticsearch performance:

#### JVM Heap Sizing

- Default heap size is usually too small for production
- General guideline: allocate 50% of available RAM to the JVM heap, but not more than 31GB
- Set min and max heap size to the same value to prevent heap resizing

```yaml
# Example JVM configuration
-Xms16g
-Xmx16g
```

#### Memory Use Cases

- **Field data cache**: Needed for aggregations on text fields
- **Node query cache**: Caches query results at the node level
- **Shard request cache**: Caches shard-level query results
- **Indexing buffer**: Memory buffer for indexing operations
- **OS file cache**: Operating system's cache for file access

Proper memory allocation example:
```yaml
# elasticsearch.yml
indices.fielddata.cache.size: 20%  # Limit fielddata cache size
indices.memory.index_buffer_size: 10%  # Limit indexing buffer
```

### CPU Considerations

CPU affects search and indexing throughput:

- More CPU cores allow for more concurrent operations
- Thread pools in Elasticsearch are sized relative to available CPU cores
- Consider CPU architecture and generation (newer CPUs can be significantly faster)

Optimizing thread pools:
```yaml
# elasticsearch.yml
thread_pool.write.size: 16  # Increase write thread pool size
thread_pool.search.size: 12  # Increase search thread pool size
```

### Storage Considerations

Storage performance directly impacts indexing and search:

- **SSDs vs. HDDs**: SSDs provide significantly better performance
- **RAID configuration**: RAID 0 offers better performance (use replication for redundancy)
- **I/O operations per second (IOPS)**: Higher IOPS improves performance
- **Throughput**: Greater throughput benefits bulk operations

### Network Considerations

Network can become a bottleneck in distributed systems:

- **Network Interface Card (NIC)**: 10 Gbps or faster recommended for production
- **Latency**: Lower latency improves coordination between nodes
- **Bandwidth**: Higher bandwidth improves data transfer rates

### Limits of Vertical Scaling

Vertical scaling has inherent limitations:

- Hardware constraints (maximum memory per server)
- JVM limitations (heap sizes beyond 31GB may cause inefficient garbage collection)
- Diminishing returns on investment
- Single point of failure risk
- Limited node-level redundancy

## Horizontal Scaling

Horizontal scaling involves adding more nodes to your Elasticsearch cluster to distribute the workload.

### When to Consider Horizontal Scaling

- When approaching resource limits on individual nodes
- To improve fault tolerance and availability
- When data volume exceeds what a single node can reasonably handle
- To handle increased query volume and concurrent users

### Cluster Size Planning

Determining the right cluster size depends on several factors:

- **Data volume**: Total size of indices including replicas
- **Expected growth**: Anticipated data growth over time
- **Query patterns**: Types and frequency of queries
- **Indexing patterns**: Volume and frequency of document indexing
- **Redundancy requirements**: Number of replica shards needed

#### Node Capacity Calculation Example

For time-series data like logs:

```
Daily index size = 100GB
Retention period = 30 days
Total primary data = 3TB
Replication factor = 1 (one replica per primary)
Total cluster data = 6TB

If each node has 2TB storage:
Minimum nodes needed = 6TB / 2TB = 3 nodes
For fault tolerance: 4-5 nodes recommended
```

### Shard Allocation

Proper shard allocation is critical for balanced horizontal scaling:

- **Primary Shards**: Define the number of primary shards when creating an index (default is 1)
- **Replica Shards**: Define the number of replica shards for redundancy and read scaling

```
# Create an index with custom shard settings
PUT /my_index
{
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1
  }
}
```

#### Shard Sizing Guidelines

- **Target shard size**: 20-40GB per shard (avoid both too small and too large shards)
- **Shard count formula**: `(daily_data * retention_days) / target_shard_size`
- **Shards per node**: Roughly 20-25 shards per GB of heap memory (including replicas)
- **Total shard count**: Consider the entire cluster's capacity to handle coordination

### Cluster Expansion

Adding nodes to an existing cluster involves several steps:

1. **Prepare the new node**: Install Elasticsearch with compatible version
2. **Configure the node**: Set appropriate settings and roles
3. **Add to the cluster**: Configure discovery settings to join the cluster
4. **Monitor rebalancing**: Elasticsearch will automatically rebalance shards

```yaml
# elasticsearch.yml for a new node
cluster.name: my-production-cluster
node.name: node-4
discovery.seed_hosts: ["node-1", "node-2", "node-3"]
```

### Shard Rebalancing

Control shard rebalancing behavior during scaling:

```yaml
# elasticsearch.yml
# Number of concurrent rebalancing operations
cluster.routing.allocation.cluster_concurrent_rebalance: 2

# Percentage threshold to trigger rebalancing
cluster.routing.allocation.balance.threshold: 3.0

# Delay before starting rebalance after node joins/leaves
cluster.routing.allocation.node_concurrent_recoveries: 2
```

### Routing and Allocation Filtering

Direct shard allocation to specific nodes:

```
# Allocate index to specific nodes
PUT /my_index/_settings
{
  "index.routing.allocation.require.box_type": "hot"
}

# Exclude a node from allocation
PUT /my_index/_settings
{
  "index.routing.allocation.exclude._name": "node-3"
}
```

## Functional Scaling

Functional scaling separates different workloads onto specialized nodes based on their roles and resource requirements.

### Node Roles and Specialization

Elasticsearch supports different node roles:

- **Master-eligible nodes**: Manage cluster state and coordination
- **Data nodes**: Store and search data
- **Ingest nodes**: Pre-process documents before indexing
- **Coordinating-only nodes**: Route requests and aggregate results
- **Machine learning nodes**: Run machine learning jobs

Configure nodes with specific roles:

```yaml
# Master node configuration
node.roles: [ master ]

# Data node configuration
node.roles: [ data ]

# Ingest node configuration
node.roles: [ ingest ]

# Coordinating-only node (no roles)
node.roles: [ ]

# Combined master-eligible and data node
node.roles: [ master, data ]
```

### Hot-Warm-Cold Architecture

Implement a data lifecycle architecture based on access patterns:

![Hot-Warm-Cold Architecture](https://i.imgur.com/tOW2Vv8.png)

- **Hot nodes**: Store recent, frequently accessed data (SSD-based)
- **Warm nodes**: Store older, less frequently accessed data (HDD-based)
- **Cold nodes**: Store historical, rarely accessed data (optimized for storage cost)
- **Frozen tier**: Archive data, loaded from snapshot on demand

Configure node attributes:

```yaml
# Hot node configuration
node.attr.data: "hot"

# Warm node configuration
node.attr.data: "warm"

# Cold node configuration
node.attr.data: "cold"
```

Use Index Lifecycle Management (ILM) to move indices between tiers:

```
PUT _ilm/policy/time_series_policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_size": "50GB",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": {
            "number_of_shards": 1
          },
          "forcemerge": {
            "max_num_segments": 1
          },
          "allocate": {
            "require": {
              "data": "warm"
            }
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "allocate": {
            "require": {
              "data": "cold"
            }
          },
          "freeze": {}
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

### Search/Indexing Specialization

Optimize nodes for specific workloads:

- **Search-focused nodes**: More CPU and memory for complex queries
- **Indexing-focused nodes**: More I/O capacity and CPU for document processing
- **Aggregation-focused nodes**: More memory for compute-intensive operations

Configure nodes with attributes to target specific workloads:

```yaml
# Search-optimized node
node.attr.workload: "search"

# Indexing-optimized node
node.attr.workload: "indexing"
```

Target indices to specific nodes:

```
PUT /search_heavy_index/_settings
{
  "index.routing.allocation.require.workload": "search"
}

PUT /indexing_heavy_index/_settings
{
  "index.routing.allocation.require.workload": "indexing"
}
```

## Data Distribution Strategies

Effective data distribution is crucial for balanced scaling and query performance.

### Sharding Strategies

#### Primary Shard Calculation

Calculate primary shards based on data volume:

```
Expected data size = 500GB
Target shard size = 25GB
Primary shards needed = 500GB / 25GB = 20 shards
```

#### Replica Shard Calculation

Calculate replicas based on redundancy and query load:

```
Primary shards = 20
For 1 replica: Total shards = 20 * 2 = 40
For 2 replicas: Total shards = 20 * 3 = 60
```

#### Time-Based Indices

Use time-based indices for time-series data:

```
# Create index template for time-based indices
PUT _index_template/logs_template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1
    }
  }
}

# Create initial index
PUT /logs-2023.01.01-000001
{
  "aliases": {
    "logs": {
      "is_write_index": true
    }
  }
}
```

### Routing

Control shard routing for better data distribution and query performance:

#### Custom Routing

Direct related documents to the same shard:

```
# Index with custom routing
POST /my_index/_doc?routing=user123
{
  "user_id": "user123",
  "message": "Document content"
}

# Search with custom routing
GET /my_index/_search?routing=user123
{
  "query": {
    "match": {
      "user_id": "user123"
    }
  }
}
```

#### Routing by User/Tenant

For multi-tenant applications, route by tenant:

```python
# Python example of tenant-based routing
def index_document(es_client, tenant_id, document):
    es_client.index(
        index="shared_index",
        document=document,
        routing=tenant_id
    )
```

### Aliases and Rollover

Use aliases and rollover for efficient index management:

```
# Create an alias pointing to multiple indices
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "logs-2023.01.*",
        "alias": "recent_logs"
      }
    }
  ]
}

# Configure automatic rollover
PUT _ilm/policy/rollover_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "50GB",
            "max_age": "1d"
          }
        }
      }
    }
  }
}
```

### Data Streams

Use data streams for time-series data:

```
# Create a data stream
PUT _data_stream/logs-app

# Index into a data stream (no need to specify index name)
POST logs-app/_doc
{
  "@timestamp": "2023-01-15T14:12:00Z",
  "message": "Application event"
}
```

## Cross-Cluster Scaling

For very large deployments, scaling across multiple clusters provides additional benefits.

### Cross-Cluster Replication (CCR)

Replicate indices across clusters for geographical distribution or disaster recovery:

```
# Configure remote cluster
PUT /_cluster/settings
{
  "persistent": {
    "cluster.remote.cluster_two.seeds": [
      "cluster2-node1:9300",
      "cluster2-node2:9300"
    ]
  }
}

# Set up follower index
PUT /follower_index/_ccr/follow
{
  "remote_cluster": "cluster_two",
  "leader_index": "leader_index"
}
```

### Cross-Cluster Search (CCS)

Search across multiple clusters:

```
# Search across local and remote cluster
GET /local_index,cluster_two:remote_index/_search
{
  "query": {
    "match": {
      "field": "value"
    }
  }
}
```

### Cross-Cluster Considerations

When implementing cross-cluster architectures:

- **Network latency**: Higher latency between clusters affects performance
- **Bandwidth requirements**: Sufficient bandwidth needed for replication
- **Consistency model**: Follower indices have eventual consistency
- **Security configuration**: Configure security across cluster boundaries
- **Disaster recovery planning**: Design for cluster or region failures

## Monitoring and Optimization

Effective scaling requires ongoing monitoring and optimization.

### Key Metrics to Monitor

- **Cluster Health**: Overall status (green, yellow, red)
- **Node Statistics**: CPU, memory, disk, and load averages
- **JVM Metrics**: Heap usage, garbage collection frequency and duration
- **Indexing Rates**: Documents per second, indexing latency
- **Search Rates**: Queries per second, query latency
- **Cache Statistics**: Hit rates for various caches
- **Thread Pool Queue Size**: For various operations (search, write, etc.)

### Monitoring with Elasticsearch APIs

```bash
# Cluster health
curl -X GET "localhost:9200/_cluster/health?pretty"

# Nodes stats
curl -X GET "localhost:9200/_nodes/stats?pretty"

# Index stats
curl -X GET "localhost:9200/_stats?pretty"

# Thread pool stats
curl -X GET "localhost:9200/_nodes/stats/thread_pool?pretty"
```

### Monitoring with Metricbeat

Automate monitoring with Metricbeat:

```yaml
metricbeat.modules:
- module: elasticsearch
  metricsets:
    - node
    - node_stats
    - index
    - index_summary
    - shard
  period: 10s
  hosts: ["http://localhost:9200"]
```

### Optimization Techniques

#### Index Optimization

Optimize indices for better performance:

```
# Force merge for read-only indices
POST /my_index/_forcemerge?max_num_segments=1

# Refresh interval adjustment
PUT /my_index/_settings
{
  "index": {
    "refresh_interval": "30s"
  }
}
```

#### Query Optimization

Improve query performance:

```
# Use filter context for better caching
GET /my_index/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "status": "active" } }
      ],
      "must": [
        { "match": { "description": "important document" } }
      ]
    }
  }
}

# Use routing for faster queries
GET /my_index/_search?routing=user123
{
  "query": {
    "match": {
      "user_id": "user123"
    }
  }
}
```

#### Caching Configuration

Tune caching for workload patterns:

```yaml
# elasticsearch.yml
indices.queries.cache.size: 5%
indices.memory.index_buffer_size: 15%
```

## Scaling Patterns and Case Studies

### Pattern: High-Volume Logging

For high-volume logging scenarios:

- Use time-based indices with rollover
- Implement hot-warm-cold architecture
- Focus on indexing performance (bulk operations, fewer replicas initially)
- Consider index lifecycle management for retention

```
Daily Volume: 500GB
Retention: 30 days
Total Data: 15TB

Architecture:
- 4 hot nodes (SSD, 2TB each): 0-7 days data
- 6 warm nodes (HDD, 4TB each): 8-30 days data
- Data streams with ILM for automated management
```

### Pattern: Search-Heavy Applications

For search-heavy applications:

- Prioritize query performance with more replicas
- Use dedicated search nodes with more CPU and memory
- Implement aggressive caching strategies
- Consider search-optimized hardware (more CPU cores)

```
Index Size: 200GB
Query Load: 500 requests per second

Architecture:
- 3 master-eligible nodes
- 4 data nodes with indices
- 6 search-optimized coordinating nodes
- Cross-cluster search for federated results
```

### Pattern: Time-Series Analytics

For time-series analytics workloads:

- Optimize for both write and aggregation performance
- Use data streams or time-based indices
- Implement downsampling for historical data
- Consider specialized nodes for heavyweight aggregations

```
Architecture:
- Dedicated ingest nodes for data preprocessing
- Hot nodes for recent data (high query load)
- Warm nodes for historical aggregations
- Automated downsampling with transforms
```

### Case Study: E-commerce Platform

An e-commerce platform scaling journey:

1. **Initial setup**: Single-node Elasticsearch for product catalog
2. **First scaling**: Vertical scaling to handle more products
3. **Search demand growth**: Added dedicated search nodes
4. **Data growth**: Implemented horizontal scaling with more data nodes
5. **Global expansion**: Deployed cross-cluster replication for regional searching
6. **Monitoring**: Implemented comprehensive monitoring and alerting

Final architecture:
- 3 master-eligible nodes for cluster management
- 12 data nodes for product catalog and user data
- 8 search-optimized nodes for handling query load
- Cross-cluster replication between 3 regional clusters
- Metricbeat and Elasticsearch monitoring stack

## Conclusion

Scaling Elasticsearch effectively requires a thoughtful approach that balances multiple factors including data volume, query patterns, and resource constraints. By understanding the various scaling strategies—vertical, horizontal, functional, and cross-cluster—you can design an architecture that meets your current needs while providing a clear path for future growth.

The most successful Elasticsearch deployments often combine multiple scaling approaches, such as using specialized node roles within a horizontally scaled cluster, or implementing a hot-warm-cold architecture with cross-cluster capabilities for disaster recovery.

Regular monitoring and optimization remain essential for maintaining performance as your deployment grows, allowing you to identify and address potential bottlenecks before they impact users.

In the next chapter, we'll explore high availability strategies for Elasticsearch to ensure your scaled deployment remains reliable and resilient.