# Elasticsearch Fundamentals

## Introduction to Elasticsearch

Elasticsearch is a distributed, RESTful search and analytics engine built on Apache Lucene. It provides a distributed, multitenant-capable full-text search engine with an HTTP web interface and schema-free JSON documents. Elasticsearch can be used for various use cases beyond search, including log analytics, application monitoring, security analytics, and business analytics.

## Core Concepts

### Documents

Documents are the basic unit of information in Elasticsearch. A document is a JSON object containing data with fields and values. Each document is stored in an index and has a unique ID.

Example document:
```json
{
  "id": "1",
  "name": "John Doe",
  "age": 30,
  "email": "john.doe@example.com",
  "account_created": "2023-01-15T14:12:00",
  "interests": ["technology", "music", "hiking"]
}
```

### Indices

An index is a collection of documents that have somewhat similar characteristics. An index is identified by a name (must be lowercase) and is used to refer to the documents in it when performing search, update, and delete operations.

Think of an index as similar to a database table in the relational database world, but with a schema that can evolve over time.

### Shards and Replicas

Elasticsearch distributes an index's documents across multiple storage containers called shards. This is how Elasticsearch scales horizontally.

- **Primary Shard**: Original shard that receives all indexing operations
- **Replica Shard**: Copy of a primary shard for redundancy and read scaling

By default, an index is created with 1 primary shard and 1 replica. This means that if you have at least two nodes in your cluster, your index will have 1 primary shard and 1 replica shard (1 complete replica) for a total of 2 shards.

### Nodes and Clusters

- **Node**: A single instance of Elasticsearch running on a server
- **Cluster**: A collection of connected nodes that share the same cluster name

A node can serve different roles:
- **Master-eligible node**: Can be elected as the master node
- **Data node**: Stores data and executes CRUD operations, searches, and aggregations
- **Ingest node**: Pre-processes documents before indexing
- **Coordinating node**: Routes client requests and distributes search operations
- **Machine learning node**: Runs machine learning jobs

## Basic Operations

### Creating and Managing Indices

Create an index:
```
PUT /customers
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  }
}
```

Get index information:
```
GET /customers
```

Delete an index:
```
DELETE /customers
```

### Indexing Documents

Index a document with a specific ID:
```
PUT /customers/_doc/1
{
  "name": "John Doe",
  "email": "john@example.com",
  "age": 30
}
```

Index a document with auto-generated ID:
```
POST /customers/_doc
{
  "name": "Jane Smith",
  "email": "jane@example.com",
  "age": 28
}
```

### Retrieving Documents

Get a document by ID:
```
GET /customers/_doc/1
```

### Updating Documents

Update a document:
```
POST /customers/_update/1
{
  "doc": {
    "age": 31
  }
}
```

### Deleting Documents

Delete a document:
```
DELETE /customers/_doc/1
```

### Bulk Operations

Elasticsearch provides a bulk API to perform multiple operations in a single request:

```
POST /_bulk
{"index":{"_index":"customers","_id":"1"}}
{"name":"John Doe","email":"john@example.com","age":30}
{"index":{"_index":"customers","_id":"2"}}
{"name":"Jane Smith","email":"jane@example.com","age":28}
```

## Searching in Elasticsearch

### Basic Search

Simple query string search:
```
GET /customers/_search?q=john
```

Using Query DSL:
```
GET /customers/_search
{
  "query": {
    "match": {
      "name": "john"
    }
  }
}
```

### Query Types

Elasticsearch offers various query types:

1. **Match Query**: Standard full-text query for analyzing and searching text fields
   ```
   GET /customers/_search
   {
     "query": {
       "match": {
         "name": "john doe"
       }
     }
   }
   ```

2. **Term Query**: Exact match for structured data like numbers, dates, and keywords
   ```
   GET /customers/_search
   {
     "query": {
       "term": {
         "age": 30
       }
     }
   }
   ```

3. **Range Query**: Find documents with field values within a certain range
   ```
   GET /customers/_search
   {
     "query": {
       "range": {
         "age": {
           "gte": 20,
           "lte": 40
         }
       }
     }
   }
   ```

4. **Bool Query**: Combine multiple queries with boolean logic
   ```
   GET /customers/_search
   {
     "query": {
       "bool": {
         "must": [
           { "match": { "name": "john" } }
         ],
         "filter": [
           { "range": { "age": { "gte": 30 } } }
         ],
         "must_not": [
           { "term": { "email": "other@example.com" } }
         ],
         "should": [
           { "match": { "interests": "technology" } }
         ]
       }
     }
   }
   ```

### Filtering vs. Querying

There are two contexts in which query clauses can operate:

- **Query context**: Answers "How well does this document match this query clause?" and calculates a relevance score
- **Filter context**: Answers "Does this document match this query clause?" with a yes/no answer (no scoring)

Filter context is generally faster and can be cached, making it ideal for narrowing down results before scoring.

### Sorting

Sort results by specific fields:
```
GET /customers/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    { "age": "desc" },
    { "name.keyword": "asc" }
  ]
}
```

### Pagination

Control result pagination:
```
GET /customers/_search
{
  "from": 0,
  "size": 20,
  "query": {
    "match_all": {}
  }
}
```

## Aggregations

Aggregations allow you to generate analytics from your documents.

### Metric Aggregations

Calculate metrics like average, min, max, sum:
```
GET /customers/_search
{
  "size": 0,
  "aggs": {
    "avg_age": {
      "avg": {
        "field": "age"
      }
    }
  }
}
```

### Bucket Aggregations

Group documents into buckets:
```
GET /customers/_search
{
  "size": 0,
  "aggs": {
    "age_ranges": {
      "range": {
        "field": "age",
        "ranges": [
          { "to": 30 },
          { "from": 30, "to": 40 },
          { "from": 40 }
        ]
      }
    }
  }
}
```

### Nested Aggregations

Combine aggregations for deeper analysis:
```
GET /customers/_search
{
  "size": 0,
  "aggs": {
    "age_ranges": {
      "range": {
        "field": "age",
        "ranges": [
          { "to": 30 },
          { "from": 30, "to": 40 },
          { "from": 40 }
        ]
      },
      "aggs": {
        "gender_count": {
          "terms": {
            "field": "gender.keyword"
          }
        }
      }
    }
  }
}
```

## Mappings

Mappings define how documents and their fields are stored and indexed. Elasticsearch can dynamically map fields, but explicit mappings give you full control.

### Dynamic Mapping

Elasticsearch automatically detects field types:
```
PUT /customers/_doc/1
{
  "name": "John Doe",
  "age": 30,
  "is_active": true,
  "join_date": "2023-01-15T14:12:00Z"
}
```

The fields will be mapped as:
- name: text field with keyword sub-field
- age: long (numeric) field
- is_active: boolean field
- join_date: date field

### Explicit Mapping

Define mappings explicitly:
```
PUT /customers
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      },
      "email": {
        "type": "keyword"
      },
      "age": {
        "type": "integer"
      },
      "account_created": {
        "type": "date"
      },
      "interests": {
        "type": "text"
      }
    }
  }
}
```

### Common Field Types

- **text**: For full-text search (analyzed)
- **keyword**: For exact matching, aggregations, and sorting
- **date**: For date values
- **numeric types**: integer, long, float, double, etc.
- **boolean**: true/false values
- **object**: For nested JSON objects
- **nested**: For arrays of objects that need to be queried independently
- **geo_point** and **geo_shape**: For geographical data

## Analyzing Text

Elasticsearch uses analyzers to convert text fields into terms for indexing and searching.

### Anatomy of an Analyzer

An analyzer consists of:
1. **Character filters**: Pre-process the text string (e.g., strip HTML)
2. **Tokenizer**: Split the string into individual terms or tokens
3. **Token filters**: Modify tokens (e.g., lowercase, remove stop words)

### Built-in Analyzers

- **standard**: The default analyzer that splits text on word boundaries and removes punctuation
- **simple**: Splits text on non-letter characters and lowercases
- **whitespace**: Splits text on whitespace
- **keyword**: Treats the entire string as a single token
- **language analyzers**: Optimized for specific languages (english, french, etc.)

### Custom Analyzers

Create custom analyzers:
```
PUT /my_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_custom_analyzer": {
          "type": "custom",
          "char_filter": [
            "html_strip"
          ],
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "stop",
            "snowball"
          ]
        }
      }
    }
  }
}
```

## Conclusion

This chapter covered the fundamental concepts of Elasticsearch, including documents, indices, basic operations, searching, aggregations, and mappings. Understanding these concepts provides the foundation for building robust search and analytics solutions with Elasticsearch.

In the next chapter, we'll explore Elasticsearch cluster architecture and how to design scalable and resilient Elasticsearch clusters.