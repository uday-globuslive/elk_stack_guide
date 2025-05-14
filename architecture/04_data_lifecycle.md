# Data Lifecycle Management

This chapter covers strategies and best practices for managing the lifecycle of data within the ELK Stack, from ingestion through hot storage, warm storage, cold storage, and eventually archival or deletion.

## Table of Contents
- [Introduction to Data Lifecycle Management](#introduction-to-data-lifecycle-management)
- [Elasticsearch Index Lifecycle Management](#elasticsearch-index-lifecycle-management)
- [Time-Based Data Management](#time-based-data-management)
- [Storage Tiers and Node Attributes](#storage-tiers-and-node-attributes)
- [Managing Index Templates](#managing-index-templates)
- [Rollup and Transform Jobs](#rollup-and-transform-jobs)
- [Data Retention Policies](#data-retention-policies)
- [Data Archiving Strategies](#data-archiving-strategies)
- [Monitoring and Maintenance](#monitoring-and-maintenance)
- [Optimizing Resource Utilization](#optimizing-resource-utilization)
- [Automated Lifecycle Management](#automated-lifecycle-management)
- [Compliance and Governance](#compliance-and-governance)

## Introduction to Data Lifecycle Management

Data Lifecycle Management (DLM) in the ELK Stack involves managing data throughout its useful life, from collection to final disposition, while balancing performance, cost, and compliance requirements.

### The Data Lifecycle

1. **Ingestion**: Data is collected and indexed into Elasticsearch
2. **Hot Phase**: Active data that requires fast access and frequent updates
3. **Warm Phase**: Less active data that still needs to be searchable
4. **Cold Phase**: Infrequently accessed data with lower performance requirements
5. **Frozen Phase**: Rarely accessed data kept for compliance or historical purposes
6. **Deletion/Archival**: Data that has reached the end of its retention period

### Benefits of Data Lifecycle Management

- **Cost Optimization**: Store data on appropriate hardware based on access patterns
- **Performance Improvement**: Keep active indices on high-performance hardware
- **Compliance**: Meet regulatory requirements for data retention and deletion
- **Resource Efficiency**: Allocate resources based on data importance and access patterns
- **Scalability**: Manage growing data volumes without proportional infrastructure growth
- **Operational Simplicity**: Automate routine data management tasks

### Key Considerations

When planning for data lifecycle management, consider:

1. **Performance Requirements**: Speed of data retrieval at each phase
2. **Cost Constraints**: Budget for storage and compute resources
3. **Retention Requirements**: Legal or business needs for data retention
4. **Access Patterns**: How often and how data is accessed over time
5. **Growth Rate**: Rate at which new data is added
6. **Recovery Needs**: Backup and restore requirements for each phase

## Elasticsearch Index Lifecycle Management

Elasticsearch provides Index Lifecycle Management (ILM) to automate the management of indices as they age.

### ILM Concepts

1. **Policy**: A set of rules defining how an index should be managed throughout its lifecycle
2. **Phase**: A stage in the lifecycle (hot, warm, cold, frozen, delete)
3. **Action**: A task performed during a phase (e.g., rollover, allocate, forcemerge)
4. **Transition**: Rules that determine when an index moves to the next phase

### Creating an ILM Policy

Define a basic ILM policy:

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
            "max_size": "50gb",
            "max_docs": 100000000
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
      "frozen": {
        "min_age": "90d",
        "actions": {
          "searchable_snapshot": {
            "snapshot_repository": "logs_repository"
          }
        }
      },
      "delete": {
        "min_age": "365d",
        "actions": {
          "delete": {
            "delete_searchable_snapshot": true
          }
        }
      }
    }
  }
}
```

### ILM Actions

Common actions used in ILM policies:

1. **Rollover**: Create a new index when the current one reaches certain thresholds
   ```json
   "rollover": {
     "max_age": "7d",
     "max_size": "50gb",
     "max_docs": 100000000
   }
   ```

2. **Shrink**: Reduce the number of shards in an index
   ```json
   "shrink": {
     "number_of_shards": 1
   }
   ```

3. **Forcemerge**: Reduce the number of segments in an index shard
   ```json
   "forcemerge": {
     "max_num_segments": 1
   }
   ```

4. **Allocate**: Control which nodes host the index shards
   ```json
   "allocate": {
     "require": {
       "data": "warm"
     }
   }
   ```

5. **Freeze**: Mark an index as read-only and minimize its memory footprint
   ```json
   "freeze": {}
   ```

6. **Searchable Snapshot**: Create a searchable snapshot of the index
   ```json
   "searchable_snapshot": {
     "snapshot_repository": "logs_repository"
   }
   ```

7. **Delete**: Remove the index
   ```json
   "delete": {}
   ```

### Applying an ILM Policy

Apply the ILM policy to an index:

```json
// Create an index with an ILM policy
PUT logs-2023.05.20
{
  "settings": {
    "index.lifecycle.name": "logs-policy",
    "index.lifecycle.rollover_alias": "logs"
  }
}

// Create an index template with an ILM policy
PUT _index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1,
      "index.lifecycle.name": "logs-policy",
      "index.lifecycle.rollover_alias": "logs"
    },
    "mappings": {
      "properties": {
        "@timestamp": {
          "type": "date"
        }
      }
    }
  }
}

// Create initial index
PUT logs-000001
{
  "aliases": {
    "logs": {
      "is_write_index": true
    }
  },
  "settings": {
    "index.lifecycle.name": "logs-policy"
  }
}
```

### Monitoring ILM

Monitor the status of ILM policies and indices:

```json
// Check ILM status
GET _ilm/status

// List all policies
GET _ilm/policy

// Check specific policy
GET _ilm/policy/logs-policy

// Check index lifecycle status
GET logs-*/_ilm/explain
```

## Time-Based Data Management

Manage time-based data effectively with date-based indices and rollover strategies.

### Date-Based Index Naming

Use a consistent naming convention for time-based indices:

```
logs-YYYY.MM.DD
metrics-YYYY.MM
events-YYYY.ww
```

Benefits of date-based naming:
- Makes age of data immediately apparent
- Simplifies querying specific time ranges
- Facilitates maintenance and cleanup
- Enables natural lifecycle progression

### Index Aliases

Use aliases to simplify data access across multiple indices:

```json
// Create an alias pointing to multiple indices
POST _aliases
{
  "actions": [
    {
      "add": {
        "index": "logs-2023.05.*",
        "alias": "logs-may-2023"
      }
    }
  ]
}

// Create a write alias for the current index
POST _aliases
{
  "actions": [
    {
      "add": {
        "index": "logs-2023.05.20",
        "alias": "logs-write",
        "is_write_index": true
      }
    }
  ]
}

// Update write alias during rollover
POST _aliases
{
  "actions": [
    {
      "remove": {
        "index": "logs-2023.05.20",
        "alias": "logs-write"
      }
    },
    {
      "add": {
        "index": "logs-2023.05.21",
        "alias": "logs-write",
        "is_write_index": true
      }
    }
  ]
}
```

### Rollover Strategies

Implement effective rollover strategies:

1. **Time-Based Rollover**:
   ```json
   "rollover": {
     "max_age": "1d"
   }
   ```

2. **Size-Based Rollover**:
   ```json
   "rollover": {
     "max_size": "50gb"
   }
   ```

3. **Document Count Rollover**:
   ```json
   "rollover": {
     "max_docs": 100000000
   }
   ```

4. **Compound Rollover Conditions**:
   ```json
   "rollover": {
     "max_age": "1d",
     "max_size": "50gb",
     "max_docs": 100000000
   }
   ```

5. **Manual Rollover**:
   ```bash
   # Manually trigger a rollover
   POST logs/_rollover
   {
     "conditions": {
       "max_age": "1d",
       "max_size": "50gb"
     }
   }
   ```

### Time Series Data Best Practices

Optimize for time series data:

1. **Shard Allocation**:
   - Use fewer shards for older indices
   - Consider the query patterns when deciding shard count
   ```json
   PUT logs-2023.05.20/_settings
   {
     "index": {
       "number_of_shards": 3
     }
   }
   ```

2. **Replication Strategy**:
   - Higher replica count for hot data
   - Reduce replicas for warm/cold data
   ```json
   PUT logs-2023.01.*/_settings
   {
     "index": {
       "number_of_replicas": 0
     }
   }
   ```

3. **Refresh Interval**:
   - Frequent refresh for hot data
   - Less frequent or disable for warm/cold data
   ```json
   PUT logs-2023.05.20/_settings
   {
     "index": {
       "refresh_interval": "1s"
     }
   }
   
   PUT logs-2023.01.*/_settings
   {
     "index": {
       "refresh_interval": "30s"
     }
   }
   ```

## Storage Tiers and Node Attributes

Implement a tiered storage architecture to optimize cost and performance.

### Defining Node Attributes

Configure different node types in `elasticsearch.yml`:

```yaml
# Hot node configuration
node.attr.data: hot
node.roles: [ data_hot, data_content ]

# Warm node configuration
node.attr.data: warm
node.roles: [ data_warm, data_content ]

# Cold node configuration
node.attr.data: cold
node.roles: [ data_cold ]

# Frozen node configuration (Elasticsearch 7.x+)
node.attr.data: frozen
node.roles: [ data_frozen ]
```

In Elasticsearch 7.12+, use the specialized data roles:

```yaml
# Hot node
node.roles: [ data_hot, data_content ]

# Warm node
node.roles: [ data_warm, data_content ]

# Cold node
node.roles: [ data_cold ]

# Frozen node
node.roles: [ data_frozen ]
```

### Shard Allocation Filtering

Control where shards are allocated based on node attributes:

```json
// Set allocation rules for an index
PUT logs-2023.01.01/_settings
{
  "index.routing.allocation.require.data": "cold"
}

// Set allocation rules for multiple indices
PUT logs-2022.*/_settings
{
  "index.routing.allocation.require.data": "cold"
}

// Include and exclude allocation rules
PUT logs-2023.05.*/_settings
{
  "index.routing.allocation.include.data": "hot,warm",
  "index.routing.allocation.exclude.data": "cold"
}
```

### Hardware Considerations for Tiers

Different hardware specifications for each tier:

1. **Hot Tier**:
   - Fastest storage (NVMe/SSD)
   - Higher CPU allocation
   - More memory per node
   - Example: 32+ CPU cores, 64GB+ RAM, NVMe storage

2. **Warm Tier**:
   - SSD or fast HDD storage
   - Moderate CPU allocation
   - Moderate memory per node
   - Example: 16 CPU cores, 32GB RAM, SSD storage

3. **Cold Tier**:
   - HDD storage
   - Lower CPU allocation
   - Less memory per node
   - Example: 8 CPU cores, 16GB RAM, HDD storage

4. **Frozen Tier**:
   - Object storage or archival HDDs
   - Minimal CPU allocation
   - Minimal memory requirements
   - Example: 4 CPU cores, 8GB RAM, archival storage

### Tier Transitions

Define transitions between tiers in ILM policies:

```json
PUT _ilm/policy/tiered-storage-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_age": "1d",
            "max_size": "50gb"
          },
          "set_priority": {
            "priority": 100
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
      "frozen": {
        "min_age": "90d",
        "actions": {
          "searchable_snapshot": {
            "snapshot_repository": "logs_repository"
          }
        }
      },
      "delete": {
        "min_age": "365d",
        "actions": {
          "delete": {
            "delete_searchable_snapshot": true
          }
        }
      }
    }
  }
}
```

## Managing Index Templates

Use index templates to define settings, mappings, and lifecycle policies for new indices.

### Component Templates

Create reusable component templates:

```json
// Create a settings component template
PUT _component_template/logs-settings
{
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1,
      "index.lifecycle.name": "logs-policy",
      "index.lifecycle.rollover_alias": "logs"
    }
  },
  "_meta": {
    "description": "Settings for log indices",
    "version": "1.0"
  }
}

// Create a mappings component template
PUT _component_template/logs-mappings
{
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": {
          "type": "date"
        },
        "message": {
          "type": "text",
          "fields": {
            "keyword": {
              "type": "keyword",
              "ignore_above": 256
            }
          }
        },
        "level": {
          "type": "keyword"
        },
        "service": {
          "type": "keyword"
        },
        "host": {
          "type": "keyword"
        }
      }
    }
  }
}

// Create an alias component template
PUT _component_template/logs-aliases
{
  "template": {
    "aliases": {
      "logs": {}
    }
  }
}
```

### Index Templates

Combine component templates into an index template:

```json
PUT _index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "index.lifecycle.name": "logs-policy"
    }
  },
  "composed_of": ["logs-settings", "logs-mappings", "logs-aliases"],
  "priority": 100,
  "_meta": {
    "description": "Template for log indices",
    "version": "1.0"
  }
}
```

### Template Prioritization

Handle multiple matching templates with priorities:

```json
// High priority template for specific log type
PUT _index_template/api-logs-template
{
  "index_patterns": ["logs-api-*"],
  "template": {
    "settings": {
      "number_of_shards": 6,
      "number_of_replicas": 2,
      "index.lifecycle.name": "api-logs-policy"
    }
  },
  "composed_of": ["logs-mappings", "logs-aliases"],
  "priority": 200,
  "_meta": {
    "description": "Template for API log indices",
    "version": "1.0"
  }
}
```

### Managing Multiple Data Types

Create templates for different data types:

```json
// Logs template
PUT _index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "index.lifecycle.name": "logs-policy"
    }
  },
  "composed_of": ["logs-settings", "logs-mappings", "logs-aliases"],
  "priority": 100
}

// Metrics template
PUT _index_template/metrics-template
{
  "index_patterns": ["metrics-*"],
  "template": {
    "settings": {
      "index.lifecycle.name": "metrics-policy"
    }
  },
  "composed_of": ["metrics-settings", "metrics-mappings", "metrics-aliases"],
  "priority": 100
}

// Events template
PUT _index_template/events-template
{
  "index_patterns": ["events-*"],
  "template": {
    "settings": {
      "index.lifecycle.name": "events-policy"
    }
  },
  "composed_of": ["events-settings", "events-mappings", "events-aliases"],
  "priority": 100
}
```

## Rollup and Transform Jobs

Use rollup and transform jobs to create summaries of historical data.

### Rollup Jobs

Create and manage rollup jobs to aggregate historical data:

```json
// Create a rollup job
PUT _rollup/job/logs_rollup
{
  "index_pattern": "logs-*",
  "rollup_index": "logs-rollup",
  "cron": "0 0 * * * ?",
  "page_size": 1000,
  "groups": {
    "date_histogram": {
      "field": "@timestamp",
      "fixed_interval": "1h",
      "time_zone": "UTC"
    },
    "terms": {
      "fields": ["service", "level", "host"]
    }
  },
  "metrics": [
    {
      "field": "response_time",
      "metrics": ["min", "max", "avg", "sum"]
    },
    {
      "field": "bytes",
      "metrics": ["min", "max", "avg", "sum"]
    }
  ]
}

// Start the rollup job
POST _rollup/job/logs_rollup/_start

// Query rollup data
GET logs-rollup/_rollup_search
{
  "size": 0,
  "aggregations": {
    "hourly": {
      "date_histogram": {
        "field": "@timestamp",
        "fixed_interval": "1h"
      },
      "aggs": {
        "service": {
          "terms": {
            "field": "service"
          },
          "aggs": {
            "avg_response": {
              "avg": {
                "field": "response_time"
              }
            }
          }
        }
      }
    }
  }
}
```

### Transform Jobs

Create and manage transform jobs for more complex data transformations:

```json
// Create a transform job
PUT _transform/logs-summary
{
  "source": {
    "index": ["logs-*"]
  },
  "dest": {
    "index": "logs-summary"
  },
  "frequency": "1h",
  "sync": {
    "time": {
      "field": "@timestamp",
      "delay": "60s"
    }
  },
  "pivot": {
    "group_by": {
      "service": { "terms": { "field": "service" } },
      "level": { "terms": { "field": "level" } },
      "day": {
        "date_histogram": {
          "field": "@timestamp",
          "calendar_interval": "1d"
        }
      }
    },
    "aggregations": {
      "count": { "value_count": { "field": "_id" } },
      "error_count": {
        "filter": {
          "term": {
            "level": "error"
          }
        }
      },
      "avg_response_time": { "avg": { "field": "response_time" } },
      "max_response_time": { "max": { "field": "response_time" } },
      "p95_response_time": {
        "percentiles": {
          "field": "response_time",
          "percents": [95]
        }
      }
    }
  }
}

// Start the transform job
POST _transform/logs-summary/_start

// Query transformed data
GET logs-summary/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "service": "api-gateway" } },
        { "range": { "day": { "gte": "now-7d/d", "lte": "now/d" } } }
      ]
    }
  },
  "sort": [
    { "day": "desc" }
  ]
}
```

### Combining Rollups and Transforms with ILM

Use rollups and transforms as part of a comprehensive data lifecycle strategy:

```json
PUT _ilm/policy/comprehensive-logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_age": "1d",
            "max_size": "50gb"
          },
          "set_priority": {
            "priority": 100
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
      "frozen": {
        "min_age": "90d",
        "actions": {
          "searchable_snapshot": {
            "snapshot_repository": "logs_repository"
          }
        }
      },
      "delete": {
        "min_age": "365d",
        "actions": {
          "delete": {
            "delete_searchable_snapshot": true
          }
        }
      }
    }
  }
}
```

Create a lifecycle process that includes:
1. Hot phase: Raw data for real-time analysis
2. Warm phase: Consolidated data for recent analysis
3. Cold phase: Move to cost-effective storage
4. Create rollups before moving to frozen
5. Frozen phase: Minimal footprint for compliance
6. Delete when retention period expires

## Data Retention Policies

Implement effective data retention policies based on business requirements.

### Defining Retention Requirements

Consider factors that influence retention periods:

1. **Regulatory Requirements**:
   - GDPR, HIPAA, SOX, etc.
   - Industry-specific regulations
   - Regional/country-specific laws

2. **Business Requirements**:
   - Operational needs (troubleshooting, analysis)
   - Historical analysis
   - Business intelligence

3. **Technical Considerations**:
   - Storage costs
   - Query performance
   - Backup/recovery time

### Implementing Different Retention Tiers

Create varying retention periods for different data types:

```json
// Security logs - long retention
PUT _ilm/policy/security-logs-policy
{
  "policy": {
    "phases": {
      "hot": { "min_age": "0ms", "actions": { "rollover": { "max_age": "1d" } } },
      "warm": { "min_age": "7d", "actions": { /* warm actions */ } },
      "cold": { "min_age": "30d", "actions": { /* cold actions */ } },
      "frozen": { "min_age": "90d", "actions": { /* frozen actions */ } },
      "delete": { "min_age": "730d", "actions": { "delete": {} } } // 2 years
    }
  }
}

// Application logs - medium retention
PUT _ilm/policy/app-logs-policy
{
  "policy": {
    "phases": {
      "hot": { "min_age": "0ms", "actions": { "rollover": { "max_age": "1d" } } },
      "warm": { "min_age": "7d", "actions": { /* warm actions */ } },
      "cold": { "min_age": "30d", "actions": { /* cold actions */ } },
      "delete": { "min_age": "90d", "actions": { "delete": {} } } // 3 months
    }
  }
}

// Metrics data - short retention
PUT _ilm/policy/metrics-policy
{
  "policy": {
    "phases": {
      "hot": { "min_age": "0ms", "actions": { "rollover": { "max_age": "1d" } } },
      "warm": { "min_age": "3d", "actions": { /* warm actions */ } },
      "delete": { "min_age": "30d", "actions": { "delete": {} } } // 1 month
    }
  }
}
```

### Retention Policy Documentation

Document your retention policies:

```markdown
# Data Retention Policy

## Overview
This document outlines the retention periods for different data types in our ELK Stack deployment.

## Data Types and Retention Periods

### Security Logs
- **Hot Phase**: 0-7 days
- **Warm Phase**: 7-30 days
- **Cold Phase**: 30-90 days
- **Frozen Phase**: 90-730 days
- **Total Retention**: 2 years (730 days)
- **Regulatory Requirement**: SOX, HIPAA

### Application Logs
- **Hot Phase**: 0-7 days
- **Warm Phase**: 7-30 days
- **Cold Phase**: 30-90 days
- **Total Retention**: 3 months (90 days)
- **Business Requirement**: Application troubleshooting

### Metrics Data
- **Hot Phase**: 0-3 days
- **Warm Phase**: 3-30 days
- **Total Retention**: 1 month (30 days)
- **Business Requirement**: Performance monitoring

## Exceptions
- **Critical Incident Data**: Retained for 1 year in separate archive
- **Financial Transaction Logs**: Retained for 7 years to comply with financial regulations
- **Security Incident Data**: Retained for 3 years from incident closure

## Review Schedule
This policy is reviewed annually or when significant changes in business or regulatory requirements occur.

## Policy Enforcement
Enforcement is automated through Index Lifecycle Management policies in Elasticsearch.
```

### Communicating Retention to Stakeholders

Create dashboards to visualize data retention:

```json
// Kibana saved object for data retention dashboard
{
  "attributes": {
    "title": "Data Retention Overview",
    "visState": "...",
    "uiStateJSON": "{}",
    "description": "Overview of data retention policies and current index states",
    "version": 1,
    "kibanaSavedObjectMeta": {
      "searchSourceJSON": "..."
    }
  },
  "type": "visualization"
}
```

## Data Archiving Strategies

Implement strategies for archiving data that must be retained but is rarely accessed.

### Snapshot Archives

Archive data using snapshot repositories:

```json
// Configure repository
PUT _snapshot/archive_repository
{
  "type": "fs",
  "settings": {
    "location": "/mnt/elasticsearch-archives",
    "compress": true
  }
}

// Create snapshots for archival
PUT _snapshot/archive_repository/logs-2022-q1
{
  "indices": "logs-2022.01.*,logs-2022.02.*,logs-2022.03.*",
  "include_global_state": false,
  "metadata": {
    "archive_date": "2023-05-01",
    "retention": "7 years",
    "archive_reason": "Regulatory compliance"
  }
}
```

### Searchable Snapshots

Use searchable snapshots for low-cost, searchable archives:

```json
// Configure repository
PUT _snapshot/searchable_archives
{
  "type": "s3",
  "settings": {
    "bucket": "elasticsearch-snapshots",
    "client": "default"
  }
}

// Mount as searchable snapshot
POST _snapshot/searchable_archives/logs-2022-q1/_mount
{
  "index": "logs-2022.01.01",
  "renamed_index": "archive-logs-2022.01.01",
  "index_settings": {
    "index.number_of_replicas": 0,
    "index.routing.allocation.include.data": "frozen"
  }
}
```

### External Archives

Archive data to external systems:

1. **S3 or Object Storage**:
   ```json
   // Use Logstash to archive to S3
   output {
     s3 {
       bucket => "log-archives"
       region => "us-east-1"
       prefix => "logs/%{+YYYY}/%{+MM}/%{+dd}"
       time_file => 15
       codec => json_lines
     }
   }
   ```

2. **Hadoop/HDFS**:
   ```json
   // Use Logstash with HDFS output plugin
   output {
     webhdfs {
       host => "hadoop-namenode"
       port => 50070
       path => "/logs/%{+YYYY}/%{+MM}/%{+dd}/%{+HH}.log"
       user => "hdfs"
     }
   }
   ```

3. **Data Lakes**:
   ```python
   # Example Python script for archiving to a data lake
   from elasticsearch import Elasticsearch
   import pandas as pd
   import boto3
   
   es = Elasticsearch(["https://elasticsearch:9200"])
   
   # Query data for archiving
   query = {
     "query": {
       "range": {
         "@timestamp": {
           "lt": "now-365d/d"
         }
       }
     },
     "size": 10000
   }
   
   result = es.search(index="logs-*", body=query, scroll="1m")
   scroll_id = result["_scroll_id"]
   hits = result["hits"]["hits"]
   
   # Process and archive data
   while len(hits) > 0:
       # Convert to DataFrame
       df = pd.DataFrame([hit["_source"] for hit in hits])
       
       # Archive to S3
       s3 = boto3.resource('s3')
       bucket = s3.Bucket('data-lake-archive')
       
       # Group by date and save as parquet
       for date, group in df.groupby(pd.to_datetime(df['@timestamp']).dt.date):
           parquet_buffer = io.BytesIO()
           group.to_parquet(parquet_buffer)
           key = f"logs/{date.year}/{date.month}/{date.day}/archive.parquet"
           bucket.put_object(Key=key, Body=parquet_buffer.getvalue())
       
       # Get next batch
       result = es.scroll(scroll_id=scroll_id, scroll="1m")
       scroll_id = result["_scroll_id"]
       hits = result["hits"]["hits"]
   ```

### Archive Index Templates

Create specialized templates for archive indices:

```json
PUT _index_template/archive-template
{
  "index_patterns": ["archive-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0,
      "index.routing.allocation.require.data": "frozen",
      "index.refresh_interval": "60s",
      "index.codec": "best_compression",
      "index.search.slowlog.threshold.query.warn": "5s"
    },
    "mappings": {
      "dynamic": false,
      "_source": {
        "enabled": true
      }
    }
  }
}
```

## Monitoring and Maintenance

Set up monitoring and maintenance procedures for data lifecycle management.

### Monitoring ILM Status

Monitor the status of ILM processes:

```json
// Check ILM status
GET _ilm/status

// Check index lifecycle information
GET _cat/indices?v&h=index,ilm.policy,lifecycle.phase,shard.max,pri.store.size

// Detailed explanation of index lifecycle
GET logs-2023.05.20/_ilm/explain

// Create a watch for ILM errors
PUT _watcher/watch/ilm_errors
{
  "trigger": {
    "schedule": {
      "interval": "1h"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": [".monitoring-es-*"],
        "body": {
          "size": 0,
          "query": {
            "bool": {
              "must": [
                { "term": { "type": "ilm_stats" } },
                { "range": { "timestamp": { "gte": "now-1h" } } }
              ]
            }
          },
          "aggs": {
            "indices": {
              "terms": {
                "field": "index_stats.index",
                "size": 100
              },
              "aggs": {
                "errors": {
                  "filter": {
                    "exists": {
                      "field": "ilm.error"
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  },
  "condition": {
    "script": {
      "source": "return ctx.payload.aggregations.indices.buckets.stream().anyMatch(bucket -> bucket.errors.doc_count > 0)"
    }
  },
  "actions": {
    "notify": {
      "email": {
        "to": ["admin@example.com"],
        "subject": "ILM Errors Detected",
        "body": {
          "html": """
          <h2>ILM Errors Detected</h2>
          <p>Errors have been detected in Index Lifecycle Management. Please investigate the following indices:</p>
          <ul>
          {{#ctx.payload.aggregations.indices.buckets}}
            {{#errors}}
              {{#doc_count}}
              <li>{{../key}} - {{doc_count}} errors</li>
              {{/doc_count}}
            {{/errors}}
          {{/ctx.payload.aggregations.indices.buckets}}
          </ul>
          """
        }
      }
    }
  }
}
```

### Maintenance Tasks

Perform regular maintenance tasks:

1. **Audit Index Sizes and Counts**:
   ```bash
   # Get index sizes
   curl -X GET "localhost:9200/_cat/indices?v&h=index,status,health,docs.count,store.size&s=store.size:desc"
   
   # Create a script for regular index auditing
   cat <<EOF > /usr/local/bin/audit-indices.sh
   #!/bin/bash
   ES_URL="http://localhost:9200"
   
   # Get all indices
   indices=\$(curl -s "\$ES_URL/_cat/indices?h=index,docs.count,store.size")
   
   # Summarize by prefix
   echo "Index Pattern,Count,Total Size,Avg Size"
   echo "\$indices" | awk '{
     pattern=gensub(/([^-]+-[^-]+).*/, "\\1*", "g", \$1);
     count[pattern]++;
     docs[pattern]+=\$2;
     size[pattern]=size[pattern]" "\$3;
   }
   END {
     for (p in count) {
       total_size=0;
       split(size[p], sizes, " ");
       for (i in sizes) {
         if (sizes[i] ~ /gb/) {
           size_val=gensub(/gb/, "", "g", sizes[i]);
           total_size+=size_val*1024;
         } else if (sizes[i] ~ /mb/) {
           size_val=gensub(/mb/, "", "g", sizes[i]);
           total_size+=size_val;
         } else if (sizes[i] ~ /kb/) {
           size_val=gensub(/kb/, "", "g", sizes[i]);
           total_size+=size_val/1024;
         } else if (sizes[i] ~ /b/) {
           size_val=gensub(/b/, "", "g", sizes[i]);
           total_size+=size_val/1024/1024;
         }
       }
       if (total_size > 1024) {
         total_size_str=sprintf("%.2f gb", total_size/1024);
       } else {
         total_size_str=sprintf("%.2f mb", total_size);
       }
       avg_size=total_size/count[p];
       if (avg_size > 1024) {
         avg_size_str=sprintf("%.2f gb", avg_size/1024);
       } else {
         avg_size_str=sprintf("%.2f mb", avg_size);
       }
       printf "%s,%d,%s,%s\\n", p, count[p], total_size_str, avg_size_str;
     }
   }' | sort -t',' -k3,3nr
   EOF
   chmod +x /usr/local/bin/audit-indices.sh
   ```

2. **Review ILM Policies**:
   ```bash
   # Get all ILM policies
   curl -X GET "localhost:9200/_ilm/policy?pretty"
   
   # Check which policies need updating
   cat <<EOF > /usr/local/bin/review-ilm-policies.sh
   #!/bin/bash
   ES_URL="http://localhost:9200"
   
   # Get all policies
   policies=\$(curl -s "\$ES_URL/_ilm/policy?pretty" | jq -r 'keys[]')
   
   echo "Policy,Phase Count,Has Delete Phase,Oldest Version"
   for policy in \$policies; do
     phase_count=\$(curl -s "\$ES_URL/_ilm/policy/\$policy?pretty" | jq '.[\$policy].policy.phases | length')
     has_delete=\$(curl -s "\$ES_URL/_ilm/policy/\$policy?pretty" | jq 'if .[\$policy].policy.phases.delete then "Yes" else "No" end')
     policy_version=\$(curl -s "\$ES_URL/_ilm/policy/\$policy?pretty" | jq '.[\$policy].version')
     echo "\$policy,\$phase_count,\$has_delete,\$policy_version"
   done
   EOF
   chmod +x /usr/local/bin/review-ilm-policies.sh
   ```

3. **Validate Data Distribution**:
   ```bash
   # Get data distribution across nodes
   curl -X GET "localhost:9200/_cat/allocation?v"
   
   # Check shard distribution across tiers
   cat <<EOF > /usr/local/bin/check-tier-distribution.sh
   #!/bin/bash
   ES_URL="http://localhost:9200"
   
   # Get nodes by tier
   hot_nodes=\$(curl -s "\$ES_URL/_cat/nodes?h=name,node.role&v" | grep data_hot | awk '{print \$1}')
   warm_nodes=\$(curl -s "\$ES_URL/_cat/nodes?h=name,node.role&v" | grep data_warm | awk '{print \$1}')
   cold_nodes=\$(curl -s "\$ES_URL/_cat/nodes?h=name,node.role&v" | grep data_cold | awk '{print \$1}')
   
   # Get shards by node
   shards=\$(curl -s "\$ES_URL/_cat/shards?h=index,shard,node&v")
   
   echo "Tier,Node,Shard Count,Indices"
   # Hot tier
   for node in \$hot_nodes; do
     count=\$(echo "\$shards" | grep "\$node" | wc -l)
     indices=\$(echo "\$shards" | grep "\$node" | awk '{print \$1}' | sort | uniq | wc -l)
     echo "hot,\$node,\$count,\$indices"
   done
   
   # Warm tier
   for node in \$warm_nodes; do
     count=\$(echo "\$shards" | grep "\$node" | wc -l)
     indices=\$(echo "\$shards" | grep "\$node" | awk '{print \$1}' | sort | uniq | wc -l)
     echo "warm,\$node,\$count,\$indices"
   done
   
   # Cold tier
   for node in \$cold_nodes; do
     count=\$(echo "\$shards" | grep "\$node" | wc -l)
     indices=\$(echo "\$shards" | grep "\$node" | awk '{print \$1}' | sort | uniq | wc -l)
     echo "cold,\$node,\$count,\$indices"
   done
   EOF
   chmod +x /usr/local/bin/check-tier-distribution.sh
   ```

### Remediation Actions

Handle common lifecycle management issues:

1. **Failed ILM Transitions**:
   ```json
   // Retry ILM execution for an index
   POST logs-2023.05.20/_ilm/retry

   // Move an index to a specific phase manually
   POST logs-2023.05.20/_ilm/move/cold

   // Remove index from ILM
   PUT logs-2023.05.20/_settings
   {
     "index.lifecycle.name": null
   }
   ```

2. **Fixing Stuck Indices**:
   ```json
   // Check for stuck indices
   GET _cat/indices?v&h=index,health,status,ilm.policy,lifecycle.phase,lifecycle.step,lifecycle.action

   // Remove index from management temporarily
   PUT logs-stuck-index/_settings
   {
     "index.lifecycle.name": null
   }

   // Fix any issues with the index
   // e.g., force merge to fix too many segments
   POST logs-stuck-index/_forcemerge?max_num_segments=1

   // Re-apply lifecycle policy
   PUT logs-stuck-index/_settings
   {
     "index.lifecycle.name": "logs-policy"
   }
   ```

3. **Handling Allocation Issues**:
   ```json
   // Check for unassigned shards
   GET _cat/shards?v&h=index,shard,prirep,state,unassigned.reason&s=state

   // Temporarily change allocation rules
   PUT logs-2023.05.20/_settings
   {
     "index.routing.allocation.require.data": null,
     "index.routing.allocation.include.data": "*"
   }

   // Once allocated, restore proper allocation
   PUT logs-2023.05.20/_settings
   {
     "index.routing.allocation.require.data": "cold",
     "index.routing.allocation.include.data": null
   }
   ```

## Optimizing Resource Utilization

Optimize resource usage throughout the data lifecycle.

### Index Optimization Techniques

Optimize indices at different lifecycle stages:

1. **Hot Phase Optimization**:
   ```json
   // Optimize for indexing performance
   PUT logs-current/_settings
   {
     "index.refresh_interval": "5s",
     "index.number_of_replicas": 1,
     "index.translog.durability": "async",
     "index.translog.sync_interval": "5s"
   }
   ```

2. **Warm Phase Optimization**:
   ```json
   // Optimize for balance of search and storage
   PUT logs-warm/_settings
   {
     "index.refresh_interval": "30s",
     "index.number_of_replicas": 1,
     "index.codec": "best_compression"
   }

   // Force merge to reduce segments
   POST logs-warm/_forcemerge?max_num_segments=1
   ```

3. **Cold Phase Optimization**:
   ```json
   // Optimize for storage efficiency
   PUT logs-cold/_settings
   {
     "index.refresh_interval": "60s",
     "index.number_of_replicas": 0,
     "index.codec": "best_compression",
     "index.search.slowlog.threshold.query.warn": "5s"
   }
   ```

### Storage Optimization

Optimize storage usage with techniques specific to each phase:

1. **Hot Phase Storage**:
   - Use higher shard counts for parallel indexing
   - Use async translog durability when safe
   - Balance between write and read performance

2. **Warm Phase Storage**:
   - Reduce shard count through shrink action
   - Force merge to reduce segment count
   - Use best_compression codec

3. **Cold Phase Storage**:
   - Reduce or eliminate replicas if using snapshots
   - Move to lower-cost storage
   - Use compression heavily

4. **Frozen Phase Storage**:
   - Use searchable snapshots
   - Mount indices with shared_cache option
   - Minimize heap memory usage

### Memory Optimization

Optimize memory usage across the data lifecycle:

1. **Field Data Cache**:
   ```json
   // Limit field data cache for certain indices
   PUT logs-2022.*/_settings
   {
     "index.fielddata.cache.max_size": "5%"
   }
   ```

2. **Query Cache**:
   ```json
   // Disable query cache for cold indices
   PUT logs-cold/_settings
   {
     "index.queries.cache.enabled": false
   }
   ```

3. **Request Cache**:
   ```json
   // Enable request cache for read-only indices
   PUT logs-2022.*/_settings
   {
     "index.requests.cache.enable": true
   }
   ```

### Resource Reservation

Reserve resources for critical operations:

```json
// Ensure ILM operations have resources
PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.disk.threshold_enabled": true,
    "cluster.routing.allocation.disk.watermark.low": "85%",
    "cluster.routing.allocation.disk.watermark.high": "90%",
    "cluster.routing.allocation.disk.watermark.flood_stage": "95%"
  }
}
```

## Automated Lifecycle Management

Implement automation for data lifecycle management.

### Scheduled Jobs

Create scheduled jobs for lifecycle management tasks:

```bash
#!/bin/bash
# ilm-maintenance.sh
ES_URL="http://localhost:9200"

# Check for stuck indices
stuck_indices=$(curl -s "$ES_URL/_cat/indices?h=index,ilm.policy,lifecycle.phase,lifecycle.step_time&v" | awk '{
  if ($3 != "" && $4 != "") {
    step_time=$4;
    current_time=systime();
    age_hours=(current_time - step_time) / 3600;
    if (age_hours > 24) {
      print $1;
    }
  }
}')

# Retry stuck indices
if [ ! -z "$stuck_indices" ]; then
  echo "Found stuck indices, attempting to retry ILM execution:"
  for index in $stuck_indices; do
    echo "Retrying $index..."
    curl -X POST "$ES_URL/$index/_ilm/retry"
  done
fi

# Check for indices without lifecycle policy
missing_policy_indices=$(curl -s "$ES_URL/_cat/indices?h=index&v" | grep -v "^index" | while read index; do
  has_policy=$(curl -s "$ES_URL/$index/_settings" | jq -r ".$index.settings.index | has(\"lifecycle\")")
  if [ "$has_policy" = "false" ]; then
    # Check if index should have a policy based on naming
    if [[ $index =~ ^(logs|metrics|events) ]]; then
      echo "$index"
    fi
  fi
done)

# Apply appropriate policies to indices missing them
if [ ! -z "$missing_policy_indices" ]; then
  echo "Found indices missing lifecycle policies:"
  for index in $missing_policy_indices; do
    if [[ $index =~ ^logs ]]; then
      policy="logs-policy"
    elif [[ $index =~ ^metrics ]]; then
      policy="metrics-policy"
    elif [[ $index =~ ^events ]]; then
      policy="events-policy"
    else
      policy="default-policy"
    fi
    
    echo "Applying $policy to $index..."
    curl -X PUT "$ES_URL/$index/_settings" -H "Content-Type: application/json" -d "{
      \"index.lifecycle.name\": \"$policy\"
    }"
  done
fi

# Audit ILM execution
echo "ILM Phase Distribution:"
curl -s "$ES_URL/_cat/indices?h=index,ilm.policy,lifecycle.phase&v" | awk '{
  if ($3 != "") {
    phases[$3]++;
  }
} END {
  for (phase in phases) {
    print phase ": " phases[phase];
  }
}'
```

### Lifecycle Management Dashboard

Create a Kibana dashboard for lifecycle management:

```json
// Elasticsearch query to get lifecycle status
GET _search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        {
          "term": {
            "event.dataset": "elasticsearch.index_management"
          }
        }
      ]
    }
  },
  "aggs": {
    "indices_by_phase": {
      "terms": {
        "field": "elasticsearch.index.lifecycle.phase",
        "size": 10
      },
      "aggs": {
        "indices": {
          "terms": {
            "field": "elasticsearch.index.name",
            "size": 1000
          }
        }
      }
    }
  }
}
```

### Automated Remediation

Create scripts for automated problem remediation:

```python
#!/usr/bin/env python3
# auto-remediate-ilm.py
import requests
import json
import time
import logging
from datetime import datetime, timedelta

# Configuration
ES_URL = "http://localhost:9200"
ES_AUTH = ("elastic", "changeme")
LOG_FILE = "/var/log/elasticsearch/ilm-remediation.log"

# Setup logging
logging.basicConfig(
    filename=LOG_FILE,
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

def get_stuck_indices():
    """Find indices stuck in ILM phases for more than 24 hours"""
    try:
        response = requests.get(f"{ES_URL}/_cat/indices?h=index,ilm.policy,lifecycle.phase,lifecycle.step_time&v", auth=ES_AUTH)
        response.raise_for_status()
        
        stuck_indices = []
        for line in response.text.splitlines():
            if not line.strip() or line.startswith("index"):
                continue
                
            parts = line.split()
            if len(parts) < 4:
                continue
                
            index_name = parts[0]
            policy = parts[1] if len(parts) > 1 else ""
            phase = parts[2] if len(parts) > 2 else ""
            step_time_str = parts[3] if len(parts) > 3 else ""
            
            if not policy or not phase or not step_time_str:
                continue
                
            try:
                # Parse step time (usually in milliseconds since epoch)
                step_time_ms = int(step_time_str)
                step_time = datetime.fromtimestamp(step_time_ms / 1000)
                age_hours = (datetime.now() - step_time).total_seconds() / 3600
                
                if age_hours > 24:
                    stuck_indices.append({
                        "index": index_name,
                        "policy": policy,
                        "phase": phase,
                        "step_time": step_time,
                        "age_hours": age_hours
                    })
            except (ValueError, TypeError):
                logging.warning(f"Could not parse step time for index {index_name}: {step_time_str}")
                
        return stuck_indices
    except Exception as e:
        logging.error(f"Error getting stuck indices: {e}")
        return []

def retry_ilm(index):
    """Retry ILM execution for an index"""
    try:
        response = requests.post(f"{ES_URL}/{index}/_ilm/retry", auth=ES_AUTH)
        response.raise_for_status()
        logging.info(f"Successfully retried ILM for index {index}")
        return True
    except Exception as e:
        logging.error(f"Error retrying ILM for index {index}: {e}")
        return False

def move_to_next_phase(index, current_phase):
    """Move an index to the next phase if it's stuck"""
    phase_order = ["hot", "warm", "cold", "frozen", "delete"]
    
    try:
        current_idx = phase_order.index(current_phase)
        if current_idx < len(phase_order) - 1:
            next_phase = phase_order[current_idx + 1]
            
            response = requests.post(
                f"{ES_URL}/{index}/_ilm/move/{next_phase}", 
                auth=ES_AUTH
            )
            response.raise_for_status()
            logging.info(f"Moved index {index} from {current_phase} to {next_phase}")
            return True
        else:
            logging.warning(f"Index {index} is in the final phase ({current_phase}), cannot move to next phase")
            return False
    except ValueError:
        logging.warning(f"Unknown phase {current_phase} for index {index}")
        return False
    except Exception as e:
        logging.error(f"Error moving index {index} to next phase: {e}")
        return False

def main():
    logging.info("Starting ILM remediation run")
    
    # Get stuck indices
    stuck_indices = get_stuck_indices()
    logging.info(f"Found {len(stuck_indices)} stuck indices")
    
    # Remediate stuck indices
    for idx_info in stuck_indices:
        index = idx_info["index"]
        phase = idx_info["phase"]
        age_hours = idx_info["age_hours"]
        
        logging.info(f"Processing stuck index {index} in phase {phase} for {age_hours:.1f} hours")
        
        # First try to retry the current phase
        if retry_ilm(index):
            logging.info(f"Retried ILM for index {index}, waiting to see if it progresses")
            continue
            
        # If index is stuck for more than 48 hours, try to move to next phase
        if age_hours > 48:
            logging.warning(f"Index {index} stuck for {age_hours:.1f} hours, attempting to move to next phase")
            move_to_next_phase(index, phase)
    
    logging.info("Completed ILM remediation run")

if __name__ == "__main__":
    main()
```

## Compliance and Governance

Implement compliance and governance controls for data management.

### Data Retention Compliance

Document and enforce compliance requirements:

```json
// Create index lifecycle policies that comply with regulations
PUT _ilm/policy/gdpr-compliant-policy
{
  "policy": {
    "_meta": {
      "description": "Policy compliant with GDPR requirements",
      "compliance": ["GDPR"],
      "data_classification": "personally_identifiable",
      "owner": "Data Protection Officer",
      "reviewed_date": "2023-05-01"
    },
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_age": "7d"
          },
          "set_priority": {
            "priority": 100
          }
        }
      },
      "warm": {
        "min_age": "30d",
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

### Audit Trails

Maintain audit trails for data lifecycle actions:

```json
// Enable system index monitoring
PUT _cluster/settings
{
  "persistent": {
    "xpack.monitoring.collection.enabled": true,
    "xpack.monitoring.collection.system.indices.usage": {
      "enabled": true
    }
  }
}

// Create a watch for lifecycle changes
PUT _watcher/watch/ilm_lifecycle_changes
{
  "trigger": {
    "schedule": {
      "interval": "1h"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": ".monitoring-es-*",
        "body": {
          "size": 0,
          "query": {
            "bool": {
              "must": [
                { "term": { "type": "index_stats" } },
                { "range": { "timestamp": { "gte": "now-1h" } } }
              ]
            }
          },
          "aggs": {
            "indices": {
              "terms": {
                "field": "index_stats.index",
                "size": 1000
              },
              "aggs": {
                "lifecycle_changes": {
                  "terms": {
                    "field": "index_stats.lifecycle.phase"
                  }
                }
              }
            }
          }
        }
      }
    }
  },
  "condition": {
    "script": {
      "source": "return true;"
    }
  },
  "actions": {
    "index_lifecycle_log": {
      "index": {
        "index": "lifecycle-audit-log",
        "doc_id": "{{ctx.execution_time}}",
        "body": {
          "timestamp": "{{ctx.execution_time}}",
          "message": "Index lifecycle changes detected",
          "changes": "{{ctx.payload.aggregations.indices.buckets}}"
        }
      }
    }
  }
}
```

### Legal Hold Procedures

Implement legal hold for data subject to litigation:

```json
// Create a policy for legal hold
PUT _ilm/policy/legal-hold-policy
{
  "policy": {
    "_meta": {
      "description": "Policy for indices under legal hold",
      "legal_hold_id": "LH-2023-05",
      "legal_case": "Case-12345",
      "hold_date": "2023-05-01",
      "authorized_by": "Legal Department"
    },
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "set_priority": {
            "priority": 100
          }
        }
      }
    }
  }
}

// Apply legal hold to specific indices
POST _indices/update_policies
{
  "indices": "logs-2023.01.*,logs-2023.02.*",
  "policy": "legal-hold-policy"
}

// Document the legal hold
PUT legal-holds/_doc/LH-2023-05
{
  "hold_id": "LH-2023-05",
  "case_number": "Case-12345",
  "hold_date": "2023-05-01",
  "expected_duration": "indefinite",
  "authorized_by": "Legal Department",
  "affected_indices": [
    "logs-2023.01.01",
    "logs-2023.01.02",
    "logs-2023.02.15"
  ],
  "reason": "Ongoing litigation related to security incident on January 15, 2023",
  "contacts": [
    {
      "name": "Jane Smith",
      "role": "Legal Counsel",
      "email": "jane.smith@example.com"
    }
  ]
}
```

### Encryption and Access Controls

Implement encryption and access controls for sensitive data:

```json
// Create role for limited access to archived data
POST _security/role/archived_logs_read
{
  "indices": [
    {
      "names": ["logs-20*"],
      "privileges": ["read", "view_index_metadata"],
      "query": {
        "range": {
          "@timestamp": {
            "gte": "now-2y",
            "lt": "now-1y"
          }
        }
      },
      "field_security": {
        "grant": ["@timestamp", "message", "level", "service"],
        "except": ["user.*", "password", "credit_card", "ssn"]
      }
    }
  ]
}
```

## Conclusion

Effective data lifecycle management is essential for maintaining performance, controlling costs, and meeting compliance requirements in Elasticsearch deployments. By implementing a comprehensive approach that includes index lifecycle management, tiered storage, and automated maintenance, you can ensure your ELK Stack remains efficient and effective as your data grows.

Remember that data lifecycle management is an ongoing process that requires regular monitoring, adjustment, and optimization. Review your policies and procedures regularly to ensure they continue to meet your evolving requirements.

In the next chapter, we'll explore cross-cluster replication and other techniques for managing data across multiple Elasticsearch clusters.