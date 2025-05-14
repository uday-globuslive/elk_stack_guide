# Querying and Aggregations

## Introduction to Elasticsearch Queries

Elasticsearch provides a powerful query language through its Query DSL (Domain Specific Language), which allows you to search and filter your data in sophisticated ways. This chapter covers the fundamentals of querying in Elasticsearch, from basic searches to complex aggregations and analytics.

The Query DSL is a JSON-based interface that gives you access to the full power of Elasticsearch's search capabilities. By understanding the query types and how to combine them, you can create precise search experiences that match your application's requirements.

## Basic Querying

### Query Structure

A basic Elasticsearch query has this structure:

```json
GET /my_index/_search
{
  "query": {
    // Query goes here
  }
}
```

Without specifying a query, Elasticsearch uses the `match_all` query:

```json
GET /my_index/_search
{
  "query": {
    "match_all": {}
  }
}
```

This returns all documents in the index.

### Term-Level Queries

Term-level queries search for exact values without analysis:

#### Term Query

Finds documents containing the exact term:

```json
GET /products/_search
{
  "query": {
    "term": {
      "category": "electronics"
    }
  }
}
```

This matches documents where the `category` field exactly equals "electronics" (not "Electronics" or "ELECTRONICS").

#### Terms Query

Matches multiple exact values:

```json
GET /products/_search
{
  "query": {
    "terms": {
      "category": ["electronics", "computers"]
    }
  }
}
```

#### Range Query

Matches values within a range:

```json
GET /products/_search
{
  "query": {
    "range": {
      "price": {
        "gte": 100,
        "lte": 200
      }
    }
  }
}
```

This finds products with prices between $100 and $200, inclusive.

For dates:

```json
GET /logs/_search
{
  "query": {
    "range": {
      "timestamp": {
        "gte": "2023-01-01",
        "lt": "2023-02-01",
        "format": "yyyy-MM-dd"
      }
    }
  }
}
```

#### Exists Query

Finds documents where a field exists:

```json
GET /products/_search
{
  "query": {
    "exists": {
      "field": "discount"
    }
  }
}
```

#### Prefix Query

Matches documents with field values starting with a prefix:

```json
GET /products/_search
{
  "query": {
    "prefix": {
      "name": "iph"
    }
  }
}
```

#### Wildcard Query

Matches documents using wildcards:

```json
GET /products/_search
{
  "query": {
    "wildcard": {
      "name": "i*e"
    }
  }
}
```

The wildcard `*` matches zero or more characters, while `?` matches exactly one character.

#### Regular Expression Query

Matches documents using regular expressions:

```json
GET /products/_search
{
  "query": {
    "regexp": {
      "sku": "[A-Z]{3}[0-9]{4}"
    }
  }
}
```

### Full-Text Queries

Full-text queries analyze the query string before searching, making them suitable for human language:

#### Match Query

The most basic full-text query:

```json
GET /articles/_search
{
  "query": {
    "match": {
      "content": "elasticsearch query"
    }
  }
}
```

This analyzes "elasticsearch query" and finds documents containing either "elasticsearch" OR "query" in the content field.

#### Match Phrase Query

Finds documents with the exact phrase:

```json
GET /articles/_search
{
  "query": {
    "match_phrase": {
      "content": "elasticsearch query"
    }
  }
}
```

This finds documents where "elasticsearch" and "query" appear next to each other in the order specified.

#### Multi-Match Query

Searches multiple fields:

```json
GET /articles/_search
{
  "query": {
    "multi_match": {
      "query": "elasticsearch guide",
      "fields": ["title", "content"]
    }
  }
}
```

Options for field boosting:

```json
GET /articles/_search
{
  "query": {
    "multi_match": {
      "query": "elasticsearch guide",
      "fields": ["title^3", "content"]
    }
  }
}
```

This weighs matches in the title field 3 times higher than matches in the content field.

#### Query String Query

Supports complex syntax including wildcards, phrases, and operators:

```json
GET /articles/_search
{
  "query": {
    "query_string": {
      "query": "(elasticsearch OR elastic) AND guide",
      "fields": ["title", "content"]
    }
  }
}
```

### Compound Queries

Compound queries combine multiple queries for more complex searches:

#### Boolean Query

The most flexible compound query:

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "category": "electronics" } }
      ],
      "filter": [
        { "range": { "price": { "gte": 100, "lte": 200 } } }
      ],
      "should": [
        { "match": { "name": "iphone" } }
      ],
      "must_not": [
        { "term": { "status": "discontinued" } }
      ]
    }
  }
}
```

This query:
- MUST match electronics category (affects relevance score)
- FILTER for prices between $100-$200 (doesn't affect score)
- SHOULD match "iphone" in name (boosts relevance if matched)
- MUST NOT match discontinued status (excludes these documents)

#### Boosting Query

Decreases the relevance of matching documents in the negative query:

```json
GET /products/_search
{
  "query": {
    "boosting": {
      "positive": {
        "match": { "name": "phone" }
      },
      "negative": {
        "term": { "name": "iphone" }
      },
      "negative_boost": 0.5
    }
  }
}
```

This reduces the score of documents containing "iphone" by multiplying it by 0.5.

## Query Context vs. Filter Context

Elasticsearch has two query execution contexts:

### Query Context

- Answers "How well does this document match the query?"
- Calculates and uses relevance scores
- Typically slower than filter context
- Example: `must` and `should` clauses in a bool query

### Filter Context

- Answers "Does this document match the query?" (yes/no)
- Doesn't calculate relevance scores
- Can be cached for better performance
- Example: `filter` and `must_not` clauses in a bool query

Best practice: Use filter context for exact matching and query context for relevance ranking.

## Sorting and Pagination

### Sorting Results

Sort by one or more fields:

```json
GET /products/_search
{
  "query": { "match_all": {} },
  "sort": [
    { "price": "asc" },
    { "rating": "desc" }
  ]
}
```

Sort by relevance and then by field:

```json
GET /products/_search
{
  "query": { "match": { "name": "phone" } },
  "sort": [
    "_score",
    { "price": "asc" }
  ]
}
```

### Pagination

Standard pagination with `from` and `size`:

```json
GET /products/_search
{
  "from": 0,  // Starting from the first result
  "size": 10, // Return 10 results
  "query": { "match_all": {} }
}
```

For the next page:

```json
GET /products/_search
{
  "from": 10, // Starting from the 11th result
  "size": 10, // Return 10 results
  "query": { "match_all": {} }
}
```

Note: The `from` + `size` cannot exceed the `index.max_result_window` setting (default: 10,000).

### Deep Pagination with Search After

For deep pagination, use `search_after`:

```json
GET /products/_search
{
  "size": 10,
  "query": { "match_all": {} },
  "sort": [
    { "price": "asc" },
    { "id": "asc" }  // Add a tiebreaker
  ],
  "search_after": [50.99, "a123"]  // Values from the last result
}
```

This approach requires:
- Consistent sort order
- Sort fields that uniquely identify documents
- The `search_after` values from the last result of the previous page

## Aggregations

Aggregations allow you to analyze and summarize your data, similar to GROUP BY in SQL.

### Bucket Aggregations

Bucket aggregations group documents based on criteria:

#### Terms Aggregation

Groups documents by field values:

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "categories": {
      "terms": {
        "field": "category",
        "size": 10
      }
    }
  }
}
```

This returns the top 10 categories and their document counts.

#### Range Aggregation

Groups documents into ranges:

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          { "to": 50 },
          { "from": 50, "to": 100 },
          { "from": 100, "to": 200 },
          { "from": 200 }
        ]
      }
    }
  }
}
```

#### Date Histogram Aggregation

Creates time-based buckets:

```json
GET /logs/_search
{
  "size": 0,
  "aggs": {
    "logs_per_day": {
      "date_histogram": {
        "field": "timestamp",
        "calendar_interval": "day"
      }
    }
  }
}
```

Other intervals include: `minute`, `hour`, `week`, `month`, and `year`.

#### Filters Aggregation

Creates buckets based on filters:

```json
GET /logs/_search
{
  "size": 0,
  "aggs": {
    "errors_by_type": {
      "filters": {
        "filters": {
          "validation_errors": { "match": { "message": "validation" } },
          "timeout_errors": { "match": { "message": "timeout" } },
          "server_errors": { "match": { "message": "server error" } }
        }
      }
    }
  }
}
```

### Metric Aggregations

Metric aggregations compute statistics over a set of documents:

#### Stats Aggregation

Computes min, max, sum, count, and avg:

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "price_stats": {
      "stats": {
        "field": "price"
      }
    }
  }
}
```

#### Extended Stats Aggregation

Includes additional statistics:

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "price_extended_stats": {
      "extended_stats": {
        "field": "price"
      }
    }
  }
}
```

This includes standard deviation, variance, etc.

#### Percentiles Aggregation

Calculates percentiles for a field:

```json
GET /response_times/_search
{
  "size": 0,
  "aggs": {
    "response_time_percentiles": {
      "percentiles": {
        "field": "duration",
        "percents": [50, 75, 95, 99]
      }
    }
  }
}
```

#### Cardinality Aggregation

Counts approximate unique values:

```json
GET /logs/_search
{
  "size": 0,
  "aggs": {
    "unique_users": {
      "cardinality": {
        "field": "user_id"
      }
    }
  }
}
```

### Pipeline Aggregations

Pipeline aggregations process the output of other aggregations:

#### Moving Average

Calculates moving averages over a date histogram:

```json
GET /sales/_search
{
  "size": 0,
  "aggs": {
    "sales_per_month": {
      "date_histogram": {
        "field": "date",
        "calendar_interval": "month"
      },
      "aggs": {
        "monthly_sales": {
          "sum": {
            "field": "amount"
          }
        },
        "sales_moving_avg": {
          "moving_avg": {
            "buckets_path": "monthly_sales"
          }
        }
      }
    }
  }
}
```

#### Cumulative Sum

Calculates running totals:

```json
GET /sales/_search
{
  "size": 0,
  "aggs": {
    "sales_per_month": {
      "date_histogram": {
        "field": "date",
        "calendar_interval": "month"
      },
      "aggs": {
        "monthly_sales": {
          "sum": {
            "field": "amount"
          }
        },
        "cumulative_sales": {
          "cumulative_sum": {
            "buckets_path": "monthly_sales"
          }
        }
      }
    }
  }
}
```

### Nested Aggregations

Aggregations can be nested to create more complex analyses:

```json
GET /products/_search
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
        },
        "price_ranges": {
          "range": {
            "field": "price",
            "ranges": [
              { "to": 50 },
              { "from": 50, "to": 100 },
              { "from": 100, "to": 200 },
              { "from": 200 }
            ]
          }
        }
      }
    }
  }
}
```

This returns:
- Top 10 categories
- Average price per category
- Price range distribution per category

## Query Templates and Stored Searches

### Search Templates

Create reusable search templates:

```json
POST /_scripts/my_search_template
{
  "script": {
    "lang": "mustache",
    "source": {
      "query": {
        "bool": {
          "must": [
            { "match": { "category": "{{category}}" } }
          ],
          "filter": [
            { "range": { "price": { "gte": "{{min_price}}", "lte": "{{max_price}}" } } }
          ]
        }
      },
      "size": "{{size}}",
      "from": "{{from}}"
    },
    "params": {
      "category": "electronics",
      "min_price": 0,
      "max_price": 100,
      "size": 10,
      "from": 0
    }
  }
}
```

Use the template:

```json
GET /products/_search/template
{
  "id": "my_search_template",
  "params": {
    "category": "smartphones",
    "min_price": 300,
    "max_price": 800,
    "size": 5,
    "from": 0
  }
}
```

### Conditionals in Templates

Use conditionals to create flexible templates:

```json
POST /_scripts/conditional_template
{
  "script": {
    "lang": "mustache",
    "source": """
    {
      "query": {
        "bool": {
          "must": [
            { "match": { "category": "{{category}}" } }
          ]
          {{#has_price_filter}}
          , "filter": [
            { "range": { "price": { "gte": {{min_price}}, "lte": {{max_price}} } } }
          ]
          {{/has_price_filter}}
        }
      },
      "size": {{size}}
    }
    """
  }
}
```

Use with conditional parameters:

```json
GET /products/_search/template
{
  "id": "conditional_template",
  "params": {
    "category": "smartphones",
    "has_price_filter": true,
    "min_price": 300,
    "max_price": 800,
    "size": 5
  }
}
```

## Query Performance and Optimization

### Query Optimization Techniques

#### Use Filter Context

Use filter context for exact matching:

```json
# Less efficient
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "term": { "category": "electronics" } },
        { "range": { "price": { "gte": 100, "lte": 200 } } }
      ]
    }
  }
}

# More efficient
GET /products/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "category": "electronics" } },
        { "range": { "price": { "gte": 100, "lte": 200 } } }
      ]
    }
  }
}
```

The second example is more efficient because:
- It doesn't calculate relevance scores
- Filter clauses are cached
- It often executes faster

#### Use Efficient Field Types

Choose field types wisely:
- Use `keyword` for exact matching and aggregations
- Use `text` only when full-text search is needed
- Use specialized types like `date`, `geo_point`, etc.

#### Limit Fields

Only request fields you need:

```json
GET /products/_search
{
  "query": { "match": { "name": "phone" } },
  "_source": ["name", "price", "category"]
}
```

Or exclude fields:

```json
GET /products/_search
{
  "query": { "match": { "name": "phone" } },
  "_source": {
    "excludes": ["description", "specifications"]
  }
}
```

#### Use Pagination Carefully

Avoid deep pagination with `from`/`size`:
- Use `search_after` for deep pagination
- Use `sort` with a unique identifier
- Consider using the point-in-time API for consistent results

### Caching Considerations

Elasticsearch automatically caches:
- Filter results (common filters may be reused)
- Frequently used aggregations
- Index and field data

To benefit from caching:
- Use filter context when possible
- Keep filters consistent
- Consider the `request_cache` for frequent searches

## Advanced Query Techniques

### Function Score Queries

Modify document scores with custom functions:

```json
GET /products/_search
{
  "query": {
    "function_score": {
      "query": { "match": { "name": "phone" } },
      "functions": [
        {
          "field_value_factor": {
            "field": "rating",
            "factor": 1.2,
            "modifier": "log1p",
            "missing": 1
          }
        },
        {
          "gauss": {
            "price": {
              "origin": "500",
              "scale": "100"
            }
          }
        }
      ],
      "boost_mode": "multiply",
      "score_mode": "avg"
    }
  }
}
```

This query:
- Starts with documents matching "phone"
- Boosts scores based on the product rating
- Decreases scores for products far from the $500 price point
- Combines the scores using multiplication

### Span Queries

For precise control over word proximity and ordering:

```json
GET /articles/_search
{
  "query": {
    "span_near": {
      "clauses": [
        { "span_term": { "content": "elasticsearch" } },
        { "span_term": { "content": "query" } }
      ],
      "slop": 5,
      "in_order": true
    }
  }
}
```

This matches documents where "elasticsearch" appears within 5 positions before "query".

### Combining Aggregations with Queries

Filter documents before aggregating:

```json
GET /products/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "category": "electronics" } }
      ]
    }
  },
  "aggs": {
    "brands": {
      "terms": {
        "field": "brand",
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

Use filtered aggregations to compare segments:

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "all_products": {
      "global": {},
      "aggs": {
        "avg_price": {
          "avg": {
            "field": "price"
          }
        }
      }
    },
    "by_category": {
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

This calculates:
- Average price across all products
- Average price by category

## Practical Examples

### E-commerce Product Search

Create a product search with relevant sorting and filtering:

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "smartphone wireless charging",
            "fields": ["name^3", "description"],
            "type": "cross_fields"
          }
        }
      ],
      "filter": [
        { "term": { "category": "smartphones" } },
        { "range": { "price": { "gte": 300, "lte": 900 } } },
        { "term": { "in_stock": true } }
      ],
      "should": [
        { "term": { "features": "5g" } },
        { "term": { "features": "wireless_charging" } }
      ]
    }
  },
  "sort": [
    "_score",
    { "price": "asc" }
  ],
  "from": 0,
  "size": 20,
  "aggs": {
    "brands": {
      "terms": {
        "field": "brand",
        "size": 10
      }
    },
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          { "to": 400 },
          { "from": 400, "to": 600 },
          { "from": 600, "to": 800 },
          { "from": 800 }
        ]
      }
    },
    "avg_rating": {
      "avg": {
        "field": "rating"
      }
    }
  }
}
```

### Log Analysis Dashboard

Query for analyzing application logs:

```json
GET /logs/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "message": "error" } }
      ],
      "filter": [
        {
          "range": {
            "@timestamp": {
              "gte": "now-1d",
              "lt": "now"
            }
          }
        },
        { "term": { "application": "payment-service" } }
      ]
    }
  },
  "sort": [
    { "@timestamp": "desc" }
  ],
  "size": 20,
  "aggs": {
    "errors_over_time": {
      "date_histogram": {
        "field": "@timestamp",
        "fixed_interval": "1h"
      }
    },
    "error_types": {
      "terms": {
        "field": "error.type",
        "size": 10
      }
    },
    "error_sources": {
      "terms": {
        "field": "error.source",
        "size": 10
      },
      "aggs": {
        "error_counts": {
          "terms": {
            "field": "error.type",
            "size": 5
          }
        }
      }
    }
  }
}
```

### Geographic Analysis

Query for location-based analysis:

```json
GET /stores/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "amenities": "wifi"
          }
        }
      ],
      "filter": [
        {
          "geo_distance": {
            "distance": "5km",
            "location": {
              "lat": 40.7128,
              "lon": -74.0060
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
          "lat": 40.7128,
          "lon": -74.0060
        },
        "order": "asc",
        "unit": "km"
      }
    }
  ],
  "size": 20,
  "aggs": {
    "store_types": {
      "terms": {
        "field": "type",
        "size": 10
      }
    },
    "avg_rating": {
      "avg": {
        "field": "rating"
      }
    },
    "by_distance": {
      "geo_distance": {
        "field": "location",
        "origin": {
          "lat": 40.7128,
          "lon": -74.0060
        },
        "ranges": [
          { "to": 1 },
          { "from": 1, "to": 2 },
          { "from": 2, "to": 5 },
          { "from": 5 }
        ],
        "unit": "km"
      }
    }
  }
}
```

## Conclusion

This chapter covered the fundamentals of querying and aggregations in Elasticsearch, from basic searches to complex analytics. By understanding and combining these features, you can create powerful search experiences and extract valuable insights from your data.

Remember these key points:
- Use the right query type for your use case
- Combine queries with bool queries for complex conditions
- Use filter context whenever possible for better performance
- Leverage aggregations for data analysis and visualization
- Optimize queries for performance, especially for large datasets

In the next chapter, we'll explore mapping and analysis, which define how your data is indexed and searched in Elasticsearch.