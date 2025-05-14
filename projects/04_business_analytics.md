# Business Analytics Dashboard with ELK Stack

This chapter guides you through building comprehensive business analytics dashboards using the Elastic Stack. You'll learn how to collect, process, and visualize business data to provide actionable insights for stakeholders across your organization.

## Table of Contents

- [Business Analytics Overview](#business-analytics-overview)
- [Data Architecture for Business Analytics](#data-architecture-for-business-analytics)
- [Data Collection Methods](#data-collection-methods)
- [Data Processing and Enrichment](#data-processing-and-enrichment)
- [Building KPI Dashboards](#building-kpi-dashboards)
- [Sales Performance Dashboard](#sales-performance-dashboard)
- [Customer Analytics Dashboard](#customer-analytics-dashboard)
- [Marketing Campaign Analysis](#marketing-campaign-analysis)
- [Financial Analytics Dashboard](#financial-analytics-dashboard)
- [Operational Metrics Dashboard](#operational-metrics-dashboard)
- [Real-time Business Monitoring](#real-time-business-monitoring)
- [Advanced Analytics with ML](#advanced-analytics-with-ml)
- [Dashboard Sharing and Reporting](#dashboard-sharing-and-reporting)
- [Best Practices](#best-practices)

## Business Analytics Overview

The Elastic Stack excels at collecting, processing, and visualizing large volumes of business data from various sources. Key benefits include:

- **Real-time analytics**: Monitor business metrics as they change
- **Unified data platform**: Collect data from multiple business systems
- **Scalable storage**: Handle large volumes of historical and real-time data
- **Powerful visualization**: Create interactive, shareable dashboards
- **Advanced analytics**: Apply machine learning for forecasting and anomaly detection

## Data Architecture for Business Analytics

A robust business analytics platform using the Elastic Stack typically includes:

1. **Data Collection Layer**: Beats, Logstash, and custom integrations to collect data from:
   - Databases (MySQL, PostgreSQL, MongoDB)
   - APIs (Salesforce, HubSpot, ServiceNow)
   - Web analytics platforms
   - E-commerce systems
   - ERP and CRM platforms

2. **Data Processing Layer**: Logstash or Elasticsearch Ingest Pipelines for:
   - Data normalization
   - Field extraction
   - Data enrichment
   - Calculations and transformations

3. **Storage Layer**: Elasticsearch indices with:
   - Optimized mappings for analytics
   - Appropriate index lifecycle policies
   - Role-based access control

4. **Analytics Layer**: Kibana dashboards for:
   - KPI monitoring
   - Trend analysis
   - Drill-down capabilities
   - Alerts on metric thresholds

## Data Collection Methods

### Database Integration

To collect data from databases:

#### JDBC Input for Logstash

```ruby
# logstash.conf for database collection
input {
  jdbc {
    jdbc_driver_library => "/path/to/mysql-connector-java-8.0.23.jar"
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://localhost:3306/ecommerce"
    jdbc_user => "username"
    jdbc_password => "password"
    schedule => "*/30 * * * *"  # Run every 30 minutes
    statement => "SELECT o.order_id, o.order_date, o.customer_id, c.customer_name, 
                  o.total_amount, o.status, o.payment_method 
                  FROM orders o JOIN customers c ON o.customer_id = c.customer_id 
                  WHERE o.order_date >= :sql_last_value"
    use_column_value => true
    tracking_column => "order_date"
    tracking_column_type => "timestamp"
  }
}

filter {
  # Convert timestamp
  date {
    match => [ "order_date", "yyyy-MM-dd HH:mm:ss" ]
    target => "order_date"
  }
  
  # Calculate order value brackets
  mutate {
    add_field => {
      "order_value_bracket" => "%{total_amount}"
    }
  }
  
  ruby {
    code => "
      amount = event.get('total_amount').to_f
      bracket = case
        when amount < 50 then 'Under $50'
        when amount < 100 then '$50-$99'
        when amount < 200 then '$100-$199'
        else '$200+'
      end
      event.set('order_value_bracket', bracket)
    "
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "ecommerce-orders-%{+YYYY.MM}"
    document_id => "%{order_id}"
  }
}
```

#### Using Filebeat with SQL Module

For more regular database collection:

```yaml
# filebeat.yml
filebeat.modules:
- module: sql
  period: 10m
  query: SELECT o.order_id, o.order_date, o.customer_id, c.customer_name, 
         o.total_amount, o.status, o.payment_method 
         FROM orders o JOIN customers c ON o.customer_id = c.customer_id 
         WHERE o.order_date >= :sql_last_value
  driver: "mysql"
  dsn: "username:password@tcp(localhost:3306)/ecommerce"
```

### API Integration

#### Collecting Data from REST APIs

Use Logstash HTTP input:

```ruby
# logstash.conf for API collection
input {
  http_poller {
    urls => {
      sales => {
        method => get
        url => "https://api.salesforce.com/services/data/v52.0/query/?q=SELECT+Id,+Name,+Amount,+CloseDate+FROM+Opportunity+WHERE+CloseDate+>+LAST_N_DAYS:30"
        headers => {
          "Authorization" => "Bearer ${SALESFORCE_TOKEN}"
          "Content-Type" => "application/json"
        }
        metadata_target => "http_poller_metadata"
      }
    }
    request_timeout => 60
    schedule => { cron => "*/15 * * * *" }
  }
}

filter {
  json {
    source => "message"
  }
  
  split {
    field => "[records]"
  }
  
  mutate {
    rename => { "[records][Id]" => "opportunity_id" }
    rename => { "[records][Name]" => "opportunity_name" }
    rename => { "[records][Amount]" => "amount" }
    rename => { "[records][CloseDate]" => "close_date" }
  }
  
  date {
    match => [ "close_date", "yyyy-MM-dd" ]
    target => "close_date"
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "salesforce-opportunities-%{+YYYY.MM}"
    document_id => "%{opportunity_id}"
  }
}
```

### Event-Driven Collection

For real-time business events, use:

1. **Kafka Input** for Logstash:
   ```ruby
   input {
     kafka {
       bootstrap_servers => "kafka:9092"
       topics => ["order-events", "inventory-events"]
       group_id => "logstash"
       codec => "json"
     }
   }
   ```

2. **Webhook Endpoints** with Logstash HTTP input:
   ```ruby
   input {
     http {
       port => 8088
       codec => "json"
     }
   }
   ```

## Data Processing and Enrichment

### Normalizing Data with Logstash

Standardize data from multiple sources:

```ruby
filter {
  # Standardize date formats
  date {
    match => [ "transaction_date", "yyyy-MM-dd'T'HH:mm:ss.SSSZ", "yyyy-MM-dd", "MM/dd/yyyy" ]
    target => "transaction_date"
  }
  
  # Normalize product categories
  translate {
    field => "product_category"
    destination => "product_category_standard"
    dictionary => {
      "electronics" => "Electronics"
      "electronic" => "Electronics"
      "computer" => "Electronics"
      "phones" => "Electronics"
      "cloths" => "Apparel"
      "clothing" => "Apparel"
      "apparel" => "Apparel"
    }
    fallback => "%{product_category}"
  }
  
  # Add business metadata
  mutate {
    add_field => {
      "business_unit" => "%{[metadata][business_unit]}"
      "region" => "%{[metadata][region]}"
    }
  }
}
```

### Geo-enrichment for Regional Analysis

Add geographical data for regional analysis:

```ruby
filter {
  # Extract country from IP address
  geoip {
    source => "customer_ip"
    target => "geo"
  }
  
  # Extract city/region from ZIP code
  translate {
    field => "zip_code"
    destination => "region"
    dictionary_path => "/etc/logstash/zip_to_region.yml"
  }
}
```

### Business Calculations 

Perform business calculations during indexing:

```ruby
filter {
  # Calculate profit margin
  ruby {
    code => "
      if event.get('sale_price') && event.get('cost')
        sale_price = event.get('sale_price').to_f
        cost = event.get('cost').to_f
        if sale_price > 0
          margin = ((sale_price - cost) / sale_price) * 100
          event.set('profit_margin', margin.round(2))
        end
      end
    "
  }
  
  # Calculate month-over-month change
  ruby {
    code => "
      if event.get('current_month_revenue') && event.get('previous_month_revenue')
        current = event.get('current_month_revenue').to_f
        previous = event.get('previous_month_revenue').to_f
        if previous > 0
          change = ((current - previous) / previous) * 100
          event.set('mom_change', change.round(2))
        end
      end
    "
  }
}
```

## Building KPI Dashboards

### Data Preparation

Before building dashboards, prepare your Elasticsearch indices:

1. Create index templates with appropriate mappings:

```json
PUT _index_template/business_analytics
{
  "index_patterns": ["sales-*", "customers-*", "marketing-*", "finance-*"],
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "transaction_date": { "type": "date" },
        "revenue": { "type": "double" },
        "cost": { "type": "double" },
        "profit": { "type": "double" },
        "profit_margin": { "type": "double" },
        "customer_id": { "type": "keyword" },
        "product_id": { "type": "keyword" },
        "product_category": { "type": "keyword" },
        "region": { "type": "keyword" },
        "sales_channel": { "type": "keyword" },
        "business_unit": { "type": "keyword" },
        "geo": {
          "properties": {
            "country_name": { "type": "keyword" },
            "region_name": { "type": "keyword" },
            "city_name": { "type": "keyword" },
            "location": { "type": "geo_point" }
          }
        }
      }
    }
  }
}
```

2. Create a Kibana index pattern for your business data:
   - Go to Kibana → Stack Management → Index Patterns → Create index pattern
   - Enter pattern: `sales-*,customers-*,marketing-*,finance-*`
   - Select timestamp field: `@timestamp`

### Dashboard Structure

Create a hierarchical dashboard structure:

1. **Executive Dashboard**: High-level KPIs and metrics
2. **Department Dashboards**: Sales, Marketing, Finance, Operations
3. **Detail Dashboards**: Product performance, Customer segments, Campaign analysis

## Sales Performance Dashboard

Create a comprehensive sales performance dashboard:

### Revenue Trends Visualization

1. Create a line chart for revenue trends:
   - Metrics: Sum of revenue
   - Buckets: X-axis = Date Histogram of transaction_date (Monthly)
   - Buckets: Split Series = Terms of sales_channel
   
2. Create a bar chart for revenue by region:
   - Metrics: Sum of revenue
   - Buckets: X-axis = Terms of region
   - Buckets: Split Series = Terms of product_category

### Sales KPI Metrics

Add metric visualizations for key KPIs:

1. **Total Revenue**: Metric Visualization
   - Aggregation: Sum
   - Field: revenue
   - Filter: transaction_date is within the selected time range

2. **Average Order Value**: Metric Visualization
   - Aggregation: Average
   - Field: order_amount
   - Filter: transaction_date is within the selected time range

3. **Sales Growth**: Metric Visualization with growth calculation
   - Use a scripted field or Vega visualization to calculate period-over-period growth

### Product Performance

Create product performance visualizations:

1. **Top Products Table**:
   - Metrics: Sum of revenue, Count of orders, Average of profit_margin
   - Buckets: Split Rows = Terms of product_name (Top 10)
   
2. **Product Category Pie Chart**:
   - Metrics: Sum of revenue
   - Buckets: Split Slices = Terms of product_category

## Customer Analytics Dashboard

Create visualizations for customer analysis:

### Customer Segmentation

1. **Customer Segments by Value**:
   - Metrics: Count of distinct customer_id
   - Buckets: X-axis = Range of lifetime_value
     - Ranges: 0-100, 100-500, 500-1000, 1000-5000, 5000+

2. **Customer Acquisition Trends**:
   - Metrics: Count of distinct customer_id
   - Buckets: X-axis = Date Histogram of first_purchase_date (Monthly)
   - Buckets: Split Series = Terms of acquisition_channel

### Customer Behavior Analysis

1. **Purchase Frequency**:
   - Metrics: Count of orders
   - Buckets: X-axis = Terms of customer_id (Top 20)

2. **Customer Retention Heatmap**:
   - Create a cohort analysis visualization using Vega or a custom visualization:
   ```
   {
     "data": {
       "url": {
         "index": "sales-*",
         "body": {
           "size": 0,
           "aggs": {
             "cohort": {
               "date_histogram": {
                 "field": "first_purchase_date",
                 "calendar_interval": "month"
               },
               "aggs": {
                 "retention": {
                   "date_histogram": {
                     "field": "transaction_date",
                     "calendar_interval": "month"
                   }
                 }
               }
             }
           }
         }
       }
     },
     "mark": "rect",
     "encoding": {
       "x": {"field": "retention", "type": "ordinal"},
       "y": {"field": "cohort", "type": "ordinal"},
       "color": {"field": "count", "type": "quantitative"}
     }
   }
   ```

3. **Customer Lifetime Value**:
   - Metrics: Average of lifetime_value
   - Buckets: X-axis = Terms of acquisition_channel
   - Buckets: Split Charts = Terms of customer_segment

## Marketing Campaign Analysis

Create a marketing campaign analysis dashboard:

### Campaign Performance

1. **Campaign ROI Comparison**:
   - Metrics: Sum of revenue, Sum of campaign_cost, Calculated field for ROI
   - Buckets: X-axis = Terms of campaign_name

2. **Campaign Conversion Funnel**:
   - Create a Timelion visualization for funnel analysis:
   ```
   .es(index=marketing-*, q='event:impression', timefield=event_date).label('Impressions'),
   .es(index=marketing-*, q='event:click', timefield=event_date).label('Clicks'),
   .es(index=marketing-*, q='event:product_view', timefield=event_date).label('Product Views'),
   .es(index=marketing-*, q='event:add_to_cart', timefield=event_date).label('Add to Cart'),
   .es(index=marketing-*, q='event:purchase', timefield=event_date).label('Purchase')
   ```

### Channel Attribution

1. **Revenue by Marketing Channel**:
   - Metrics: Sum of revenue
   - Buckets: X-axis = Terms of marketing_channel
   - Buckets: Split Series = Terms of attribution_model

2. **Multi-touch Attribution Model**:
   - Create a custom visualization using Vega for advanced attribution:
   ```
   {
     "data": {
       "url": {
         "index": "marketing-attribution-*",
         "body": {
           "size": 0,
           "aggs": {
             "channels": {
               "terms": {
                 "field": "marketing_channel",
                 "size": 10
               },
               "aggs": {
                 "attribution": {
                   "terms": {
                     "field": "attribution_type",
                     "size": 5
                   },
                   "aggs": {
                     "revenue": {
                       "sum": {
                         "field": "attributed_revenue"
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
     "mark": "bar",
     "encoding": {
       "x": {"field": "channels", "type": "nominal"},
       "y": {"field": "revenue", "type": "quantitative"},
       "color": {"field": "attribution", "type": "nominal"}
     }
   }
   ```

## Financial Analytics Dashboard

Create financial analytics visualizations:

### Revenue and Profit Analysis

1. **Revenue vs. Profit Trend**:
   - Metrics: Sum of revenue, Sum of profit
   - Buckets: X-axis = Date Histogram of transaction_date (Monthly)
   
2. **Profit Margin by Business Unit**:
   - Metrics: Average of profit_margin
   - Buckets: X-axis = Terms of business_unit
   - Buckets: Split Series = Date Histogram of transaction_date (Quarterly)

### Expense Tracking

1. **Expense Categories Breakdown**:
   - Metrics: Sum of expense_amount
   - Buckets: X-axis = Terms of expense_category
   - Buckets: Split Series = Terms of business_unit

2. **Budget vs. Actual**:
   - Create a Timelion visualization for budget comparison:
   ```
   .es(index=finance-budget-*, q='type:budget', timefield=period_date).label('Budget'),
   .es(index=finance-actuals-*, q='type:actual', timefield=transaction_date).label('Actual'),
   .es(index=finance-budget-*, q='type:budget', timefield=period_date)
    .subtract(.es(index=finance-actuals-*, q='type:actual', timefield=transaction_date))
    .label('Variance').color('#FF7E62')
   ```

## Operational Metrics Dashboard

Track operational performance:

### Supply Chain Metrics

1. **Inventory Levels**:
   - Metrics: Sum of inventory_quantity
   - Buckets: X-axis = Terms of product_category
   - Buckets: Split Series = Terms of warehouse_location

2. **Order Fulfillment Time**:
   - Metrics: Average of fulfillment_time_hours
   - Buckets: X-axis = Date Histogram of order_date (Weekly)
   - Buckets: Split Series = Terms of fulfillment_center

### Customer Service Metrics

1. **Support Ticket Volume**:
   - Metrics: Count of tickets
   - Buckets: X-axis = Date Histogram of created_date (Daily)
   - Buckets: Split Series = Terms of ticket_category

2. **Average Resolution Time**:
   - Metrics: Average of resolution_time_hours
   - Buckets: X-axis = Terms of support_team
   - Buckets: Split Series = Terms of ticket_priority

## Real-time Business Monitoring

Create real-time dashboards for business monitoring:

### Real-time Sales Dashboard

1. Configure dashboard for auto-refresh (every 30 seconds)
2. Add visualizations focused on the last hour of data:
   - Current hour's sales
   - Orders per minute line chart
   - Active cart count

### Threshold Alerting

Set up alerts for business metrics:

1. Go to Kibana → Stack Management → Rules and Connectors
2. Create rule for low inventory:
   ```json
   {
     "trigger": {
       "schedule": {
         "interval": "1h"
       }
     },
     "input": {
       "search": {
         "request": {
           "indices": ["inventory-*"],
           "body": {
             "query": {
               "bool": {
                 "filter": [
                   { "range": { "stock_quantity": { "lt": "reorder_level" } } }
                 ]
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
       "notify_email": {
         "email": {
           "to": "inventory@example.com",
           "subject": "Low Stock Alert",
           "body": "The following products are below reorder levels: {{ctx.payload.hits.hits}}"
         }
       }
     }
   }
   ```

3. Create rule for sales anomalies:
   - Use ML job to detect anomalies in sales data
   - Trigger alert when anomaly score exceeds threshold

## Advanced Analytics with ML

Implement machine learning for business insights:

### Sales Forecasting

Create a machine learning job for sales forecasting:

1. Go to Kibana → Machine Learning → Anomaly Detection → Create job
2. Configure job:
   - Data source: sales-*
   - Analysis type: Multi-metric
   - Fields to analyze: Sum of revenue
   - Split data by: product_category
   - Bucket span: 1 day
   - Model memory limit: 20MB

3. Create a forecast visualization:
   - Add ML results to dashboard
   - Show forecasted values for next 7-30 days

### Anomaly Detection for Business Metrics

Set up anomaly detection for key metrics:

1. Create ML job for revenue anomalies:
   ```json
   {
     "job_id": "revenue_anomalies",
     "description": "Detect anomalies in revenue by channel",
     "groups": ["business", "sales"],
     "analysis_config": {
       "bucket_span": "1h",
       "detectors": [
         {
           "detector_description": "Sum of revenue by channel",
           "function": "sum",
           "field_name": "revenue",
           "by_field_name": "sales_channel"
         }
       ],
       "influencers": [
         "sales_channel",
         "region",
         "product_category"
       ]
     },
     "data_description": {
       "time_field": "transaction_date"
     }
   }
   ```

2. Create a dashboard showing anomalies:
   - Anomaly timeline visualization
   - Top influencers visualization
   - Annotated metric charts

### Customer Segmentation with ML

Use ML to create customer segments:

1. Export customer data to a CSV
2. Use K-means clustering in a Jupyter notebook
3. Import results back into Elasticsearch
4. Create visualizations of customer segments

## Dashboard Sharing and Reporting

### Embedding Dashboards

Create embedded dashboards for stakeholders:

1. Generate public or authenticated URL for dashboard
2. Embed in internal portals or applications
3. Set up appropriate permissions

### Automated Reporting

Configure automated report generation:

1. Go to Kibana → Stack Management → Reporting
2. Set up a scheduled report for the Executive Dashboard:
   - Daily report sent at 8 AM
   - PDF format
   - Email to leadership team
   - Include annotations and comments

### CSV Export

Enable data export for further analysis:

1. Configure reporting role with export privileges
2. Set up saved searches for key data sets
3. Create export button on dashboards for ad-hoc exports

## Best Practices

### Dashboard Design Best Practices

1. **Focus on clarity**:
   - Start with key metrics (KPIs)
   - Use consistent color schemes
   - Include context and comparison (vs. previous period, goals)
   - Group related visualizations together

2. **Optimize for audience**:
   - Executive dashboards: high-level, summary metrics
   - Operational dashboards: detailed, actionable metrics
   - Analytical dashboards: exploratory, drill-down capabilities

3. **Apply visualization best practices**:
   - Use appropriate chart types (line for trends, bar for comparisons)
   - Avoid chart junk and 3D visualizations
   - Ensure text is readable
   - Use color purposefully

### Data Management Best Practices

1. **Index optimization**:
   - Use daily/monthly indices with appropriate shard counts
   - Implement index lifecycle management
   - Use rollups for historical data

2. **Field management**:
   - Consistent naming conventions
   - Appropriate field types (keyword vs. text)
   - Judicious use of scripted fields

3. **Query optimization**:
   - Filter before aggregating
   - Use appropriate date ranges
   - Cache frequent queries

### Security and Access Control

1. **Role-based access**:
   - Create roles for different departments
   - Restrict sensitive financial data
   - Implement field-level security for PII

2. **Data anonymization**:
   - Anonymize customer information where appropriate
   - Aggregate individual data for public dashboards

## Conclusion

Business analytics dashboards using the Elastic Stack provide organizations with powerful tools for monitoring performance, identifying trends, and making data-driven decisions. By following the implementation guidelines and best practices outlined in this chapter, you can create comprehensive analytics solutions that deliver actionable insights across your organization.

The flexibility of the Elastic Stack allows you to start with basic visualizations and progressively add more sophisticated analytics capabilities as your needs evolve. Whether you're tracking sales performance, customer behavior, or operational efficiency, Elastic provides the foundation for effective business intelligence.