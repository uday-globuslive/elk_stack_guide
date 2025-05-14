# Kibana Dashboards

This chapter explores how to create, configure, and use Kibana dashboards to combine multiple visualizations into cohesive, interactive displays that provide comprehensive insights into your data.

## Table of Contents
- [Dashboard Fundamentals](#dashboard-fundamentals)
- [Creating Dashboards](#creating-dashboards)
- [Adding and Arranging Visualizations](#adding-and-arranging-visualizations)
- [Dashboard Controls and Filters](#dashboard-controls-and-filters)
- [Time Range Controls](#time-range-controls)
- [Dashboard Options](#dashboard-options)
- [Saving and Sharing](#saving-and-sharing)
- [Drilldowns and Actions](#drilldowns-and-actions)
- [Dashboard Best Practices](#dashboard-best-practices)
- [Real-World Dashboard Examples](#real-world-dashboard-examples)

## Dashboard Fundamentals

Dashboards in Kibana are collections of visualizations, saved searches, and controls that provide a comprehensive view of your data. They allow you to:

- Monitor multiple metrics and trends at a glance
- Investigate patterns and correlations across different data points
- Share insights with stakeholders
- Create operational views for specific use cases
- Tell data stories through structured visual presentations

### Dashboard Architecture

A Kibana dashboard consists of these key components:

1. **Container**: The overall dashboard layout and settings
2. **Panels**: Individual visualizations, saved searches, and markdown elements
3. **Controls**: Interactive elements for filtering and time range selection
4. **Filters**: Global dashboard filters that affect all panels
5. **Query Bar**: Free-text search functionality that filters all panels

## Creating Dashboards

### Basic Dashboard Creation

To create a new dashboard:

1. Navigate to **Dashboard** in the Kibana sidebar
2. Click **Create dashboard**
3. Add visualizations using the **Add** button
4. Arrange panels by dragging and repositioning
5. Resize panels by dragging panel edges
6. Apply filters or search queries as needed
7. Set the time range to display relevant data
8. Click **Save** to name and store your dashboard

### Dashboard from Saved Objects

Create a dashboard based on existing saved objects:

1. Navigate to **Management > Saved Objects**
2. Select visualizations you want to include
3. Click **Add to dashboard** from the actions menu
4. Choose to create a new dashboard
5. Arrange and configure as needed
6. Save the dashboard

## Adding and Arranging Visualizations

### Types of Dashboard Panels

Dashboards can include several types of panels:

- **Saved Visualizations**: Any visualization created in Kibana
- **Saved Searches**: Display search results with document data
- **Controls**: Interactive elements for filtering
- **Markdown**: Text panels for documentation, context, or instructions

### Adding Panels

Add content to your dashboard:

1. Click **Edit** to enter dashboard edit mode
2. Click **Add** to see available panels:
   - Visualization: Choose from saved visualizations
   - Saved Search: Choose from saved searches
   - Controls: Add interactive filters
   - Markdown: Add formatted text

3. For each selected panel, click **Add** to place it on the dashboard

### Panel Arrangement

Organize your dashboard layout:

1. **Grid Layout**: Panels snap to a grid system
2. **Drag and Drop**: Reposition panels by dragging their headers
3. **Resize**: Adjust panel dimensions by dragging edges or corners
4. **Panel Options**:
   - Click the panel header gear icon for options
   - Expand to full screen
   - Edit the visualization
   - Delete the panel

### Panel Groups

Organize related panels into groups (available in newer Kibana versions):

1. Select multiple panels by clicking while holding Shift
2. Right-click and select **Group panels**
3. The grouped panels can be:
   - Moved together
   - Resized proportionally
   - Collapsed/expanded as a unit

### Advanced Layout Techniques

Fine-tune your dashboard layout:

1. **Aspect Ratios**: Maintain consistent panel sizes
2. **Negative Space**: Use empty areas to create visual hierarchy
3. **Color Coordination**: Apply consistent color schemes across panels
4. **Clear Hierarchy**: Place important metrics at the top/left

## Dashboard Controls and Filters

### Filter Types

Apply filters to focus your dashboard on specific data:

1. **Global Filters**: Applied to all dashboard panels
2. **Panel-Specific Filters**: Applied only to individual visualizations
3. **Search Bar**: Free-text search across all panels

### Adding Filters

Add filters to your dashboard:

1. Click **Add filter** in the filter bar
2. Select field, operator, and value:
   - Field: The data field to filter on
   - Operator: equals, contains, is between, exists, etc.
   - Value: The specific value(s) to filter for

3. Optional: Set a label for the filter
4. Click **Save** to apply the filter

Example filter configuration:
```
Field: response.keyword
Operator: is
Value: 404
Label: Error Responses
```

### Input Controls

Add interactive filter controls (available in newer Kibana versions):

1. In edit mode, click **Add > Controls**
2. Select control type:
   - **Options List**: Dropdown with predefined values
   - **Range Slider**: Select numeric ranges
   - **Date Picker**: Select date ranges

3. Configure the control:
   - **Options List**:
     - Field: The field to filter
     - Size: Number of options to display
     - Order: How to sort options

Example Options List control:
```
Control Type: Options List
Field: host.keyword
Label: Hostname
Size: 10
Order: Alphabetical
```

### Filter Interactions

Interact with dashboard filters:

1. **Enable/Disable**: Toggle filters on and off
2. **Edit**: Modify existing filters
3. **Pin**: Keep filters when navigating away
4. **Invert**: Reverse the filter logic (exclude instead of include)
5. **Temporary Filters**: Apply filters that don't persist when saved

### Global vs. Panel-Specific Filtering

Understand filter scope:

1. **Global Filters**: Applied from the dashboard filter bar
2. **Visualization Filters**: Set within individual visualizations
3. **Filter Inheritance**: Panel-specific filters don't affect other panels

## Time Range Controls

### Time Range Selector

Configure the time window for your dashboard:

1. Use the time range selector in the upper right
2. Choose from preset options:
   - Quick selectors (Last 15 minutes, Last 24 hours, etc.)
   - Relative options (minutes, hours, days, etc. ago)
   - Absolute date/time range

3. Set a custom time range:
   - Start and end dates/times
   - Relative to now options

### Auto-Refresh

Set your dashboard to update automatically:

1. Click the auto-refresh selector next to the time range
2. Choose a refresh interval:
   - Off (manual refresh only)
   - 5 seconds, 10 seconds, 30 seconds
   - 1 minute, 5 minutes, 15 minutes, etc.

3. The dashboard will refresh with the latest data at the specified interval

### Time-Based Controls

Create custom time controls:

1. Add a dashboard time filter
2. Configure minimum and maximum time bounds
3. Use relative or absolute times
4. Add animation controls for time series exploration

## Dashboard Options

### Display Options

Configure overall dashboard appearance:

1. Click **Options** in edit mode
2. Set dashboard display settings:
   - **Dark/Light Theme**: Change color scheme
   - **Text Size**: Adjust panel title font size
   - **Hide/Show Features**:
     - Filter bar
     - Query input
     - Time picker
     - Panel titles

### Color Themes

Apply consistent color schemes:

1. Navigate to **Management > Advanced Settings**
2. Search for "visualization:colorMapping"
3. Define custom color mappings for data values

Example color mapping:
```json
{
  "error": "#E7664C",  
  "warn": "#F9BA8F",
  "info": "#7DE2D1"
}
```

### Data Settings

Configure how data is loaded and processed:

1. **Auto-Refresh**:
   - Set refresh intervals
   - Configure refresh notifications

2. **Query Settings**:
   - Default query language (KQL or Lucene)
   - Auto-refresh on filter changes

### Accessibility Options

Make dashboards more accessible:

1. Configure high-contrast mode
2. Set text size scaling
3. Ensure keyboard navigation support
4. Use colorblind-friendly palettes

## Saving and Sharing

### Saving Dashboards

Save your dashboard configuration:

1. Click **Save** in the top right
2. Provide a title and optional description
3. Choose to save as a new dashboard or update existing
4. Optionally store the current time range with the dashboard

### URL Sharing

Share dashboard access via URLs:

1. Click **Share** in the top navigation
2. Choose a sharing option:
   - **Short URL**: Generated shortened link
   - **Snapshot**: URL to an image version
   - **Embed Code**: HTML for embedding in websites

3. Configure URL options:
   - Include current time range
   - Preserve current filters
   - Set auto-refresh options

### Export Options

Export dashboards for backup or transfer:

1. Navigate to **Management > Saved Objects**
2. Select the dashboard(s) to export
3. Click **Export**
4. Save the resulting JSON file

To import:
1. Navigate to **Management > Saved Objects**
2. Click **Import**
3. Select the JSON file to import
4. Resolve any conflicts

### Reporting

Generate PDF or PNG reports:

1. Open the dashboard to report
2. Click **Share > PDF Reports**
3. Configure report options:
   - Paper size and orientation
   - Header and footer text
   - Include time range
   - Preserve filters

4. Generate the report or schedule periodic delivery

## Drilldowns and Actions

### Dashboard Drilldowns

Create interactive workflows between dashboards:

1. In edit mode, select a visualization
2. Click **Actions > Create drilldown**
3. Choose drilldown type:
   - **Dashboard**: Navigate to another dashboard
   - **URL**: Open external links
   - **Custom**: Execute custom actions

4. Configure the trigger:
   - On click
   - On select
   - On hovering

Example dashboard drilldown configuration:
```
Type: Dashboard Drilldown
Target Dashboard: Host Details
Trigger: On click
Pass Filters: Yes
Pass Time Range: Yes
```

### URL Drilldowns

Link to external resources:

1. Create a URL drilldown
2. Configure the URL template with variables:
   - `{{value}}`: Selected value
   - `{{key}}`: Selected field
   - `{{timestamp}}`: Current time

Example URL drilldown:
```
URL Template: https://docs.example.com/errors/{{value}}
Open in: New tab
```

### Custom Actions

Define custom behaviors:

1. Create a custom action
2. Define the trigger event
3. Configure what happens on trigger
4. Set conditions for when the action is available

## Dashboard Best Practices

### Design Principles

Follow these guidelines for effective dashboards:

1. **Purpose-Driven Design**:
   - Define the dashboard's specific purpose
   - Include only relevant visualizations
   - Create separate dashboards for different use cases

2. **Information Hierarchy**:
   - Most important metrics at the top left
   - Group related information together
   - Use size to indicate importance

3. **Visual Clarity**:
   - Avoid cluttered layouts
   - Maintain consistent formatting
   - Use enough white space

4. **Context and Documentation**:
   - Add markdown panels for explanations
   - Include data source information
   - Document known issues or caveats

### Performance Optimization

Optimize dashboard load times:

1. **Limit Visualization Count**:
   - Keep dashboards to 15-20 panels maximum
   - Split complex dashboards into multiple simpler ones

2. **Optimize Queries**:
   - Use efficient aggregations
   - Limit the number of terms/buckets
   - Apply appropriate index patterns

3. **Date Range Management**:
   - Avoid excessively large time ranges
   - Use appropriate time intervals
   - Implement data rollups for historical data

4. **Filter Utilization**:
   - Apply filters at the dashboard level
   - Use query filters before aggregation filters
   - Leverage index patterns with appropriate mappings

### Organizational Patterns

Structure your dashboards logically:

1. **Dashboard Hierarchy**:
   - Overview dashboards with high-level metrics
   - Detail dashboards with specific focus areas
   - Drill-down paths between levels

2. **Naming Conventions**:
   - Consistent, descriptive dashboard names
   - Include purpose and scope in title
   - Use prefixes for categorization

3. **Tagging System**:
   - Apply tags for easy discovery
   - Tag by purpose, team, or application
   - Maintain tag consistency

## Real-World Dashboard Examples

### Log Monitoring Dashboard

Purpose: Monitor application logs and error trends

Components:
1. **Top-Level Metrics**:
   - Total log count
   - Error rate percentage
   - Average response time

2. **Time-Series Trends**:
   - Log volume by level (error, warn, info)
   - Error count by application
   - Response time trend

3. **Error Analysis**:
   - Top error messages (table)
   - Error distribution by service (pie chart)
   - Geographic distribution of errors (map)

4. **Interactive Elements**:
   - Host filter dropdown
   - Service selector
   - Error type filter

### Infrastructure Monitoring Dashboard

Purpose: Monitor server health and resource utilization

Components:
1. **System Metrics**:
   - CPU utilization (gauge)
   - Memory usage (gauge)
   - Disk space (gauge)
   - Network throughput (time series)

2. **Host Overview**:
   - Host status (metric)
   - Host count by status (pie chart)
   - Top hosts by resource usage (bar chart)

3. **Detailed Metrics**:
   - Process count (metric)
   - Load average (time series)
   - IO wait (time series)
   - Swap usage (time series)

4. **Heat Maps**:
   - CPU usage by host (heat map)
   - Memory usage by host (heat map)

### Business Analytics Dashboard

Purpose: Track business KPIs and metrics

Components:
1. **Revenue Metrics**:
   - Total revenue (metric)
   - Revenue trend (time series)
   - Revenue by product (bar chart)

2. **Customer Metrics**:
   - New customers (metric)
   - Customer retention (gauge)
   - Geographic distribution (map)

3. **Product Performance**:
   - Top products (table)
   - Product category breakdown (pie chart)
   - Stock levels (bar chart)

4. **Marketing Effectiveness**:
   - Conversion rates (gauge)
   - Traffic sources (pie chart)
   - Campaign performance (bar chart)

## Conclusion

Kibana dashboards are powerful tools for visualizing and exploring your Elasticsearch data. By combining multiple visualizations, implementing interactive controls, and following design best practices, you can create intuitive, insightful dashboards that help stakeholders understand complex data at a glance.

In the next chapter, we'll explore Kibana's Canvas and Lens features, which provide additional visualization capabilities beyond standard dashboards.