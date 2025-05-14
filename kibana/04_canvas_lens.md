# Canvas and Lens

This chapter explores Kibana's Canvas and Lens features, which provide additional, powerful ways to visualize and present your Elasticsearch data beyond standard visualizations and dashboards.

## Table of Contents
- [Kibana Canvas Overview](#kibana-canvas-overview)
- [Getting Started with Canvas](#getting-started-with-canvas)
- [Canvas Elements and Workpads](#canvas-elements-and-workpads)
- [Working with Data in Canvas](#working-with-data-in-canvas)
- [Advanced Canvas Features](#advanced-canvas-features)
- [Canvas Best Practices](#canvas-best-practices)
- [Kibana Lens Overview](#kibana-lens-overview)
- [Creating Visualizations with Lens](#creating-visualizations-with-lens)
- [Advanced Lens Features](#advanced-lens-features)
- [Comparing Canvas and Lens](#comparing-canvas-and-lens)
- [Integration with Dashboards](#integration-with-dashboards)

## Kibana Canvas Overview

Canvas is a data presentation and visualization tool within Kibana that allows you to create custom, pixel-perfect, presentation-ready visualizations that can be shared and updated in real-time.

### Key Features

- **Free-form Layout**: Precise positioning of elements without grid constraints
- **Custom Design**: Highly customizable visual elements with CSS styling
- **Dynamic Data**: Live connections to Elasticsearch and other data sources
- **Expressions**: Powerful expression language for data transformations
- **Interactive Elements**: Add user inputs and controls
- **Presentation Mode**: Full-screen presentation capabilities
- **Shareable**: Export to PDF or share URLs

### Canvas vs. Traditional Dashboards

| Feature | Canvas | Dashboards |
|---------|--------|------------|
| Layout | Free-form placement | Grid-based |
| Design customization | Extensive | Limited |
| Data sources | Multiple, including external | Primarily Elasticsearch |
| Interactivity | Custom interactions | Standard filters |
| Purpose | Presentations, infographics, status boards | Data analysis, monitoring |
| Learning curve | Steeper | More straightforward |

## Getting Started with Canvas

### Accessing Canvas

1. In Kibana, navigate to **Canvas** in the left navigation menu
2. Click **Create workpad** to start a new Canvas project
3. Select a template or start with a blank workpad

### Canvas Interface

The Canvas interface consists of:

1. **Toolbar**: Top menu with workpad actions and settings
2. **Element Menu**: Library of elements you can add
3. **Workpad**: Canvas area where you build your presentation
4. **Element Settings**: Panel for configuring selected elements
5. **Expression Editor**: Advanced expression editing panel

### Creating Your First Workpad

1. Start with a blank workpad or template
2. Set workpad properties:
   - Name: Give your workpad a descriptive name
   - Size: Set dimensions (e.g., 1920×1080 for presentations)
   - Background: Choose color or image

3. Add your first element:
   - Click **Add element**
   - Choose an element type (e.g., Markdown, Metric, Chart)
   - Position and resize on the canvas

4. Configure the element:
   - Set data source
   - Style the element
   - Add interactivity if needed

5. Save your workpad:
   - Click **Save** in the top menu
   - Provide a name and optional description

## Canvas Elements and Workpads

### Core Element Types

Canvas provides various element types for different visualization needs:

- **Charts and Graphs**:
  - Line, bar, area charts
  - Pie and donut charts
  - Coordinate maps

- **Metrics and KPIs**:
  - Metric display
  - Gauge
  - Progress indicators

- **Tables and Lists**:
  - Data tables
  - Data lists
  - Timelion series

- **Text and Markdown**:
  - Markdown text
  - Text elements
  - Formatted text

- **Shapes and Images**:
  - Basic shapes
  - Image uploads
  - Icons

- **Controls**:
  - Dropdown inputs
  - Time filter
  - Button controls

### Element Customization

Customize elements to match your design requirements:

1. **Visual Styling**:
   - Border properties (width, color, style)
   - Background colors and gradients
   - Opacity and blending
   - Padding and margin
   - Shadows and effects

2. **Typography**:
   - Font family, size, and weight
   - Text color and alignment
   - Line height and spacing
   - Text transformations

3. **Position and Layout**:
   - Absolute positioning with x/y coordinates
   - Element rotation
   - Z-index (layering)
   - Alignment tools

### Workpad Management

Organize your Canvas workpads:

1. **Pages**: Create multi-page presentations
   - Add page: Create a new blank page
   - Duplicate page: Copy existing page
   - Reorder pages: Drag to reposition

2. **Workpad Settings**:
   - Dimensions and resolution
   - Background colors or images
   - Auto-refresh settings
   - Time range settings

3. **Version Control**:
   - Autosave functionality
   - Restore previous versions
   - Export/import workpads

## Working with Data in Canvas

### Data Sources

Canvas supports multiple data sources:

1. **Elasticsearch**: Query your Elasticsearch indices
   - Index pattern selection
   - Lucene or KQL query language
   - Aggregation framework support

2. **Timelion**: Time series data with expressions
   - Mathematical functions
   - Moving averages and trends
   - Multiple series comparison

3. **Static Data**: Hardcoded values for prototyping
   - CSV data input
   - JSON objects
   - Manual data entry

4. **Demo Data**: Sample datasets for learning
   - Web logs
   - Flight data
   - eCommerce orders

### Working with Elasticsearch Data

Connect an element to Elasticsearch:

1. Select an element
2. Under **Data**, choose **Elasticsearch SQL Query**
3. Select your index pattern
4. Write an SQL query:
   ```sql
   SELECT response.keyword, COUNT(*) as count 
   FROM "logs-*" 
   WHERE timestamp > NOW() - INTERVAL 24 HOUR 
   GROUP BY response.keyword 
   ORDER BY count DESC 
   LIMIT 5
   ```
5. Apply the query and configure how data maps to the element

### Canvas Expressions

Use Canvas's expression language for data manipulation:

1. **Basic Expression Structure**:
   ```
   elasticsearch | essql query="SELECT * FROM my_index" | 
   pointseries x="timestamp" y="value" | 
   plot
   ```

2. **Expression Components**:
   - Functions: Processing operations
   - Arguments: Parameters for functions
   - Pipes: Connect functions in a sequence

3. **Common Functions**:
   - `elasticsearch`: Query Elasticsearch
   - `math`: Perform calculations
   - `filters`: Apply data filters
   - `sort`: Order data
   - `mapColumn`: Transform data columns
   - `render`: Output visualization

4. **Example Expression**:
   ```
   elasticsearch 
   | essql query="SELECT timestamp, value FROM my_index"
   | math "value / 1000" as="kilovalue"
   | mapColumn timestamp="time" kilovalue="count"
   | pointseries x="time" y="count"
   | plot defaultStyle={seriesStyle lines=3}
   ```

### Data Transformation in Canvas

Manipulate data before visualization:

1. **Filtering**:
   ```
   elasticsearch | essql query="..." | 
   filters expression="count > 100"
   ```

2. **Mathematical Operations**:
   ```
   elasticsearch | essql query="..." | 
   math "bytes / (1024*1024)" as="megabytes"
   ```

3. **Column Renaming**:
   ```
   elasticsearch | essql query="..." | 
   mapColumn timestamp="time" value="count"
   ```

4. **Data Pivoting**:
   ```
   elasticsearch | essql query="..." | 
   pivot rows="browser" columns="country" values="count"
   ```

## Advanced Canvas Features

### Animation and Transitions

Add movement to your presentations:

1. **Element Transitions**:
   - Fade-in/fade-out
   - Slide and scale
   - Rotation effects

2. **Dynamic Updates**:
   - Auto-refreshing data
   - Progress animations
   - Animated charts

3. **Page Transitions**:
   - Slide transitions
   - Fade transitions
   - Custom transition effects

### Interactivity

Create interactive Canvas workpads:

1. **User Inputs**:
   - Dropdown selectors
   - Text inputs
   - Button elements

2. **Element Actions**:
   - Click behaviors
   - Hover effects
   - Filter application

3. **Variables and State**:
   - Global variables
   - Local element state
   - Data-driven variables

Example of an interactive filter:
```
dropdown
| var name="selectedHost" 
| elasticsearch
| essql query="SELECT DISTINCT host.keyword FROM logs-*"
| render
```

Then use the variable in another element:
```
elasticsearch
| essql query="SELECT * FROM logs-* WHERE host.keyword = '{{selectedHost}}'"
| render
```

### Scripted Elements

Extend Canvas with custom JavaScript:

1. **Custom Functions**:
   - Define reusable functions
   - Process data with JavaScript
   - Create custom visualizations

2. **API Integration**:
   - Connect to external APIs
   - Fetch and process JSON data
   - Integrate third-party services

3. **Advanced Interactions**:
   - Custom event handling
   - Complex state management
   - Conditional logic

Example of a custom JavaScript function:
```javascript
filters 
| custom
  fn={
    return context.rows.map(row => {
      return {
        ...row,
        upper_text: row.text.toUpperCase()
      };
    });
  }
| render
```

## Canvas Best Practices

### Design Guidelines

Create effective Canvas presentations:

1. **Visual Hierarchy**:
   - Emphasize important information
   - Group related elements
   - Use size and color for importance

2. **Consistency**:
   - Maintain consistent styling
   - Use a limited color palette
   - Establish a grid system

3. **White Space**:
   - Don't overcrowd the canvas
   - Allow elements to breathe
   - Group related elements with spacing

4. **Typography**:
   - Use readable fonts
   - Establish a type hierarchy
   - Limit to 2-3 font families

### Performance Optimization

Ensure smooth Canvas performance:

1. **Data Efficiency**:
   - Limit data volume
   - Use appropriate time ranges
   - Apply filters early in pipelines

2. **Element Count**:
   - Keep element count reasonable
   - Combine similar elements when possible
   - Remove unnecessary elements

3. **Expression Optimization**:
   - Simplify complex expressions
   - Break into multiple elements when needed
   - Cache results when appropriate

### Common Use Cases

Effective applications for Canvas:

1. **Executive Dashboards**:
   - Clean, minimalist design
   - Focus on key metrics
   - Custom branding

2. **Status Displays**:
   - Large, clear indicators
   - Real-time updates
   - Alert highlighting

3. **Data Stories**:
   - Multiple sequential pages
   - Narrative text elements
   - Progressive data revelations

4. **Infographics**:
   - Rich visual design
   - Custom illustrations
   - Data-driven graphics

## Kibana Lens Overview

Kibana Lens is a drag-and-drop interface for creating visualizations quickly without requiring deep knowledge of Elasticsearch aggregations or Kibana's visualization editors.

### Key Features

- **Intuitive Interface**: Drag-and-drop field selection
- **Suggestion System**: Intelligent visualization recommendations
- **Dynamic Switching**: Change visualization types while preserving settings
- **Rapid Iteration**: Quick exploration of different visualization options
- **Integration**: Seamless inclusion in Kibana dashboards

### Lens vs. Traditional Visualization Editor

| Feature | Lens | Visualization Editor |
|---------|------|----------------------|
| Learning curve | Lower | Higher |
| Flexibility | Good for common use cases | More customizable |
| Speed of creation | Very fast | More time-consuming |
| Advanced options | Limited | Comprehensive |
| Visualization types | Common types | Full range |
| Aggregations | Simplified | Complete control |

## Creating Visualizations with Lens

### Getting Started with Lens

1. In Kibana, navigate to **Visualize**
2. Click **Create visualization**
3. Select **Lens** visualization
4. Choose a data source (index pattern)

### Basic Lens Interface

The Lens interface consists of:

1. **Field List**: Available fields from your data source
2. **Visualization Pane**: Preview of the current visualization
3. **Configuration Panel**: Options for the current visualization
4. **Layer Panel**: Manage multiple data layers
5. **Suggestion Panel**: Alternative visualization suggestions

### Building a Visualization

Create visualizations by dragging and dropping fields:

1. **Select fields**:
   - Drag fields from the field list to the visualization
   - Drop on the appropriate area (X-axis, Y-axis, breakdown)

2. **Choose visualization type**:
   - Select from available chart types
   - Lens suggests appropriate types for your data

3. **Configure options**:
   - Set aggregation type (count, average, sum, etc.)
   - Adjust chart properties
   - Add multiple metrics or dimensions

4. **Apply changes**:
   - Preview updates in real-time
   - Save the visualization when complete

### Example: Creating a Bar Chart

1. Drag a categorical field (e.g., `response.keyword`) to the X-axis
2. Lens automatically creates a bar chart with count aggregation
3. Drag a numeric field (e.g., `bytes`) to the Y-axis to change the metric
4. Select "Average" as the aggregation type
5. Drag another field (e.g., `host.keyword`) to the "Break down by" area to create grouped bars
6. Adjust colors, labels, and other settings as needed

### Example: Creating a Pie Chart

1. Click the visualization type selector and choose "Pie"
2. Drag a categorical field (e.g., `geo.src`) to the "Slice by" area
3. Lens automatically creates a pie chart showing the distribution
4. Drag a numeric field to the "Size by" area to change from count to another metric
5. Adjust slice limit, other settings as needed

## Advanced Lens Features

### Multi-Dimension Visualizations

Create complex visualizations with multiple dimensions:

1. **Multiple Metrics**:
   - Add multiple Y-axis fields
   - Compare different measurements
   - Use dual axis options when available

2. **Multiple Breakdowns**:
   - Add secondary dimensions (color, size, etc.)
   - Create hierarchical visualizations
   - Show relationships between dimensions

3. **Cross-Filtering**:
   - Enable interactive filtering
   - Click elements to filter dashboard
   - Create drill-down experiences

### Formula and References

Use formulas to create calculated metrics:

1. **Math Operations**:
   - Create derived fields from existing data
   - Apply mathematical formulas
   - Calculate ratios and percentages

2. **Example Formulas**:
   - Error rate: `count(status:error) / count() * 100`
   - Conversion rate: `sum(transactions) / count(visitors) * 100`
   - Growth rate: `(current - previous) / previous * 100`

3. **Field References**:
   - Reference other fields in formulas
   - Use results from other layers
   - Create comparative metrics

### Lens Layers

Work with multiple data layers in a single visualization:

1. **Add Layer**:
   - Click "Add layer" button
   - Configure independent data query
   - Set visualization properties

2. **Layer Use Cases**:
   - Compare different time periods
   - Overlay actual vs. target values
   - Combine different data sources

3. **Layer-Specific Settings**:
   - Independent colors and styles
   - Separate data queries
   - Different aggregation methods

### Custom Palette and Styling

Customize the visual appearance:

1. **Color Palettes**:
   - Select predefined palettes
   - Create custom color schemes
   - Apply semantic coloring (e.g., green for good, red for bad)

2. **Value Formatting**:
   - Format numbers (decimal places, thousands separators)
   - Apply unit labels (%, $, etc.)
   - Custom value templates

3. **Legend Options**:
   - Position and layout
   - Truncation settings
   - Show/hide functionality

## Comparing Canvas and Lens

### When to Use Canvas

Canvas is best for:

- Presentation-ready visualizations
- Custom-designed infographics
- Executive dashboards with specific branding
- Interactive data stories
- Status boards and displays
- Pixel-perfect layouts with precise positioning

### When to Use Lens

Lens is best for:

- Quick data exploration
- Standard analytical visualizations
- Dashboard components
- Ad-hoc analysis
- Team members with limited Elasticsearch knowledge
- Rapid visualization iteration

### Combining Canvas and Lens

Leverage both tools together:

1. **Lens for Exploration**:
   - Use Lens to quickly explore data
   - Identify key visualizations
   - Discover important patterns

2. **Canvas for Presentation**:
   - Transfer insights to Canvas
   - Enhance with custom styling
   - Add context and narrative
   - Create polished presentations

3. **Complementary Workflow**:
   - Lens for analysts
   - Canvas for stakeholder communication
   - Dashboard for operational monitoring

## Integration with Dashboards

### Adding Lens to Dashboards

Incorporate Lens visualizations in dashboards:

1. Create a visualization in Lens
2. Save the visualization
3. Add to a dashboard from:
   - The save dialog
   - Dashboard "Add" panel
   - Visualization library

### Adding Canvas to Dashboards

Include Canvas workpads in dashboards:

1. Create a Canvas workpad
2. Save the workpad
3. In a dashboard, add a Canvas embed panel
4. Select the workpad to embed
5. Configure display options

### Sharing and Exporting

Share your Canvas and Lens creations:

1. **Direct URLs**:
   - Share direct links
   - Set time ranges
   - Include current filters

2. **Embed Options**:
   - Embed in web pages
   - Include in applications
   - Embed in iframes

3. **Export Formats**:
   - PDF reports
   - PNG images
   - Raw data exports

## Conclusion

Kibana Canvas and Lens provide powerful, complementary tools for visualizing and presenting Elasticsearch data. Lens offers an intuitive, rapid approach to creating standard visualizations, while Canvas enables pixel-perfect, highly customized data presentations. By understanding when and how to use each tool, you can create compelling visualizations that communicate data insights effectively to any audience.

In the next chapter, we'll explore Kibana's alerting and reporting capabilities, which allow you to automate notifications and generate scheduled reports based on your Elasticsearch data.