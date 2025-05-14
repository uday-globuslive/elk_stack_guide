# Kibana Fundamentals

## Introduction to Kibana

Kibana is an open-source data visualization and exploration platform designed to work with Elasticsearch. As the "K" in the ELK Stack, Kibana provides a user-friendly interface for exploring, visualizing, and analyzing data stored in Elasticsearch indices.

### Core Functions of Kibana

1. **Data Exploration**: Search and filter data with a powerful query language
2. **Data Visualization**: Create charts, graphs, maps, and other visualizations
3. **Dashboard Creation**: Combine visualizations into comprehensive dashboards
4. **Management**: Configure Elasticsearch indices, spaces, users, and more
5. **Monitoring**: Track the health and performance of the Elastic Stack
6. **Advanced Analytics**: Perform anomaly detection, forecasting, and machine learning

### Key Features

- **Intuitive Interface**: User-friendly UI for exploring and visualizing data
- **Real-time Analysis**: See results immediately as you query and filter
- **Responsive Design**: Dashboards adapt to different screen sizes
- **Powerful Query Language**: Uses Elasticsearch's query DSL and Kibana Query Language (KQL)
- **Extensibility**: Plugin architecture allows custom extensions
- **Security Integration**: Role-based access control and authentication options

## Kibana Architecture

### Component Overview

Kibana consists of several core components:

1. **Web Server**: Node.js-based server that hosts the Kibana web application
2. **Backend Services**: Handle saved objects, user management, and other functionality
3. **Frontend Application**: Browser-based UI built with React
4. **Plugin System**: Extensible architecture for adding features

### Communication Flow

![Kibana Architecture](https://i.imgur.com/9lxDmIY.png)

1. Users access Kibana through a web browser
2. Kibana's web server serves the frontend application
3. The frontend application communicates with Kibana's backend services
4. Kibana's backend services communicate with Elasticsearch
5. Results are returned to the frontend for display

### Storage in Kibana

Kibana stores its configuration data in Elasticsearch indices:

- **Saved Objects**: Index patterns, visualizations, dashboards, etc.
- **State**: System information, plugin data, etc.
- **Space Data**: Space-specific configurations and objects
- **Security Data**: Role mappings, user information (when security is enabled)

## Installing Kibana

### System Requirements

Minimum requirements:
- 2GB RAM (4GB+ recommended)
- Modern web browser (Chrome, Firefox, Safari, or Edge)
- Connection to Elasticsearch
- Node.js (bundled with Kibana)

### Installation Methods

#### Package Repositories

**Debian/Ubuntu**:
```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
sudo apt-get install apt-transport-https
echo "deb https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-8.x.list
sudo apt-get update && sudo apt-get install kibana
```

**RHEL/CentOS**:
```bash
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
cat << EOF | sudo tee /etc/yum.repos.d/elasticsearch.repo
[elasticsearch]
name=Elasticsearch repository for 8.x packages
baseurl=https://artifacts.elastic.co/packages/8.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF
sudo yum install kibana
```

#### Archive Installation

Download and extract the archive:
```bash
curl -L -O https://artifacts.elastic.co/downloads/kibana/kibana-8.8.0-linux-x86_64.tar.gz
tar -xzvf kibana-8.8.0-linux-x86_64.tar.gz
cd kibana-8.8.0
```

#### Docker

```bash
docker pull docker.elastic.co/kibana/kibana:8.8.0
docker run --link elasticsearch:elasticsearch -p 5601:5601 docker.elastic.co/kibana/kibana:8.8.0
```

### Directory Structure

After installation, you'll find these important directories:

- `bin/`: Executable files including the `kibana` command
- `config/`: Configuration files including `kibana.yml`
- `data/`: Data files
- `logs/`: Log files
- `plugins/`: Custom plugins

## Basic Configuration

### Configuration File

The main configuration file is `kibana.yml`, located in the `config` directory.

Key configuration options:

```yaml
# Server settings
server.host: "0.0.0.0"  # Listen on all interfaces
server.port: 5601  # Default port

# Elasticsearch connection
elasticsearch.hosts: ["http://localhost:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "password"

# Kibana-specific settings
kibana.index: ".kibana"  # Index for storing Kibana configuration
kibana.defaultAppId: "home"  # Default page

# Logging
logging.dest: stdout  # Log to standard output
logging.level: "info"  # Log level

# Path settings
path.data: /var/lib/kibana  # Data storage path
```

### Starting Kibana

#### Systemd Service (Package Install)

```bash
sudo systemctl start kibana
sudo systemctl enable kibana
```

#### Direct Start (Archive Install)

```bash
bin/kibana
```

#### Docker

```bash
docker run --name kibana \
  --link elasticsearch:elasticsearch \
  -p 5601:5601 \
  -e "ELASTICSEARCH_HOSTS=http://elasticsearch:9200" \
  docker.elastic.co/kibana/kibana:8.8.0
```

### Verifying Installation

Open a web browser and navigate to `http://localhost:5601`. You should see the Kibana welcome page.

## Kibana User Interface

### Home Page

The home page provides:
- Quick links to common tasks
- Documentation and tutorials
- Recently accessed items
- Status information

### Main Navigation

The main menu includes:
- **Analytics**: Discover, Dashboard, Canvas, Maps, ML
- **Observability**: APM, Logs, Metrics, Uptime
- **Security**: SIEM, Endpoint
- **Management**: Stack Management, Dev Tools

### Discover

The Discover page allows you to explore your data:
- Select an index pattern
- Search with KQL or Lucene query syntax
- View document fields and values
- Apply filters and time ranges
- Save searches for later use

![Kibana Discover](https://i.imgur.com/MFFyKT5.png)

### Dashboards

The Dashboard page lets you create dashboards from visualizations:
- Add visualizations to a dashboard
- Arrange visualizations in a grid
- Apply filters that affect all visualizations
- Customize time ranges and refresh intervals
- Save and share dashboards

![Kibana Dashboard](https://i.imgur.com/xgFJLpV.png)

### Visualize

The Visualize page is for creating visualizations:
- Choose from multiple visualization types
- Configure data sources and metrics
- Customize appearance and behavior
- Save visualizations for use in dashboards

### Canvas

Canvas allows you to create custom presentations:
- Design free-form layouts
- Add visualizations, images, and text
- Create interactive elements
- Build infographic-style displays

### Maps

The Maps application enables geospatial visualization:
- Plot points, lines, and shapes on maps
- Use multiple map layers
- Color and size features based on metrics
- Integrate with other Kibana visualizations

### Dev Tools

Dev Tools provides developer utilities:
- **Console**: Interface for Elasticsearch API requests
- **Search Profiler**: Analyze search performance
- **Grok Debugger**: Test Grok patterns
- **Painless Lab**: Test Painless scripts

### Stack Management

Stack Management offers configuration options:
- **Index Patterns**: Define patterns for accessing Elasticsearch indices
- **Saved Objects**: Manage visualizations, dashboards, etc.
- **Advanced Settings**: Configure Kibana behaviors
- **Spaces**: Create and manage spaces
- **Security**: Configure users, roles, and permissions (if X-Pack security is enabled)

## Working with Index Patterns

### What Are Index Patterns?

Index patterns define which Elasticsearch indices Kibana uses for analysis and visualization. They can match exact index names or use wildcards to match multiple indices.

### Creating an Index Pattern

1. Navigate to Stack Management > Index Patterns
2. Click "Create index pattern"
3. Enter a pattern (e.g., `logs-*` to match all indices that start with "logs-")
4. Select a timestamp field if your data has time-based information
5. Click "Create index pattern"

### Configuring Field Properties

After creating an index pattern, you can configure how fields are displayed and used:

1. Go to the index pattern's detail page
2. Configure field-specific options:
   - **Format**: How the field is displayed (date format, URL, etc.)
   - **Popularity**: Fields with higher popularity appear higher in lists
   - **Visibility**: Whether fields appear in field lists

### Runtime Fields

Create computed fields without reindexing:

1. In your index pattern, go to the Runtime Fields tab
2. Click "Add runtime field"
3. Define the field with a Painless script:

```painless
emit(doc['bytes'].value / 1024)
```

## Searching and Querying Data

### Kibana Query Language (KQL)

KQL is a simple, yet powerful query language for filtering data:

- **Field matching**: `field:value`
- **Boolean logic**: `and`, `or`, `not`
- **Range queries**: `field > value`, `field <= value`
- **Wildcard searches**: `field:value*`
- **Nested field access**: `parent.child:value`

Examples:
```
# Match documents where status is "error"
status:error

# Match documents where status is "error" AND service is "api"
status:error and service:api

# Match documents where status is "error" OR status is "warning"
status:(error or warning)

# Match documents where bytes is greater than 1000
bytes > 1000

# Match documents where path starts with "/api"
path:/api*
```

### Lucene Query Syntax

Kibana also supports Elasticsearch's Lucene query syntax:

- **Field matching**: `field:value`
- **Boolean operators**: `AND`, `OR`, `NOT`
- **Grouping**: `(term1 OR term2) AND term3`
- **Range queries**: `field:[value1 TO value2]`
- **Wildcard searches**: `field:val*`

Examples:
```
# Match documents where status is "error"
status:error

# Match documents where status is "error" AND service is "api"
status:error AND service:api

# Match documents where bytes is between 1000 and 2000
bytes:[1000 TO 2000]

# Match documents where the message field contains "error" or "exception"
message:(error OR exception)
```

### Filtering

Filters can be applied in several ways:

1. Click the add filter button (+) and define your filter
2. Click on values in the document table or visualizations
3. Use the KQL bar to enter a query

Filters can be:
- **Enabled or disabled** without removing them
- **Pinned** to keep them active across Kibana sessions
- **Inverted** to negate their effect
- **Edited** to change their parameters
- **Deleted** when no longer needed

### Time Range Selection

For time-based data, select a time range:

1. Use the time picker in the top-right corner
2. Choose from preset ranges (Last 15 minutes, Last 24 hours, etc.)
3. Define a custom range with absolute or relative times
4. Configure refresh intervals for auto-updating

## Creating Visualizations

### Visualization Types

Kibana offers many visualization types:

- **Aggregation Based**:
  - Line, Area, and Bar charts
  - Pie charts
  - Data Table
  - Metric
  - Gauge
  - Heat Map
  - Tag Cloud

- **Coordinate Map Based**:
  - Coordinate Map
  - Region Map
  - Heat Map

- **Time Series Visual Builder (TSVB)**:
  - Time Series
  - Metric
  - Top N
  - Gauge
  - Markdown

- **Others**:
  - Vega & Vega-Lite
  - Timelion
  - Controls

### Creating a Basic Visualization

To create a bar chart visualization:

1. Go to Visualize and click "Create visualization"
2. Select "Bar" as the visualization type
3. Choose an index pattern
4. Configure the Y-axis with an aggregation (e.g., Count, Average)
5. Configure the X-axis with a bucket (e.g., Date Histogram, Terms)
6. Click "Update" to see your visualization
7. Customize appearance in the "Options" tab
8. Save your visualization with a title and description

### Metrics and Buckets

Visualizations are built using metrics and buckets:

- **Metrics**: Calculations performed on document fields (count, average, sum, etc.)
- **Buckets**: Ways to segment or group the data (by date, by term, by range, etc.)

Example configurations:

**Line chart by time**:
- Metric: Average of 'response_time' field
- Bucket: Date Histogram on '@timestamp' field with 1-hour interval

**Pie chart by status**:
- Metric: Count of documents
- Bucket: Terms aggregation on 'status' field

### Advanced Visualization Features

**Sub-buckets**:
Add multiple bucket levels to create more complex visualizations:
1. Add a "Date Histogram" bucket for the X-axis
2. Add a "Terms" sub-bucket to split the series
3. This creates a multi-series chart grouped by terms

**Sibling Pipelines**:
Calculate metrics based on other metrics:
1. Add a "Count" metric
2. Add a "Derivative" metric that calculates the rate of change of the count
3. This shows the change rate of your documents over time

**Series Options**:
Customize individual series in a visualization:
- Change colors
- Modify chart type
- Adjust data formatting
- Add annotations

## Building Dashboards

### Creating a Dashboard

1. Go to Dashboard and click "Create dashboard"
2. Click "Add" to add visualizations
3. Select from existing visualizations or create new ones
4. Arrange visualizations by dragging and resizing
5. Save the dashboard with a title and description

### Dashboard Features

**Filters and Queries**:
- Apply filters and queries that affect all visualizations
- Create visualization-specific filters
- Save filters with the dashboard

**Time Range**:
- Set a global time range for all visualizations
- Enable "Use dates from data" to let each visualization use its own time range

**Refresh Controls**:
- Set automatic refresh intervals (5s, 10s, 30s, etc.)
- Manually refresh data with the refresh button

**View Options**:
- Toggle fullscreen mode
- Adjust color theme (light/dark)
- Edit visualization display settings

### Dashboard Drilldowns

Create interactive dashboards with drilldowns:

1. Edit a dashboard
2. Select a visualization
3. Click "Create drilldown"
4. Choose "Dashboard drilldown"
5. Configure the target dashboard and how to pass context
6. Save the drilldown

Now when users click on the visualization, they'll navigate to the target dashboard with context maintained.

### Sharing Dashboards

Share dashboards in several ways:

- **Direct Link**: Copy the URL to share
- **Saved Object Export**: Export the dashboard definition for import in another Kibana instance
- **PDF/PNG Export**: Generate a static image or PDF document
- **Embedded Iframe**: Embed the dashboard in another web page
- **CSV Export**: Export visualization data as CSV

## Lens

Lens is Kibana's drag-and-drop interface for creating visualizations.

### Getting Started with Lens

1. Go to Visualize and click "Create visualization"
2. Select "Lens" as the visualization type
3. Choose an index pattern
4. Drag fields from the field list to the visualization builder
5. Lens automatically suggests appropriate visualization types
6. Switch between visualization types using the chart type selector
7. Save your visualization when finished

### Lens Features

**Automatic Suggestions**:
Lens suggests visualizations based on the data you're working with.

**Formula Support**:
Write simple formulas to transform your data:
```
count() / max(bytes)
```

**Layer Support**:
Add multiple data layers to a single visualization.

**Dimension Configuration**:
Configure how dimensions are displayed:
- Field formatting
- Custom labels
- Value scaling
- Sort order

## Canvas

Canvas is a presentation tool for creating pixel-perfect visualizations.

### Canvas Basics

1. Go to Canvas and click "Create workpad"
2. Set the workpad size and background
3. Add elements using the "Add" button
4. Configure elements with data sources and display options
5. Arrange elements using drag, resize, and alignment tools
6. Save your workpad

### Canvas Elements

Canvas offers various elements:

- **Visualizations**: Connect to Elasticsearch data
- **Shapes**: Add geometric shapes
- **Text**: Add text with markdown support
- **Images**: Add static images or image URLs
- **Containers**: Group elements together

### Canvas Expressions

Canvas uses an expression language for data processing:

```
elasticsearch
| essql query="SELECT category, sum(sales) as total_sales FROM sales GROUP BY category"
| pointseries x="category" y="total_sales"
| plot
```

This pipeline:
1. Fetches data from Elasticsearch
2. Uses Elasticsearch SQL to query sales data
3. Formats the data as a point series
4. Plots the data as a chart

### Canvas Presentations

Create interactive presentations with Canvas:

1. Add multiple pages to your workpad
2. Create transitions between pages
3. Add interactive elements like filters and buttons
4. Present in fullscreen mode
5. Auto-play presentations with auto-refresh

## Kibana Spaces

Spaces let you organize dashboards and visualizations for different teams or purposes.

### Creating a Space

1. Go to Stack Management > Spaces
2. Click "Create space"
3. Provide a name, description, and optional icon
4. Configure which features are available in the space
5. Click "Create space"

### Using Spaces

Switch between spaces using the space selector in the main navigation.

Each space has:
- Its own set of saved objects (dashboards, visualizations, etc.)
- Feature access controls
- Isolated URLs (e.g., `/s/marketing/app/dashboards`)

### Sharing Objects Between Spaces

Copy objects from one space to another:

1. Go to Stack Management > Saved Objects
2. Select the objects you want to share
3. Click "Copy to space"
4. Select the target space(s)
5. Resolve any conflicts
6. Click "Copy" to complete the process

## Kibana Security (X-Pack)

X-Pack security provides authentication and authorization features.

### Authentication Methods

Kibana supports multiple authentication methods:

- **Basic Authentication**: Username and password
- **SAML**: Single sign-on with identity providers
- **OpenID Connect**: Integration with OAuth providers
- **PKI**: Client certificate authentication
- **Kerberos**: Integrated Windows authentication
- **Token Authentication**: API tokens for services

### Role-Based Access Control (RBAC)

Define permissions with roles:

1. Go to Stack Management > Security > Roles
2. Create a role with specific privileges:
   - **Elasticsearch privileges**: Index-level and cluster-level permissions
   - **Kibana privileges**: Feature access and saved object permissions
   - **Space privileges**: Per-space permissions

### User Management

Create and manage users:

1. Go to Stack Management > Security > Users
2. Click "Create user"
3. Provide username, password, and full name
4. Assign roles to the user
5. Click "Create user"

### API Keys

Generate API keys for programmatic access:

1. Go to Stack Management > Security > API Keys
2. Click "Create API key"
3. Provide a name and expiration
4. Assign privileges
5. Click "Create"

## Kibana Plugins

Extend Kibana functionality with plugins.

### Official Plugins

Elastic provides many official plugins:

- **Maps**: Geospatial visualization
- **Canvas**: Free-form layouts
- **Graph**: Relationship visualization
- **ML**: Machine learning
- **Reporting**: PDF and CSV export
- **Alerting**: Create and manage alerts

### Third-Party Plugins

You can install third-party plugins using:

```bash
bin/kibana-plugin install [plugin_url]
```

### Developing Custom Plugins

Create your own plugins to extend Kibana:

1. Generate a plugin skeleton using the Kibana plugin generator
2. Implement your functionality
3. Build and package your plugin
4. Install it in your Kibana instance

## Monitoring and Alerting

### Stack Monitoring

Monitor your Elastic Stack with built-in tools:

1. Enable Stack Monitoring in `kibana.yml`:
```yaml
monitoring.ui.enabled: true
```
2. Go to Stack Monitoring
3. View health and performance metrics for Elasticsearch, Kibana, Logstash, and Beats
4. Drill down into specific components for detailed metrics

### Alerting

Create alerts based on data conditions:

1. Go to Stack Management > Alerting > Rules
2. Click "Create rule"
3. Select the rule type (e.g., index threshold, anomaly)
4. Configure conditions and actions
5. Set check interval and notification settings
6. Save the rule

Alert examples:
- Notify when error count exceeds threshold
- Alert on abnormal system metrics
- Trigger when documents matching specific criteria appear

### Watcher

Use Elasticsearch Watcher for more advanced alerting:

1. Go to Stack Management > Watcher
2. Create a new watch with:
   - **Trigger**: When to evaluate (e.g., schedule)
   - **Input**: What data to fetch
   - **Condition**: When to alert
   - **Actions**: What to do when condition is met
3. Test and save your watch

## Conclusion

This chapter covered the fundamentals of Kibana, including its architecture, installation, configuration, and core features. Kibana provides a powerful yet user-friendly interface for exploring, visualizing, and analyzing data stored in Elasticsearch.

In the next chapters, we'll dive deeper into specific aspects of Kibana, including advanced visualizations, Canvas workpads, Kibana scripting, and best practices for production deployments.