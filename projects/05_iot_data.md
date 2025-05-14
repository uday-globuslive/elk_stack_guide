# IoT Data Processing with the ELK Stack

This chapter explores how to build a comprehensive IoT (Internet of Things) data processing solution using the Elastic Stack. You'll learn how to collect, store, analyze, and visualize data from IoT devices at scale, enabling real-time monitoring, analytics, and alerting.

## Table of Contents

- [IoT and the Elastic Stack](#iot-and-the-elastic-stack)
- [IoT Data Architecture](#iot-data-architecture)
- [Data Collection Strategies](#data-collection-strategies)
- [Data Processing Pipelines](#data-processing-pipelines)
- [IoT Data Storage](#iot-data-storage)
- [Real-time Monitoring Dashboard](#real-time-monitoring-dashboard)
- [Anomaly Detection](#anomaly-detection)
- [Time Series Analysis](#time-series-analysis)
- [Geospatial Analytics](#geospatial-analytics)
- [Edge Processing](#edge-processing)
- [Alerting and Notifications](#alerting-and-notifications)
- [IoT Security Monitoring](#iot-security-monitoring)
- [Scaling for Production](#scaling-for-production)
- [Case Study: Smart Factory](#case-study-smart-factory)
- [Best Practices](#best-practices)

## IoT and the Elastic Stack

The Elastic Stack is particularly well-suited for IoT data processing due to its:

1. **Scalability**: Handle millions of data points per second
2. **Time series optimizations**: Efficiently store and query time-based data
3. **Geospatial capabilities**: Track and analyze device locations
4. **Real-time processing**: Immediate insights from streaming data
5. **Machine learning**: Detect anomalies and forecast trends
6. **Visualization**: Interactive dashboards for IoT monitoring

Common IoT use cases include:

- **Industrial IoT**: Factory monitoring, predictive maintenance
- **Smart Cities**: Traffic monitoring, public safety, infrastructure monitoring
- **Connected Vehicles**: Fleet management, telematics
- **Environmental Monitoring**: Weather stations, air quality sensors
- **Healthcare**: Remote patient monitoring, medical device tracking
- **Smart Agriculture**: Soil sensors, irrigation control, crop monitoring
- **Energy Management**: Smart meters, grid monitoring, consumption analytics

## IoT Data Architecture

A typical IoT architecture with the Elastic Stack includes:

1. **IoT Devices Layer**: Physical sensors and actuators collecting data
2. **Edge Layer**: Local processing and filtering (optional)
3. **Transport Layer**: MQTT, HTTP, CoAP, or custom protocols
4. **Ingest Layer**: Logstash, Beats, or Elastic Agent for data collection
5. **Storage Layer**: Elasticsearch for data storage and indexing
6. **Analytics Layer**: Kibana for visualization, ML for anomaly detection
7. **Action Layer**: Alerting and response

### Reference Architecture Diagram

```
┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│  IoT Devices  │     │  Edge Gateway  │     │  MQTT Broker  │
│  - Sensors    │────▶│  - Filtering   │────▶│  - Message    │
│  - Actuators  │     │  - Aggregation │     │    Routing    │
└───────────────┘     └───────────────┘     └───────┬───────┘
                                                    │
┌────────────────────────────────────────────┐     │
│              Elastic Stack                  │     │
│                                             │     │
│  ┌───────────────┐     ┌───────────────┐   │     │
│  │   Logstash    │     │ Elasticsearch │   │     │
│  │  - Collection │◀────┤  - Storage    │◀──┼─────┘
│  │  - Processing │────▶│  - Indexing   │   │
│  └───────────────┘     └───────┬───────┘   │
│                                │           │
│  ┌───────────────┐     ┌───────▼───────┐   │
│  │  Machine      │     │    Kibana     │   │
│  │  Learning     │◀───▶│  - Dashboards │   │
│  │  - Anomalies  │     │  - Alerting   │   │
│  └───────────────┘     └───────────────┘   │
└────────────────────────────────────────────┘
```

## Data Collection Strategies

### MQTT Integration

MQTT (Message Queuing Telemetry Transport) is a lightweight protocol ideal for IoT devices. Configure Logstash to collect MQTT messages:

```ruby
# logstash.conf for MQTT collection
input {
  mqtt {
    host => "mqtt-broker.example.com"
    port => 1883
    topics => ["sensors/#"]
    username => "${MQTT_USER}"
    password => "${MQTT_PASSWORD}"
    client_id => "logstash_collector"
    codec => "json"
  }
}

filter {
  # Add device metadata
  if [topic] =~ "sensors/([^/]+)/([^/]+)" {
    grok {
      match => { "topic" => "sensors/%{DATA:device.type}/%{DATA:device.id}" }
    }
  }
  
  # Ensure timestamp is properly formatted
  date {
    match => [ "timestamp", "ISO8601", "UNIX", "UNIX_MS" ]
    target => "@timestamp"
    remove_field => [ "timestamp" ]
  }
  
  # Data validation and enrichment
  ruby {
    code => '
      # Validate temperature in reasonable range
      if event.get("temperature")
        temp = event.get("temperature").to_f
        if temp < -100 || temp > 200
          event.set("temperature_valid", false)
        else
          event.set("temperature_valid", true)
        end
      end
    '
  }
}

output {
  elasticsearch {
    hosts => ["https://elasticsearch:9200"]
    index => "iot-data-%{+YYYY.MM.dd}"
    user => "${ES_USER}"
    password => "${ES_PASSWORD}"
    ssl_certificate_verification => true
    cacert => "/etc/logstash/ca.crt"
  }
}
```

### HTTP Data Collection

For devices using HTTP/REST:

```ruby
# logstash.conf for HTTP collection
input {
  http {
    port => 8080
    codec => "json"
  }
}

filter {
  # Extract device info from headers
  mutate {
    add_field => {
      "device.id" => "%{[headers][x-device-id]}"
      "device.type" => "%{[headers][x-device-type]}"
      "device.firmware" => "%{[headers][x-firmware-version]}"
    }
  }
  
  # Remove headers from final event
  mutate {
    remove_field => [ "headers" ]
  }
}

output {
  elasticsearch {
    hosts => ["https://elasticsearch:9200"]
    index => "iot-data-%{+YYYY.MM.dd}"
    user => "${ES_USER}"
    password => "${ES_PASSWORD}"
  }
}
```

### Edge Collection with Elastic Agent

For edge devices running Linux (like Raspberry Pi):

1. Install Elastic Agent:
   ```bash
   curl -L -O https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.8.0-linux-arm64.tar.gz
   tar xzvf elastic-agent-8.8.0-linux-arm64.tar.gz
   cd elastic-agent-8.8.0-linux-arm64
   ```

2. Configure with Fleet:
   ```bash
   sudo ./elastic-agent install --url=https://fleet-server:8220 \
     --enrollment-token=<enrollment-token> \
     --ca-sha256=<ca-fingerprint>
   ```

3. Configure custom metrics collection in Fleet:
   ```yaml
   # metricbeat module for custom sensor data
   - module: mqtt
     metricsets: ["stats"]
     enabled: true
     period: 10s
     hosts: ["tcp://mqtt-broker:1883"]
   
   - module: system
     metricsets: ["cpu", "memory", "network", "process"]
     enabled: true
     period: 10s
     processes: ['.*']
   ```

## Data Processing Pipelines

### Real-time Data Transformation

Process incoming data with Logstash or Elasticsearch Ingest Pipelines:

#### Logstash Pipeline for Sensor Normalization

```ruby
filter {
  # Normalize sensor readings
  mutate {
    convert => {
      "temperature" => "float"
      "humidity" => "float"
      "pressure" => "float"
      "battery" => "float"
    }
  }
  
  # Convert units if needed
  if [temperature_unit] == "F" {
    ruby {
      code => "
        temp_f = event.get('temperature').to_f
        temp_c = (temp_f - 32) * 5.0/9.0
        event.set('temperature', temp_c.round(2))
        event.set('temperature_unit', 'C')
      "
    }
  }
  
  # Calculate derived metrics
  ruby {
    code => "
      # Calculate heat index if temp and humidity are available
      if event.get('temperature') && event.get('humidity')
        temp = event.get('temperature').to_f
        hum = event.get('humidity').to_f
        
        # Simplified heat index calculation
        hi = 0.5 * (temp + 61.0 + ((temp - 68.0) * 1.2) + (hum * 0.094))
        event.set('heat_index', hi.round(2))
      end
    "
  }
}
```

#### Elasticsearch Ingest Pipeline

```json
PUT _ingest/pipeline/iot-sensor-pipeline
{
  "description": "IoT sensor data processing pipeline",
  "processors": [
    {
      "date": {
        "field": "timestamp",
        "formats": ["ISO8601", "UNIX", "UNIX_MS"],
        "target_field": "@timestamp"
      }
    },
    {
      "convert": {
        "field": "temperature",
        "type": "float"
      }
    },
    {
      "convert": {
        "field": "humidity",
        "type": "float" 
      }
    },
    {
      "script": {
        "lang": "painless",
        "source": "if (ctx.temperature != null && ctx.humidity != null) { ctx.dewpoint = ctx.temperature - ((100 - ctx.humidity) / 5); }"
      }
    },
    {
      "geoip": {
        "field": "device_ip",
        "target_field": "geo"
      }
    }
  ]
}
```

### Data Aggregation Strategies

For high-frequency sensor data, consider pre-aggregation:

```ruby
filter {
  # Group by time window and device
  aggregate {
    task_id => "%{device.id}"
    code => "
      map['temperature_readings'] ||= []
      map['temperature_readings'] << event.get('temperature')
      map['reading_count'] ||= 0
      map['reading_count'] += 1
      
      # Calculate statistics every 10 readings or after 30 seconds
      if map['reading_count'] >= 10 || (event.get('@timestamp').time - map['first_reading_time']) > 30000
        event.set('temperature_min', map['temperature_readings'].min)
        event.set('temperature_max', map['temperature_readings'].max)
        event.set('temperature_avg', map['temperature_readings'].sum / map['temperature_readings'].size)
        event.set('reading_count', map['reading_count'])
        map.clear
      else
        # Don't create an event yet
        event.cancel
      end
    "
    push_map_as_event_on_timeout => true
    timeout => 30
    timeout_tags => ['aggregation_timeout']
  }
}
```

## IoT Data Storage

### Optimizing Elasticsearch for IoT Data

Configure Elasticsearch indices for time series data:

```json
PUT _index_template/iot-data-template
{
  "index_patterns": ["iot-data-*"],
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1,
      "index.mapping.total_fields.limit": 2000,
      "index.refresh_interval": "5s",
      "index.routing.allocation.total_shards_per_node": 3
    },
    "mappings": {
      "dynamic_templates": [
        {
          "strings_as_keywords": {
            "match_mapping_type": "string",
            "mapping": {
              "type": "keyword",
              "ignore_above": 256
            }
          }
        }
      ],
      "properties": {
        "@timestamp": { "type": "date" },
        "device": {
          "properties": {
            "id": { "type": "keyword" },
            "type": { "type": "keyword" },
            "firmware": { "type": "keyword" }
          }
        },
        "location": { "type": "geo_point" },
        "temperature": { "type": "float" },
        "humidity": { "type": "float" },
        "pressure": { "type": "float" },
        "battery": { "type": "float" },
        "heat_index": { "type": "float" },
        "dewpoint": { "type": "float" }
      }
    }
  }
}
```

### Index Lifecycle Management

Set up lifecycle management for IoT data:

```json
PUT _ilm/policy/iot-data-policy
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
          "forcemerge": {
            "max_num_segments": 1
          },
          "shrink": {
            "number_of_shards": 1
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

### Data Rollups for Long-term Storage

Create rollups for historical IoT data:

```json
PUT _rollup/job/iot-hourly-rollup
{
  "index_pattern": "iot-data-*",
  "rollup_index": "iot-data-hourly",
  "cron": "0 */1 * * * ?",
  "page_size": 1000,
  "groups": {
    "date_histogram": {
      "field": "@timestamp",
      "fixed_interval": "1h",
      "time_zone": "UTC"
    },
    "terms": {
      "fields": ["device.id", "device.type"]
    }
  },
  "metrics": [
    {
      "field": "temperature",
      "metrics": ["min", "max", "avg", "sum", "value_count"]
    },
    {
      "field": "humidity",
      "metrics": ["min", "max", "avg", "sum", "value_count"]
    },
    {
      "field": "pressure",
      "metrics": ["min", "max", "avg", "sum", "value_count"]
    },
    {
      "field": "battery",
      "metrics": ["min", "max", "avg", "sum", "value_count"]
    }
  ]
}
```

## Real-time Monitoring Dashboard

### Device Status Dashboard

Create a Kibana dashboard for real-time device monitoring:

1. **Device Status Overview** (Metric Visualization):
   - Count of online devices
   - Count of offline devices
   - Average battery level
   - Data points in last hour

2. **Device Status Table** (Data Table):
   - Columns: Device ID, Type, Last Seen, Status, Battery Level
   - Sort by Last Seen (descending)
   - Color coding for status and battery levels

3. **Device Status Timeline** (Line Chart):
   - Y-axis: Count of devices
   - X-axis: Time (@timestamp)
   - Split by status (online/offline)

### Sensor Readings Dashboard

Create visualizations for sensor data:

1. **Temperature Gauge** (Gauge Visualization):
   - Aggregation: Average
   - Field: temperature
   - Range: -10 to 50 degrees
   - Colors: Blue to Red

2. **Environment Readings** (Line Chart):
   - Y-axis: Multiple metrics (temperature, humidity, pressure)
   - X-axis: Time (@timestamp)
   - Split by device.id

3. **Heatmap by Time of Day** (Heatmap):
   - Y-axis: Hour of day
   - X-axis: Day of week
   - Color: Average temperature
   - Filter: Last 2 weeks

### Dashboard Configuration

Sample dashboard configuration:

```javascript
// Kibana Dashboard Object
{
  "attributes": {
    "title": "IoT Monitoring Dashboard",
    "hits": 0,
    "description": "Real-time monitoring of IoT devices",
    "panelsJSON": "[
      {\"panelIndex\":\"1\",\"gridData\":{\"x\":0,\"y\":0,\"w\":12,\"h\":8,\"i\":\"1\"},\"version\":\"7.10.0\",\"panelRefName\":\"panel_0\"},
      {\"panelIndex\":\"2\",\"gridData\":{\"x\":12,\"y\":0,\"w\":12,\"h\":8,\"i\":\"2\"},\"version\":\"7.10.0\",\"panelRefName\":\"panel_1\"},
      {\"panelIndex\":\"3\",\"gridData\":{\"x\":0,\"y\":8,\"w\":24,\"h\":12,\"i\":\"3\"},\"version\":\"7.10.0\",\"panelRefName\":\"panel_2\"}
    ]",
    "optionsJSON": "{\"hidePanelTitles\":false,\"useMargins\":true}",
    "timeRestore": true,
    "refreshInterval": {
      "value": 10000,
      "pause": false
    },
    "kibanaSavedObjectMeta": {
      "searchSourceJSON": "{\"query\":{\"language\":\"kuery\",\"query\":\"\"},\"filter\":[]}"
    }
  }
}
```

## Anomaly Detection

### Machine Learning for IoT Data

Configure anomaly detection for sensor data:

1. Go to Kibana → Machine Learning → Anomaly Detection → Create job
2. Configure a multi-metric job:
   - Data source: iot-data-*
   - Analysis type: Multi-metric
   - Fields to analyze: temperature, humidity, pressure, battery
   - Split data by: device.id
   - Bucket span: 5m
   - Job ID: iot-sensor-anomalies

### Anomaly Job Configuration

```json
PUT _ml/anomaly_detectors/iot-sensor-anomalies
{
  "description": "IoT sensor anomaly detection",
  "groups": ["iot"],
  "analysis_config": {
    "bucket_span": "5m",
    "detectors": [
      {
        "detector_description": "high_mean(temperature) partitioned by device.id",
        "function": "high_mean",
        "field_name": "temperature",
        "by_field_name": "device.id"
      },
      {
        "detector_description": "low_mean(battery) partitioned by device.id",
        "function": "low_mean",
        "field_name": "battery",
        "by_field_name": "device.id"
      },
      {
        "detector_description": "metric(temperature,mean) partitioned by device.id",
        "function": "metric",
        "field_name": "temperature",
        "by_field_name": "device.id",
        "over_field_name": null,
        "partition_field_name": null,
        "metric": "mean",
        "detector_index": 2
      }
    ],
    "influencers": [
      "device.id",
      "device.type"
    ]
  },
  "data_description": {
    "time_field": "@timestamp",
    "time_format": "epoch_ms"
  },
  "model_plot_config": {
    "enabled": true,
    "terms": "device.id"
  }
}
```

### Anomaly Dashboard

Create an anomaly dashboard:

1. **Anomaly Timeline** (ML Single Metric Viewer):
   - Job: iot-sensor-anomalies
   - Detector: temperature
   - Show anomaly timeline

2. **Anomaly Heatmap** (ML Anomaly Explorer):
   - Jobs: iot-sensor-anomalies
   - Show heatmap of anomalies

3. **Anomaly Details Table** (ML Anomaly Explorer Table):
   - Display detailed anomaly information
   - Columns: Time, Score, Device ID, Actual, Typical, Influencers

## Time Series Analysis

### Trend Analysis Dashboard

Create time series visualizations:

1. **Long-term Temperature Trends** (Line Chart with Moving Average):
   - Metrics: Average of temperature
   - Buckets: X-axis = Date Histogram of @timestamp (Daily)
   - Metrics: Moving average of temperature (Window: 7 days)

2. **Seasonality Analysis** (Timelion Visualization):
   ```
   .es(index=iot-data-*,metric=avg:temperature,timefield=@timestamp)
   .fit(mode='poly', 5).label('Trend')
   .lines(width=1).color(gray),
   .es(index=iot-data-*,metric=avg:temperature,timefield=@timestamp)
   .label('Temperature')
   ```

3. **Pattern Detection** (ML Single Metric Viewer):
   - Job: iot-seasonal-patterns
   - Detector: temperature
   - Show seasonality patterns

### Forecasting

Configure forecasting for sensor data:

1. Set up a forecast job:
   ```json
   POST _ml/anomaly_detectors/iot-temperature-forecast/_forecast
   {
     "duration": "7d",
     "expires_in": "14d"
   }
   ```

2. Create forecast visualizations:
   - Metrics: ML forecast data
   - Show prediction intervals (95% confidence)
   - Compare with actual values

## Geospatial Analytics

### Device Location Tracking

Create geospatial visualizations:

1. **Device Location Map** (Coordinate Map):
   - Geospatial field: location
   - Aggregation: Terms of device.id
   - Color by: Latest temperature values

2. **Movement Heatmap** (Heatmap on Map):
   - Geospatial field: location
   - Heatmap intensity based on point density
   - Filter: Moving devices only

3. **Route Tracking** (Line Map):
   - Geospatial field: location
   - Connect points by device.id
   - Color by device.type

### Geofencing Alerts

Set up geofence alerting:

```json
PUT _watcher/watch/geofence-alert
{
  "trigger": {
    "schedule": {
      "interval": "5m"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": ["iot-data-*"],
        "body": {
          "query": {
            "bool": {
              "must": [
                {
                  "range": {
                    "@timestamp": {
                      "gte": "now-5m"
                    }
                  }
                }
              ],
              "filter": {
                "geo_distance": {
                  "distance": "100m",
                  "location": {
                    "lat": 40.7128,
                    "lon": -74.0060
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
    "compare": {
      "ctx.payload.hits.total": {
        "gt": 0
      }
    }
  },
  "actions": {
    "send_email": {
      "email": {
        "to": "alerts@example.com",
        "subject": "Geofence Alert: Device entered restricted area",
        "body": {
          "html": "Device ID: {{ctx.payload.hits.hits.0._source.device.id}} has entered a restricted area."
        }
      }
    }
  }
}
```

## Edge Processing

### Edge Data Filtering

Configure Elastic Agent for edge filtering:

```yaml
# Elastic Agent config with filtering
agent:
  monitoring:
    enabled: true
    logs: true
    metrics: true

inputs:
  - type: mqtt
    id: mqtt-input
    name: MQTT Input
    use_output: elasticsearch
    streams:
      - data_stream:
          type: metrics
          dataset: mqtt.sensor
        protocol_version: 3.1.1
        topics:
          - sensors/#
        host: mqtt-broker:1883
        client_id: elastic-agent
        username: ${MQTT_USER}
        password: ${MQTT_PASSWORD}
        processors:
          - drop_event:
              when:
                conditions:
                  - equals:
                      mqtt.topic: "sensors/+/heartbeat"
          - drop_event:
              when:
                conditions:
                  - range:
                      field: temperature
                      lt: -100
                      gt: 200
          - include_fields:
              fields: ["device_id", "temperature", "humidity", "battery", "location"]
```

### Edge Aggregation

Set up edge aggregation for high-frequency data:

```yaml
# Elastic Agent config with aggregation
inputs:
  - type: mqtt
    id: mqtt-input
    name: MQTT Input
    use_output: elasticsearch
    streams:
      - data_stream:
          type: metrics
          dataset: mqtt.sensor
        protocol_version: 3.1.1
        topics:
          - sensors/#
        host: mqtt-broker:1883
        client_id: elastic-agent
        username: ${MQTT_USER}
        password: ${MQTT_PASSWORD}
        processors:
          - aggregate:
              fields:
                - name: temperature
                  initial: null
                  metrics:
                    - avg
                    - max
                    - min
                - name: humidity
                  initial: null
                  metrics:
                    - avg
                - name: battery
                  initial: null
                  metrics:
                    - avg
              target_fields:
                - temperature_stats
                - humidity_stats
                - battery_stats
              interval: 5m
```

## Alerting and Notifications

### Condition-based Alerts

Set up threshold-based alerts:

```json
PUT _watcher/watch/temperature-threshold-alert
{
  "trigger": {
    "schedule": {
      "interval": "5m"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": ["iot-data-*"],
        "body": {
          "size": 0,
          "query": {
            "bool": {
              "must": [
                {
                  "range": {
                    "@timestamp": {
                      "gte": "now-5m"
                    }
                  }
                },
                {
                  "range": {
                    "temperature": {
                      "gt": 35
                    }
                  }
                }
              ]
            }
          },
          "aggs": {
            "devices": {
              "terms": {
                "field": "device.id",
                "size": 10
              },
              "aggs": {
                "max_temp": {
                  "max": {
                    "field": "temperature"
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
    "compare": {
      "ctx.payload.aggregations.devices.buckets": {
        "not_empty": true
      }
    }
  },
  "actions": {
    "send_email": {
      "email": {
        "to": "alerts@example.com",
        "subject": "Temperature Alert: High temperature detected",
        "body": {
          "html": "The following devices have reported high temperatures:<br><ul>{{#ctx.payload.aggregations.devices.buckets}}<li>Device: {{key}} - Temperature: {{max_temp.value}}°C</li>{{/ctx.payload.aggregations.devices.buckets}}</ul>"
        }
      }
    },
    "webhook": {
      "webhook": {
        "scheme": "https",
        "host": "alerts.example.com",
        "port": 443,
        "method": "post",
        "path": "/api/alerts",
        "params": {},
        "headers": {
          "Content-Type": "application/json"
        },
        "body": "{\"alert_type\":\"temperature\",\"devices\":{{#toJson}}ctx.payload.aggregations.devices.buckets{{/toJson}}}"
      }
    }
  }
}
```

### Anomaly-based Alerts

Set up alerts based on ML anomalies:

```json
POST _watcher/watch/ml-anomaly-alert
{
  "trigger": {
    "schedule": {
      "interval": "15m"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": [".ml-anomalies-*"],
        "body": {
          "size": 0,
          "query": {
            "bool": {
              "filter": [
                {
                  "term": {
                    "job_id": "iot-sensor-anomalies"
                  }
                },
                {
                  "range": {
                    "timestamp": {
                      "gte": "now-15m"
                    }
                  }
                },
                {
                  "range": {
                    "anomaly_score": {
                      "gte": 75
                    }
                  }
                }
              ]
            }
          },
          "aggs": {
            "device": {
              "terms": {
                "field": "by_field_value",
                "size": 10
              },
              "aggs": {
                "max_score": {
                  "max": {
                    "field": "anomaly_score"
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
    "compare": {
      "ctx.payload.hits.total.value": {
        "gt": 0
      }
    }
  },
  "actions": {
    "send_email": {
      "email": {
        "to": "alerts@example.com",
        "subject": "IoT Anomaly Alert: Unusual sensor readings detected",
        "body": {
          "html": "Anomalies detected for the following devices:<br><ul>{{#ctx.payload.aggregations.device.buckets}}<li>Device: {{key}} - Anomaly Score: {{max_score.value}}</li>{{/ctx.payload.aggregations.device.buckets}}</ul>"
        }
      }
    }
  }
}
```

## IoT Security Monitoring

### Device Authentication Monitoring

Set up monitoring for device authentication:

```ruby
# Logstash configuration for security monitoring
filter {
  if [topic] =~ "devices/auth" {
    # Extract device authentication data
    mutate {
      add_field => {
        "event.category" => "authentication"
        "event.type" => "info"
      }
    }
    
    # Check for suspicious access patterns
    if [source.ip] {
      elasticsearch {
        hosts => ["https://elasticsearch:9200"]
        index => "iot-auth-*"
        query => "source.ip:%{[source.ip]} AND device.id:%{[device.id]} AND event.outcome:success"
        fields => { "@timestamp" => "last_auth_time" }
        user => "logstash_read"
        password => "${ES_PASSWORD}"
      }
      
      # Flag potential device spoofing
      if [last_auth_time] and [device.ip] != [last_known_ip] {
        mutate {
          add_field => {
            "security.threat.detected" => true
            "security.threat.description" => "Potential device spoofing - IP changed"
          }
        }
      }
    }
  }
}
```

### Firmware Verification

Monitor firmware versions and updates:

```json
PUT _watcher/watch/firmware-alert
{
  "trigger": {
    "schedule": {
      "interval": "1h"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": ["iot-data-*"],
        "body": {
          "size": 0,
          "query": {
            "bool": {
              "must": [
                {
                  "range": {
                    "@timestamp": {
                      "gte": "now-1h"
                    }
                  }
                }
              ],
              "must_not": [
                {
                  "terms": {
                    "device.firmware": ["1.2.5", "1.2.6", "1.3.0"]
                  }
                }
              ]
            }
          },
          "aggs": {
            "outdated_firmware": {
              "terms": {
                "field": "device.firmware",
                "size": 10
              },
              "aggs": {
                "devices": {
                  "terms": {
                    "field": "device.id",
                    "size": 100
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
    "compare": {
      "ctx.payload.hits.total.value": {
        "gt": 0
      }
    }
  },
  "actions": {
    "send_email": {
      "email": {
        "to": "security@example.com",
        "subject": "Firmware Alert: Outdated firmware detected",
        "body": {
          "html": "The following devices are running outdated firmware:<br><ul>{{#ctx.payload.aggregations.outdated_firmware.buckets}}<li>Firmware version {{key}} - {{devices.buckets.length}} devices</li>{{/ctx.payload.aggregations.outdated_firmware.buckets}}</ul>"
        }
      }
    }
  }
}
```

## Scaling for Production

### High-throughput Architecture

For production IoT deployments:

1. **Multiple data ingestion paths**:
   - Kafka as a message buffer
   - Multiple Logstash instances
   - Load balancing for HTTP endpoints

2. **Elasticsearch cluster sizing**:
   - Hot-warm-cold architecture
   - Dedicated master, data, and ingest nodes
   - Appropriate JVM and OS settings

3. **Sample configuration for hot nodes**:
   ```yaml
   # Hot node configuration
   node.name: es-hot-01
   node.roles: [data_hot, data_content, ingest]
   
   # Memory settings
   bootstrap.memory_lock: true
   
   # Storage settings
   path.data: /mnt/high_performance_disk/elasticsearch/data
   
   # Performance settings
   thread_pool.write.queue_size: 1000
   indices.memory.index_buffer_size: 30%
   ```

### Disaster Recovery

Configure cross-cluster replication for disaster recovery:

```json
PUT _cluster/settings
{
  "persistent": {
    "cluster.remote.dr_cluster.seeds": [
      "es-dr-01:9300",
      "es-dr-02:9300"
    ]
  }
}

PUT _ccr/auto_follow/iot-data
{
  "remote_cluster": "dr_cluster",
  "leader_index_patterns": ["iot-data-*"],
  "follow_index_pattern": "{{leader_index}}",
  "settings": {
    "index.number_of_replicas": 1
  }
}
```

## Case Study: Smart Factory

### Project Overview

Implement IoT monitoring for a smart factory with:

1. **200+ temperature and vibration sensors** on manufacturing equipment
2. **50+ environmental sensors** throughout the facility
3. **25 production line monitoring systems**

### Data Architecture

Implement a comprehensive data architecture:

```
┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│  Factory      │     │  Edge Gateway  │     │  Kafka Cluster│
│  Sensors      │────▶│  - Filtering   │────▶│  - Message    │
│  - Equipment  │     │  - Aggregation │     │    Buffering  │
│  - Environment│     │                │     │               │
└───────────────┘     └───────────────┘     └───────┬───────┘
                                                    │
┌────────────────────────────────────────────┐     │
│              Elastic Stack                  │     │
│                                             │     │
│  ┌───────────────┐     ┌───────────────┐   │     │
│  │   Logstash    │     │ Elasticsearch │   │     │
│  │  - Collection │◀────┤  - Hot-Warm   │◀──┼─────┘
│  │  - Processing │────▶│  - ILM/Rollups│   │
│  └───────────────┘     └───────┬───────┘   │
│                                │           │
│  ┌───────────────┐     ┌───────▼───────┐   │
│  │  Machine      │     │    Kibana     │   │
│  │  Learning     │◀───▶│  - Dashboards │   │
│  │  - Predictive │     │  - Alerts     │   │
│  │  Maintenance  │     │  - Reports    │   │
│  └───────────────┘     └───────────────┘   │
└────────────────────────────────────────────┘
```

### Key Visualizations

1. **Factory Floor Map** (Custom Vega visualization):
   - 2D layout of factory floor
   - Equipment and sensor locations
   - Color-coded by temperature or status
   - Clickable for detailed information

2. **Equipment Health Heatmap**:
   - Y-axis: Equipment IDs
   - X-axis: Time
   - Color: Anomaly score
   - Filter: Last 24 hours

3. **Predictive Maintenance Dashboard**:
   - ML-based failure predictions
   - Anomaly scores for vibration patterns
   - Maintenance schedule recommendations
   - Historical failure correlation

### Alerting Strategy

Implement multi-level alerting:

1. **Level 1 - Information**:
   - Minor deviations from normal
   - Log only, no immediate action

2. **Level 2 - Warning**:
   - Significant deviations
   - Email to maintenance staff
   - Add to maintenance queue

3. **Level 3 - Critical**:
   - Severe anomalies or pre-failure conditions
   - SMS and email alerts
   - Automated production line slowdown
   - Immediate maintenance dispatch

## Best Practices

### IoT Data Collection Best Practices

1. **Minimize data at the source**:
   - Filter irrelevant data at the edge
   - Use delta encoding for sensor readings
   - Pre-aggregate high-frequency measurements

2. **Optimize network usage**:
   - Use binary protocols (MQTT, Protocol Buffers)
   - Implement batch sending for non-critical data
   - Use compression for larger payloads

3. **Implement data validation**:
   - Set realistic min/max thresholds
   - Detect impossible state transitions
   - Flag suspicious patterns

### Storage and Retention Best Practices

1. **Implement tiered storage**:
   - Hot tier: Recent, high-resolution data (SSD)
   - Warm tier: Medium-term, medium-resolution data
   - Cold tier: Long-term, aggregated data

2. **Define appropriate retention policies**:
   - Raw data: 7-30 days
   - Hourly aggregations: 6-12 months
   - Daily aggregations: 3-5 years

3. **Optimize index management**:
   - Smaller shards for hot data (faster reallocation)
   - Larger shards for cold data (storage efficiency)
   - Force-merge and shrink older indices

### Security Best Practices

1. **Implement device authentication**:
   - Unique credentials per device
   - Certificate-based authentication
   - Token rotation policies

2. **Secure data transmission**:
   - TLS for all connections
   - Payload encryption for sensitive data
   - Message signing for critical commands

3. **Monitor for security anomalies**:
   - Unusual connection patterns
   - Unexpected firmware versions
   - Abnormal message volumes or payloads

## Conclusion

The Elastic Stack provides a powerful platform for IoT data processing, enabling real-time monitoring, analysis, and visualization of sensor data at scale. By implementing the architecture and best practices outlined in this chapter, organizations can build robust IoT solutions that deliver actionable insights and value from their connected devices.

Whether you're monitoring industrial equipment, tracking environmental conditions, or analyzing connected products, the Elastic Stack's ability to ingest, process, store, and analyze time series data makes it an excellent choice for IoT applications of all kinds. As IoT continues to grow and evolve, the flexibility and scalability of the Elastic Stack ensure that your data infrastructure can grow and adapt with your IoT deployments.