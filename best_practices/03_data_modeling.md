# Data Modeling Best Practices

This chapter covers best practices for effective data modeling in Elasticsearch, from index design to optimizing mappings.

## Table of Contents
- [Introduction to Data Modeling](#introduction-to-data-modeling)
- [Index Design Strategies](#index-design-strategies)
- [Mapping Optimization](#mapping-optimization)
- [Type Selection](#type-selection)
- [Document Structure](#document-structure)
- [Relationships in Elasticsearch](#relationships-in-elasticsearch)
- [Time-Based Data Strategies](#time-based-data-strategies)
- [Optimizing for Different Use Cases](#optimizing-for-different-use-cases)
- [Modeling for Scale](#modeling-for-scale)
- [Common Modeling Patterns](#common-modeling-patterns)
- [Migrating and Evolving Models](#migrating-and-evolving-models)
- [Data Modeling Validation and Testing](#data-modeling-validation-and-testing)

## Introduction to Data Modeling

Effective data modeling is critical for achieving optimal performance, scalability, and usability in Elasticsearch. Unlike traditional relational databases, Elasticsearch requires a different approach to data modeling that leverages its distributed document-oriented architecture.

### Elasticsearch Data Model Fundamentals

Elasticsearch's data model consists of several key components:

1. **Documents**: JSON objects that contain your data.
2. **Indices**: Collections of documents with similar characteristics.
3. **Mappings**: Define how documents and their fields are stored and indexed.
4. **Settings**: Configuration options for indices, including sharding and replication.

### Key Principles for Elasticsearch Data Modeling

When designing data models for Elasticsearch, consider these guiding principles:

1. **Denormalization**: Unlike relational databases, Elasticsearch often performs better with denormalized data.
2. **Search-First Design**: Model data to optimize for your search patterns, not just for storage.
3. **Scale Considerations**: Design with future scale in mind, considering sharding strategies and data volume growth.
4. **Query Efficiency**: Structure data to minimize the number and complexity of queries needed.
5. **Storage Efficiency**: Balance search performance against storage requirements.

### The Importance of Data Modeling

Proper data modeling impacts:

- **Query Performance**: A well-designed model can make queries 10-100x faster.
- **Storage Efficiency**: Good models can reduce storage needs by 30-50%.
- **Indexing Speed**: Efficient models can improve indexing throughput by 25-75%.
- **User Experience**: Better models lead to more responsive applications and dashboards.
- **Cluster Stability**: Effective modeling reduces memory pressure and CPU usage.

## Index Design Strategies

### Index Naming Conventions

Consistent, meaningful index naming improves management and supports automation:

```
<application or data source>-<data type or entity>-<YYYY.MM.DD>
```

Examples:
- `logs-apache-2023.06.15`
- `metrics-system-2023.06.15`
- `products-inventory`

Best practices:
- Use lowercase letters only.
- Use hyphens (-) as separators.
- Include date suffixes for time-series data.
- Keep names descriptive but concise.

### Index Granularity

Choose index granularity based on your use case:

1. **Single Index**: Suitable for small amounts of data (< 10 GB) with uniform access patterns.
   ```
   PUT /products
   ```

2. **Daily Indices**: Ideal for logs and time-series data with predictable daily volume.
   ```
   PUT /logs-2023.06.15
   ```

3. **Weekly/Monthly Indices**: Good for time-series data with lower daily volume.
   ```
   PUT /logs-2023.06
   ```

4. **Entity-Based Indices**: Separate indices for different entity types.
   ```
   PUT /users
   PUT /orders
   PUT /products
   ```

Considerations for selecting granularity:
- **Data volume**: Higher volumes benefit from higher granularity.
- **Retention policy**: Different retention needs favor separate indices.
- **Query patterns**: Frequently queried together = same index.
- **Delete patterns**: Data deleted together should be in the same index.

### Shard Sizing and Allocation

Proper shard sizing ensures optimal performance:

- **Target shard size**: 20-40 GB per shard.
- **Primary shards**: Set based on expected data volume: `data_size / target_shard_size`.
- **Replicas**: Typically 1 for production (more for critical workloads).

```json
PUT /logs-2023.06.15
{
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1
  }
}
```

For time-series data with varying volumes, consider using ILM for dynamic shard counts:

```json
PUT _ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "30GB",
            "max_age": "1d"
          },
          "set_priority": {
            "priority": 100
          }
        }
      }
    }
  }
}
```

### Settings Optimization

Configure index settings for your specific use case:

```json
PUT /my-index
{
  "settings": {
    "index": {
      "number_of_shards": 5,
      "number_of_replicas": 1,
      "refresh_interval": "5s",
      "codec": "best_compression",
      "routing.allocation.total_shards_per_node": 2,
      "analysis": {
        "analyzer": {
          "content_analyzer": {
            "type": "custom",
            "tokenizer": "standard",
            "filter": ["lowercase", "stop", "snowball"]
          }
        }
      }
    }
  }
}
```

Key settings to consider:
- **refresh_interval**: Lower for faster search visibility, higher for better indexing performance.
- **codec**: `best_compression` for storage efficiency, `default` for performance.
- **blocks.write**: Enable to prevent writing to an index.
- **mapping.total_fields.limit**: Adjust for complex documents.
- **analysis**: Define custom analyzers and filters.

## Mapping Optimization

### Dynamic vs. Explicit Mappings

**Dynamic mapping** (automatically created):
```json
PUT /my-dynamic-index/_doc/1
{
  "title": "Elasticsearch Guide",
  "published": true,
  "views": 1000,
  "publish_date": "2023-06-15"
}
```

Elasticsearch auto-detects field types, which is convenient but can lead to problems.

**Explicit mapping** (recommended for production):
```json
PUT /my-explicit-index
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "published": {
        "type": "boolean"
      },
      "views": {
        "type": "integer"
      },
      "publish_date": {
        "type": "date",
        "format": "yyyy-MM-dd"
      }
    }
  }
}
```

Best practices:
- Always use explicit mappings in production.
- Consider disabling dynamic mapping to prevent mapping explosions:

```json
PUT /my-strict-index
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      /* defined fields here */
    }
  }
}
```

### Multi-field Mappings

Index the same data in multiple ways for different query types:

```json
PUT /my-index
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "english",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          },
          "search_as_you_type": {
            "type": "search_as_you_type"
          },
          "completion": {
            "type": "completion"
          }
        }
      }
    }
  }
}
```

This approach enables:
- Full-text search using the main `text` field.
- Exact matching and aggregations using the `keyword` sub-field.
- Autocomplete functionality using the `search_as_you_type` sub-field.
- Type-ahead suggestions using the `completion` sub-field.

### Mapping Parameters

Select field-specific mapping parameters for your use case:

```json
PUT /my-detailed-index
{
  "mappings": {
    "properties": {
      "description": {
        "type": "text",
        "analyzer": "english",
        "search_analyzer": "standard",
        "index_options": "positions",
        "term_vector": "with_positions_offsets",
        "norms": false,
        "similarity": "BM25"
      },
      "email": {
        "type": "keyword",
        "ignore_above": 100,
        "normalizer": "lowercase"
      },
      "user_id": {
        "type": "keyword",
        "doc_values": false
      },
      "timestamp": {
        "type": "date",
        "format": "epoch_millis||strict_date_optional_time"
      },
      "price": {
        "type": "scaled_float",
        "scaling_factor": 100
      },
      "location": {
        "type": "geo_point"
      },
      "content": {
        "type": "text",
        "index_phrases": true
      }
    }
  }
}
```

Key parameters to consider:
- **analyzer/search_analyzer**: Define how text is processed.
- **index_options**: Control what information to store for search.
- **norms**: Disable for fields not used for relevance scoring.
- **doc_values**: Disable for fields not used in sorting or aggregations.
- **ignore_above**: Limit the length of indexed keyword fields.
- **normalizer**: Apply pre-processing for keyword fields.
- **copy_to**: Create custom combinations of fields.

### Mapping Templates

Use index templates for consistent mapping across multiple indices:

```json
PUT _index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
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
        },
        "service": {
          "type": "keyword"
        },
        "trace_id": {
          "type": "keyword",
          "doc_values": false
        }
      }
    }
  },
  "priority": 100
}
```

Consider using component templates for reusable mapping sections:

```json
PUT _component_template/logs-mappings
{
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "message": { "type": "text" }
      }
    }
  }
}

PUT _component_template/logs-settings
{
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1
    }
  }
}

PUT _index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "composed_of": ["logs-mappings", "logs-settings"],
  "priority": 100
}
```

## Type Selection

### Field Type Selection Guide

Choosing the right field type improves both search performance and storage efficiency:

| Data Category | Use Case | Recommended Type |
|---------------|----------|------------------|
| Text content | Full-text search | `text` |
| Identifiers | Exact matching | `keyword` |
| Whole numbers | Numeric range queries | `integer`, `long` |
| Decimal numbers | High precision needed | `double`, `float` |
| Decimal numbers | Storage efficient | `scaled_float` |
| True/false values | Boolean filtering | `boolean` |
| Time data | Date/time operations | `date` |
| Geo-coordinates | Location-based search | `geo_point` |
| Geo-shapes | Area-based search | `geo_shape` |
| IP addresses | Network analysis | `ip` |
| Nested objects | Preserving object relations | `nested` |
| Object data | Simple attributes | `object` |
| Range values | Min/max pairs | `integer_range`, `date_range` |
| Structured array | Multiple values | `keyword` array |
| Relational joins | Parent/child | `join` |

### Numeric Type Selection

Choose the appropriate numeric type based on range and precision needs:

```json
PUT /products
{
  "mappings": {
    "properties": {
      "product_id": {
        "type": "keyword"
      },
      "quantity": {
        "type": "integer"  // For values that fit in 32 bits
      },
      "price": {
        "type": "scaled_float",
        "scaling_factor": 100  // Store $10.99 as 1099
      },
      "distance_miles": {
        "type": "half_float"  // 16-bit float for less precision
      },
      "scientific_measurement": {
        "type": "double"  // When full precision is needed
      },
      "huge_count": {
        "type": "long"  // For very large integers
      }
    }
  }
}
```

Guidelines:
- Use the smallest numeric type that covers your range.
- `scaled_float` is excellent for prices (fixed decimal precision).
- `half_float` for metrics where some precision loss is acceptable.
- `integer` for most count fields.

### Text vs. Keyword

Understanding when to use `text` vs. `keyword` is fundamental:

```json
PUT /users
{
  "mappings": {
    "properties": {
      "user_id": {
        "type": "keyword"  // Exact matching only
      },
      "email": {
        "type": "keyword"  // Typically used for exact matching
      },
      "username": {
        "type": "text",    // Searchable by parts
        "fields": {
          "keyword": {     // Also available for exact matching
            "type": "keyword"
          }
        }
      },
      "bio": {
        "type": "text"     // Only full-text search needed
      },
      "status": {
        "type": "keyword"  // Enumerated value
      },
      "tags": {
        "type": "keyword"  // Array of exact values
      }
    }
  }
}
```

General rules:
- Use `keyword` for:
  - Identifiers
  - Emails
  - URLs
  - Status codes
  - Categories/tags
  - Any field used for aggregations, sorting, or exact filtering
- Use `text` for:
  - Human-readable text content
  - Fields that need to be searchable by partial matches
  - Content that benefits from language analysis
- Use multi-fields (`text` with `keyword` sub-field) for:
  - Fields that need both full-text search and aggregations/sorting
  - Fields where you're uncertain about future use

### Object vs. Nested vs. Join

Choose the right relationship model based on query needs:

```json
// Object type (simple parent-child)
PUT /blog
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text"
      },
      "author": {          // Object type (default)
        "properties": {
          "name": {
            "type": "text"
          },
          "email": {
            "type": "keyword"
          }
        }
      },
      "tags": {
        "type": "keyword"
      }
    }
  }
}

// Nested type (for preserving object relationships)
PUT /products
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text"
      },
      "variations": {      // Nested type for array of objects
        "type": "nested",
        "properties": {
          "color": {
            "type": "keyword"
          },
          "size": {
            "type": "keyword"
          },
          "price": {
            "type": "scaled_float",
            "scaling_factor": 100
          }
        }
      }
    }
  }
}

// Join type (for complex relationships)
PUT /department_store
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text"
      },
      "entity_type": {    // Join field for parent-child
        "type": "join",
        "relations": {
          "department": "product"
        }
      }
    }
  }
}

// Indexing parent document
POST /department_store/_doc/dep1
{
  "name": "Electronics",
  "entity_type": "department"
}

// Indexing child document
POST /department_store/_doc/prod1?routing=dep1
{
  "name": "Smartphone",
  "price": 699.99,
  "entity_type": {
    "name": "product",
    "parent": "dep1"
  }
}
```

Recommendations:
- Use **objects** for simple nested attributes that don't need independent querying.
- Use **nested** when you need to maintain the relationship between fields in an array of objects.
- Use **join** only when necessary, as it has performance implications and requires routing.
- Consider complete **denormalization** as an alternative to complex relationships.

## Document Structure

### Flattening vs. Nesting

Choose between flat and nested structures based on query needs:

**Flat structure** (simpler, faster for most queries):
```json
PUT /orders/_doc/1
{
  "order_id": "ORD-12345",
  "customer_id": "CUST-6789",
  "customer_name": "Jane Smith",
  "customer_email": "jane@example.com",
  "order_date": "2023-06-15",
  "order_status": "shipped",
  "order_amount": 129.99,
  "shipping_address_line1": "123 Main St",
  "shipping_address_city": "Boston",
  "shipping_address_state": "MA",
  "shipping_address_zip": "02108",
  "billing_address_line1": "123 Main St",
  "billing_address_city": "Boston",
  "billing_address_state": "MA",
  "billing_address_zip": "02108"
}
```

**Nested structure** (preserves object relationships):
```json
PUT /orders/_doc/1
{
  "order_id": "ORD-12345",
  "order_date": "2023-06-15",
  "order_status": "shipped",
  "order_amount": 129.99,
  "customer": {
    "customer_id": "CUST-6789",
    "name": "Jane Smith",
    "email": "jane@example.com"
  },
  "addresses": [
    {
      "type": "shipping",
      "line1": "123 Main St",
      "city": "Boston",
      "state": "MA",
      "zip": "02108"
    },
    {
      "type": "billing",
      "line1": "123 Main St",
      "city": "Boston",
      "state": "MA",
      "zip": "02108"
    }
  ]
}
```

Guidelines:
- Flat structures are simpler and generally faster for most queries.
- Nested structures are better for maintaining object relationships.
- If querying relationships isn't important, prefer flattening.
- Combine approaches by denormalizing frequently accessed fields.

### Handling Arrays

Arrays in Elasticsearch require careful design to ensure efficient querying:

```json
// Simple array of primitives
PUT /products/_doc/1
{
  "name": "Portable Speaker",
  "tags": ["bluetooth", "wireless", "audio", "portable"]
}

// Array of objects (nested to maintain relationship)
PUT /products
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text"
      },
      "reviews": {
        "type": "nested",
        "properties": {
          "user": {
            "type": "keyword"
          },
          "rating": {
            "type": "byte"
          },
          "text": {
            "type": "text"
          },
          "date": {
            "type": "date"
          }
        }
      }
    }
  }
}

PUT /products/_doc/1
{
  "name": "Wireless Headphones",
  "reviews": [
    {
      "user": "user123",
      "rating": 5,
      "text": "Great sound quality and comfortable to wear.",
      "date": "2023-01-15"
    },
    {
      "user": "user456",
      "rating": 4,
      "text": "Good battery life but a bit heavy.",
      "date": "2023-02-21"
    }
  ]
}
```

Best practices:
- Use `keyword` type for arrays used for filtering and aggregation.
- Always use `nested` type for arrays of objects to preserve relationships.
- Consider performance implications of nested arrays (each nested document is indexed separately).
- For large nested arrays, consider using child documents instead.

### Field Naming Conventions

Consistent field naming improves maintainability and usability:

```json
PUT /users
{
  "mappings": {
    "properties": {
      "user_id": { "type": "keyword" },            // Use snake_case
      "firstName": { "type": "text" },             // Or camelCase
      "email_address": { "type": "keyword" },      // Be consistent
      "account": {
        "properties": {
          "account_creation_date": { "type": "date" },  // Logical prefix
          "account_status": { "type": "keyword" }
        }
      },
      "pmt_method": { "type": "keyword" }          // Avoid abbreviations
    }
  }
}
```

Guidelines:
- Choose a naming convention (snake_case or camelCase) and be consistent.
- Group related fields with common prefixes.
- Use descriptive names over abbreviations.
- Use plural names for array fields.
- Consider field name length limitations in aggregations and scripts.

### Document Size Considerations

Document size affects performance and resource usage:

```json
// Prefer smaller documents
PUT /events/_doc/1
{
  "event_id": "evt-12345",
  "timestamp": "2023-06-15T10:30:45Z",
  "user_id": "usr-6789",
  "action": "login",
  "source_ip": "192.168.1.1",
  "device": "mobile",
  "status": "success"
}
```

Best practices:
- Keep documents under 10MB (hard limit is 100MB).
- Target size of 1KB to 10KB for most use cases.
- Consider splitting very large documents:
  - Based on logical boundaries.
  - Using parent-child relationships for related data.
  - Implementing document references for large-volume relationships.
- Avoid excessive nesting (more than 3-4 levels).

## Relationships in Elasticsearch

### Denormalization

Denormalization improves query performance by reducing join operations:

```json
// Denormalized order document
PUT /orders/_doc/1
{
  "order_id": "ORD-12345",
  "order_date": "2023-06-15T10:30:00Z",
  "status": "shipped",
  "amount": 129.99,
  
  // Denormalized customer information
  "customer": {
    "id": "CUST-6789",
    "name": "Jane Smith",
    "email": "jane@example.com",
    "tier": "gold",
    "account_age_years": 3
  },
  
  // Denormalized product information
  "items": [
    {
      "product_id": "PROD-1001",
      "name": "Wireless Headphones",
      "category": "Electronics",
      "price": 79.99,
      "quantity": 1
    },
    {
      "product_id": "PROD-2002",
      "name": "Smartphone Case",
      "category": "Accessories",
      "price": 24.99,
      "quantity": 2
    }
  ],
  
  // Denormalized shipping information
  "shipping": {
    "method": "express",
    "carrier": "FedEx",
    "tracking_number": "FX123456789",
    "estimated_delivery": "2023-06-18"
  }
}
```

Benefits:
- Faster read operations
- Simplified queries
- Reduced complexity

Drawbacks:
- Increased storage requirements
- Update complexity (multiple documents may need updates)
- Data consistency challenges

When to denormalize:
- Read-heavy workloads
- When related data changes infrequently
- When query performance is critical

### Document References

Use document references for related data that doesn't fit denormalization:

```json
// Customer document
PUT /customers/_doc/CUST-6789
{
  "customer_id": "CUST-6789",
  "name": "Jane Smith",
  "email": "jane@example.com",
  "tier": "gold",
  "registration_date": "2020-03-10",
  "address": {
    "line1": "123 Main St",
    "city": "Boston",
    "state": "MA",
    "zip": "02108"
  }
}

// Order document with reference to customer
PUT /orders/_doc/ORD-12345
{
  "order_id": "ORD-12345",
  "customer_id": "CUST-6789",  // Reference to customer
  "order_date": "2023-06-15T10:30:00Z",
  "status": "shipped",
  "amount": 129.99,
  "items": [
    {
      "product_id": "PROD-1001",  // Reference to product
      "quantity": 1,
      "price": 79.99
    },
    {
      "product_id": "PROD-2002",  // Reference to product
      "quantity": 2,
      "price": 24.99
    }
  ]
}
```

Combining this with client-side joins or application-level joins:

```javascript
// Pseudocode for application-level join
async function getOrderWithDetails(orderId) {
  // Get order document
  const order = await elasticsearch.get({
    index: 'orders',
    id: orderId
  });
  
  // Get referenced customer
  const customer = await elasticsearch.get({
    index: 'customers',
    id: order._source.customer_id
  });
  
  // Get referenced products
  const productIds = order._source.items.map(item => item.product_id);
  const products = await elasticsearch.mget({
    index: 'products',
    body: {
      ids: productIds
    }
  });
  
  // Combine data
  return {
    ...order._source,
    customer: customer._source,
    items: order._source.items.map(item => ({
      ...item,
      product: products.docs.find(p => p._id === item.product_id)._source
    }))
  };
}
```

When to use references:
- When referenced entities are large
- When referenced entities change frequently
- When join operations are infrequent
- When storage efficiency is important

### Parent-Child Relationships

For complex relationships, use the join datatype:

```json
// Define the join relationship
PUT /company
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text"
      },
      "employee_count": {
        "type": "integer"
      },
      "my_join_field": {
        "type": "join",
        "relations": {
          "department": "employee"
        }
      }
    }
  }
}

// Index a parent document (department)
PUT /company/_doc/dept1
{
  "name": "Engineering",
  "employee_count": 50,
  "my_join_field": "department"
}

// Index a child document (employee)
PUT /company/_doc/emp1?routing=dept1
{
  "name": "John Doe",
  "age": 34,
  "position": "Senior Developer",
  "my_join_field": {
    "name": "employee",
    "parent": "dept1"
  }
}

// Parent-child join query
GET /company/_search
{
  "query": {
    "has_parent": {
      "parent_type": "department",
      "query": {
        "term": {
          "name": "engineering"
        }
      }
    }
  }
}
```

Important considerations:
- Parent and child documents must be indexed on the same shard (using routing).
- Has performance implications for large datasets.
- Use sparingly and only when really needed.
- Not suitable for relationships with high cardinality.

### Nested Documents

For arrays of objects where relationships matter:

```json
// Define nested mapping
PUT /products
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text"
      },
      "variations": {
        "type": "nested",
        "properties": {
          "color": {
            "type": "keyword"
          },
          "size": {
            "type": "keyword"
          },
          "price": {
            "type": "float"
          },
          "stock": {
            "type": "integer"
          }
        }
      }
    }
  }
}

// Index document with nested objects
PUT /products/_doc/1
{
  "name": "Cotton T-Shirt",
  "variations": [
    {
      "color": "blue",
      "size": "S",
      "price": 19.99,
      "stock": 25
    },
    {
      "color": "blue",
      "size": "M",
      "price": 19.99,
      "stock": 15
    },
    {
      "color": "red",
      "size": "M",
      "price": 22.99,
      "stock": 10
    }
  ]
}

// Nested query
GET /products/_search
{
  "query": {
    "nested": {
      "path": "variations",
      "query": {
        "bool": {
          "must": [
            { "term": { "variations.color": "blue" } },
            { "term": { "variations.size": "M" } }
          ]
        }
      }
    }
  }
}
```

When to use nested documents:
- For arrays of objects where the relationship between fields is important.
- When you need to query against combinations of fields within the objects.
- When the number of nested objects per document is reasonable (typically < 100).

## Time-Based Data Strategies

### Time-Series Index Patterns

For time-series data, design indices with time in mind:

```json
// Daily index pattern
PUT /logs-2023.06.15
{
  "settings": {
    "number_of_shards": 3,
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

// Template for daily indices
PUT _index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 3,
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
  },
  "composed_of": []
}
```

Best practices:
- Use time-based index names (`logs-YYYY.MM.DD`).
- Right-size shards based on expected daily data volume.
- Use index templates to ensure consistent mappings.
- Consider time-based queries in your index pattern design.

### Index Lifecycle Management

Use ILM to automate time-series index management:

```json
// Define ILM policy
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
          "allocate": {
            "number_of_replicas": 1,
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
            "number_of_replicas": 0,
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

// Create an index with the policy
PUT /logs-000001
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "index.lifecycle.name": "logs-policy",
    "index.lifecycle.rollover_alias": "logs"
  },
  "aliases": {
    "logs": {
      "is_write_index": true
    }
  }
}
```

Benefits:
- Automated lifecycle management
- Optimized storage and performance
- Simplified retention management
- Tiered storage capabilities

### Date Histogram Aggregations

Optimize for time-based aggregations:

```json
// Efficient time-series aggregation
GET /logs-*/_search
{
  "size": 0,
  "aggs": {
    "errors_over_time": {
      "date_histogram": {
        "field": "@timestamp",
        "calendar_interval": "hour",
        "format": "yyyy-MM-dd HH:mm"
      },
      "aggs": {
        "error_levels": {
          "terms": {
            "field": "level",
            "include": ["ERROR", "FATAL"]
          }
        }
      }
    }
  },
  "query": {
    "range": {
      "@timestamp": {
        "gte": "now-7d",
        "lte": "now"
      }
    }
  }
}
```

Optimization strategies:
- Use `date_histogram` aggregations with appropriate intervals.
- Apply filters before aggregations to reduce dataset size.
- Set `size: 0` when only aggregation results are needed.
- Use index patterns that match the time range of your query.
- Consider pre-calculated aggregations for complex dashboards.

## Optimizing for Different Use Cases

### Search-Optimized Mappings

Configure mappings for optimal search performance:

```json
PUT /blog_posts
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "english_stop",
            "english_stemmer"
          ]
        }
      },
      "filter": {
        "english_stop": {
          "type": "stop",
          "stopwords": "_english_"
        },
        "english_stemmer": {
          "type": "stemmer",
          "language": "english"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "my_analyzer",
        "boost": 2.0,
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "content": {
        "type": "text",
        "analyzer": "my_analyzer"
      },
      "tags": {
        "type": "keyword"
      },
      "author": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      },
      "publish_date": {
        "type": "date"
      }
    }
  }
}
```

Key search optimizations:
- Configure appropriate analyzers and filters.
- Use multi-fields for both full text and exact matching.
- Set field boosts for relevance control.
- Enable indexing of positions and offsets for phrase queries.

### Aggregation-Optimized Mappings

Configure mappings for efficient aggregations:

```json
PUT /products
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "category": {
        "type": "keyword"
      },
      "brand": {
        "type": "keyword"
      },
      "price": {
        "type": "scaled_float",
        "scaling_factor": 100
      },
      "color": {
        "type": "keyword"
      },
      "description": {
        "type": "text",
        "doc_values": false,  // Disable for fields not used in aggregations
        "norms": false
      },
      "review_count": {
        "type": "integer"
      },
      "avg_rating": {
        "type": "half_float"
      },
      "created_at": {
        "type": "date"
      }
    }
  }
}
```

Key aggregation optimizations:
- Use `keyword` type for fields used in terms aggregations.
- Use `scaled_float` for numeric fields to reduce precision where appropriate.
- Enable `doc_values` for fields used in aggregations (enabled by default).
- Disable `doc_values` for fields not used in aggregations.
- Consider cardinality when designing fields for aggregations.

### Geospatial Data Modeling

Optimize for geographic search and analytics:

```json
PUT /locations
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
      "category": {
        "type": "keyword"
      },
      "location": {
        "type": "geo_point"
      },
      "area": {
        "type": "geo_shape"
      },
      "address": {
        "properties": {
          "street": {
            "type": "text"
          },
          "city": {
            "type": "keyword"
          },
          "state": {
            "type": "keyword"
          },
          "zip": {
            "type": "keyword"
          },
          "country": {
            "type": "keyword"
          }
        }
      },
      "opening_hours": {
        "type": "text"
      },
      "tags": {
        "type": "keyword"
      }
    }
  }
}

// Index a document with geo data
PUT /locations/_doc/1
{
  "name": "Central Park",
  "category": "Park",
  "location": {
    "lat": 40.785091,
    "lon": -73.968285
  },
  "area": {
    "type": "polygon",
    "coordinates": [
      [
        [-73.958, 40.794],
        [-73.949, 40.783],
        [-73.973, 40.778],
        [-73.982, 40.789],
        [-73.958, 40.794]
      ]
    ]
  },
  "address": {
    "street": "Central Park",
    "city": "New York",
    "state": "NY",
    "zip": "10022",
    "country": "USA"
  },
  "tags": ["park", "recreation", "tourism"]
}

// Geo distance query
GET /locations/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "term": {
            "category": "Park"
          }
        }
      ],
      "filter": [
        {
          "geo_distance": {
            "distance": "5km",
            "location": {
              "lat": 40.78,
              "lon": -73.97
            }
          }
        }
      ]
    }
  },
  "sort": [
    {
      "_geo_distance": {
        "location": {
          "lat": 40.78,
          "lon": -73.97
        },
        "order": "asc",
        "unit": "km"
      }
    }
  ]
}
```

Key geospatial optimizations:
- Use `geo_point` for point locations (coordinates).
- Use `geo_shape` for areas (polygons, multipolygons).
- Create indexes for frequent geo-queries.
- Use geo_distance filters rather than queries for better performance.
- Consider precision requirements when using geo_grid aggregations.

## Modeling for Scale

### Sharding Strategy

Design sharding strategies for growth:

```json
// Initial shard settings for a growing index
PUT /products
{
  "settings": {
    "number_of_shards": 6,     // Plan for future growth
    "number_of_replicas": 1,
    "index.routing.allocation.total_shards_per_node": 2  // Distribute shards
  },
  "mappings": {
    "properties": { /* ... */ }
  }
}

// Time-based indices with appropriate sharding
PUT /logs-2023.06.15
{
  "settings": {
    "number_of_shards": 3,     // Based on daily volume
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": { /* ... */ }
  }
}
```

Sharding best practices:
- Target shard size: 20GB to 40GB per shard.
- Consider growth pattern: plan for 3-6 months ahead.
- For time-series data: size shards based on daily/weekly volume.
- Distribution: use routing to control shard allocation.
- Over-sharding hurts performance; under-sharding limits scalability.
- Consider query patterns when setting shard count.

### Routing for Scalability

Custom routing improves shard distribution and query performance:

```json
// Define routing in mappings
PUT /logs
{
  "mappings": {
    "_routing": {
      "required": true
    },
    "properties": {
      "tenant_id": {
        "type": "keyword"
      },
      "timestamp": {
        "type": "date"
      },
      "message": {
        "type": "text"
      }
    }
  }
}

// Index with custom routing
PUT /logs/_doc/1?routing=tenant123
{
  "tenant_id": "tenant123",
  "timestamp": "2023-06-15T12:30:00Z",
  "message": "User login successful"
}

// Query with routing for shard selection
GET /logs/_search?routing=tenant123
{
  "query": {
    "match": {
      "message": "login"
    }
  }
}
```

When to use custom routing:
- Multi-tenant applications
- When data has a natural partitioning key
- To improve query performance for filtered datasets
- To reduce shard searches for frequent query patterns

### Field Limits and Mapping Explosion

Prevent mapping explosion in high-cardinality fields:

```json
// Strict dynamic mapping
PUT /logs
{
  "settings": {
    "index.mapping.total_fields.limit": 2000,
    "index.mapping.depth.limit": 20,
    "index.mapping.nested_fields.limit": 50,
    "index.mapping.nested_objects.limit": 10000
  },
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "known_field1": {
        "type": "text"
      },
      "known_field2": {
        "type": "keyword"
      },
      // Only explicitly defined fields are allowed
    }
  }
}

// Runtime fields for dynamic data
PUT /logs/_mapping
{
  "runtime": {
    "parsed_agent": {
      "type": "keyword",
      "script": {
        "source": """
          String agent = doc['user_agent'].value;
          if (agent.contains('Chrome')) {
            emit('Chrome');
          } else if (agent.contains('Firefox')) {
            emit('Firefox');
          } else {
            emit('Other');
          }
        """
      }
    }
  }
}
```

Preventing mapping explosion:
- Use `dynamic: "strict"` for controlled mappings.
- Define explicit mappings for known fields.
- Use `flattened` type for objects with unknown structures.
- Set appropriate limits on fields and nesting.
- Consider runtime fields for dynamic data analysis.

## Common Modeling Patterns

### Event Data Model

Pattern for modeling time-based events:

```json
PUT /events
{
  "mappings": {
    "properties": {
      "event_id": {
        "type": "keyword"
      },
      "@timestamp": {
        "type": "date"
      },
      "event_type": {
        "type": "keyword"
      },
      "event_source": {
        "type": "keyword"
      },
      "event_action": {
        "type": "keyword"
      },
      "actor": {
        "properties": {
          "id": {
            "type": "keyword"
          },
          "name": {
            "type": "keyword"
          },
          "type": {
            "type": "keyword"
          }
        }
      },
      "target": {
        "properties": {
          "id": {
            "type": "keyword"
          },
          "name": {
            "type": "keyword"
          },
          "type": {
            "type": "keyword"
          }
        }
      },
      "context": {
        "properties": {
          "ip": {
            "type": "ip"
          },
          "user_agent": {
            "type": "text",
            "fields": {
              "keyword": {
                "type": "keyword",
                "ignore_above": 256
              }
            }
          },
          "request_id": {
            "type": "keyword"
          },
          "session_id": {
            "type": "keyword"
          }
        }
      },
      "metadata": {
        "type": "flattened"  // For unknown custom metadata
      }
    }
  }
}
```

Key characteristics:
- Common event fields for consistency
- Structured actor and target objects
- Context information for environmental data
- Flattened type for variable metadata
- Optimized for time-series queries

### Log Data Model

Pattern for modeling log data:

```json
PUT /logs
{
  "mappings": {
    "properties": {
      "@timestamp": {
        "type": "date"
      },
      "log": {
        "properties": {
          "level": {
            "type": "keyword"
          },
          "logger": {
            "type": "keyword"
          },
          "origin": {
            "properties": {
              "file": {
                "properties": {
                  "name": {
                    "type": "keyword"
                  },
                  "line": {
                    "type": "integer"
                  }
                }
              },
              "function": {
                "type": "keyword"
              }
            }
          }
        }
      },
      "message": {
        "type": "text",
        "fields": {
          "raw": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "process": {
        "properties": {
          "name": {
            "type": "keyword"
          },
          "pid": {
            "type": "long"
          },
          "thread": {
            "properties": {
              "id": {
                "type": "long"
              },
              "name": {
                "type": "keyword"
              }
            }
          }
        }
      },
      "service": {
        "properties": {
          "name": {
            "type": "keyword"
          },
          "version": {
            "type": "keyword"
          },
          "environment": {
            "type": "keyword"
          }
        }
      },
      "host": {
        "properties": {
          "name": {
            "type": "keyword"
          },
          "ip": {
            "type": "ip"
          }
        }
      },
      "error": {
        "properties": {
          "message": {
            "type": "text"
          },
          "stack_trace": {
            "type": "text",
            "index": false
          },
          "type": {
            "type": "keyword"
          }
        }
      },
      "tags": {
        "type": "keyword"
      }
    }
  }
}
```

Key characteristics:
- Structured format following ECS (Elastic Common Schema)
- Hierarchical organization of related fields
- Multi-field type for message (searchable text + raw keyword)
- Tags for flexible categorization
- Error handling for exception details

### E-commerce Product Model

Pattern for e-commerce products:

```json
PUT /products
{
  "mappings": {
    "properties": {
      "product_id": {
        "type": "keyword"
      },
      "name": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "description": {
        "type": "text"
      },
      "brand": {
        "type": "keyword"
      },
      "categories": {
        "type": "keyword"
      },
      "price": {
        "type": "scaled_float",
        "scaling_factor": 100
      },
      "discount_price": {
        "type": "scaled_float",
        "scaling_factor": 100
      },
      "availability": {
        "type": "boolean"
      },
      "in_stock": {
        "type": "boolean"
      },
      "stock_quantity": {
        "type": "integer"
      },
      "rating": {
        "type": "float"
      },
      "review_count": {
        "type": "integer"
      },
      "attributes": {
        "type": "nested",
        "properties": {
          "name": {
            "type": "keyword"
          },
          "value": {
            "type": "keyword"
          },
          "searchable_value": {
            "type": "text"
          }
        }
      },
      "variations": {
        "type": "nested",
        "properties": {
          "variation_id": {
            "type": "keyword"
          },
          "attributes": {
            "type": "nested",
            "properties": {
              "name": {
                "type": "keyword"
              },
              "value": {
                "type": "keyword"
              }
            }
          },
          "price": {
            "type": "scaled_float",
            "scaling_factor": 100
          },
          "discount_price": {
            "type": "scaled_float",
            "scaling_factor": 100
          },
          "stock_quantity": {
            "type": "integer"
          },
          "sku": {
            "type": "keyword"
          }
        }
      },
      "created_at": {
        "type": "date"
      },
      "updated_at": {
        "type": "date"
      },
      "tags": {
        "type": "keyword"
      },
      "images": {
        "properties": {
          "url": {
            "type": "keyword"
          },
          "alt": {
            "type": "text"
          }
        }
      }
    }
  }
}
```

Key characteristics:
- Keyword fields for faceted navigation
- Nested attributes and variations
- Optimized for both search and filtering
- Scaled_float for price values
- Multi-field support for sorting and searching

## Migrating and Evolving Models

### Reindexing Strategies

Safely evolve your data model with reindexing:

```json
// Create a new index with updated mapping
PUT /customers_v2
{
  "mappings": {
    "properties": {
      "id": {
        "type": "keyword"
      },
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
      "address": {
        "properties": {
          "street": {
            "type": "text"
          },
          "city": {
            "type": "keyword"
          },
          "state": {
            "type": "keyword"
          },
          "zip": {
            "type": "keyword"
          },
          "country": {
            "type": "keyword"
          },
          "location": {
            "type": "geo_point"  // New field in v2
          }
        }
      },
      "phone": {
        "type": "keyword"
      }
    }
  }
}

// Reindex data from old to new index
POST /_reindex
{
  "source": {
    "index": "customers_v1"
  },
  "dest": {
    "index": "customers_v2"
  },
  "script": {
    "source": """
      // Transform data during reindex
      if (ctx._source.containsKey('address')) {
        if (ctx._source.address.containsKey('lat') && ctx._source.address.containsKey('lon')) {
          ctx._source.address.location = [ctx._source.address.lon, ctx._source.address.lat];
          ctx._source.address.remove('lat');
          ctx._source.address.remove('lon');
        }
      }
    """
  }
}

// Create alias to switch traffic
POST /_aliases
{
  "actions": [
    {
      "remove": {
        "index": "customers_v1",
        "alias": "customers"
      }
    },
    {
      "add": {
        "index": "customers_v2",
        "alias": "customers"
      }
    }
  ]
}
```

Best practices for reindexing:
- Use aliases to minimize downtime.
- Transform data during reindexing with scripts.
- Use the Task API to monitor reindexing progress.
- Consider batching for large datasets.
- Test reindexing in a non-production environment first.

### Zero-Downtime Index Updates

Perform updates without disrupting operations:

1. **Create alias for transparent switching**:

```json
// Initial setup
PUT /products_v1
{
  "mappings": {
    "properties": { /* original mapping */ }
  }
}

POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "products_v1",
        "alias": "products"
      }
    }
  ]
}
```

2. **Create new index with updated mapping**:

```json
PUT /products_v2
{
  "mappings": {
    "properties": { /* updated mapping */ }
  }
}
```

3. **Reindex data to new index**:

```json
POST /_reindex
{
  "source": {
    "index": "products_v1"
  },
  "dest": {
    "index": "products_v2"
  }
}
```

4. **Switch alias to new index**:

```json
POST /_aliases
{
  "actions": [
    {
      "remove": {
        "index": "products_v1",
        "alias": "products"
      }
    },
    {
      "add": {
        "index": "products_v2",
        "alias": "products"
      }
    }
  ]
}
```

5. **Use dual write approach for high-write environments**:

```json
// Temporary dual-write strategy
// Client application writes to both indices
PUT /products_v1/_doc/123
{
  /* document data */
}

PUT /products_v2/_doc/123
{
  /* document data with updated structure */
}
```

### Update By Query for In-Place Changes

Make in-place updates when possible:

```json
// Add a new field to existing documents
POST /customers/_update_by_query
{
  "script": {
    "source": "ctx._source.account_status = 'active'",
    "lang": "painless"
  },
  "query": {
    "bool": {
      "must_not": {
        "exists": {
          "field": "account_status"
        }
      }
    }
  }
}

// Transform existing values
POST /products/_update_by_query
{
  "script": {
    "source": """
      if (ctx._source.price_usd != null) {
        ctx._source.price = ctx._source.price_usd;
        ctx._source.currency = 'USD';
        ctx._source.remove('price_usd');
      }
    """,
    "lang": "painless"
  },
  "query": {
    "exists": {
      "field": "price_usd"
    }
  }
}
```

When to use update by query:
- Adding new fields with default values
- Making field name changes
- Simple data transformations
- Enriching existing data

Limitations:
- Cannot change field types
- Cannot change how fields are indexed
- May cause performance impact on large indices

## Data Modeling Validation and Testing

### Model Testing Approach

A systematic approach to validate data models:

1. **Create a test index**:

```json
PUT /products_test
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      // Your proposed mapping
    }
  }
}
```

2. **Index sample documents**:

```json
// Index sample documents
POST /products_test/_bulk
{"index":{"_id":"1"}}
{"name":"Product 1","category":"Electronics","price":199.99,"tags":["wireless","bluetooth"]}
{"index":{"_id":"2"}}
{"name":"Product 2","category":"Clothing","price":59.99,"tags":["summer","cotton"]}
```

3. **Test query patterns**:

```json
// Test search query
GET /products_test/_search
{
  "query": {
    "bool": {
      "must": [
        {"match": {"name": "product"}}
      ],
      "filter": [
        {"term": {"category": "electronics"}},
        {"range": {"price": {"gte": 100}}}
      ]
    }
  }
}

// Test aggregation
GET /products_test/_search
{
  "size": 0,
  "aggs": {
    "categories": {
      "terms": {
        "field": "category",
        "size": 10
      },
      "aggs": {
        "avg_price": {
          "avg": {
            "field": "price"
          }
        }
      }
    }
  }
}
```

4. **Evaluate query performance**:

```json
GET /products_test/_search
{
  "query": {
    "match": {"name": "product"}
  },
  "profile": true
}
```

### Model Validation Checklist

Use this checklist to validate your data model:

1. **Mapping Validation**:
   - [ ] Field types match the data and query patterns
   - [ ] Text analyzer configuration is appropriate
   - [ ] Multi-fields are defined where needed
   - [ ] Nested fields are used when object relationships matter
   - [ ] Field limits are set appropriately

2. **Query Performance**:
   - [ ] Common queries execute efficiently
   - [ ] Sorting operations work as expected
   - [ ] Aggregations produce correct results
   - [ ] Response times are within acceptable limits

3. **Data Integrity**:
   - [ ] Data can be indexed without errors
   - [ ] JSON structures are valid
   - [ ] Field values conform to the expected types
   - [ ] Date formats are parsed correctly

4. **Scaling Considerations**:
   - [ ] Shard count is appropriate for the data volume
   - [ ] Index size projections are reasonable
   - [ ] Mapping won't lead to field explosion

5. **Use Case Coverage**:
   - [ ] The model supports all required query types
   - [ ] The model supports all required aggregations
   - [ ] The model balances search and aggregation needs

### Performance Evaluation Script

Use this script to evaluate model performance:

```bash
#!/bin/bash
# Data Model Performance Test Script

# Variables
INDEX="test_model"
DOCUMENT_COUNT=10000
QUERY_COUNT=100

# Create test index
echo "Creating test index..."
curl -XPUT "localhost:9200/$INDEX" -H 'Content-Type: application/json' -d '{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "field1": {"type": "keyword"},
      "field2": {"type": "text"},
      "field3": {"type": "integer"},
      "nested_field": {
        "type": "nested",
        "properties": {
          "sub1": {"type": "keyword"},
          "sub2": {"type": "integer"}
        }
      }
    }
  }
}'

# Generate and index test data
echo "Generating and indexing $DOCUMENT_COUNT documents..."
for i in $(seq 1 $DOCUMENT_COUNT); do
  curl -XPOST "localhost:9200/$INDEX/_doc" -H 'Content-Type: application/json' -d '{
    "field1": "value'$i'",
    "field2": "This is a sample text value for document '$i'",
    "field3": '"$i"',
    "nested_field": [
      {"sub1": "a", "sub2": '"$i"'},
      {"sub1": "b", "sub2": '"$(($i+100))"'}
    ]
  }' > /dev/null 2>&1
  
  # Show progress
  if (( i % 1000 == 0 )); then
    echo "  Indexed $i documents..."
  fi
done

# Refresh index
curl -XPOST "localhost:9200/$INDEX/_refresh" > /dev/null

# Test queries
echo "Testing query performance..."

# Test 1: Simple term query
echo "Test 1: Simple term query"
time for i in $(seq 1 $QUERY_COUNT); do
  curl -s -XGET "localhost:9200/$INDEX/_search" -H 'Content-Type: application/json' -d '{
    "query": {
      "term": {
        "field1": "value'$(($RANDOM % $DOCUMENT_COUNT + 1))'"
      }
    }
  }' > /dev/null
done

# Test 2: Full text query
echo "Test 2: Full text query"
time for i in $(seq 1 $QUERY_COUNT); do
  curl -s -XGET "localhost:9200/$INDEX/_search" -H 'Content-Type: application/json' -d '{
    "query": {
      "match": {
        "field2": "sample text"
      }
    }
  }' > /dev/null
done

# Test 3: Nested query
echo "Test 3: Nested query"
time for i in $(seq 1 $QUERY_COUNT); do
  curl -s -XGET "localhost:9200/$INDEX/_search" -H 'Content-Type: application/json' -d '{
    "query": {
      "nested": {
        "path": "nested_field",
        "query": {
          "bool": {
            "must": [
              {"term": {"nested_field.sub1": "a"}},
              {"range": {"nested_field.sub2": {"gt": '$(($RANDOM % 5000))'}}
            ]
          }
        }
      }
    }
  }' > /dev/null
done

# Test 4: Aggregation
echo "Test 4: Aggregation"
time for i in $(seq 1 $QUERY_COUNT); do
  curl -s -XGET "localhost:9200/$INDEX/_search" -H 'Content-Type: application/json' -d '{
    "size": 0,
    "aggs": {
      "range": {
        "range": {
          "field": "field3",
          "ranges": [
            {"to": 2500},
            {"from": 2500, "to": 5000},
            {"from": 5000, "to": 7500},
            {"from": 7500}
          ]
        }
      }
    }
  }' > /dev/null
done

# Clean up
echo "Cleaning up..."
curl -XDELETE "localhost:9200/$INDEX" > /dev/null

echo "Performance testing complete!"
```

## Summary

Effective data modeling is the foundation for building high-performing, scalable, and efficient Elasticsearch applications. By following the best practices outlined in this chapter, you can design data models that:

1. **Optimize search performance** through appropriate mappings and analyzers
2. **Reduce storage requirements** by selecting the right field types
3. **Improve query efficiency** with denormalization and appropriate relationship models
4. **Support scalability** with proper sharding and routing strategies
5. **Enable easy evolution** through well-planned migration strategies

Remember that data modeling is both an art and a science. The optimal model for your use case depends on your specific requirements, query patterns, and scale. Always test your models with representative data and queries before deploying to production.

## References

- [Elasticsearch Field Data Types](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-types.html)
- [Elasticsearch Mapping Parameters](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-params.html)
- [Index Templates](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-templates.html)
- [Index Lifecycle Management](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html)
- [Reindex API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-reindex.html)