# Kibana Visualizations

This chapter covers Kibana's visualization capabilities, which allow you to create powerful visual representations of your Elasticsearch data.

## Table of Contents
- [Visualization Types](#visualization-types)
- [Creating Visualizations](#creating-visualizations)
- [Aggregations and Metrics](#aggregations-and-metrics)
- [Visual Builder](#visual-builder)
- [Time Series Visual Builder (TSVB)](#time-series-visual-builder-tsvb)
- [Visualization Options and Settings](#visualization-options-and-settings)
- [Saving and Sharing](#saving-and-sharing)
- [Advanced Visualization Techniques](#advanced-visualization-techniques)

## Visualization Types

Kibana offers a wide range of visualization types to represent your data:

### Basic Visualizations

- **Line, Area, and Bar Charts**
  - Ideal for time-series data and trends
  - Support for multiple series, stacking, and color schemes

- **Pie and Donut Charts**
  - Effective for showing proportions and distributions
  - Support for multiple slices and drilldowns

- **Data Tables**
  - Tabular view of aggregated data
  - Sortable columns and pagination options

- **Metrics**
  - Single-value displays (counters, gauges)
  - Support for trends and thresholds

### Specialized Visualizations

- **Heat Maps**
  - Display data density across two dimensions
  - Color coding for value intensity

- **Tag Clouds**
  - Visual representation of text frequency
  - Customizable sizing and orientation

- **Coordinate Maps and Region Maps**
  - Geospatial data visualization
  - Support for points, lines, and regions

- **Controls**
  - Interactive input elements
  - Range sliders, dropdown selectors, and options lists

- **Timelion**
  - Time series data visualization with its own expression language
  - Mathematical functions and data transformations

- **Vega and Vega-Lite**
  - Custom visualizations using the Vega grammar
  - Support for complex, interactive charts

## Creating Visualizations

### Basic Workflow

1. From Kibana, navigate to **Visualize**
2. Click **Create visualization** to see available visualization types
3. Select a visualization type
4. Choose a source:
   - New search
   - Saved search
   - Index pattern
5. Configure the visualization:
   - Set metrics (y-axis values)
   - Set buckets (data groupings)
   - Adjust options and settings
6. Save the visualization

### Example: Creating a Basic Bar Chart

1. Select **Bar** visualization
2. Choose an index pattern
3. Add Y-axis metric:
   - Aggregation: Count, Average, Sum, etc.
   - Field: Numeric field to aggregate
   - Custom Label: Optional display name
4. Add X-axis bucket:
   - Aggregation: Terms, Date Histogram, Range, etc.
   - Field: Field to group by
   - Size: Number of values to display
   - Order: How to sort results
5. Click **Update** to apply changes

### Example: Creating a Pie Chart

```
1. Select "Pie" visualization
2. Choose an index pattern
3. Add Metric:
   - Aggregation: Count
4. Add Bucket:
   - Split Slices
   - Aggregation: Terms
   - Field: response.keyword
   - Size: 5
5. Click "Update"
```

This creates a pie chart showing the distribution of HTTP response codes.

## Aggregations and Metrics

Aggregations are at the core of Kibana visualizations, allowing you to summarize and analyze data.

### Metric Aggregations

Calculate single values based on a set of documents:

- **Count**: Number of documents
- **Average**: Mean value of a numeric field
- **Sum**: Total of a numeric field
- **Min/Max**: Minimum/maximum value
- **Median/Percentiles**: Statistical distributions
- **Standard Deviation**: Statistical variance measure
- **Cardinality**: Count of unique values
- **Top Hit**: Sample values from top documents

### Bucket Aggregations

Group documents into buckets based on criteria:

- **Date Histogram**: Time-based bucketing
- **Histogram**: Numeric range bucketing
- **Terms**: Group by field values
- **Range**: Custom numeric ranges
- **Date Range**: Custom date ranges
- **Filters**: Custom query-based buckets
- **Significant Terms**: Statistically significant terms
- **Geohash Grid**: Geospatial bucketing

### Advanced Aggregations

- **Parent Pipeline Aggregations**:
  - Derivative: Rate of change
  - Cumulative Sum: Running total
  - Moving Average: Smoothed trends
  
- **Sibling Pipeline Aggregations**:
  - Average Bucket: Average of other buckets
  - Sum Bucket: Total across buckets
  - Min/Max Bucket: Extremes across buckets

## Visual Builder

Visual Builder is a powerful tool for creating complex visualizations with an intuitive interface.

### Types of Visual Builder Panels

- **Time Series**: Line, bar, and area charts
- **Metric**: Single value displays
- **Top N**: Ranked lists with bars
- **Gauge**: Dial or arc displays
- **Markdown**: Text with data variables
- **Table**: Tabular data display

### Example: Creating a Time Series Visualization

1. Select **Visual Builder** from visualization types
2. Choose **Time Series** panel type
3. Define data series:
   - Select metrics aggregation (Count, Average, etc.)
   - Choose field to aggregate
   - Set group by options (Terms, Filters, etc.)
4. Add multiple series for comparison
5. Configure panel options:
   - Chart type (line, bar, area)
   - Color scheme
   - Axis settings
   - Legend positioning

```yaml
# Example configuration
Panel Type: Time Series
Data:
  - Aggregation: Count
    Group By: Terms
    Field: response.keyword
    Top: 5 values
Options:
  Chart Type: Line
  Stacked: true
  Legend: top-right
  Y-Axis: logarithmic
```

### Formulas and Calculations

Visual Builder supports mathematical formulas between series:

```
params.series1 + params.series2          # Addition
params.series1 - params.series2          # Subtraction
params.series1 * 100                     # Multiplication
params.series1 / params.series2          # Division
Math.max(params.series1, params.series2) # Maximum value
```

Example: Calculate error rate percentage:
```
(params.errors / params.total) * 100
```

## Time Series Visual Builder (TSVB)

TSVB is designed specifically for time-based data visualization and analysis.

### Key Features

- **Multiple visualization types** in one interface
- **Annotations** for marking significant events
- **Math functions** for data transformation
- **Color rules** for dynamic styling
- **Template variables** for dynamic text
- **Split series** by terms or filters

### Time Series Example

Creating a CPU usage visualization with thresholds:

1. Add metrics:
   - Aggregation: Average
   - Field: system.cpu.user.pct
   
2. Add another metric:
   - Aggregation: Average
   - Field: system.cpu.system.pct

3. Configure options:
   - Stack series
   - Set different colors
   - Add threshold markers at 70% and 90%
   - Display a legend

4. Add annotations:
   - Source: deployment events
   - Display marker for system restarts

## Visualization Options and Settings

### Data Display Options

- **Data Formatting**:
  - Number formats (decimal places, thousands separators)
  - Date formats
  - String transformations

- **Scale Types**:
  - Linear, logarithmic, square root
  - Min/max bounds
  - Extended bounds

- **Orientation and Direction**:
  - Horizontal/vertical
  - Left-to-right/right-to-left
  - Ascending/descending

### Styling Options

- **Colors and Palettes**:
  - Custom color schemes
  - Color rules based on values
  - Gradient or stepped color ranges

- **Labels and Legends**:
  - Axis labels and titles
  - Legend position and format
  - Value labels

- **Grid and Axes**:
  - Grid lines
  - Axis scaling
  - Multiple Y-axes

### Interactive Elements

- **Tooltips**:
  - Custom tooltip content
  - Tooltip formatting

- **Drilldowns**:
  - Click behavior
  - URL actions
  - Filter actions

## Saving and Sharing

### Saving Visualizations

Save visualizations for later use:

1. Click **Save** in the visualization editor
2. Provide a title and optional description
3. Choose to save it as a new visualization or update an existing one
4. Optionally, add to an existing dashboard

### Exporting and Importing

Share visualizations between Kibana instances:

1. **Exporting**:
   - Navigate to Management > Saved Objects
   - Select visualizations to export
   - Click Export to download a JSON file

2. **Importing**:
   - Navigate to Management > Saved Objects
   - Click Import
   - Upload previously exported JSON file
   - Resolve any conflicts

### URL Sharing

Share direct links to visualizations:

1. Open a saved visualization
2. Click **Share**
3. Choose from sharing options:
   - Direct link
   - Embedded iframe code
   - Snapshot (PNG image)

4. Configure time range options:
   - Current time range
   - Set specific time range
   - Auto-refresh settings

## Advanced Visualization Techniques

### Data Transformations

Manipulate data before visualization:

- **Scripted Fields**:
  - Create custom fields using Painless script
  - Perform calculations on existing fields
  - Example: Convert bytes to megabytes
    ```
    doc['bytes'].value / (1024*1024)
    ```

- **Lens Formulas**:
  - Apply mathematical operations
  - Combine multiple metrics
  - Example: Calculate percentage
    ```
    (count(response.keyword=="404") / count()) * 100
    ```

### Multi-Series Visualizations

Create complex, multi-level visualizations:

1. **Split Series**:
   - Divide data into multiple series
   - Use terms, filters, or ranges
   - Apply sub-aggregations

2. **Split Charts**:
   - Create small multiples
   - Compare different dimensions
   - Show correlations

Example: HTTP Response Codes by Host and Method
```
Y-axis: Count
X-axis: Date Histogram (timestamp)
Split Series: Terms (response.keyword)
Split Chart: Terms (host.keyword)
```

### Adding Context with Annotations

Enhance time series visualizations with event markers:

1. Configure an annotation data source:
   - Index pattern with event data
   - Time field for positioning
   - Field for annotation text

2. Style the annotations:
   - Icon or line style
   - Color coding
   - Hover text

Example: Marking Deployments on Error Rate Chart
```
Data Source: deployments-*
Time Field: timestamp
Text Field: message
Icon: tag
Color: #F00
```

### Custom Vega Visualizations

For specialized visualization needs, use Vega specifications:

```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v4.json",
  "data": {
    "url": {
      "%context%": true,
      "%timefield%": "@timestamp",
      "index": "logs-*",
      "body": {
        "size": 0,
        "aggs": {
          "histogram": {
            "date_histogram": {
              "field": "@timestamp",
              "interval": "1h"
            },
            "aggs": {
              "errors": {
                "filter": {
                  "term": {
                    "level": "error"
                  }
                }
              }
            }
          }
        }
      }
    },
    "format": {"property": "aggregations.histogram.buckets"}
  },
  "mark": "bar",
  "encoding": {
    "x": {
      "field": "key",
      "type": "temporal",
      "title": "Time"
    },
    "y": {
      "field": "errors.doc_count",
      "type": "quantitative",
      "title": "Error Count"
    }
  }
}
```

## Conclusion

Kibana's visualization capabilities provide powerful tools for exploring and understanding your data. By mastering the various visualization types and their configuration options, you can create insightful representations of your data that help identify patterns, trends, and anomalies.

In the next chapter, we will explore how to combine multiple visualizations into comprehensive dashboards that provide a complete view of your data.