# Filter Plugins

## Introduction to Filter Plugins

Filter plugins are the processing workhorses of Logstash, transforming raw data into structured, enriched events. They allow you to parse, modify, add, remove, and manipulate fields in your events as they pass through the Logstash pipeline.

This chapter covers key filter plugins, their configuration options, and common use cases to help you effectively process and transform your data.

## Grok Filter

The Grok filter is one of the most powerful and commonly used filters. It parses unstructured data into structured fields using pattern matching.

### Basic Configuration

```ruby
filter {
  grok {
    match => { "message" => "%{IP:client} %{WORD:method} %{URIPATHPARAM:request} %{NUMBER:status} %{NUMBER:bytes}" }
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `match` | Hash of field-pattern pairs | Required | `{ "message" => "%{PATTERN}" }` |
| `patterns_dir` | Directory of custom patterns | None | `["/path/to/patterns"]` |
| `pattern_definitions` | Custom patterns | None | `{ "MY_PATTERN" => "foo\\w+" }` |
| `break_on_match` | Stop after first match | true | `false` |
| `named_captures_only` | Keep only named captures | true | `false` |
| `keep_empty_captures` | Keep empty captures | false | `true` |
| `tag_on_failure` | Tags to add on failure | `["_grokparsefailure"]` | `["parse_failed"]` |
| `tag_on_timeout` | Tags to add on timeout | `["_groktimeout"]` | `["timeout"]` |
| `timeout_millis` | Pattern match timeout | 30000 | `10000` |

### Common Grok Patterns

Grok comes with many predefined patterns:

- **IP**: Matches IPv4 or IPv6 addresses
- **NUMBER**: Matches decimal numbers
- **WORD**: Matches word characters
- **TIMESTAMP_ISO8601**: Matches ISO 8601 timestamps
- **HTTPDATE**: Matches HTTP date format
- **COMMONAPACHELOG**: Matches Apache common log format
- **COMBINEDAPACHELOG**: Matches Apache combined log format

Full pattern examples:

```ruby
# Apache log parsing
grok {
  match => { "message" => "%{COMBINEDAPACHELOG}" }
}

# Syslog parsing
grok {
  match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}" }
}

# Custom application log parsing
grok {
  match => { "message" => "\[%{TIMESTAMP_ISO8601:timestamp}\] \[%{LOGLEVEL:level}\] %{JAVACLASS:class}: %{GREEDYDATA:message}" }
}
```

### Multiple Pattern Matching

You can specify multiple patterns for fallback:

```ruby
filter {
  grok {
    match => {
      "message" => [
        "%{COMBINEDAPACHELOG}",
        "%{COMMONAPACHELOG}",
        "\[%{TIMESTAMP_ISO8601:timestamp}\] %{LOGLEVEL:level} %{GREEDYDATA:msg}"
      ]
    }
  }
}
```

### Custom Grok Patterns

Define custom patterns:

```ruby
filter {
  grok {
    pattern_definitions => {
      "APP_ID" => "[A-Z]{2}[0-9]{6}"
      "TX_CODE" => "[A-Z]{2}[0-9]{2}"
    }
    match => {
      "message" => "Transaction %{APP_ID:app_id} with code %{TX_CODE:tx_code} %{GREEDYDATA:details}"
    }
  }
}
```

Or from pattern files:

```ruby
filter {
  grok {
    patterns_dir => ["/etc/logstash/patterns"]
    match => { "message" => "%{MY_CUSTOM_PATTERN:field}" }
  }
}
```

### Grok Debugging

Use the Grok Debugger in Kibana or online tools to test patterns. When troubleshooting:

1. Check grok failures with `_grokparsefailure` tag
2. Use multiple patterns from most specific to most general
3. Start with small patterns and build up gradually
4. Use named pattern pieces for reusability and clarity

### Best Practices for Grok

- Use existing patterns when possible
- Create reusable custom patterns for application logs
- Start with a sample of representative logs
- Be cautious with resource-intensive patterns (e.g., wildcard anchors)
- Optimize patterns with explicit anchors (^ and $)
- Monitor for grok failures and adjust patterns accordingly
- Set timeouts for complex patterns to prevent blocking

## Mutate Filter

The Mutate filter performs general field transformations like renaming, removing, replacing, and modifying fields.

### Basic Configuration

```ruby
filter {
  mutate {
    convert => { "status" => "integer" }
    rename => { "old_field" => "new_field" }
    add_field => { "new_field" => "new_value" }
    remove_field => [ "unwanted_field" ]
  }
}
```

### Key Operations

| Operation | Description | Example |
|-----------|-------------|---------|
| `convert` | Convert field type | `convert => { "bytes" => "integer" }` |
| `rename` | Rename fields | `rename => { "old" => "new" }` |
| `replace` | Replace field value | `replace => { "status" => "OK" }` |
| `update` | Update existing field | `update => { "name" => "John" }` |
| `add_field` | Add new fields | `add_field => { "full_name" => "%{first} %{last}" }` |
| `remove_field` | Remove fields | `remove_field => ["temp", "debug"]` |
| `gsub` | Replace using regex | `gsub => [ "field", "pattern", "replacement" ]` |
| `uppercase` | Convert to uppercase | `uppercase => [ "field1", "field2" ]` |
| `lowercase` | Convert to lowercase | `lowercase => [ "field1", "field2" ]` |
| `capitalize` | Capitalize first letter | `capitalize => [ "field1", "field2" ]` |
| `strip` | Remove leading/trailing whitespace | `strip => [ "field1", "field2" ]` |
| `split` | Split field into array | `split => { "tags" => "," }` |
| `join` | Join array into string | `join => { "array" => "," }` |
| `merge` | Merge with another field | `merge => { "dest_field" => "src_field" }` |
| `coerce` | Coerce field to array | `coerce => { "field" => "array" }` |

### Type Conversion

Convert field types with mutate:

```ruby
filter {
  mutate {
    convert => {
      "bytes" => "integer"
      "duration" => "float"
      "active" => "boolean"
      "tags" => "array"
      "day_of_week" => "string"
    }
  }
}
```

### String Manipulation

Perform string operations:

```ruby
filter {
  mutate {
    gsub => [
      # Replace all spaces with underscores
      "path", " ", "_",
      # Replace multiple slashes with single slash
      "url", "/+", "/"
    ]
    
    # Convert field values to lowercase
    lowercase => [ "user_agent", "method" ]
    
    # Strip whitespace
    strip => [ "message" ]
    
    # Split comma-separated values into array
    split => { "tags" => "," }
  }
}
```

### Conditional Field Operations

Use mutate with conditionals:

```ruby
filter {
  if [status] == 404 {
    mutate {
      add_field => { "error_type" => "not_found" }
      add_tag => [ "error" ]
    }
  } else if [status] >= 500 {
    mutate {
      add_field => { "error_type" => "server_error" }
      add_tag => [ "error", "alert" ]
    }
  }
}
```

### Best Practices for Mutate

- Group similar operations together
- Use type conversion early in the pipeline
- Remove temporary fields to reduce event size
- Use rename instead of add_field + remove_field
- Be careful with gsub performance on large fields
- Leverage field reference syntax for dynamic values

## Date Filter

The Date filter parses dates from fields and uses them for the @timestamp field or target fields.

### Basic Configuration

```ruby
filter {
  date {
    match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
    target => "@timestamp"
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `match` | Array with field and formats | Required | `[ "timestamp", "ISO8601" ]` |
| `target` | Target timestamp field | "@timestamp" | `"event_time"` |
| `timezone` | Timezone for parsing | UTC | `"America/Los_Angeles"` |
| `locale` | Locale for parsing | "en" | `"fr"` |
| `tag_on_failure` | Tags to add on failure | `["_dateparsefailure"]` | `["invalid_date"]` |

### Multiple Date Formats

Parse dates with multiple potential formats:

```ruby
filter {
  date {
    match => [ "timestamp", 
      "yyyy-MM-dd'T'HH:mm:ss.SSSZ",
      "yyyy-MM-dd'T'HH:mm:ssZ", 
      "yyyy-MM-dd HH:mm:ss",
      "MM/dd/yyyy HH:mm:ss",
      "dd/MMM/yyyy:HH:mm:ss Z",
      "UNIX",
      "UNIX_MS"
    ]
    target => "@timestamp"
    timezone => "UTC"
  }
}
```

### UNIX Timestamps

Parse UNIX timestamps (seconds or milliseconds since epoch):

```ruby
filter {
  date {
    match => [ "timestamp", "UNIX" ]
    target => "@timestamp"
  }
}

filter {
  date {
    match => [ "timestamp", "UNIX_MS" ]
    target => "@timestamp"
  }
}
```

### Multiple Date Fields

Process multiple date fields:

```ruby
filter {
  # Parse created_at into @timestamp
  date {
    match => [ "created_at", "yyyy-MM-dd HH:mm:ss" ]
    target => "@timestamp"
  }
  
  # Parse updated_at into its own field
  date {
    match => [ "updated_at", "yyyy-MM-dd HH:mm:ss" ]
    target => "updated_timestamp"
  }
}
```

### Best Practices for Date Filter

- List date formats from most specific to most general
- Include the timezone in your pattern if present in the data
- Set explicit timezone when the source timezone is known
- Use a consistent timestamp field (usually @timestamp)
- Use locale settings for language-specific date names
- Monitor for date parsing failures
- Consider performance impact of complex date patterns

## JSON Filter

The JSON filter parses JSON strings into structured data.

### Basic Configuration

```ruby
filter {
  json {
    source => "message"
    target => "parsed_json"
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `source` | Source field containing JSON | "message" | `"json_data"` |
| `target` | Target field for parsed JSON | root document | `"parsed"` |
| `skip_on_invalid_json` | Skip filter on invalid JSON | false | `true` |
| `tag_on_failure` | Tags for invalid JSON | `["_jsonparsefailure"]` | `["invalid_json"]` |

### Parsing JSON at Root Level

Parse JSON directly into the event:

```ruby
filter {
  json {
    source => "message"
    # No target means fields go to the root event
  }
}
```

### Parsing JSON into a Field

Parse JSON into a specific field:

```ruby
filter {
  json {
    source => "message"
    target => "data"
  }
}
```

### Handling Nested JSON

Process nested JSON structures:

```ruby
filter {
  # First parse the outer JSON
  json {
    source => "message"
    target => "parsed"
  }
  
  # Then parse a nested JSON string
  if [parsed][metadata] {
    json {
      source => "[parsed][metadata]"
      target => "[parsed][metadata_parsed]"
    }
  }
}
```

### Best Practices for JSON Filter

- Validate JSON structure before processing
- Use target field for complex nested structures
- Apply type conversion after JSON parsing
- Consider performance with large JSON objects
- Handle nested JSON structures carefully
- Set tags for JSON parsing failures

## KV Filter

The KV (Key-Value) filter extracts key-value pairs from strings.

### Basic Configuration

```ruby
filter {
  kv {
    source => "message"
    field_split => "&"
    value_split => "="
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `source` | Source field containing KV pairs | "message" | `"query_string"` |
| `field_split` | Character(s) between pairs | " " | `"&"` |
| `value_split` | Character(s) between key and value | "=" | `":"` |
| `target` | Target for extracted fields | root document | `"parameters"` |
| `include_keys` | Keys to include | None | `[ "user", "id" ]` |
| `exclude_keys` | Keys to exclude | None | `[ "password" ]` |
| `default_keys` | Default keys for values | None | `{ "method" => "GET" }` |
| `allow_duplicate_values` | Allow duplicate keys | true | `false` |
| `transform_key` | Transform keys | "noop" | `"lowercase"` |
| `transform_value` | Transform values | "noop" | `"lowercase"` |
| `prefix` | Add prefix to keys | None | `"param_"` |

### URL Query String Parsing

Parse URL query parameters:

```ruby
filter {
  kv {
    source => "query_string"
    field_split => "&"
    value_split => "="
    target => "params"
  }
}
```

### Log Message Parsing

Parse structured log messages:

```ruby
filter {
  kv {
    source => "message"
    field_split => " "
    value_split => "="
    allow_duplicate_values => false
    include_keys => ["duration", "status", "user", "path"]
  }
}
```

### Nested Field Processing

Process KV pairs into nested structures:

```ruby
filter {
  kv {
    source => "message"
    field_split => " "
    value_split => "="
    target => "details"
    prefix => "param_"
  }
}
```

### Key/Value Transformation

Transform keys and values:

```ruby
filter {
  kv {
    source => "message"
    transform_key => "lowercase"
    transform_value => "uppercase"
    include_brackets => true    # Parse [key]=value
  }
}
```

### Best Practices for KV Filter

- Choose appropriate field and value separators
- Use include_keys to extract only needed fields
- Apply exclude_keys for sensitive data
- Use prefix for namespace clarity
- Consider performance with large values
- Apply additional filters after KV for type conversion
- Test with representative data samples

## GeoIP Filter

The GeoIP filter adds geographic location data based on IP addresses.

### Basic Configuration

```ruby
filter {
  geoip {
    source => "client_ip"
    target => "geo"
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `source` | Source field containing IP | Required | `"clientip"` |
| `target` | Target field for geo data | "geoip" | `"geo"` |
| `database` | Path to custom database | Built-in | `"/path/to/GeoLite2-City.mmdb"` |
| `default_database_type` | Database type | "GeoLite2-City" | `"GeoLite2-ASN"` |
| `fields` | Fields to extract | All | `["city_name", "country_name", "location"]` |
| `tag_on_failure` | Tags on failure | `["_geoip_lookup_failure"]` | `["geoip_failed"]` |
| `cache_size` | LRU cache size | 1000 | `5000` |

### Standard GeoIP Lookup

Basic IP address enrichment:

```ruby
filter {
  geoip {
    source => "clientip"
    # Uses default database and target field
  }
}
```

### Custom Database and Fields

Use a custom database and select specific fields:

```ruby
filter {
  geoip {
    source => "ip_address"
    database => "/etc/logstash/GeoIP/GeoLite2-City.mmdb"
    target => "geo"
    fields => ["city_name", "country_name", "country_code", "region_name", "location"]
    tag_on_failure => ["geoip_lookup_failed"]
  }
}
```

### ASN Database

Look up Autonomous System information:

```ruby
filter {
  geoip {
    source => "ip_address"
    default_database_type => "GeoLite2-ASN"
    target => "asn"
  }
}
```

### Multiple IP Field Processing

Process multiple IP fields:

```ruby
filter {
  # Process source IP
  geoip {
    source => "src_ip"
    target => "src_geo"
  }
  
  # Process destination IP
  geoip {
    source => "dst_ip"
    target => "dst_geo"
  }
}
```

### Best Practices for GeoIP Filter

- Regularly update the GeoIP database for accuracy
- Select only needed fields to reduce event size
- Use tagging for failed lookups
- Be aware of privacy regulations for IP geolocation
- Set up caching for repeated lookups
- Consider using both City and ASN databases
- Convert coordinates to geo_point for Elasticsearch

## User Agent Filter

The User Agent filter parses user agent strings into structured data.

### Basic Configuration

```ruby
filter {
  useragent {
    source => "user_agent"
    target => "ua"
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `source` | Source field | Required | `"useragent"` |
| `target` | Target field | "user_agent" | `"ua"` |
| `regexes` | Path to regexes file | Built-in | `"/path/to/regexes.yaml"` |
| `prefix` | Field prefix | None | `"browser_"` |
| `lru_cache_size` | Cache size | 1000 | `5000` |

### Basic User Agent Parsing

Parse user agent into structured data:

```ruby
filter {
  useragent {
    source => "agent"
    target => "user_agent"
  }
}
```

The output includes fields like:
- `name` (browser name)
- `os` (operating system)
- `device` (device type)
- `major`, `minor`, `patch` (browser version)
- And more metadata

### Custom Target and Prefix

Customize field names:

```ruby
filter {
  useragent {
    source => "http_user_agent"
    target => "browser"
    prefix => "ua_"
  }
}
```

This would create fields like `browser.ua_name`, `browser.ua_os`, etc.

### Best Practices for User Agent Filter

- Apply the filter early to facilitate downstream decisions
- Update the regexes file regularly
- Use caching for repeated user agent strings
- Consider performance impact for high-volume logs
- Extract only needed fields to reduce event size
- Test with common user agents in your environment

## Ruby Filter

The Ruby filter executes custom Ruby code to process events.

### Basic Configuration

```ruby
filter {
  ruby {
    code => "event.set('field', 'value')"
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `code` | Ruby code to execute | Required | `"event.set('field', 'value')"` |
| `path` | Path to Ruby script file | None | `"/path/to/script.rb"` |
| `script_params` | Parameters for the script | None | `{ "param1" => "value1" }` |
| `tag_on_exception` | Tags on exception | `["_rubyexception"]` | `["ruby_error"]` |

### Simple Field Manipulation

Set, modify, or compute fields:

```ruby
filter {
  ruby {
    code => '
      # Create a new field
      event.set("timestamp_str", Time.now.strftime("%Y-%m-%d %H:%M:%S"))
      
      # Modify existing field
      if event.get("status")
        status = event.get("status").to_i
        event.set("status_category", 
          case status
            when 100..199 then "informational"
            when 200..299 then "success"
            when 300..399 then "redirection"
            when 400..499 then "client_error"
            when 500..599 then "server_error"
            else "unknown"
          end)
      end
    '
  }
}
```

### Complex Transformations

Perform complex data transformations:

```ruby
filter {
  ruby {
    code => '
      # Extract values from nested structures
      headers = event.get("[request][headers]")
      if headers
        # Extract specific headers
        event.set("[request][content_type]", headers["content-type"]) if headers["content-type"]
        event.set("[request][auth]", headers["authorization"]) if headers["authorization"]
      end
      
      # Compute values
      bytes = event.get("bytes").to_i
      duration = event.get("duration").to_f
      if duration > 0
        event.set("throughput", bytes / duration)
      end
      
      # Format complex field
      name = event.get("first_name") || ""
      last = event.get("last_name") || ""
      event.set("full_name", "#{name} #{last}".strip)
    '
  }
}
```

### Using Script Files

Keep complex logic in external files:

```ruby
# /etc/logstash/scripts/transform.rb
def filter(event)
  # Complex processing logic here
  event.set("processed_time", Time.now.to_i)
  return [event]
end
```

```ruby
filter {
  ruby {
    path => "/etc/logstash/scripts/transform.rb"
    script_params => { "environment" => "production" }
  }
}
```

### Best Practices for Ruby Filter

- Use for complex transformations not possible with other filters
- Keep scripts simple and focused
- Implement proper error handling
- Consider performance impact for high-volume pipelines
- Use script files for reusable, complex logic
- Test thoroughly before deployment
- Monitor for exceptions and performance issues

## Aggregate Filter

The Aggregate filter correlates events belonging to the same task.

### Basic Configuration

```ruby
filter {
  aggregate {
    task_id => "%{transaction_id}"
    code => "map['duration'] ||= 0; map['duration'] += event.get('request_time')"
    push_map_as_event_on_timeout => true
    timeout => 120
    timeout_tags => ["aggregation_timeout"]
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `task_id` | Task identifier | Required | `"%{id}"` |
| `code` | Ruby code to execute | Required | `"map['count'] ||= 0; map['count'] += 1"` |
| `map_action` | Action for map creation | "create_or_update" | `"create"` |
| `end_of_task` | Expression for task completion | None | `"event.get('status') == 'complete'"` |
| `timeout` | Task timeout in seconds | 1800 | `300` |
| `timeout_tags` | Tags for timeout events | None | `["timeout"]` |
| `push_map_as_event_on_timeout` | Generate event on timeout | false | `true` |
| `push_previous_map_as_event` | Generate event for previous task | false | `true` |
| `timeout_code` | Code for timeout event | None | `"event.set('status', 'timeout')"` |

### Request-Response Correlation

Correlate request and response events:

```ruby
filter {
  if [type] == "request" {
    aggregate {
      task_id => "%{transaction_id}"
      map_action => "create"
      code => "
        map['start_time'] = event.get('@timestamp')
        map['client_ip'] = event.get('client_ip')
        map['request'] = event.get('request')
      "
    }
  } else if [type] == "response" {
    aggregate {
      task_id => "%{transaction_id}"
      map_action => "update"
      code => "
        map['end_time'] = event.get('@timestamp')
        map['response_code'] = event.get('response_code')
        
        # Calculate duration if start_time exists
        if map['start_time']
          start_time = map['start_time']
          end_time = event.get('@timestamp')
          event.set('duration_ms', (end_time - start_time) * 1000)
        end
        
        # Copy fields from start event
        event.set('client_ip', map['client_ip'])
        event.set('request', map['request'])
      "
      end_of_task => true
    }
  }
}
```

### Session Tracking

Track user sessions:

```ruby
filter {
  aggregate {
    task_id => "%{user_id}"
    code => "
      # Initialize if first event
      map['event_count'] ||= 0
      map['pages'] ||= []
      map['first_seen'] ||= event.get('@timestamp')
      
      # Update tracking data
      map['event_count'] += 1
      map['last_seen'] = event.get('@timestamp')
      map['pages'] << event.get('page') if event.get('page')
      map['pages'].uniq!
      
      # Add aggregated data to current event
      event.set('session_event_count', map['event_count'])
      event.set('session_duration_s', map['last_seen'] - map['first_seen'])
      event.set('session_pages', map['pages'].size)
    "
    timeout => 1800  # 30 minutes session timeout
    push_map_as_event_on_timeout => true
    timeout_code => "
      event.set('type', 'session_summary')
      event.set('user_id', task_id)
      event.set('session_duration_s', map['last_seen'] - map['first_seen'])
      event.set('total_pages', map['pages'].size)
      event.set('event_count', map['event_count'])
    "
  }
}
```

### Best Practices for Aggregate Filter

- Choose unique and reliable task IDs
- Set appropriate timeouts based on task duration
- Consider memory usage for long-running tasks
- Use map_action for clear task boundaries
- Implement error handling for incomplete tasks
- Test with various task durations and edge cases
- Monitor memory consumption and performance

## Clone Filter

The Clone filter creates a duplicate of an event, with optional modifications.

### Basic Configuration

```ruby
filter {
  clone {
    clones => ["backup"]
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `clones` | Array of tags for clones | Required | `["copy1", "copy2"]` |

### Basic Cloning

Create a tagged copy of events:

```ruby
filter {
  clone {
    clones => ["archive"]
  }
}
```

### Multiple Clones with Modifications

Create multiple modified copies:

```ruby
filter {
  clone {
    clones => ["error", "warning"]
  }
  
  if "error" in [tags] {
    mutate {
      replace => { "log_level" => "error" }
      add_field => { "alert" => "true" }
    }
  } else if "warning" in [tags] {
    mutate {
      replace => { "log_level" => "warning" }
    }
  }
}
```

### Workflow Branching

Use clone to create multiple processing paths:

```ruby
filter {
  clone {
    clones => ["realtime", "batch"]
  }
  
  if "realtime" in [tags] {
    # Apply minimal processing for realtime analysis
    mutate {
      remove_field => ["detailed_data", "raw_payload"]
    }
  } else if "batch" in [tags] {
    # Apply extensive processing for batch analysis
    # ...extensive processing filters...
  }
}
```

### Best Practices for Clone Filter

- Use sparingly to avoid excessive event multiplication
- Consider performance and resource implications
- Apply modifications to make clones distinct and useful
- Use tags to track clone lineage
- Consider placing clone filter early in the pipeline
- Be aware of interactions with aggregations and metrics

## Drop Filter

The Drop filter discards events that match specific conditions.

### Basic Configuration

```ruby
filter {
  drop {
    percentage => 90
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `percentage` | Percentage to drop (1-100) | None | `50` |
| `periodic_flush` | Handle events on flush | false | `true` |

### Conditional Dropping

Drop events based on content:

```ruby
filter {
  if [loglevel] == "DEBUG" {
    drop { }
  }
}
```

### Sampling

Keep only a sample of events:

```ruby
filter {
  # Keep only 10% of events
  drop {
    percentage => 90
  }
}
```

### Filtering Specific Events

Drop specific event types:

```ruby
filter {
  if [source] == "monitoring" and [severity] == "info" {
    drop { }
  }
  
  if [http_code] == 200 and [path] =~ "^/health" {
    drop { }
  }
}
```

### Best Practices for Drop Filter

- Be specific with conditions to avoid dropping valuable data
- Consider using conditionals for clarity
- Use sampling only when statistically appropriate
- Document dropped event types for reference
- Consider alternative filtering at the source where possible
- Add metrics before drop to track discard rates

## Translate Filter

The Translate filter performs lookups to convert field values.

### Basic Configuration

```ruby
filter {
  translate {
    field => "status_code"
    destination => "status_description"
    dictionary => {
      "200" => "OK"
      "201" => "Created"
      "400" => "Bad Request"
      "404" => "Not Found"
      "500" => "Server Error"
    }
  }
}
```

### Key Options

| Option | Description | Default | Example |
|--------|-------------|---------|---------|
| `field` | Source field for translation | Required | `"error_code"` |
| `destination` | Destination field | Required | `"error_description"` |
| `dictionary` | Dictionary of translations | Required unless dictionary_path | `{"404" => "Not Found"}` |
| `dictionary_path` | Path to dictionary file | None | `"/path/to/dict.yaml"` |
| `exact` | Match entire string | true | `false` |
| `regex` | Use regex for matching | false | `true` |
| `fallback` | Value for unmatched entries | None | `"Unknown value"` |
| `refresh_interval` | Dictionary refresh interval | 300 | `60` |
| `override` | Overwrite existing destination | false | `true` |

### Dictionary File

Use an external dictionary file:

```ruby
filter {
  translate {
    field => "country_code"
    destination => "country_name"
    dictionary_path => "/etc/logstash/dictionaries/country_codes.yaml"
    fallback => "Unknown Country"
    refresh_interval => 300
  }
}
```

Example dictionary YAML file:
```yaml
# country_codes.yaml
US: "United States"
CA: "Canada"
UK: "United Kingdom"
FR: "France"
DE: "Germany"
# ...more entries
```

### Regular Expression Matching

Use regex patterns for flexible matching:

```ruby
filter {
  translate {
    field => "user_agent"
    destination => "device_type"
    regex => true
    dictionary => {
      "iPhone|iPad" => "iOS"
      "Android" => "Android"
      "Windows" => "Windows"
      "Macintosh" => "Mac"
      "Linux" => "Linux"
    }
    fallback => "Other"
  }
}
```

### Best Practices for Translate Filter

- Use dictionary files for large mappings
- Set appropriate refresh intervals for updated dictionaries
- Provide fallback values for unmatched entries
- Consider case sensitivity in matching
- Test dictionary mappings with representative data
- Use regex mode selectively due to performance impact
- Document dictionary sources and maintenance

## Conclusion

Filter plugins are the core transformation components in Logstash pipelines. They allow you to parse, structure, enrich, and manipulate your data to prepare it for storage and analysis.

When designing your filter chain, consider:

1. **Order matters**: Arrange filters to build on each other's results
2. **Performance impact**: Some filters (like Grok and Ruby) can be resource-intensive
3. **Error handling**: Use conditionals and tagging for problematic events
4. **Simplicity**: Start simple and add complexity as needed
5. **Testing**: Validate your filter chain with representative data samples

The right combination of filters will transform raw, unstructured data into valuable, structured information ready for analysis in Elasticsearch and visualization in Kibana.

In the next chapter, we'll explore output plugins that deliver your processed data to various destinations, including Elasticsearch.