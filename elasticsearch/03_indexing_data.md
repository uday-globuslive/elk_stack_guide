# Indexing Data in Elasticsearch

## Introduction to Indexing

Indexing is the process of adding documents to Elasticsearch, making them searchable. This chapter covers everything you need to know about efficiently indexing data in Elasticsearch, from basic document operations to advanced bulk processing techniques and optimization strategies.

Elasticsearch organizes data in indices, which are collections of documents with similar characteristics. Each document is a JSON object containing fields with values. When you index a document, Elasticsearch adds it to an index and makes it available for search almost immediately.

## Basic Document Operations

### Creating Documents

There are several ways to create documents in Elasticsearch:

#### Specifying Document ID

When you know the ID you want to assign to your document:

```
PUT /customers/_doc/1
{
  "name": "John Smith",
  "email": "john.smith@example.com",
  "age": 35,
  "address": {
    "city": "New York",
    "state": "NY",
    "zip": "10001"
  },
  "interests": ["technology", "music", "hiking"]
}
```

This request:
- Creates a document in the `customers` index
- Assigns ID `1` to the document
- Will create the index if it doesn't exist (with default settings)
- Will replace any existing document with the same ID

#### Auto-generating Document ID

When you don't care about the specific ID:

```
POST /customers/_doc
{
  "name": "Jane Doe",
  "email": "jane.doe@example.com",
  "age": 28,
  "address": {
    "city": "San Francisco",
    "state": "CA",
    "zip": "94105"
  },
  "interests": ["art", "travel", "photography"]
}
```

Elasticsearch will generate a unique ID automatically and return it in the response.

### Reading Documents

Retrieve documents by their ID:

```
GET /customers/_doc/1
```

Response:
```json
{
  "_index": "customers",
  "_id": "1",
  "_version": 1,
  "_seq_no": 0,
  "_primary_term": 1,
  "found": true,
  "_source": {
    "name": "John Smith",
    "email": "john.smith@example.com",
    "age": 35,
    "address": {
      "city": "New York",
      "state": "NY",
      "zip": "10001"
    },
    "interests": ["technology", "music", "hiking"]
  }
}
```

### Updating Documents

There are two ways to update documents:

#### Complete Document Update

Replace the entire document:

```
PUT /customers/_doc/1
{
  "name": "John Smith",
  "email": "john.smith.updated@example.com",
  "age": 36,
  "address": {
    "city": "Brooklyn",
    "state": "NY",
    "zip": "11201"
  },
  "interests": ["technology", "music", "hiking", "cooking"]
}
```

This completely replaces the existing document.

#### Partial Document Update

Update only specific fields:

```
POST /customers/_update/1
{
  "doc": {
    "age": 36,
    "address": {
      "city": "Brooklyn",
      "zip": "11201"
    },
    "interests": ["technology", "music", "hiking", "cooking"]
  }
}
```

This only updates the specified fields, leaving others unchanged.

#### Scripted Updates

Use scripts for more complex updates:

```
POST /customers/_update/1
{
  "script": {
    "source": "ctx._source.age += params.age_increment; ctx._source.interests.add(params.new_interest)",
    "lang": "painless",
    "params": {
      "age_increment": 1,
      "new_interest": "yoga"
    }
  }
}
```

This example:
- Increments the age by 1
- Adds a new interest to the array

### Deleting Documents

Remove documents by their ID:

```
DELETE /customers/_doc/1
```

For deleting multiple documents based on a query, use the Delete By Query API:

```
POST /customers/_delete_by_query
{
  "query": {
    "match": {
      "address.state": "NY"
    }
  }
}
```

This deletes all documents where `address.state` is "NY".

## Bulk Operations

For efficient indexing of multiple documents, use the Bulk API.

### Bulk API Basics

The Bulk API allows you to perform multiple operations in a single request, significantly improving indexing performance:

```
POST /_bulk
{"index":{"_index":"customers","_id":"1"}}
{"name":"John Smith","email":"john@example.com","age":35}
{"index":{"_index":"customers","_id":"2"}}
{"name":"Jane Doe","email":"jane@example.com","age":28}
{"update":{"_index":"customers","_id":"1"}}
{"doc":{"age":36}}
{"delete":{"_index":"customers","_id":"3"}}
```

Note the format:
- Each operation is defined by an action line and (optionally) a data line
- No commas between lines
- Each line must end with a newline character, including the last line

The available action types are:
- `index`: Add or replace a document
- `create`: Add a document, failing if it already exists
- `update`: Partially update a document
- `delete`: Delete a document (no data line required)

### Bulk Request Sizing

Optimal bulk request size depends on several factors:

- **Document Size**: For average-sized documents (1KB), start with 5-15MB per bulk request
- **Memory Configuration**: Keep request size below 50% of JVM heap if possible
- **Hardware**: Faster hardware can handle larger bulk sizes
- **Monitoring**: Check CPU, memory, and I/O during bulk operations to find optimal size

A general guideline is to aim for 1,000-5,000 documents per bulk request, but your specific scenario may require adjustments.

### Concurrent Bulk Requests

For maximum throughput, send multiple bulk requests concurrently:

```python
# Python example using elasticsearch-py
from elasticsearch import Elasticsearch
from elasticsearch.helpers import parallel_bulk
import concurrent.futures

es = Elasticsearch([{'host': 'localhost', 'port': 9200}])

def generate_documents():
    for i in range(10000):
        yield {
            "_index": "customers",
            "_id": i,
            "_source": {
                "name": f"Customer {i}",
                "age": 20 + (i % 50)
            }
        }

# Process bulk operations in parallel
for success, info in parallel_bulk(
        es,
        generate_documents(),
        thread_count=4,
        chunk_size=1000,
        max_chunk_bytes=5 * 1024 * 1024
    ):
    if not success:
        print(f"A document failed: {info}")
```

Best practices for concurrent bulk requests:
- Use a number of parallel connections equal to 2-3× the number of CPU cores
- Monitor client and server CPU and memory
- Adjust connection count, batch size, and request frequency based on monitoring

## Indexing Settings and Performance

### Optimizing for Indexing Performance

When bulk indexing, consider these performance optimizations:

#### Index Refresh Interval

By default, Elasticsearch refreshes indices every second, making new documents available for search. During bulk indexing, you can increase this interval:

```
PUT /my_index/_settings
{
  "index": {
    "refresh_interval": "30s"
  }
}
```

For maximum indexing performance, you can disable refreshing entirely until indexing is complete:

```
PUT /my_index/_settings
{
  "index": {
    "refresh_interval": "-1"
  }
}
```

Don't forget to restore it afterward:

```
PUT /my_index/_settings
{
  "index": {
    "refresh_interval": "1s"
  }
}
```

#### Translog Settings

The translog ensures durability of operations. For bulk indexing, you can adjust its settings:

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

Setting `translog.durability` to `async` means Elasticsearch will sync the translog to disk every `sync_interval` instead of after each request.

#### Number of Replicas

Temporarily reducing the number of replicas during initial bulk indexing can improve performance:

```
PUT /my_index/_settings
{
  "index": {
    "number_of_replicas": 0
  }
}
```

Add replicas back after indexing completes:

```
PUT /my_index/_settings
{
  "index": {
    "number_of_replicas": 1
  }
}
```

### Indexing-Optimized Mappings

Designing your mappings with indexing performance in mind:

```
PUT /my_index
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "index.codec": "best_compression"
  },
  "mappings": {
    "properties": {
      "timestamp": {
        "type": "date"
      },
      "text_content": {
        "type": "text",
        "norms": false,
        "index_options": "docs"
      },
      "status": {
        "type": "keyword"
      },
      "less_important_field": {
        "type": "text",
        "index": false
      }
    }
  }
}
```

Indexing optimizations in this mapping:
- Setting `norms: false` reduces memory usage for scoring-related data
- Using `index_options: docs` stores minimum information needed for search
- Setting `index: false` for fields that don't need to be searched
- Using `best_compression` codec to save disk space

### Dynamic vs. Explicit Mappings

#### Dynamic Mapping

By default, Elasticsearch uses dynamic mapping to automatically define how fields are indexed:

```
POST /dynamic_index/_doc/1
{
  "name": "John Smith",
  "age": 35,
  "is_active": true,
  "registration_date": "2023-01-15T14:12:00Z",
  "tags": ["customer", "premium"]
}
```

Elasticsearch will:
- Map `name` as a `text` field with a `keyword` sub-field
- Map `age` as a `long` field
- Map `is_active` as a `boolean` field
- Map `registration_date` as a `date` field
- Map `tags` as a `text` field with a `keyword` sub-field

#### Controlling Dynamic Mapping

You can control dynamic mapping behavior:

```
PUT /controlled_dynamic_index
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "name": { "type": "text" },
      "age": { "type": "integer" }
    }
  }
}
```

Dynamic mapping options:
- `true`: Automatically add fields (default)
- `false`: Ignore new fields but include them in `_source`
- `strict`: Reject documents with unmapped fields

#### Dynamic Templates

For more control over automatically mapped fields:

```
PUT /template_index
{
  "mappings": {
    "dynamic_templates": [
      {
        "strings_as_keywords": {
          "match_mapping_type": "string",
          "mapping": {
            "type": "keyword"
          }
        }
      },
      {
        "integers_as_longs": {
          "match_mapping_type": "long",
          "mapping": {
            "type": "integer"
          }
        }
      }
    ]
  }
}
```

This will:
- Map all string fields as `keyword` instead of `text`
- Map all long integer fields as `integer` (which uses less space)

## Advanced Indexing Techniques

### Ingest Pipelines

Ingest pipelines allow you to transform and enrich documents before indexing:

```
PUT _ingest/pipeline/customer_pipeline
{
  "description": "Process customer documents",
  "processors": [
    {
      "date": {
        "field": "registration_timestamp",
        "formats": ["epoch_millis", "yyyy-MM-dd'T'HH:mm:ss.SSSZ"],
        "target_field": "registration_date"
      }
    },
    {
      "grok": {
        "field": "address",
        "patterns": ["%{GREEDYDATA:street}, %{WORD:city}, %{WORD:state} %{POSINT:zip}"]
      }
    },
    {
      "remove": {
        "field": "registration_timestamp"
      }
    },
    {
      "set": {
        "field": "indexed_at",
        "value": "{{_ingest.timestamp}}"
      }
    }
  ]
}
```

Use the pipeline when indexing:

```
POST /customers/_doc?pipeline=customer_pipeline
{
  "name": "John Smith",
  "registration_timestamp": 1673789520000,
  "address": "123 Main St, Boston, MA 02108"
}
```

Common ingest processors:
- `date`: Parse date strings
- `grok`: Parse unstructured text
- `gsub`: String replacement
- `script`: Execute a script
- `set`: Set field values
- `remove`: Remove fields
- `rename`: Rename fields
- `geoip`: Add location data based on IP addresses
- `user_agent`: Parse user agent strings

### Reindexing Data

The Reindex API allows you to copy documents from one index to another:

```
POST /_reindex
{
  "source": {
    "index": "old_customers"
  },
  "dest": {
    "index": "new_customers"
  }
}
```

More complex reindexing with transformations:

```
POST /_reindex
{
  "source": {
    "index": "old_customers",
    "query": {
      "range": {
        "age": {
          "gte": 21
        }
      }
    }
  },
  "dest": {
    "index": "new_customers"
  },
  "script": {
    "source": "ctx._source.adult = true; ctx._source.age_category = (ctx._source.age < 30) ? 'young_adult' : 'adult'",
    "lang": "painless"
  }
}
```

This example:
- Selects documents with age ≥ 21
- Adds an `adult` field set to `true`
- Adds an `age_category` field based on the age

#### Handling Large Reindex Operations

For large datasets, use these options:

```
POST /_reindex?wait_for_completion=false
{
  "source": {
    "index": "old_customers",
    "size": 1000
  },
  "dest": {
    "index": "new_customers"
  }
}
```

The `wait_for_completion=false` parameter returns a task ID that you can use to check progress:

```
GET /_tasks/oTUltX4IQMOUUVeiohTt8A:12345
```

### Upserts

Perform an update if a document exists, or create it if it doesn't:

```
POST /customers/_update/1
{
  "script": {
    "source": "ctx._source.visits += params.visit_count",
    "lang": "painless",
    "params": {
      "visit_count": 1
    }
  },
  "upsert": {
    "name": "New Customer",
    "visits": 1,
    "created_at": "{{_now}}"
  }
}
```

This will:
- Increment the `visits` field if document exists
- Create a new document with the given fields if it doesn't exist

### Update By Query

Update multiple documents matching a query:

```
POST /customers/_update_by_query
{
  "script": {
    "source": "ctx._source.updated = true; ctx._source.last_updated = params.timestamp",
    "lang": "painless",
    "params": {
      "timestamp": "2023-01-15T14:12:00Z"
    }
  },
  "query": {
    "range": {
      "age": {
        "gte": 30
      }
    }
  }
}
```

This updates all customer documents with `age` ≥ 30.

## Data Streams and Time Series Data

### Introduction to Data Streams

Data streams are ideal for time-series data like logs, metrics, and events. They simplify index management by:

- Automatically creating timestamped backing indices
- Rolling over indices based on age or size
- Applying lifecycle policies
- Providing a single target for indexing and search

### Creating a Data Stream

First, create an index template:

```
PUT _index_template/logs_template
{
  "index_patterns": ["logs-*"],
  "data_stream": {},
  "template": {
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 1
    },
    "mappings": {
      "properties": {
        "@timestamp": {
          "type": "date"
        },
        "message": {
          "type": "text"
        },
        "level": {
          "type": "keyword"
        }
      }
    }
  }
}
```

The data stream is created automatically when you index data:

```
POST /logs-app/_doc
{
  "@timestamp": "2023-01-15T14:12:00Z",
  "message": "Application started",
  "level": "INFO"
}
```

### Managing Data Streams

List data streams:

```
GET /_data_stream
```

Get information about a specific data stream:

```
GET /_data_stream/logs-app
```

Delete a data stream:

```
DELETE /_data_stream/logs-app
```

### Data Stream Lifecycle Management

Use Index Lifecycle Management (ILM) with data streams:

```
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
        "min_age": "2d",
        "actions": {
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

PUT _index_template/logs_template
{
  "index_patterns": ["logs-*"],
  "data_stream": {},
  "template": {
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 1,
      "index.lifecycle.name": "logs_policy"
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
```

## Handling Failures and Retries

### Handling Indexing Failures

#### Understanding Failure Responses

Example failure response:

```json
{
  "took": 30,
  "errors": true,
  "items": [
    {
      "index": {
        "_index": "customers",
        "_id": "1",
        "status": 400,
        "error": {
          "type": "mapper_parsing_exception",
          "reason": "failed to parse field [age] of type [integer]",
          "caused_by": {
            "type": "number_format_exception",
            "reason": "For input string: \"not_a_number\""
          }
        }
      }
    },
    {
      "index": {
        "_index": "customers",
        "_id": "2",
        "_version": 1,
        "result": "created",
        "status": 201
      }
    }
  ]
}
```

This response indicates:
- The first document failed with a parsing error for the `age` field
- The second document was successfully indexed

#### Retry Strategies

For client-side retry handling:

```python
# Python example with retry logic
from elasticsearch import Elasticsearch
from elasticsearch.helpers import bulk
import time

es = Elasticsearch([{'host': 'localhost', 'port': 9200}])

def index_with_retries(documents, max_retries=3, initial_backoff=2):
    retry_docs = documents
    failures = []
    
    for attempt in range(max_retries):
        success, failed = bulk(
            es, 
            retry_docs, 
            stats_only=False,
            raise_on_error=False
        )
        
        if not failed:
            return success, failures
        
        # Extract failed documents for retry
        retry_docs = [doc for doc in failed]
        
        # Wait with exponential backoff
        if attempt < max_retries - 1:
            time.sleep(initial_backoff * (2 ** attempt))
    
    # Add remaining failures to the total failures
    failures.extend(retry_docs)
    return success, failures
```

### Concurrent Indexing and Optimistic Concurrency Control

Elasticsearch uses optimistic concurrency control to prevent unexpected overwrites:

```
PUT /customers/_doc/1?if_seq_no=10&if_primary_term=2
{
  "name": "Updated Name",
  "email": "updated@example.com"
}
```

This update will only succeed if:
- The document has a sequence number of 10
- The document has a primary term of 2

You get these values from the previous GET or update response:

```json
{
  "_index": "customers",
  "_id": "1",
  "_version": 2,
  "_seq_no": 10,
  "_primary_term": 2,
  "found": true,
  "_source": {
    // Document contents
  }
}
```

## Index Templates and Component Templates

### Component Templates

Component templates define reusable building blocks for index templates:

```
PUT _component_template/logs_mappings
{
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "message": { "type": "text" },
        "level": { "type": "keyword" },
        "service": { "type": "keyword" }
      }
    }
  }
}

PUT _component_template/logs_settings
{
  "template": {
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 1,
      "index.lifecycle.name": "logs_policy",
      "index.lifecycle.rollover_alias": "logs"
    }
  }
}
```

### Index Templates

Index templates use component templates to define complete index configuration:

```
PUT _index_template/logs_template
{
  "index_patterns": ["logs-*"],
  "template": {
    "aliases": {
      "all_logs": {}
    }
  },
  "composed_of": ["logs_mappings", "logs_settings"],
  "priority": 100
}
```

This template:
- Applies to indices matching the `logs-*` pattern
- Uses mappings from `logs_mappings` component template
- Uses settings from `logs_settings` component template
- Adds an alias called `all_logs` to each matching index
- Has a priority of 100 (higher priority templates take precedence)

## Real-World Indexing Examples

### Batch Loading Data from Files

Loading a CSV file using Logstash:

```ruby
# Logstash config for batch loading
input {
  file {
    path => "/data/customers.csv"
    start_position => "beginning"
    sincedb_path => "/dev/null"
  }
}

filter {
  csv {
    columns => ["id", "name", "email", "age", "registration_date", "status"]
    separator => ","
  }
  
  date {
    match => ["registration_date", "yyyy-MM-dd"]
    target => "registration_date"
  }
  
  mutate {
    convert => {
      "age" => "integer"
      "id" => "integer"
    }
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "customers"
    document_id => "%{id}"
    action => "index"
    
    # Performance settings
    bulk_size => 1000
    workers => 4
  }
}
```

### Streaming Log Ingestion

Streaming application logs using Filebeat:

```yaml
# filebeat.yml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/app/*.log
  json.keys_under_root: true
  json.add_error_key: true
  
setup.template.enabled: true
setup.template.name: "logs"
setup.template.pattern: "logs-*"

output.elasticsearch:
  hosts: ["http://elasticsearch:9200"]
  pipeline: "log_enrichment"
  index: "logs-%{+yyyy.MM.dd}"
```

With an ingest pipeline for enrichment:

```
PUT _ingest/pipeline/log_enrichment
{
  "description": "Enrich log entries",
  "processors": [
    {
      "geoip": {
        "field": "client_ip",
        "target_field": "geo",
        "ignore_missing": true
      }
    },
    {
      "user_agent": {
        "field": "user_agent_string",
        "target_field": "user_agent",
        "ignore_missing": true
      }
    },
    {
      "date": {
        "field": "timestamp",
        "formats": ["ISO8601", "UNIX"],
        "target_field": "@timestamp",
        "ignore_failure": true
      }
    }
  ]
}
```

### Near Real-Time Metrics Ingestion

Indexing metrics data using Metricbeat:

```yaml
# metricbeat.yml
metricbeat.modules:
- module: system
  metricsets: ["cpu", "memory", "network", "filesystem"]
  period: 10s
  
- module: elasticsearch
  metricsets: ["node", "node_stats"]
  period: 30s
  hosts: ["http://elasticsearch:9200"]
  
setup.template.enabled: true
setup.template.name: "metrics"
setup.template.pattern: "metrics-*"

output.elasticsearch:
  hosts: ["http://elasticsearch:9200"]
  index: "metrics-%{+yyyy.MM.dd}"
```

## Conclusion

Efficient indexing is critical for Elasticsearch performance. This chapter covered various approaches to indexing data, from basic document operations to advanced techniques like bulk indexing, data streams, and ingest pipelines.

Key takeaways:
- Use bulk operations for efficient indexing of multiple documents
- Configure index settings for optimal indexing performance
- Design mappings with performance in mind
- Use ingest pipelines for document preprocessing
- Implement data streams for time-series data
- Handle failures gracefully with retry strategies

In the next chapter, we'll explore querying and aggregations to effectively search and analyze the data you've indexed.