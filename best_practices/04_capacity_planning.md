# Capacity Planning for the ELK Stack

This chapter covers best practices for capacity planning, resource allocation, and scaling strategies for the ELK Stack.

## Table of Contents
- [Introduction to Capacity Planning](#introduction-to-capacity-planning)
- [Resource Requirements](#resource-requirements)
- [Sizing Elasticsearch Clusters](#sizing-elasticsearch-clusters)
- [Sizing Logstash Deployments](#sizing-logstash-deployments)
- [Sizing Kibana Deployments](#sizing-kibana-deployments)
- [Sizing Beats Deployments](#sizing-beats-deployments)
- [Storage Planning](#storage-planning)
- [Network Capacity Planning](#network-capacity-planning)
- [Growth Planning](#growth-planning)
- [Performance Benchmarking](#performance-benchmarking)
- [Capacity Planning Tools](#capacity-planning-tools)
- [Multi-Tier Architectures](#multi-tier-architectures)
- [Cloud vs. On-Premises Considerations](#cloud-vs-on-premises-considerations)

## Introduction to Capacity Planning

Proper capacity planning is essential for ensuring your ELK Stack deployment has adequate resources to handle your workloads efficiently, maintain performance, and accommodate future growth. This chapter provides comprehensive guidance for planning and allocating resources across all components of the ELK Stack.

### Why Capacity Planning Matters

Effective capacity planning helps:

1. **Ensure performance and stability**: Properly sized systems provide consistent performance and reduce outages.
2. **Optimize costs**: Right-sized infrastructure minimizes wasted resources.
3. **Plan for growth**: Anticipate future needs and scale smoothly.
4. **Prevent bottlenecks**: Identify and address resource constraints before they impact users.
5. **Improve user experience**: Maintain query responsiveness and indexing throughput.

### Capacity Planning Methodology

A structured approach to capacity planning includes:

1. **Requirements gathering**: Understand workload patterns, data volumes, and performance expectations.
2. **Workload characterization**: Analyze the nature of your workload (search-heavy, ingest-heavy, or balanced).
3. **Resource estimation**: Calculate initial resource requirements based on workload.
4. **Component sizing**: Size each component of the ELK Stack appropriately.
5. **Validation**: Test with representative workloads to verify performance.
6. **Growth planning**: Project future resource needs based on expected growth.
7. **Monitoring and adjustment**: Continuously monitor and adjust as needed.

## Resource Requirements

### Memory Requirements

Memory is often the most critical resource for the ELK Stack, particularly for Elasticsearch.

#### Elasticsearch Memory Guidelines

- **JVM Heap**: Set to 50% of available RAM, but no more than 31GB per node.
- **System memory**: The remaining 50% is used for the operating system, page cache, and other processes.

```yaml
# Example Elasticsearch JVM configuration
-Xms16g
-Xmx16g
```

**Minimum memory recommendations**:

| Elasticsearch Node Type | Minimum RAM | Recommended RAM |
|-------------------------|-------------|-----------------|
| Master nodes            | 8GB         | 16-32GB         |
| Data nodes              | 16GB        | 32-64GB         |
| Ingest nodes            | 8GB         | 16-32GB         |
| Coordinating nodes      | 8GB         | 16-32GB         |
| Machine Learning nodes  | 16GB        | 32-64GB         |

#### Logstash Memory Guidelines

- **JVM Heap**: Typically 4GB is sufficient for most deployments.
- **System memory**: Additional memory for the operating system.

```yaml
# Example Logstash JVM configuration
-Xms4g
-Xmx4g
```

**Factors affecting Logstash memory needs**:
- Number and complexity of pipelines
- Event size and throughput
- Plugin usage, particularly aggregation plugins
- Queue size for persistent queues

#### Kibana Memory Guidelines

- **Node.js memory**: Default limit is 1GB, but should be adjusted based on usage.
- **Memory per user**: ~100-200MB per concurrent user is a reasonable starting point.

```yaml
# Example Kibana memory configuration in kibana.yml
server.maxPayloadBytes: 5242880
```

```bash
# Setting Node.js memory limit
NODE_OPTIONS="--max-old-space-size=4096"
```

### CPU Requirements

CPU needs vary across components and workloads.

#### Elasticsearch CPU Guidelines

- **Rule of thumb**: 1-2 CPU cores per 32GB of JVM heap.
- **Search-heavy workloads**: More CPU cores benefit search operations.
- **Ingest-heavy workloads**: More CPU cores benefit indexing operations.

**Recommended CPU cores**:

| Elasticsearch Node Type | Minimum vCPUs | Recommended vCPUs |
|-------------------------|---------------|-------------------|
| Master nodes            | 4             | 8-16              |
| Data nodes              | 8             | 16-32             |
| Ingest nodes            | 4             | 8-16              |
| Coordinating nodes      | 4             | 8-16              |
| Machine Learning nodes  | 8             | 16-32             |

#### Logstash CPU Guidelines

- **Rule of thumb**: 1 CPU core per 1,000 events per second (EPS).
- **Workers**: Typically set to the number of CPU cores available.

```ruby
# Example Logstash pipeline configuration
pipeline.workers: 8
```

**Factors affecting Logstash CPU needs**:
- Event throughput
- Complexity of filtering operations
- Number of pipeline workers
- Use of CPU-intensive plugins (like grok patterns)

#### Kibana CPU Guidelines

- **Minimum**: 2 CPU cores for basic deployments.
- **Recommended**: 4-8 CPU cores for production deployments.
- **Heavy visualization**: More CPU cores benefit complex dashboards.

### Storage Requirements

Storage capacity and performance are critical for Elasticsearch.

#### Elasticsearch Storage Guidelines

- **Raw data size**: Base calculation on expected data volume.
- **Indexing overhead**: Primary shards typically require 10-15% more space than raw data.
- **Replication**: Each replica multiplies storage needs.
- **Buffer**: Add 20-30% additional capacity for growth and operations.

**Storage capacity formula**:
```
Total storage = (Raw data size × 1.15 × (1 + number of replicas)) × 1.3
```

**Storage types**:
- **SSD/NVMe**: Recommended for all production deployments.
- **SAN/NAS**: Usable but may introduce performance and reliability issues.
- **HDD**: Acceptable only for cold data or archive nodes.

#### Logstash Storage Guidelines

- **Persistent queues**: If enabled, require storage space.
- **Log files**: Storage for Logstash's own logs.

**Storage formula for persistent queues**:
```
Queue storage = Queue memory size × Number of pipelines × Buffer factor (1.5)
```

#### Beats Storage Guidelines

- **Registry files**: Storage for tracking file positions.
- **Buffer files**: Temporary storage for events during network outages.

## Sizing Elasticsearch Clusters

### Node Count and Roles

Proper node count and role allocation are essential for balanced clusters.

#### Master Nodes

- **Recommendation**: 3 dedicated master nodes for production (always an odd number).
- **Memory**: 8-32GB, depending on cluster size.
- **Storage**: 50-100GB SSD for logs and state.

```yaml
# Master node configuration
node.roles: [ master ]
```

#### Data Nodes

- **Calculation**: Based on total data size, shard count, and query volume.
- **Formula**: 
  ```
  Number of data nodes = Ceiling(Total storage needed / Storage per node)
  ```
- **Shard allocation**: 20-30 shards per GB of heap memory.

```yaml
# Data node configuration
node.roles: [ data ]
```

#### Ingest Nodes

- **Calculation**: Based on ingest volume and preprocessing complexity.
- **Formula**:
  ```
  Number of ingest nodes = Ceiling(Total events per second / 5,000)
  ```

```yaml
# Ingest node configuration
node.roles: [ ingest ]
```

#### Coordinating Nodes

- **Calculation**: Based on query volume and complexity.
- **Formula**:
  ```
  Number of coordinating nodes = Ceiling(Concurrent queries / 10)
  ```

```yaml
# Coordinating node configuration (no roles)
node.roles: [ ]
```

### Shard Sizing and Count

Proper shard sizing is critical for performance.

#### Shard Size Guidelines

- **Optimal shard size**: Between 20GB and 40GB.
- **Maximum shards per node**: ~20-25 shards per GB of heap memory.

**Shard count formula**:
```
Number of primary shards = Ceiling(Expected index size / Optimal shard size)
```

#### Index Sizing Examples

| Index Size | Optimal Primary Shards | Configuration |
|------------|------------------------|---------------|
| 10GB       | 1                      | 1 primary shard |
| 50GB       | 2-3                    | 2 primary shards |
| 100GB      | 3-5                    | 3 primary shards |
| 500GB      | 13-25                  | 20 primary shards |
| 1TB        | 25-50                  | 30 primary shards |

```json
// Example index creation with calculated shard count
PUT /logs-2023.09.01
{
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1
  }
}
```

### Replica Configuration

Replicas provide redundancy and improve search performance.

- **Minimum recommendation**: 1 replica for production environments.
- **High availability**: 2 replicas for critical data.
- **Search-intensive**: Additional replicas can improve search throughput.

**Storage impact formula**:
```
Total shards = Primary shards × (1 + Number of replicas)
```

```json
// Example of adjusting replica count
PUT /logs-2023.09.01/_settings
{
  "number_of_replicas": 1
}
```

### Cluster Examples

#### Small Cluster (≤ 100GB data)

- 3 combined master/data nodes
- 16GB RAM per node
- 4-8 CPU cores per node
- 50GB storage per node
- 1 replica for redundancy

```yaml
# Small cluster node configuration
node.roles: [ master, data ]
```

#### Medium Cluster (100GB-1TB data)

- 3 dedicated master nodes (8GB RAM, 4 cores)
- 5-10 data nodes (32GB RAM, 8-16 cores)
- 2 coordinating nodes (16GB RAM, 8 cores)
- 1 replica for redundancy

```yaml
# Medium cluster - master node configuration
node.roles: [ master ]

# Medium cluster - data node configuration
node.roles: [ data ]

# Medium cluster - coordinating node configuration
node.roles: [ ]
```

#### Large Cluster (1TB-10TB data)

- 3 dedicated master nodes (16GB RAM, 8 cores)
- 20-40 data nodes (64GB RAM, 16 cores)
- 5 coordinating nodes (32GB RAM, 16 cores)
- 3 ingest nodes (32GB RAM, 16 cores)
- 1-2 replicas for redundancy and performance

```yaml
# Large cluster roles are separated completely
```

#### Extra Large Cluster (10TB+ data)

- 3-5 dedicated master nodes (32GB RAM, 16 cores)
- 50+ data nodes, potentially with hot/warm architecture
- 10+ coordinating nodes
- 5+ ingest nodes
- Multi-zone deployment for high availability

## Sizing Logstash Deployments

### Instance Count Calculation

Determine the appropriate number of Logstash instances:

- **Based on events per second (EPS)**:
  ```
  Number of instances = Ceiling(Total EPS / EPS per instance)
  ```
  
- **EPS per instance** varies based on:
  - Event complexity
  - Filter operations
  - Hardware resources
  - Pipeline configuration

#### Logstash Throughput Benchmarks

| Configuration | Simple Events | Complex Events |
|---------------|---------------|----------------|
| 4 cores, 8GB RAM | 10,000-15,000 EPS | 3,000-7,000 EPS |
| 8 cores, 16GB RAM | 20,000-30,000 EPS | 7,000-15,000 EPS |
| 16 cores, 32GB RAM | 40,000-60,000 EPS | 15,000-30,000 EPS |

### Pipeline Configuration

Optimize Logstash pipelines for performance:

- **Workers**: Set to match available CPU cores.
  ```ruby
  pipeline.workers: 8
  ```
  
- **Batch size**: Increase for higher throughput.
  ```ruby
  pipeline.batch.size: 500
  ```
  
- **Multiple pipelines**: Separate high-volume or complex processing.
  ```yaml
  # pipelines.yml
  - pipeline.id: main
    pipeline.workers: 4
    pipeline.batch.size: 250
    queue.type: persisted
    path.config: "/etc/logstash/conf.d/main/*.conf"
  - pipeline.id: metrics
    pipeline.workers: 2
    pipeline.batch.size: 1000
    path.config: "/etc/logstash/conf.d/metrics/*.conf"
  ```

### Queue Sizing

Configure queue settings based on event flow reliability needs:

- **Memory queue**: Default, no persistence.
  ```ruby
  queue.type: memory
  queue.max_events: 1000
  ```

- **Persisted queue**: Recommended for production.
  ```ruby
  queue.type: persisted
  queue.max_bytes: 4gb
  path.queue: "/path/to/queue/directory"
  ```

### Logstash Deployment Examples

#### Small Deployment (≤ 5,000 EPS)

- 2 Logstash instances
- 4 CPU cores per instance
- 8GB RAM per instance
- Memory queues
- Single pipeline

#### Medium Deployment (5,000-20,000 EPS)

- 3-5 Logstash instances
- 8 CPU cores per instance
- 16GB RAM per instance
- Persistent queues
- Multiple pipelines for different data types

#### Large Deployment (20,000+ EPS)

- 6+ Logstash instances
- 16 CPU cores per instance
- 32GB RAM per instance
- Persistent queues
- Multiple pipelines with specific optimizations
- Load balancing across instances

## Sizing Kibana Deployments

### Instance Count Calculation

Determine the appropriate number of Kibana instances:

- **Based on concurrent users**:
  ```
  Number of instances = Ceiling(Total concurrent users / Users per instance)
  ```
  
- **Users per instance** varies based on:
  - Dashboard complexity
  - Refresh frequency
  - Data volume
  - Hardware resources

#### Kibana User Capacity Benchmarks

| Configuration | Simple Dashboards | Complex Dashboards |
|---------------|-------------------|-------------------|
| 2 cores, 4GB RAM | 20-30 users | 5-10 users |
| 4 cores, 8GB RAM | 40-60 users | 10-20 users |
| 8 cores, 16GB RAM | 80-120 users | 20-40 users |

### Memory Allocation

Configure memory settings based on usage patterns:

- **Node.js heap size**:
  ```bash
  NODE_OPTIONS="--max-old-space-size=4096"
  ```
  
- **Maximum payload size**:
  ```yaml
  server.maxPayloadBytes: 5242880
  ```

### Kibana Deployment Examples

#### Small Deployment (≤ 50 users)

- 2 Kibana instances
- 2 CPU cores per instance
- 4GB RAM per instance
- Load balancer for high availability

#### Medium Deployment (50-200 users)

- 3-4 Kibana instances
- 4 CPU cores per instance
- 8GB RAM per instance
- Load balancer with session affinity

#### Large Deployment (200+ users)

- 5+ Kibana instances
- 8 CPU cores per instance
- 16GB RAM per instance
- Load balancer with session affinity
- Caching layer for dashboards
- Separate reporting instances

## Sizing Beats Deployments

### Filebeat Sizing

Resource requirements for Filebeat instances:

- **CPU usage**: Generally lightweight, 5-10% of one core for standard deployments.
- **Memory usage**: 
  - Base: ~100MB
  - Per harvesters: ~2-4MB per active file being harvested
  - Registry size: Grows with the number of files

```yaml
# Filebeat configuration for resource control
filebeat.spool_size: 2048
filebeat.idle_timeout: "5s"
filebeat.registry.flush: 5s

# Limit active harvesters
filebeat.harvester.limit: 500
```

### Metricbeat Sizing

Resource requirements for Metricbeat instances:

- **CPU usage**: Varies based on collection interval and module count.
- **Memory usage**: ~100-200MB base plus ~10MB per module.

```yaml
# Metricbeat configuration for resource control
metricbeat.max_start_delay: "10s"

# Adjust collection frequency for resource control
metricbeat.modules:
- module: system
  metricsets: ["cpu", "memory"]
  period: "30s"
```

### Beats Deployment at Scale

For large environments:

- **Host resources**: Allocate ~0.5 CPU core and ~512MB RAM per server.
- **Network impact**: Configure appropriate batching and compression.
  ```yaml
  output.elasticsearch:
    bulk_max_size: 1000
    compression_level: 3
  ```
- **Load balancing**: Use multiple Elasticsearch or Logstash endpoints.
  ```yaml
  output.elasticsearch:
    hosts: ["es-1:9200", "es-2:9200", "es-3:9200"]
    loadbalance: true
  ```

## Storage Planning

### Capacity Estimation

Calculate storage needs based on data retention policies:

- **Raw data size**: Estimate the average size of raw logs or metrics.
- **Daily volume**: Calculate daily data volume.
  ```
  Daily volume = Events per day × Average event size
  ```
- **Total storage**: Calculate total storage needed based on retention.
  ```
  Total storage = Daily volume × Retention days × (1 + Replica factor) × Overhead factor (1.2)
  ```

#### Daily Volume Examples

| Data Source | Events/Day | Avg Size | Daily Volume |
|-------------|-----------|----------|--------------|
| Web servers | 50M | 500 bytes | 25GB |
| Application logs | 20M | 1KB | 20GB |
| Network devices | 10M | 300 bytes | 3GB |
| System metrics | 100M | 200 bytes | 20GB |

### Index Lifecycle Management (ILM)

Configure ILM to optimize storage use:

```json
PUT _ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_age": "1d",
            "max_size": "50GB"
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
          "allocate": {
            "require": {
              "data": "warm"
            }
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
            "require": {
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

### Storage Tiering

Implement storage tiering for cost-effective retention:

- **Hot tier**: High-performance SSDs for active indices.
- **Warm tier**: Balanced storage for less active data.
- **Cold tier**: Lower-cost storage for infrequently accessed data.
- **Frozen tier**: Archive storage for historical data.

```yaml
# Node configuration for hot tier
node.attr.data: hot

# Node configuration for warm tier
node.attr.data: warm

# Node configuration for cold tier
node.attr.data: cold
```

### Snapshot Strategy

Plan for backup storage:

- **Frequency**: Daily snapshots are typical.
- **Repository size**: 
  ```
  Snapshot repository size = Primary shard size × Snapshot frequency × Retention period
  ```
- **Incremental nature**: After the first full snapshot, incremental snapshots require less space.

```json
PUT _snapshot/backup_repository
{
  "type": "fs",
  "settings": {
    "location": "/path/to/backups"
  }
}
```

## Network Capacity Planning

### Bandwidth Requirements

Calculate bandwidth needs for different components:

#### Elasticsearch Inter-Node Traffic

- **Node-to-node traffic**: Depends on shard replication and cluster operations.
- **Formula**: 
  ```
  Bandwidth = Indexing rate × 2 × Document size × Replica factor
  ```

#### Logstash to Elasticsearch Traffic

- **Formula**:
  ```
  Bandwidth = Events per second × Average event size
  ```

#### Beats to Logstash/Elasticsearch Traffic

- **Formula**:
  ```
  Bandwidth = Agents × Events per agent per second × Average event size
  ```

#### Client to Kibana Traffic

- **Formula**:
  ```
  Bandwidth = Users × Dashboard size × Refresh frequency
  ```

### Network Topology Planning

Design network topology for optimal performance:

- **Co-locate related services**: Place Elasticsearch nodes in the same rack/zone when possible.
- **Separate node types**: Consider dedicated networks for client, replication, and management traffic.
- **Minimize latency**: Keep latency between Elasticsearch nodes under 5ms.

```yaml
# Elasticsearch network configuration
network.host: 0.0.0.0
transport.host: _site_
network.publish_host: 10.0.1.1
```

### Load Balancer Configuration

Configure load balancers appropriately:

- **Elasticsearch**: TCP load balancing for the REST API (port 9200).
- **Kibana**: HTTP/HTTPS load balancing with session persistence.
- **Logstash**: TCP load balancing for Beats input (typically port 5044).

## Growth Planning

### Data Growth Projection

Project data growth to plan for future capacity:

- **Historical growth rate**: Analyze past trends in data volume.
- **New data sources**: Account for planned additional sources.
- **Business growth**: Factor in business expansion plans.

**Growth formula**:
```
Future capacity = Current capacity × (1 + Monthly growth rate)^Months
```

### Capacity Expansion Strategy

Plan for smooth capacity expansion:

- **Elasticsearch**:
  - Add data nodes for storage and indexing capacity
  - Add dedicated nodes for specific roles
  - Shard rebalancing occurs automatically

- **Logstash**:
  - Add instances for increased throughput
  - Scale horizontally with load balancing

- **Kibana**:
  - Add instances for more concurrent users
  - Scale horizontally behind load balancer

### Growth Triggers

Establish metrics-based triggers for expansion:

- **CPU utilization**: Sustaining >70% utilization.
- **Memory pressure**: JVM heap usage consistently >75%.
- **Disk usage**: Reaching 70% of allocated storage.
- **Query latency**: Increasing beyond acceptable thresholds.
- **Indexing throughput**: Approaching maximum capacity.

## Performance Benchmarking

### Elasticsearch Benchmarking

Techniques for benchmarking Elasticsearch performance:

- **Rally**: Official Elasticsearch benchmark suite.
  ```bash
  esrally --distribution-version=7.10.0 --track=http_logs
  ```

- **Index benchmarking**:
  ```bash
  # Create a test index
  curl -X PUT "localhost:9200/test_index"
  
  # Bulk index test data
  curl -X POST "localhost:9200/test_index/_bulk" -H "Content-Type: application/json" --data-binary @bulk_data.json
  ```

- **Search benchmarking**:
  ```bash
  # Run test queries
  for i in {1..1000}; do
    curl -s -X GET "localhost:9200/test_index/_search" -H "Content-Type: application/json" -d'
    {
      "query": {
        "match": {
          "field": "value"
        }
      }
    }' > /dev/null
  done
  ```

### Logstash Benchmarking

Techniques for benchmarking Logstash performance:

- **Input generator plugin**:
  ```ruby
  input {
    generator {
      count => 1000000
      lines => ["line 1", "line 2", "line 3"]
      threads => 4
    }
  }
  output {
    null {}
  }
  ```

- **Benchmark filter**:
  ```ruby
  filter {
    ruby {
      code => "event.set('@benchmark_start', Time.now.to_f)"
    }
    # Your filters here
    ruby {
      code => "
        start = event.get('@benchmark_start')
        elapsed = Time.now.to_f - start
        event.set('elapsed_time', elapsed)
      "
    }
  }
  ```

### Full Stack Benchmarking

Test the entire ELK Stack:

- **Data generation**: Use tools like Gatling or custom scripts to generate realistic data flows.
- **End-to-end timing**: Measure from data generation to visualization.
- **Scenario testing**: Test various workload patterns (spikes, sustained load, etc.).

## Capacity Planning Tools

### Monitoring-Based Planning

Use monitoring data for capacity planning:

- **Elastic Stack Monitoring**: Built-in monitoring for all components.
  ```yaml
  # Enable monitoring in elasticsearch.yml
  xpack.monitoring.collection.enabled: true
  ```

- **Metricbeat**: Collect detailed metrics.
  ```yaml
  metricbeat.modules:
  - module: elasticsearch
    metricsets: ["node", "node_stats", "cluster_stats", "index"]
    period: 10s
    hosts: ["http://elasticsearch:9200"]
  ```

- **Prometheus and Grafana**: For advanced monitoring and visualization.

### Forecast-Based Planning

Use forecasting tools to predict resource needs:

- **Elasticsearch Machine Learning**:
  ```json
  PUT _ml/anomaly_detectors/disk_space_forecast
  {
    "description": "Forecast disk space usage",
    "analysis_config": {
      "bucket_span": "1h",
      "detectors": [
        {
          "detector_description": "disk usage",
          "function": "mean",
          "field_name": "disk.used_percent"
        }
      ]
    },
    "data_description": {
      "time_field": "@timestamp"
    }
  }
  ```

### Performance Analysis Tools

Tools for analyzing ELK Stack performance:

- **Profile API**: Detailed insight into query execution.
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

- **Hot Threads API**: Identify CPU-intensive operations.
  ```bash
  GET /_nodes/hot_threads
  ```

- **Task Management API**: Monitor long-running tasks.
  ```bash
  GET /_tasks
  ```

## Multi-Tier Architectures

### Hot-Warm-Cold Architecture

Implement tiered architecture for optimal resource usage:

- **Hot tier**: Active indexing and querying.
  ```yaml
  # Hot node configuration
  node.attr.data: hot
  ```
  
- **Warm tier**: Recent data, less querying.
  ```yaml
  # Warm node configuration
  node.attr.data: warm
  ```
  
- **Cold tier**: Historical data, infrequent access.
  ```yaml
  # Cold node configuration
  node.attr.data: cold
  ```
  
- **Frozen tier**: Archived data, minimal resources.
  ```yaml
  # Frozen node configuration
  node.attr.data: frozen
  ```

### Data Lifecycle Policies

Configure index lifecycle policies for each tier:

```json
PUT _ilm/policy/tiered-policy
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
        "min_age": "2d",
        "actions": {
          "allocate": {
            "require": {
              "data": "warm"
            }
          },
          "forcemerge": {
            "max_num_segments": 1
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
          "searchable_snapshot": {
            "snapshot_repository": "backup_repository"
          }
        }
      },
      "frozen": {
        "min_age": "90d",
        "actions": {
          "searchable_snapshot": {
            "snapshot_repository": "backup_repository"
          }
        }
      },
      "delete": {
        "min_age": "365d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

### Resource Allocation by Tier

Allocate appropriate resources to each tier:

| Tier | Node Count | CPU | RAM | Storage | Storage Type |
|------|------------|-----|-----|---------|--------------|
| Hot | 3-6 | 16+ cores | 64GB+ | 1-2TB | NVMe/SSD |
| Warm | 2-4 | 8-16 cores | 32-64GB | 4-10TB | SSD |
| Cold | 2-3 | 4-8 cores | 16-32GB | 10-50TB | HDD/Archive |
| Frozen | 1-2 | 4 cores | 16GB | - | - |

## Cloud vs. On-Premises Considerations

### Cloud-Specific Sizing

Considerations for cloud deployments:

- **Instance types**: Match instance capabilities to workload needs.
- **Managed services**: Factor in managed service benefits for capacity planning.
- **Autoscaling**: Leverage cloud autoscaling capabilities.
- **Storage options**: Utilize cloud storage tiers.

**AWS Example**:
- **Hot tier**: i3.2xlarge (8 cores, 61GB RAM, NVMe storage)
- **Warm tier**: r5.2xlarge (8 cores, 64GB RAM) with EBS gp3 volumes
- **Cold tier**: r5.xlarge (4 cores, 32GB RAM) with EBS st1 volumes

**Azure Example**:
- **Hot tier**: Standard_E8s_v3 (8 cores, 64GB RAM) with Premium SSD
- **Warm tier**: Standard_E8s_v3 with Standard SSD
- **Cold tier**: Standard_E4s_v3 (4 cores, 32GB RAM) with Standard HDD

**GCP Example**:
- **Hot tier**: n2-standard-16 (16 cores, 64GB RAM) with SSD persistent disks
- **Warm tier**: n2-standard-8 (8 cores, 32GB RAM) with balanced persistent disks
- **Cold tier**: n2-standard-4 (4 cores, 16GB RAM) with standard persistent disks

### On-Premises Sizing

Considerations for on-premises deployments:

- **Hardware procurement**: Plan for acquisition lead time.
- **Rack space and power**: Factor in physical infrastructure constraints.
- **Redundancy**: Include hardware redundancy in planning.
- **Networking**: Consider physical network topology and bandwidth.

**On-Premises Hardware Example**:
- **Hot tier**: Dell PowerEdge R740xd with 16 cores, 128GB RAM, 8TB NVMe
- **Warm tier**: Dell PowerEdge R740 with 16 cores, 64GB RAM, 20TB SSD
- **Cold tier**: Dell PowerEdge R740 with 8 cores, 32GB RAM, 100TB HDD

### Hybrid Considerations

Planning for hybrid cloud/on-premises deployments:

- **Data locality**: Keep hot data close to processing.
- **Cross-environment replication**: Plan for bandwidth and latency.
- **Consistent monitoring**: Unified view across environments.

## Summary

Effective capacity planning for the ELK Stack requires understanding workload characteristics, component requirements, and growth patterns. By following the guidelines in this chapter, you can:

1. **Right-size your initial deployment** to meet performance requirements.
2. **Optimize resource allocation** across all components.
3. **Implement cost-effective storage strategies** with appropriate tiering.
4. **Plan for growth** with clear expansion triggers and strategies.
5. **Validate performance** through comprehensive benchmarking.

Remember that capacity planning is an ongoing process. Regularly revisit your capacity plan as workloads change, data volumes grow, and new requirements emerge.

## References

- [Elasticsearch Hardware Sizing Guidelines](https://www.elastic.co/guide/en/elasticsearch/guide/current/hardware.html)
- [Elasticsearch Capacity Planning](https://www.elastic.co/guide/en/elasticsearch/guide/current/capacity-planning.html)
- [Logstash Performance Tuning](https://www.elastic.co/guide/en/logstash/current/performance-troubleshooting.html)
- [Kibana Production Planning](https://www.elastic.co/guide/en/kibana/current/production.html)
- [Beats System Requirements](https://www.elastic.co/guide/en/beats/libbeat/current/getting-started.html)
- [Index Lifecycle Management](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html)