# Mapping and Analysis

## Introduction to Mapping

Mapping in Elasticsearch defines how documents and their fields are stored and indexed. It's similar to a database schema, dictating the data types for each field and how they should be processed. Understanding mapping is crucial for designing efficient and effective Elasticsearch indices.

There are two types of mappings in Elasticsearch:

1. **Dynamic mapping**: Elasticsearch automatically detects and assigns field types based on incoming data
2. **Explicit mapping**: You define the field types and configurations yourself

## Field Types

Elasticsearch supports many field types. Here are the most commonly used ones:

### Core Types

#### String Types

- **text**: For full-text search. Text fields are analyzed and broken into searchable terms.
  ```json
  "description": {
    "type": "text"
  }
  ```

- **keyword**: For exact matching, sorting, and aggregations. Keyword fields are not analyzed.
  ```json
  "email": {
    "type": "keyword"
  }
  ```

#### Numeric Types

- **long**: 64-bit signed integer
- **integer**: 32-bit signed integer
- **short**: 16-bit signed integer
- **byte**: 8-bit signed integer
- **double**: Double-precision floating point
- **float**: Single-precision floating point
- **half_float**: Half-precision floating point
- **scaled_float**: Floating point that is scaled by a fixed factor

```json
"price": {
  "type": "double"
},
"quantity": {
  "type": "integer"
},
"discount_percentage": {
  "type": "scaled_float",
  "scaling_factor": 100
}
```

#### Date Type

- **date**: Handles dates and times in various formats
  ```json
  "created_at": {
    "type": "date",
    "format": "strict_date_optional_time||epoch_millis"
  }
  ```

#### Boolean Type

- **boolean**: Represents true/false values
  ```json
  "is_active": {
    "type": "boolean"
  }
  ```

#### Binary Type

- **binary**: Base64 encoded binary data
  ```json
  "image_thumbnail": {
    "type": "binary"
  }
  ```

#### Range Types

- **integer_range**, **float_range**, **long_range**, **double_range**, **date_range**
  ```json
  "age_range": {
    "type": "integer_range"
  }
  ```

### Complex Types

#### Object Type

Represents a JSON object with nested fields
```json
"address": {
  "type": "object",
  "properties": {
    "street": { "type": "text" },
    "city": { "type": "keyword" },
    "zip": { "type": "keyword" }
  }
}
```

#### Nested Type

Like object type, but allows querying nested objects independently
```json
"comments": {
  "type": "nested",
  "properties": {
    "author": { "type": "keyword" },
    "text": { "type": "text" },
    "date": { "type": "date" }
  }
}
```

### Specialized Types

#### Geo Types

- **geo_point**: Represents a latitude/longitude point
  ```json
  "location": {
    "type": "geo_point"
  }
  ```

- **geo_shape**: Represents complex shapes like polygons
  ```json
  "area": {
    "type": "geo_shape"
  }
  ```

#### IP Type

- **ip**: Stores IPv4 and IPv6 addresses
  ```json
  "visitor_ip": {
    "type": "ip"
  }
  ```

#### Completion Type

- **completion**: Used for auto-complete suggestions
  ```json
  "suggest": {
    "type": "completion"
  }
  ```

## Explicit Mappings

### Creating an Explicit Mapping

Define all field mappings when creating an index:

```json
PUT /products
{
  "mappings": {
    "properties": {
      "id": { "type": "keyword" },
      "name": { 
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      },
      "description": { "type": "text" },
      "category": { "type": "keyword" },
      "price": { "type": "double" },
      "created_at": { "type": "date" },
      "is_active": { "type": "boolean" },
      "tags": { "type": "keyword" },
      "rating": { "type": "float" },
      "location": { "type": "geo_point" },
      "metadata": {
        "type": "object",
        "properties": {
          "manufacturer": { "type": "keyword" },
          "color": { "type": "keyword" },
          "weight": { "type": "float" }
        }
      }
    }
  }
}
```

### Field Mapping Parameters

Beyond basic types, you can configure fields with additional parameters:

#### Index Parameter

Controls whether a field is indexed for search:
```json
"serial_number": {
  "type": "keyword",
  "index": false
}
```
This makes the field retrievable but not searchable.

#### Analyzer Parameter

Specifies the analyzer to use for a text field:
```json
"description": {
  "type": "text",
  "analyzer": "english"
}
```

#### Fields Parameter

Creates multi-fields for different purposes:
```json
"title": {
  "type": "text",
  "analyzer": "standard",
  "fields": {
    "keyword": {
      "type": "keyword",
      "ignore_above": 256
    },
    "search": {
      "type": "text",
      "analyzer": "english"
    }
  }
}
```
This lets you:
- Full-text search on `title`
- Exact match, sort, and aggregate on `title.keyword`
- English-specific search on `title.search`

#### Ignore Above Parameter

Ignores strings longer than the specified length:
```json
"sku": {
  "type": "keyword",
  "ignore_above": 256
}
```

#### Null Value Parameter

Substitutes a value when the field is null:
```json
"rating": {
  "type": "float",
  "null_value": 0.0
}
```

#### Copy To Parameter

Copies the field value to another field:
```json
"first_name": {
  "type": "text",
  "copy_to": "full_name"
},
"last_name": {
  "type": "text",
  "copy_to": "full_name"
},
"full_name": {
  "type": "text"
}
```
This creates a combined field for searching.

#### Doc Values Parameter

Disables doc values to save storage (at the cost of aggregation ability):
```json
"description": {
  "type": "text",
  "doc_values": false
}
```

#### Store Parameter

Stores the original field value separately:
```json
"description": {
  "type": "text",
  "store": true
}
```
This allows retrieving the field without accessing `_source`.

### Updating Mappings

You can add new fields to existing mappings:

```json
PUT /products/_mapping
{
  "properties": {
    "discount": { "type": "double" },
    "availability": { "type": "keyword" }
  }
}
```

However, you cannot:
- Change the type of an existing field
- Change most field parameters once set
- Remove a field from the mapping

To make such changes, you must reindex your data:

1. Create a new index with the updated mapping
2. Use the Reindex API to copy data to the new index
3. Switch any aliases to point to the new index

Example reindex operation:
```json
POST /_reindex
{
  "source": {
    "index": "products_old"
  },
  "dest": {
    "index": "products_new"
  }
}
```

## Dynamic Mapping

### How Dynamic Mapping Works

When you index a document with a new field, Elasticsearch automatically determines its type:

Field Value | Mapped Type
------------|------------
`true` or `false` | `boolean`
Number with no decimal places | `long`
Number with decimal places | `double`
Date string (ISO format) | `date`
String matching IP format | `ip`
String | `text` field with a `keyword` sub-field

```json
POST /dynamic_index/_doc/1
{
  "name": "John Smith",
  "age": 30,
  "is_active": true,
  "created_at": "2023-01-15T14:12:00Z",
  "email": "john@example.com",
  "ip_address": "192.168.1.1"
}
```

The resulting dynamic mapping:

```json
{
  "dynamic_index": {
    "mappings": {
      "properties": {
        "age": { "type": "long" },
        "created_at": { "type": "date" },
        "email": { 
          "type": "text",
          "fields": {
            "keyword": { "type": "keyword", "ignore_above": 256 }
          }
        },
        "ip_address": { "type": "ip" },
        "is_active": { "type": "boolean" },
        "name": { 
          "type": "text",
          "fields": {
            "keyword": { "type": "keyword", "ignore_above": 256 }
          }
        }
      }
    }
  }
}
```

### Controlling Dynamic Mapping

Control dynamic mapping behavior at the index or object level:

```json
PUT /controlled_index
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "name": { "type": "text" },
      "created_at": { "type": "date" },
      "metadata": {
        "type": "object",
        "dynamic": true,
        "properties": {}
      }
    }
  }
}
```

Dynamic settings:
- `true`: Automatically add new fields (default)
- `false`: Ignore new fields (they'll be in `_source` but not indexed)
- `strict`: Reject documents with unmapped fields

### Dynamic Templates

Dynamic templates provide fine-grained control over auto-mapping:

```json
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
      },
      {
        "geo_points": {
          "match": "location_*",
          "mapping": {
            "type": "geo_point"
          }
        }
      }
    ]
  }
}
```

Template matching conditions:
- `match_mapping_type`: Matches by JSON type
- `match`: Matches by field name (exact or wildcard)
- `match_pattern`: How to interpret the `match` parameter (default, regex)
- `path_match`: Matches by the full path to a field
- `path_unmatch`: Inverse of `path_match`

## Analysis Overview

Analysis is the process of converting text into tokens (terms) for indexing. It happens during both indexing and searching.

### The Analysis Process

Analysis involves three main components:

1. **Character filters**: Pre-process the string (e.g., strip HTML)
2. **Tokenizer**: Split the string into tokens (e.g., on whitespace)
3. **Token filters**: Modify tokens (e.g., lowercase, stemming)

An analyzer combines these components:

```
Text -> Character Filters -> Tokenizer -> Token Filters -> Tokens
```

For example, with the standard analyzer:
```
"The quick brown fox!" -> tokenizer -> ["The", "quick", "brown", "fox"] -> lowercase filter -> ["the", "quick", "brown", "fox"]
```

### Built-in Analyzers

Elasticsearch includes several pre-configured analyzers:

#### Standard Analyzer

The default analyzer for text fields.
- Tokenizes by Unicode Text Segmentation algorithm
- Removes most punctuation
- Lowercases tokens
- Removes stop words (optional)

```json
"description": {
  "type": "text",
  "analyzer": "standard"
}
```

#### Simple Analyzer

- Splits text on anything that's not a letter
- Lowercases tokens

```json
"description": {
  "type": "text",
  "analyzer": "simple"
}
```

#### Whitespace Analyzer

- Splits text on whitespace only
- Preserves case

```json
"tags_string": {
  "type": "text",
  "analyzer": "whitespace"
}
```

#### Language Analyzers

Optimized for specific languages, with appropriate stemming and stop words:

```json
"description_en": {
  "type": "text",
  "analyzer": "english"
},
"description_fr": {
  "type": "text",
  "analyzer": "french"
}
```

Elasticsearch supports many languages, including Arabic, Dutch, English, French, German, Italian, and more.

## Custom Analysis

### Creating Custom Analyzers

Define custom analyzers with specific components:

```json
PUT /custom_analysis
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
            "asciifolding",
            "my_stop",
            "my_synonyms"
          ]
        }
      },
      "filter": {
        "my_stop": {
          "type": "stop",
          "stopwords": ["the", "and", "or", "a", "an", "of"]
        },
        "my_synonyms": {
          "type": "synonym",
          "synonyms": [
            "laptop, notebook",
            "phone, mobile, cell phone",
            "tv, television"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "product_description": {
        "type": "text",
        "analyzer": "my_custom_analyzer"
      }
    }
  }
}
```

### Character Filters

Character filters pre-process the string before tokenization:

#### HTML Strip

Removes HTML tags:
```json
"char_filter": {
  "my_html_strip": {
    "type": "html_strip",
    "escaped_tags": ["b"]
  }
}
```

#### Mapping

Replaces characters with specified replacements:
```json
"char_filter": {
  "emoticons": {
    "type": "mapping",
    "mappings": [
      ":) => _happy_",
      ":( => _sad_"
    ]
  }
}
```

#### Pattern Replace

Uses regular expressions for replacements:
```json
"char_filter": {
  "camel_case_breaker": {
    "type": "pattern_replace",
    "pattern": "(?<=\\p{Lower})(?=\\p{Upper})",
    "replacement": " "
  }
}
```

### Tokenizers

Tokenizers split the string into tokens:

#### Standard Tokenizer

Uses Unicode Text Segmentation:
```json
"tokenizer": {
  "my_standard": {
    "type": "standard",
    "max_token_length": 10
  }
}
```

#### Whitespace Tokenizer

Splits on whitespace:
```json
"tokenizer": {
  "my_whitespace": {
    "type": "whitespace"
  }
}
```

#### Pattern Tokenizer

Splits using a regular expression:
```json
"tokenizer": {
  "comma_tokenizer": {
    "type": "pattern",
    "pattern": ","
  }
}
```

#### N-Gram Tokenizer

Creates n-grams of specified length:
```json
"tokenizer": {
  "my_ngram": {
    "type": "ngram",
    "min_gram": 2,
    "max_gram": 3,
    "token_chars": ["letter", "digit"]
  }
}
```

#### Edge N-Gram Tokenizer

Creates n-grams from the start of tokens:
```json
"tokenizer": {
  "autocomplete": {
    "type": "edge_ngram",
    "min_gram": 2,
    "max_gram": 10
  }
}
```

### Token Filters

Token filters modify tokens after tokenization:

#### Lowercase Filter

Converts tokens to lowercase:
```json
"filter": {
  "my_lowercase": {
    "type": "lowercase"
  }
}
```

#### Stop Filter

Removes stop words:
```json
"filter": {
  "my_stop": {
    "type": "stop",
    "stopwords": ["and", "the", "in"]
  }
}
```

#### Synonym Filter

Adds synonyms:
```json
"filter": {
  "my_synonyms": {
    "type": "synonym",
    "synonyms_path": "synonyms.txt"
  }
}
```

#### Stemmer Filter

Reduces words to their stem:
```json
"filter": {
  "my_stemmer": {
    "type": "stemmer",
    "language": "english"
  }
}
```

#### N-Gram Filter

Creates n-grams from tokens:
```json
"filter": {
  "my_ngram": {
    "type": "ngram",
    "min_gram": 2,
    "max_gram": 3
  }
}
```

## Multi-field Mappings

Create multiple interpretations of a field for different purposes:

```json
PUT /products
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "standard",
        "fields": {
          "keyword": {
            "type": "keyword"
          },
          "english": {
            "type": "text",
            "analyzer": "english"
          },
          "autocomplete": {
            "type": "text",
            "analyzer": "autocomplete_analyzer"
          }
        }
      }
    }
  },
  "settings": {
    "analysis": {
      "analyzer": {
        "autocomplete_analyzer": {
          "tokenizer": "autocomplete_tokenizer",
          "filter": ["lowercase"]
        }
      },
      "tokenizer": {
        "autocomplete_tokenizer": {
          "type": "edge_ngram",
          "min_gram": 2,
          "max_gram": 10,
          "token_chars": ["letter", "digit"]
        }
      }
    }
  }
}
```

With this mapping, you can:
- Search full-text with `name`
- Filter, sort, and aggregate with `name.keyword`
- Search with English stemming using `name.english`
- Implement autocomplete with `name.autocomplete`

## Mapping Best Practices

### Controlling Index Size

Use these techniques to reduce index size:

1. **Disable `doc_values` for fields not used in sorting or aggregations**:
   ```json
   "description": {
     "type": "text",
     "doc_values": false
   }
   ```

2. **Disable `norms` for fields where relevance scoring isn't needed**:
   ```json
   "tags": {
     "type": "keyword",
     "norms": false
   }
   ```

3. **Set `index: false` for fields only needed for retrieval**:
   ```json
   "internal_id": {
     "type": "keyword",
     "index": false
   }
   ```

4. **Optimize `index_options` for text fields**:
   ```json
   "description": {
     "type": "text",
     "index_options": "freqs"  // One of: docs, freqs, positions, offsets
   }
   ```

5. **Select precise numeric types**:
   ```json
   "small_number": {
     "type": "short"  // Instead of long
   }
   ```

### Optimization for Search

For better search performance:

1. **Use `keyword` fields for exact matching and aggregations**

2. **Create specific text analyzers for search requirements**:
   ```json
   "product_name": {
     "type": "text",
     "analyzer": "english",
     "fields": {
       "standard": {
         "type": "text",
         "analyzer": "standard"
       }
     }
   }
   ```

3. **Use n-grams or edge n-grams for partial matching**:
   ```json
   "autocomplete_field": {
     "type": "text",
     "analyzer": "edge_ngram_analyzer"
   }
   ```

4. **Define `fielddata: true` only when necessary** (for text fields that need sorting/aggregations):
   ```json
   "uncommon_text_field_to_aggregate": {
     "type": "text",
     "fielddata": true
   }
   ```

### Handling Nested Data

For nested objects and arrays:

1. **Use `nested` type for arrays of objects that need independent querying**:
   ```json
   "comments": {
     "type": "nested",
     "properties": {
       "user_id": { "type": "keyword" },
       "text": { "type": "text" },
       "date": { "type": "date" }
     }
   }
   ```

2. **Use `object` type for single objects or when nested queries aren't needed**:
   ```json
   "address": {
     "type": "object",
     "properties": {
       "street": { "type": "text" },
       "city": { "type": "keyword" },
       "zip": { "type": "keyword" }
     }
   }
   ```

3. **Consider field flattening for simpler structures**:
   ```json
   "user_name": { "type": "keyword" },
   "user_email": { "type": "keyword" }
   // Instead of nested user object
   ```

### Common Mapping Patterns

#### Search-Optimized Text Field

```json
"title": {
  "type": "text",
  "analyzer": "standard",
  "fields": {
    "keyword": {
      "type": "keyword",
      "ignore_above": 256
    },
    "english": {
      "type": "text",
      "analyzer": "english"
    }
  }
}
```

#### Autocomplete Field

```json
PUT /autocomplete_index
{
  "mappings": {
    "properties": {
      "suggest": {
        "type": "text",
        "analyzer": "autocomplete_index",
        "search_analyzer": "standard"
      }
    }
  },
  "settings": {
    "analysis": {
      "analyzer": {
        "autocomplete_index": {
          "tokenizer": "autocomplete_tokenizer",
          "filter": ["lowercase"]
        }
      },
      "tokenizer": {
        "autocomplete_tokenizer": {
          "type": "edge_ngram",
          "min_gram": 2,
          "max_gram": 20
        }
      }
    }
  }
}
```

#### Multi-Language Field

```json
"description": {
  "type": "object",
  "properties": {
    "en": {
      "type": "text",
      "analyzer": "english"
    },
    "fr": {
      "type": "text",
      "analyzer": "french"
    },
    "es": {
      "type": "text",
      "analyzer": "spanish"
    }
  }
}
```

## Testing Analysis and Mappings

### Testing Analyzers

Use the Analyze API to test analyzers:

```json
POST /_analyze
{
  "analyzer": "standard",
  "text": "The quick brown fox jumps over the lazy dog!"
}
```

Result:
```json
{
  "tokens": [
    {
      "token": "the",
      "start_offset": 0,
      "end_offset": 3,
      "type": "<ALPHANUM>",
      "position": 0
    },
    // Other tokens...
  ]
}
```

Test a custom analyzer:

```json
POST /my_index/_analyze
{
  "analyzer": "my_custom_analyzer",
  "text": "The quick brown fox jumps over the lazy dog!"
}
```

Test individual components:

```json
POST /_analyze
{
  "tokenizer": "standard",
  "filter": ["lowercase", "stop"],
  "text": "The quick brown fox jumps over the lazy dog!"
}
```

### Validating Mappings

Verify mapping with the Get Mapping API:

```json
GET /my_index/_mapping
```

Check a specific field's mapping:

```json
GET /my_index/_mapping/field/title
```

## Runtime Fields

Runtime fields are computed at query time without reindexing data:

```json
PUT /my_index/_mapping
{
  "runtime": {
    "price_with_tax": {
      "type": "double",
      "script": {
        "source": "emit(doc['price'].value * 1.2)"
      }
    }
  }
}
```

Use runtime fields in queries:

```json
GET /my_index/_search
{
  "fields": ["price", "price_with_tax"],
  "query": {
    "range": {
      "price_with_tax": {
        "gte": 100
      }
    }
  }
}
```

Runtime fields are useful for:
- Testing field transformations before reindexing
- Performing calculations on existing fields
- Adding fields that change frequently
- Reducing index storage size

## Conclusion

Effective mapping and analysis strategies are crucial for Elasticsearch performance and search quality. By carefully designing your mappings and analyzers, you can create powerful search experiences while keeping your indices efficient and maintainable.

Best practices to remember:
- Choose appropriate field types based on your data and query patterns
- Use multi-fields for different search and aggregation requirements
- Create custom analyzers for specific text processing needs
- Test your mappings and analysis with sample documents
- Optimize for both search performance and index size
- Consider runtime fields for dynamic calculations

In the next chapter, we'll explore performance optimization techniques to ensure your Elasticsearch deployment is fast and efficient.