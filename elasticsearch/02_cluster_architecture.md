# Cluster Architecture

## Understanding Elasticsearch Clusters

An Elasticsearch cluster is a collection of connected nodes (Elasticsearch instances) that work together to store, search, and manage data. The distributed nature of Elasticsearch provides several benefits:

- **Scalability**: Add more nodes to handle larger data volumes and higher query loads
- **High Availability**: Duplicate data across nodes to prevent data loss
- **Fault Tolerance**: Continue operating even if some nodes fail
- **Performance**: Distribute work across multiple nodes for faster processing

## Node Roles and Types

Elasticsearch nodes can serve different roles in a cluster:

### Master-Eligible Nodes

- Responsible for cluster-wide management and configuration operations
- Maintain the cluster state, which includes:
  - The list of nodes in the cluster
  - The list of indices and their settings
  - Shard allocation information
  - Other cluster-wide settings and metadata
- Handle lightweight cluster management tasks
- Are NOT intended for data storage or search operations in production

Configuration:
```yaml
node.roles: [ master ]
```

### Data Nodes

- Store data and perform CRUD operations, searches, and aggregations
- Handle the bulk of the resource-intensive work in a cluster
- Can be further specialized into:
  - **Hot**: For recent, frequently accessed data (SSD-backed)
  - **Warm**: For older, less frequently accessed data (HDD-backed)
  - **Cold**: For historical, rarely accessed data (optimized for cost)
  - **Frozen**: For archived data, loaded from snapshot when needed

Configuration:
```yaml
node.roles: [ data ]
```

For tiered data nodes:
```yaml
node.roles: [ data_hot ]
node.roles: [ data_warm ]
node.roles: [ data_cold ]
node.roles: [ data_frozen ]
```

### Ingest Nodes

- Pre-process documents before indexing
- Execute ingest pipelines that can transform, enrich, or filter documents
- Similar to a lightweight version of Logstash built into Elasticsearch

Configuration:
```yaml
node.roles: [ ingest ]
```

### Coordinating-Only Nodes

- Route requests to appropriate data nodes
- Gather and merge results from data nodes
- Act as a load balancer and smart router for the cluster
- Help offload client connection overhead from data nodes

Configuration:
```yaml
node.roles: [ ]
```

### Machine Learning Nodes

- Run machine learning jobs for anomaly detection and forecasting
- Handle compute-intensive ML tasks separately from other operations

Configuration:
```yaml
node.roles: [ ml ]
```

### Multi-Role Nodes

- A single node can fulfill multiple roles
- Common for smaller clusters or development environments

Configuration:
```yaml
node.roles: [ master, data, ingest ]
```

## Cluster Formation and Discovery

When Elasticsearch starts, it needs to discover other nodes and form a cluster:

### Discovery Process

1. Node starts and looks for other nodes using seed hosts
2. Master-eligible nodes participate in master election
3. Elected master node manages the cluster state
4. Nodes join the cluster and are assigned roles

### Configuration Parameters

- `cluster.name`: Identify which cluster to join
- `discovery.seed_hosts`: List of hosts to contact for discovery
- `cluster.initial_master_nodes`: Bootstrap list of master-eligible nodes

Example configuration:
```yaml
cluster.name: production-cluster
discovery.seed_hosts: ["es-node1", "es-node2", "es-node3"]
cluster.initial_master_nodes: ["es-node1", "es-node2", "es-node3"]
```

## Sharding and Data Distribution

### Understanding Shards

- **Primary Shards**: Original shards that receive indexing operations
- **Replica Shards**: Copies of primary shards for redundancy and read scaling

Shards allow Elasticsearch to:
- Distribute data across nodes
- Provide parallel processing
- Enable redundancy for fault tolerance

### Shard Allocation

The master node determines shard allocation based on:
- Available nodes
- Current shard distribution
- Allocation settings
- Disk space
- Custom allocation rules

### Shard Sizing and Count

Guidelines for shard sizing:
- Aim for shards between 10-50GB in size
- Avoid shards that are too small (< 1GB) or too large (> 100GB)
- Total shard count should be a multiple of data nodes (ideally 1-3 shards per GB of heap memory)

### Index Settings for Sharding

```
PUT /my_index
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1
  }
}
```

### Index Lifecycle Management (ILM)

ILM helps manage indices as they age, automatically:
- Creating new indices when they reach a certain size or age
- Moving indices between data tiers (hot → warm → cold → frozen)
- Reducing replica count for older indices
- Deleting indices after a retention period

Example ILM policy:
```
PUT _ilm/policy/logs_policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_size": "50gb",
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
          "set_priority": {
            "priority": 0
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

## Resilience and Fault Tolerance

### Handling Node Failures

When a node fails:
1. Master node detects the failure via missed heartbeats
2. Master updates the cluster state
3. Missing primary shards are promoted from replicas on other nodes
4. New replica shards are allocated to maintain redundancy

### Split Brain Prevention

Split brain occurs when multiple nodes believe they are the master, potentially causing data inconsistency.

Prevention measures:
- Configure a proper quorum with `discovery.zen.minimum_master_nodes` (pre-7.0)
- Use the default automatic quorum in Elasticsearch 7.0+ (floor(n/2) + 1, where n is master-eligible nodes)
- Implement a minimum of three master-eligible nodes

### Gateway and Recovery

- Gateway stores cluster state and shard data for recovery after full cluster restart
- Recovery process rebuilds indices from shard data

## Network Design and Communication

### Node-to-Node Communication

- Transport layer uses port 9300 (by default)
- Used for internal cluster communication
- Optimized binary protocol

### Client-to-Node Communication

- REST API uses port 9200 (by default)
- Used for external client communication
- HTTP-based JSON protocol

### Network Configuration

```yaml
# Bind to specific network interface
network.host: 192.168.1.10

# Customize port settings
http.port: 9200
transport.port: 9300

# Set a specific publish address (the address other nodes use to contact this node)
network.publish_host: 192.168.1.10
```

## Common Cluster Architectures

### Small Cluster (3 Nodes)

All nodes serve all roles:
- 3 nodes with master, data, and ingest roles
- Suitable for development or small production workloads
- Simple to set up and manage

### Medium Cluster (5-9 Nodes)

Dedicated roles for improved stability:
- 3 dedicated master-eligible nodes
- 3-6 data nodes
- 1-2 coordinating/ingest nodes
- Suitable for moderate production workloads

### Large Cluster (10+ Nodes)

Specialized nodes for optimal performance:
- 3 dedicated master-eligible nodes
- 2+ coordinating nodes
- 2+ ingest nodes (if needed)
- 4+ hot data nodes (SSD-based)
- 4+ warm data nodes (HDD-based)
- 2+ cold data nodes (if needed)
- Suitable for enterprise workloads with high volume or complex requirements

### Cross-Cluster Architecture

For multi-datacenter or global deployments:
- Multiple clusters in different locations
- Cross-cluster replication (CCR) for data redundancy
- Cross-cluster search (CCS) for unified search experience
- Each cluster follows one of the architectures above

## Hardware Considerations

### Memory

- Elasticsearch is memory-intensive due to its reliance on filesystem cache
- JVM heap size: Set to 50% of available RAM, but no more than 32GB
- Leave remainder for filesystem cache
- Recommended minimum: 16GB RAM per node, 64GB+ for production data nodes

### CPU

- Elasticsearch is CPU-intensive for searches and aggregations
- More cores help with concurrent operations
- Recommended minimum: 4-8 cores for data nodes

### Storage

- SSDs strongly recommended for hot data
- RAID 0 provides better performance (rely on Elasticsearch replication instead of RAID for redundancy)
- NAS/SAN generally not recommended due to network latency
- Local storage preferred for performance

### Network

- High-bandwidth, low-latency network important for cluster communication
- 10 Gbps recommended for heavy workloads
- Isolate cluster traffic from client traffic when possible

## Cluster Monitoring and Management

### Key Metrics to Monitor

- Cluster health status (green, yellow, red)
- Node status and resource usage (CPU, memory, disk I/O)
- JVM heap usage and garbage collection
- Search and indexing performance
- Shard allocation and distribution
- Query latency and throughput

### Monitoring Tools

- Elasticsearch's built-in monitoring
- Metricbeat with Elasticsearch/Kibana modules
- Third-party monitoring tools (Datadog, New Relic, etc.)

### Management Tools

- Kibana's Stack Monitoring
- Elasticsearch API
- Cerebro (third-party open-source tool)
- Elastic Cloud Enterprise (ECE) for large deployments

## Cluster Scaling Strategies

### Vertical Scaling

- Add more resources (CPU, memory, disk) to existing nodes
- Simpler but has limits
- Requires node restart

### Horizontal Scaling

- Add more nodes to the cluster
- More flexible and scalable
- Can be done without downtime

### Scaling Process

1. Add new nodes to the cluster
2. Allow automatic shard rebalancing or use shard allocation filtering
3. Monitor cluster health during rebalancing
4. Remove old nodes if replacing (one at a time)

## Best Practices for Cluster Architecture

1. **Dedicated Master Nodes**: Always use dedicated master nodes in production
2. **Node Redundancy**: Have at least 3 master-eligible nodes and ensure data redundancy
3. **Role Separation**: Separate roles for stability and performance in larger clusters
4. **Shard Management**: Keep shard count reasonable (avoid over-sharding)
5. **Cross-Zone Distribution**: Distribute nodes across availability zones
6. **Regular Backups**: Use snapshots for regular backups
7. **Capacity Planning**: Plan for growth and monitor resource usage
8. **Hardware Choices**: Use appropriate hardware for each node role
9. **Network Configuration**: Ensure reliable, low-latency network between nodes
10. **Security**: Implement proper security measures (encryption, authentication, authorization)

## Conclusion

Designing an effective Elasticsearch cluster architecture requires understanding the various node roles, sharding mechanisms, and resource requirements. By following the best practices outlined in this chapter, you can create a scalable, resilient, and high-performing Elasticsearch deployment tailored to your specific needs.

In the next chapter, we'll dive into indexing data in Elasticsearch, exploring various approaches to efficiently get your data into the system.