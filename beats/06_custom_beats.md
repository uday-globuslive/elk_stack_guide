# Custom Beat Development

This chapter covers the process of developing custom Beats for specialized use cases, extending the Elastic Stack's data collection capabilities to meet unique requirements.

## Table of Contents
- [Understanding Beat Architecture](#understanding-beat-architecture)
- [Development Environment Setup](#development-environment-setup)
- [Creating a New Beat](#creating-a-new-beat)
- [Beat Components](#beat-components)
- [Implementing Data Collection](#implementing-data-collection)
- [Processing and Transformation](#processing-and-transformation)
- [Testing Your Beat](#testing-your-beat)
- [Packaging and Distribution](#packaging-and-distribution)
- [Deploying Custom Beats](#deploying-custom-beats)
- [Maintenance and Upgrades](#maintenance-and-upgrades)
- [Advanced Development Topics](#advanced-development-topics)
- [Case Studies](#case-studies)

## Understanding Beat Architecture

Before developing a custom Beat, it's essential to understand the architecture and components of the Beats platform.

### Beats Platform Overview

Beats are lightweight data shippers built on a common framework called libbeat. The architecture consists of several key components:

1. **Beat Core**: The main application logic specific to the Beat's data collection purpose
2. **Libbeat**: Shared library providing common functionality
3. **Publisher Pipeline**: Processes and publishes events
4. **Outputs**: Sends data to destinations like Elasticsearch or Logstash
5. **Processors**: Transform events before publishing
6. **Modules**: Optional modular configurations (for some Beats)

### Component Interactions

```
Data Source → Beat Core → Publisher Pipeline → Outputs
                 ↑              ↑
                 |              |
              Config         Processors
```

1. **Data Collection**: The Beat collects data from its source
2. **Event Creation**: Raw data is converted to events
3. **Processing**: Events are processed and enriched
4. **Publishing**: Events are sent to the configured outputs

### Beat Lifecycle

A Beat's lifecycle follows these stages:

1. **Initialization**: Load configuration and initialize components
2. **Setup**: Establish connections and prepare for data collection
3. **Run**: Collect, process, and publish data continuously
4. **Cleanup**: Release resources and shut down cleanly

## Development Environment Setup

Setting up a proper development environment is crucial for successful Beat development.

### Prerequisites

1. **Go Installation**:
   ```bash
   # Install Go 1.17 or later
   wget https://golang.org/dl/go1.17.8.linux-amd64.tar.gz
   sudo tar -C /usr/local -xzf go1.17.8.linux-amd64.tar.gz
   export PATH=$PATH:/usr/local/go/bin
   ```

2. **Git**:
   ```bash
   sudo apt-get install git
   ```

3. **Python 3**:
   ```bash
   sudo apt-get install python3 python3-pip python3-venv
   ```

4. **Development Tools**:
   ```bash
   sudo apt-get install make gcc
   ```

### Setting Up the Development Environment

1. **Clone Beats Repository**:
   ```bash
   git clone https://github.com/elastic/beats.git
   cd beats
   ```

2. **Install Mage**:
   ```bash
   git clone https://github.com/magefile/mage
   cd mage
   go run bootstrap.go
   cd ..
   ```

3. **Set Up GOPATH**:
   ```bash
   export GOPATH=$HOME/go
   export PATH=$PATH:$GOPATH/bin
   ```

4. **Verify Setup**:
   ```bash
   cd beats
   make setup
   ```

## Creating a New Beat

Create a new custom Beat using the Beat generator.

### Using the Beat Generator

The Beat generator creates a new Beat with the necessary structure and boilerplate code:

```bash
cd beats
make setup
make update
cd dev-tools/generator
go run main.go -path=~/mybeat -project=github.com/yourusername/mybeat -module=mymodule
```

This creates a new Beat project with the following structure:

```
mybeat/
├── _meta/            # Metadata files
├── beater/           # Beat-specific code
├── config/           # Configuration
├── include/          # Include files
├── magefile.go       # Build instructions
├── main.go           # Entry point
├── main_test.go      # Main tests
└── mybeat.yml        # Default configuration
```

### Customizing Beat Metadata

Edit metadata files to describe your Beat:

1. **Edit _meta/beat.yml**:
   ```yaml
   name: mybeat
   description: "My custom Beat for collecting specialized data"
   ```

2. **Update _meta/fields.yml**:
   ```yaml
   - key: mybeat
     title: MyBeat
     description: >
       Fields specific to MyBeat.
     fields:
       - name: field1
         type: keyword
         description: "Description of field1"
       - name: field2
         type: long
         description: "Description of field2"
   ```

3. **Create a Logo** (optional):
   - Create a 32x32 PNG image
   - Save it as `_meta/beat-logo.png`

### Defining Configuration

Define the configuration structure in `config/config.go`:

```go
package config

import (
	"time"
)

// Config defines the structure of MyBeat configuration
type Config struct {
	Period          time.Duration `config:"period"`
	Source          string        `config:"source"`
	IncludePatterns []string      `config:"include_patterns"`
	ExcludePatterns []string      `config:"exclude_patterns"`
	Timeout         time.Duration `config:"timeout"`
	MaxItems        int           `config:"max_items"`
}

// DefaultConfig provides default settings
var DefaultConfig = Config{
	Period:          1 * time.Minute,
	Source:          "default-source",
	IncludePatterns: []string{"*"},
	ExcludePatterns: []string{},
	Timeout:         30 * time.Second,
	MaxItems:        1000,
}
```

Update `mybeat.yml` with default configuration values:

```yaml
###################### MyBeat Configuration Example #######################

mybeat:
  # Collection period
  period: 1m
  
  # Data source configuration
  source: "default-source"
  
  # Patterns to include
  include_patterns:
    - "*"
  
  # Patterns to exclude
  exclude_patterns: []
  
  # Connection timeout
  timeout: 30s
  
  # Maximum items to collect
  max_items: 1000

#================================ General =====================================

# The name of the shipper that publishes the network data. It can be used to group
# all the transactions sent by a single shipper in the web interface.
name: "mybeat"

#================================ Outputs =====================================

# Configure what output to use when sending the data collected by the beat.

#-------------------------- Elasticsearch output ------------------------------
output.elasticsearch:
  # Array of hosts to connect to.
  hosts: ["localhost:9200"]

  # Protocol - either `http` (default) or `https`.
  #protocol: "https"

  # Authentication credentials
  #username: "elastic"
  #password: "changeme"
```

## Beat Components

A custom Beat is composed of several key components that work together to collect and ship data.

### The Beater Interface

The core of a Beat implements the `beat.Beater` interface:

```go
// beater/mybeat.go
package beater

import (
	"fmt"
	"time"

	"github.com/elastic/beats/v7/libbeat/beat"
	"github.com/elastic/beats/v7/libbeat/common"
	"github.com/elastic/beats/v7/libbeat/logp"

	"github.com/yourusername/mybeat/config"
)

// MyBeat configuration.
type MyBeat struct {
	done   chan struct{}
	config config.Config
	client beat.Client
}

// New creates an instance of mybeat.
func New(b *beat.Beat, cfg *common.Config) (beat.Beater, error) {
	c := config.DefaultConfig
	if err := cfg.Unpack(&c); err != nil {
		return nil, fmt.Errorf("error reading config file: %v", err)
	}

	bt := &MyBeat{
		done:   make(chan struct{}),
		config: c,
	}
	return bt, nil
}

// Run starts mybeat.
func (bt *MyBeat) Run(b *beat.Beat) error {
	logp.Info("mybeat is running! Config: %+v", bt.config)

	var err error
	bt.client, err = b.Publisher.Connect()
	if err != nil {
		return err
	}

	ticker := time.NewTicker(bt.config.Period)
	for {
		select {
		case <-bt.done:
			return nil
		case <-ticker.C:
			// Collect and publish data
			err := bt.collectAndPublish(b)
			if err != nil {
				logp.Err("Error collecting data: %v", err)
			}
		}
	}
}

// Stop stops mybeat.
func (bt *MyBeat) Stop() {
	logp.Info("mybeat is stopping")
	close(bt.done)
	bt.client.Close()
}

// collectAndPublish collects data and publishes events
func (bt *MyBeat) collectAndPublish(b *beat.Beat) error {
	// Implement your data collection logic here
	// This is a placeholder for your actual implementation
	event := beat.Event{
		Timestamp: time.Now(),
		Fields: common.MapStr{
			"type":       "mybeat",
			"source":     bt.config.Source,
			"sample":     "This is a sample event",
			"value":      42,
			"collection": "successful",
		},
	}
	
	bt.client.Publish(event)
	logp.Info("Event published: %v", event)
	
	return nil
}
```

### Main Entry Point

The `main.go` file serves as the entry point for your Beat:

```go
// main.go
package main

import (
	"os"

	"github.com/elastic/beats/v7/libbeat/cmd"
	"github.com/elastic/beats/v7/libbeat/cmd/instance"

	"github.com/yourusername/mybeat/beater"
)

var RootCmd = cmd.GenRootCmdWithSettings(beater.New, instance.Settings{
	Name:          "mybeat",
	Version:       "1.0.0",
	HasDashboards: true,
})

func main() {
	if err := RootCmd.Execute(); err != nil {
		os.Exit(1)
	}
}
```

### Event Fields Definition

Define your Beat's fields in `_meta/fields.yml`:

```yaml
- key: mybeat
  title: MyBeat
  description: >
    Fields specific to MyBeat.
  fields:
    - name: source
      type: keyword
      description: >
        The source of the data.
    - name: sample
      type: keyword
      description: >
        Sample field description.
    - name: value
      type: long
      description: >
        Value field description.
    - name: collection
      type: keyword
      description: >
        Information about the collection status.
```

## Implementing Data Collection

Implement the logic to collect data specific to your Beat's purpose.

### Data Collection Strategies

Choose the appropriate data collection approach:

1. **Polling**: Periodically fetch data
   ```go
   ticker := time.NewTicker(bt.config.Period)
   for {
       select {
       case <-bt.done:
           return nil
       case <-ticker.C:
           // Collect data on each tick
           data := bt.collectData()
           bt.publishEvents(data)
       }
   }
   ```

2. **Event-Driven**: React to external events
   ```go
   // Set up event source connection
   eventSource := setupEventSource(bt.config)
   
   // Listen for events
   for {
       select {
       case <-bt.done:
           return nil
       case event := <-eventSource:
           // Process and publish event
           bt.processAndPublish(event)
       }
   }
   ```

3. **Batch Processing**: Collect and process data in batches
   ```go
   ticker := time.NewTicker(bt.config.Period)
   for {
       select {
       case <-bt.done:
           return nil
       case <-ticker.C:
           // Collect batch of data
           batch := bt.collectBatch()
           
           // Process and publish batch
           bt.processBatch(batch)
       }
   }
   ```

### Example: REST API Data Collection

Implement collection from a REST API:

```go
// Helper function to collect data from an API
func (bt *MyBeat) collectData() ([]common.MapStr, error) {
	// Create a context with timeout
	ctx, cancel := context.WithTimeout(context.Background(), bt.config.Timeout)
	defer cancel()
	
	// Create HTTP client
	client := &http.Client{
		Timeout: bt.config.Timeout,
	}
	
	// Prepare request
	req, err := http.NewRequestWithContext(ctx, "GET", bt.config.Source, nil)
	if err != nil {
		return nil, fmt.Errorf("error creating request: %v", err)
	}
	
	// Add headers if needed
	req.Header.Add("Accept", "application/json")
	
	// Execute request
	resp, err := client.Do(req)
	if err != nil {
		return nil, fmt.Errorf("error making request: %v", err)
	}
	defer resp.Body.Close()
	
	// Check status code
	if resp.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("received non-200 status code: %d", resp.StatusCode)
	}
	
	// Parse response
	var data []map[string]interface{}
	if err := json.NewDecoder(resp.Body).Decode(&data); err != nil {
		return nil, fmt.Errorf("error parsing response: %v", err)
	}
	
	// Transform to MapStr objects
	var result []common.MapStr
	for _, item := range data {
		// Apply include/exclude patterns
		if bt.shouldInclude(item) {
			result = append(result, common.MapStr(item))
		}
	}
	
	// Limit number of items if configured
	if bt.config.MaxItems > 0 && len(result) > bt.config.MaxItems {
		result = result[:bt.config.MaxItems]
	}
	
	return result, nil
}

// Helper function to determine if an item should be included
func (bt *MyBeat) shouldInclude(item map[string]interface{}) bool {
	itemJson, _ := json.Marshal(item)
	itemStr := string(itemJson)
	
	// Check exclude patterns
	for _, pattern := range bt.config.ExcludePatterns {
		if matched, _ := filepath.Match(pattern, itemStr); matched {
			return false
		}
	}
	
	// Check include patterns
	if len(bt.config.IncludePatterns) == 0 {
		return true
	}
	
	for _, pattern := range bt.config.IncludePatterns {
		if matched, _ := filepath.Match(pattern, itemStr); matched {
			return true
		}
	}
	
	return false
}
```

### Example: File Monitoring

Implement monitoring of a directory for new files:

```go
// Set up file watcher
func (bt *MyBeat) watchDirectory() error {
	watcher, err := fsnotify.NewWatcher()
	if err != nil {
		return fmt.Errorf("error creating file watcher: %v", err)
	}
	
	// Add the directory to watch
	if err := watcher.Add(bt.config.Source); err != nil {
		return fmt.Errorf("error watching directory: %v", err)
	}
	
	// Start watching for events
	go func() {
		for {
			select {
			case <-bt.done:
				watcher.Close()
				return
			case event := <-watcher.Events:
				if event.Op&fsnotify.Create == fsnotify.Create {
					// New file created
					bt.processNewFile(event.Name)
				}
			case err := <-watcher.Errors:
				logp.Err("Watcher error: %v", err)
			}
		}
	}()
	
	return nil
}

// Process a new file
func (bt *MyBeat) processNewFile(path string) {
	// Check file patterns
	if !bt.matchesFilePattern(path) {
		return
	}
	
	// Process the file
	data, err := bt.readAndParseFile(path)
	if err != nil {
		logp.Err("Error processing file %s: %v", path, err)
		return
	}
	
	// Create and publish event
	event := beat.Event{
		Timestamp: time.Now(),
		Fields: common.MapStr{
			"type":       "mybeat",
			"source":     path,
			"data":       data,
			"collection": "file",
		},
	}
	
	bt.client.Publish(event)
}
```

## Processing and Transformation

Transform and enrich the collected data before publishing events.

### Building the Event

Create well-structured events from your data:

```go
// Process collected data into events
func (bt *MyBeat) processData(data []common.MapStr) []beat.Event {
	var events []beat.Event
	
	for _, item := range data {
		// Create base event
		event := beat.Event{
			Timestamp: time.Now(),
			Fields: common.MapStr{
				"type":   "mybeat",
				"source": bt.config.Source,
			},
		}
		
		// Add data fields
		for key, value := range item {
			event.Fields[key] = value
		}
		
		// Add metadata
		event.Fields["host.name"] = bt.hostname
		
		events = append(events, event)
	}
	
	return events
}
```

### Adding Metadata

Enrich events with useful context:

```go
// Get system information
hostname, _ := os.Hostname()

// Add metadata to every event
for i := range events {
	events[i].Fields.DeepUpdate(common.MapStr{
		"host": common.MapStr{
			"name": hostname,
		},
		"agent": common.MapStr{
			"name":    "mybeat",
			"type":    "mybeat",
			"version": "1.0.0",
		},
		"event": common.MapStr{
			"created": time.Now(),
			"module":  "mymodule",
		},
	})
}
```

### Implementing Processors

Add custom processors for data transformation:

```go
// Custom processor implementation
func NewConvertProcessor(c *common.Config) (beat.Processor, error) {
	config := struct {
		Field      string `config:"field"`
		TargetField string `config:"target_field"`
		Type       string `config:"type"`
	}{}
	
	err := c.Unpack(&config)
	if err != nil {
		return nil, fmt.Errorf("error unpacking convert processor config: %v", err)
	}
	
	processor := &convertProcessor{
		field:       config.Field,
		targetField: config.TargetField,
		targetType:  config.Type,
	}
	
	return processor, nil
}

type convertProcessor struct {
	field       string
	targetField string
	targetType  string
}

func (p *convertProcessor) Run(event *beat.Event) (*beat.Event, error) {
	// Get field value
	value, err := event.GetValue(p.field)
	if err != nil {
		return event, nil // Field not found, skip conversion
	}
	
	// Convert value based on target type
	var convertedValue interface{}
	switch p.targetType {
	case "string":
		convertedValue = fmt.Sprintf("%v", value)
	case "int":
		if strValue, ok := value.(string); ok {
			if intValue, err := strconv.Atoi(strValue); err == nil {
				convertedValue = intValue
			}
		}
	case "float":
		if strValue, ok := value.(string); ok {
			if floatValue, err := strconv.ParseFloat(strValue, 64); err == nil {
				convertedValue = floatValue
			}
		}
	case "bool":
		if strValue, ok := value.(string); ok {
			if boolValue, err := strconv.ParseBool(strValue); err == nil {
				convertedValue = boolValue
			}
		}
	default:
		return event, fmt.Errorf("unsupported conversion type: %s", p.targetType)
	}
	
	// Set target field with converted value
	if convertedValue != nil {
		_, err = event.PutValue(p.targetField, convertedValue)
		if err != nil {
			return event, fmt.Errorf("failed to set target field: %v", err)
		}
	}
	
	return event, nil
}
```

### Filtering Events

Filter events based on criteria:

```go
// Filter events before publishing
func (bt *MyBeat) filterEvents(events []beat.Event) []beat.Event {
	var filtered []beat.Event
	
	for _, event := range events {
		// Apply filtering conditions
		value, _ := event.GetValue("value")
		
		// Skip events with low values
		if intValue, ok := value.(int); ok && intValue < 10 {
			logp.Debug("filter", "Skipping event with low value: %d", intValue)
			continue
		}
		
		// Include event in filtered result
		filtered = append(filtered, event)
	}
	
	return filtered
}
```

## Testing Your Beat

Implement tests to ensure your Beat functions correctly.

### Unit Testing

Create unit tests for your Beat's components:

```go
// beater/mybeat_test.go
package beater

import (
	"testing"
	"time"

	"github.com/elastic/beats/v7/libbeat/beat"
	"github.com/elastic/beats/v7/libbeat/common"
	
	"github.com/yourusername/mybeat/config"
)

// MockClient for testing
type MockClient struct {
	PublishedEvents []beat.Event
}

func (c *MockClient) Publish(event beat.Event) {
	c.PublishedEvents = append(c.PublishedEvents, event)
}

func (c *MockClient) PublishAll(events []beat.Event) {
	c.PublishedEvents = append(c.PublishedEvents, events...)
}

func (c *MockClient) Close() error {
	return nil
}

// Test the New function
func TestNew(t *testing.T) {
	cfg := common.MustNewConfigFrom(map[string]interface{}{
		"period": "5s",
		"source": "test-source",
	})
	
	b := &beat.Beat{
		Info: beat.Info{
			Name:    "testbeat",
			Version: "1.0.0",
		},
	}
	
	beater, err := New(b, cfg)
	if err != nil {
		t.Fatalf("Error creating beater: %v", err)
	}
	
	mybeat, ok := beater.(*MyBeat)
	if !ok {
		t.Fatalf("Beater is not a MyBeat")
	}
	
	if mybeat.config.Period != 5*time.Second {
		t.Errorf("Period not set correctly, got %v", mybeat.config.Period)
	}
	
	if mybeat.config.Source != "test-source" {
		t.Errorf("Source not set correctly, got %v", mybeat.config.Source)
	}
}

// Test the collectAndPublish function
func TestCollectAndPublish(t *testing.T) {
	bt := &MyBeat{
		done: make(chan struct{}),
		config: config.Config{
			Period: 1 * time.Second,
			Source: "test-source",
		},
	}
	
	mockClient := &MockClient{}
	bt.client = mockClient
	
	b := &beat.Beat{
		Info: beat.Info{
			Name:    "testbeat",
			Version: "1.0.0",
		},
	}
	
	err := bt.collectAndPublish(b)
	if err != nil {
		t.Fatalf("Error in collectAndPublish: %v", err)
	}
	
	if len(mockClient.PublishedEvents) != 1 {
		t.Fatalf("Expected 1 published event, got %d", len(mockClient.PublishedEvents))
	}
	
	event := mockClient.PublishedEvents[0]
	source, _ := event.GetValue("source")
	if source != "test-source" {
		t.Errorf("Expected source 'test-source', got %v", source)
	}
}
```

### Integration Testing

Test your Beat with real dependencies:

```go
// tests/system/test_base.py
from mybeat import BaseTest

class Test(BaseTest):
    def test_base(self):
        """
        Basic test with mybeat
        """
        self.render_config_template(
            period="1s",
            source="test-source"
        )
        mybeat = self.start_beat()
        self.wait_until(lambda: self.output_lines() > 0, max_timeout=20)
        mybeat.check_kill_and_wait()
        
        output = self.read_output()
        self.assertGreater(len(output), 0)
        
        evt = output[0]
        self.assertEqual(evt["source"], "test-source")

    def test_fields(self):
        """
        Test field values
        """
        self.render_config_template(
            period="1s",
            source="test-source"
        )
        mybeat = self.start_beat()
        self.wait_until(lambda: self.output_lines() > 0, max_timeout=20)
        mybeat.check_kill_and_wait()
        
        output = self.read_output()
        self.assertGreater(len(output), 0)
        
        evt = output[0]
        self.assertEqual(evt["type"], "mybeat")
        self.assertEqual(evt["collection"], "successful")
        self.assertEqual(evt["value"], 42)
```

### Benchmarking

Create benchmarks to measure performance:

```go
// beater/mybeat_benchmark_test.go
package beater

import (
	"testing"
	"time"

	"github.com/elastic/beats/v7/libbeat/beat"
	
	"github.com/yourusername/mybeat/config"
)

func BenchmarkCollectAndPublish(b *testing.B) {
	bt := &MyBeat{
		done: make(chan struct{}),
		config: config.Config{
			Period: 1 * time.Second,
			Source: "test-source",
		},
	}
	
	mockClient := &MockClient{}
	bt.client = mockClient
	
	beatInfo := &beat.Beat{
		Info: beat.Info{
			Name:    "testbeat",
			Version: "1.0.0",
		},
	}
	
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		_ = bt.collectAndPublish(beatInfo)
	}
}

func BenchmarkProcessData(b *testing.B) {
	bt := &MyBeat{
		config: config.Config{
			Source: "test-source",
		},
	}
	
	// Prepare test data
	var data []common.MapStr
	for i := 0; i < 1000; i++ {
		data = append(data, common.MapStr{
			"field1": "value1",
			"field2": i,
			"field3": true,
		})
	}
	
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		_ = bt.processData(data)
	}
}
```

## Packaging and Distribution

Package your Beat for distribution and deployment.

### Building the Beat

Use the Mage build tool to build your Beat:

```bash
cd ~/mybeat
mage build
```

This will compile your Beat into an executable in the `build/` directory.

### Creating Packages

Generate packages for different operating systems:

```bash
# Create packages for all platforms
mage package

# Create packages for specific platforms
mage -v package:linux
mage -v package:darwin
mage -v package:windows
```

This creates packages in the `build/distributions/` directory.

### Docker Image

Create a Docker image for your Beat:

```bash
# Build Docker image
mage buildDocker

# Or create a custom Dockerfile
cat > Dockerfile <<EOF
FROM golang:1.17 AS builder
WORKDIR /go/src/github.com/yourusername/mybeat
COPY . .
RUN go build -o /go/bin/mybeat

FROM debian:buster-slim
COPY --from=builder /go/bin/mybeat /usr/bin/mybeat
COPY mybeat.yml /etc/mybeat/mybeat.yml
USER root
CMD ["/usr/bin/mybeat", "-e", "-c", "/etc/mybeat/mybeat.yml"]
EOF

docker build -t mybeat:latest .
```

### Documentation

Create documentation for your Beat:

1. **README.md**:
   ```markdown
   # MyBeat
   
   MyBeat is a custom Beat for collecting specialized data from XYZ sources.
   
   ## Description
   
   MyBeat collects data from [describe data sources] and ships it to Elasticsearch or Logstash.
   
   ## Features
   
   - Feature 1: Description
   - Feature 2: Description
   - Feature 3: Description
   
   ## Getting Started
   
   ### Prerequisites
   
   - Elasticsearch 7.x or later
   - [Any other requirements]
   
   ### Installation
   
   1. Download the appropriate package for your platform
   2. Extract the package
   3. Edit the configuration file `mybeat.yml`
   4. Start MyBeat
   
   ```bash
   ./mybeat -c mybeat.yml
   ```
   
   ## Configuration
   
   ```yaml
   mybeat:
     # Collection period
     period: 1m
     
     # Data source configuration
     source: "your-data-source"
     
     # Include patterns
     include_patterns: ["*"]
   ```
   
   ## License
   
   [Your license information]
   ```

2. **Generate Reference Documentation**:
   ```bash
   mage docs
   ```

## Deploying Custom Beats

Strategies for deploying and managing your custom Beat.

### Deployment Options

Choose the appropriate deployment method:

1. **Standard Package Installation**:
   ```bash
   # Debian/Ubuntu
   sudo dpkg -i mybeat-1.0.0-amd64.deb
   
   # RHEL/CentOS
   sudo rpm -vi mybeat-1.0.0-x86_64.rpm
   
   # Windows
   # Run installer or unzip and install as a service
   PS> .\mybeat.exe install
   ```

2. **Docker Deployment**:
   ```bash
   docker run -d \
     --name mybeat \
     -v $(pwd)/mybeat.yml:/etc/mybeat/mybeat.yml \
     mybeat:latest
   ```

3. **Kubernetes Deployment**:
   ```yaml
   # mybeat-deployment.yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: mybeat
     namespace: monitoring
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: mybeat
     template:
       metadata:
         labels:
           app: mybeat
       spec:
         containers:
         - name: mybeat
           image: mybeat:latest
           imagePullPolicy: Always
           volumeMounts:
           - name: config
             mountPath: /etc/mybeat
         volumes:
         - name: config
           configMap:
             name: mybeat-config
   ---
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: mybeat-config
     namespace: monitoring
   data:
     mybeat.yml: |
       mybeat:
         period: 1m
         source: "production-service"
       output.elasticsearch:
         hosts: ["elasticsearch:9200"]
   ```

### Configuration Management

Manage configurations across environments:

1. **Environment-Specific Configurations**:
   ```yaml
   # Development configuration
   mybeat:
     period: 30s
     source: "dev-source"
     include_patterns: ["*"]
   
   # Production configuration
   mybeat:
     period: 5m
     source: "prod-source"
     include_patterns: ["critical-*", "important-*"]
     exclude_patterns: ["test-*"]
   ```

2. **Configuration Templates**:
   ```yaml
   mybeat:
     period: ${MYBEAT_PERIOD:1m}
     source: ${MYBEAT_SOURCE:default-source}
     include_patterns: ${MYBEAT_INCLUDE_PATTERNS:["*"]}
     exclude_patterns: ${MYBEAT_EXCLUDE_PATTERNS:[]}
   ```

3. **Configuration Management Tools**:
   - Ansible
   - Chef
   - Puppet
   - Salt

### Monitoring Your Beat

Monitor the health and performance of your custom Beat:

1. **Enable Internal Metrics**:
   ```yaml
   # mybeat.yml
   monitoring.enabled: true
   monitoring.elasticsearch:
     hosts: ["monitoring-es:9200"]
     username: "elastic"
     password: "changeme"
   ```

2. **External Monitoring with Metricbeat**:
   ```yaml
   # metricbeat.yml
   metricbeat.modules:
   - module: beat
     period: 10s
     hosts: ["http://mybeat:5066"]
     xpack.enabled: true
   ```

3. **Log Monitoring**:
   ```yaml
   # mybeat.yml
   logging.level: info
   logging.to_files: true
   logging.files:
     path: /var/log/mybeat
     name: mybeat
     keepfiles: 7
     permissions: 0644
   ```

## Maintenance and Upgrades

Guidelines for maintaining your custom Beat over time.

### Version Compatibility

Ensure compatibility with Elastic Stack releases:

1. **Dependency Management**:
   - Use the appropriate libbeat version that matches your target Elasticsearch version
   - Update dependencies when upgrading
   - Test with multiple Elasticsearch versions

2. **Version Pinning**:
   ```go
   // main.go
   var RootCmd = cmd.GenRootCmdWithSettings(beater.New, instance.Settings{
       Name:          "mybeat",
       Version:       "1.0.0",
       HasDashboards: true,
       ElasticLicenseCLI: elastic.NewLicensingCLI(
           "7.10.0", // Minimum supported ES version
           "", // License
           elastic.LicenseStrictMode,
       ),
   })
   ```

### Upgrade Strategy

Plan for safe upgrades:

1. **Semantic Versioning**:
   - Major version: Incompatible API changes
   - Minor version: New functionality in a backward-compatible manner
   - Patch version: Backward-compatible bug fixes

2. **Upgrade Process**:
   - Test in a staging environment
   - Update documentation
   - Communicate breaking changes
   - Provide migration scripts if necessary

3. **Backwards Compatibility**:
   - Maintain compatible configuration formats
   - Support deprecated fields with warnings
   - Provide fallback mechanisms

### Continuous Integration

Set up CI for your Beat:

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Set up Go
      uses: actions/setup-go@v2
      with:
        go-version: 1.17
        
    - name: Install dependencies
      run: |
        go get -u github.com/magefile/mage
        
    - name: Build
      run: mage build
      
    - name: Test
      run: mage test
      
    - name: Package
      run: mage package
      
    - name: Upload artifacts
      uses: actions/upload-artifact@v2
      with:
        name: mybeat-packages
        path: build/distributions/
```

## Advanced Development Topics

Advanced topics for experienced Beat developers.

### Implementing Modules

Create modular configurations for your Beat:

1. **Module Structure**:
   ```
   mybeat/
   ├── module/
   │   └── mymodule/
   │       ├── _meta/
   │       │   ├── config.yml
   │       │   └── docs.asciidoc
   │       ├── config/
   │       │   └── config.go
   │       └── mymodule.go
   ```

2. **Module Implementation**:
   ```go
   // module/mymodule/mymodule.go
   package mymodule

   import (
       "fmt"
       "time"

       "github.com/elastic/beats/v7/libbeat/beat"
       "github.com/elastic/beats/v7/libbeat/common"
       "github.com/elastic/beats/v7/libbeat/logp"
   )

   type Module struct {
       BaseConfig BaseConfig
   }

   type BaseConfig struct {
       Enabled bool          `config:"enabled"`
       Period  time.Duration `config:"period"`
       Source  string        `config:"source"`
   }

   var DefaultBaseConfig = BaseConfig{
       Enabled: true,
       Period:  1 * time.Minute,
       Source:  "default-source",
   }

   // Init initializes the module
   func (m *Module) Init(cfg *common.Config) error {
       // Load the base config
       c := DefaultBaseConfig
       if err := cfg.Unpack(&c); err != nil {
           return fmt.Errorf("error unpacking config: %v", err)
       }
       m.BaseConfig = c

       return nil
   }

   // Collect retrieves data and returns events
   func (m *Module) Collect() ([]beat.Event, error) {
       var events []beat.Event

       // Collect data here
       // ...

       return events, nil
   }
   ```

3. **Module Configuration**:
   ```yaml
   # _meta/config.yml
   - module: mymodule
     enabled: true
     period: 1m
     source: "default-source"
   ```

### Custom Dashboards

Create custom dashboards for your Beat:

1. **Dashboard Structure**:
   ```
   mybeat/
   ├── _meta/
   │   └── kibana/
   │       ├── dashboard/
   │       │   └── mybeat-overview.json
   │       ├── search/
   │       │   └── mybeat-search.json
   │       └── visualization/
   │           ├── mybeat-stats.json
   │           └── mybeat-histogram.json
   ```

2. **Dashboard Export/Import**:
   ```bash
   # Export dashboards from Kibana to your Beat
   cd ~/mybeat
   go run dev-tools/cmd/dashboards/export_dashboards.go \
     -es https://localhost:9200 \
     -dashboard 12345678-your-dashboard-id \
     -output _meta/kibana

   # Import dashboards to Kibana
   mybeat setup --dashboards
   ```

### Custom Indexing

Customize the Elasticsearch index configuration:

1. **Index Template**:
   ```json
   // _meta/elasticsearch/index-template.json
   {
     "index_patterns": ["mybeat-*"],
     "mappings": {
       "properties": {
         "custom_field": {
           "type": "keyword",
           "ignore_above": 1024
         },
         "numeric_value": {
           "type": "scaled_float",
           "scaling_factor": 100
         },
         "timestamp": {
           "type": "date"
         }
       }
     },
     "settings": {
       "number_of_shards": 1,
       "number_of_replicas": 1
     }
   }
   ```

2. **Custom Index Name**:
   ```yaml
   output.elasticsearch:
     index: "mybeat-%{[agent.version]}-%{+yyyy.MM.dd}"
     indices:
       - index: "critical-%{[agent.version]}-%{+yyyy.MM.dd}"
         when.contains:
           tags: "critical"
       - index: "normal-%{[agent.version]}-%{+yyyy.MM.dd}"
         when.contains:
           tags: "normal"
   ```

### Extending the Beat Framework

Create custom extensions for the Beat framework:

1. **Custom Publisher Pipeline Processor**:
   ```go
   package processors

   import (
       "github.com/elastic/beats/v7/libbeat/beat"
       "github.com/elastic/beats/v7/libbeat/common"
       "github.com/elastic/beats/v7/libbeat/processors"
   )

   func init() {
       processors.RegisterPlugin("my_custom_processor", newMyCustomProcessor)
   }

   func newMyCustomProcessor(c *common.Config) (beat.Processor, error) {
       config := struct {
           Field string `config:"field"`
       }{}

       err := c.Unpack(&config)
       if err != nil {
           return nil, err
       }

       processor := &myCustomProcessor{
           field: config.Field,
       }

       return processor, nil
   }

   type myCustomProcessor struct {
       field string
   }

   func (p *myCustomProcessor) Run(event *beat.Event) (*beat.Event, error) {
       // Process the event
       // ...
       return event, nil
   }

   func (p *myCustomProcessor) String() string {
       return "my_custom_processor"
   }
   ```

2. **Custom Output**:
   ```go
   package outputs

   import (
       "context"

       "github.com/elastic/beats/v7/libbeat/beat"
       "github.com/elastic/beats/v7/libbeat/common"
       "github.com/elastic/beats/v7/libbeat/outputs"
   )

   func init() {
       outputs.RegisterType("my_custom_output", makeMyCustomOutput)
   }

   func makeMyCustomOutput(
       _ outputs.IndexManager,
       beat beat.Info,
       observer outputs.Observer,
       cfg *common.Config,
   ) (outputs.Group, error) {
       config := defaultConfig
       if err := cfg.Unpack(&config); err != nil {
           return outputs.Fail(err)
       }

       // Create client
       client := &myCustomClient{
           beat:   beat,
           config: config,
           observer: observer,
       }

       return outputs.Success(1, 0, client)
   }

   type myCustomClient struct {
       beat     beat.Info
       config   config
       observer outputs.Observer
   }

   func (c *myCustomClient) Close() error {
       return nil
   }

   func (c *myCustomClient) Publish(ctx context.Context, batch publisher.Batch) error {
       events := batch.Events()
       c.observer.NewBatch(len(events))

       // Process and send events
       // ...

       batch.ACK()
       return nil
   }
   ```

## Case Studies

Real-world examples of custom Beat development.

### Case Study 1: API Monitoring Beat

A Beat for monitoring multiple REST APIs:

```go
// APIBeat collects metrics from REST APIs
package beater

import (
	"encoding/json"
	"fmt"
	"net/http"
	"time"

	"github.com/elastic/beats/v7/libbeat/beat"
	"github.com/elastic/beats/v7/libbeat/common"
	"github.com/elastic/beats/v7/libbeat/logp"

	"github.com/yourusername/apibeat/config"
)

// APIBeat configuration
type APIBeat struct {
	done   chan struct{}
	config config.Config
	client beat.Client
	logger *logp.Logger
}

// New creates an instance of apibeat
func New(b *beat.Beat, cfg *common.Config) (beat.Beater, error) {
	c := config.DefaultConfig
	if err := cfg.Unpack(&c); err != nil {
		return nil, fmt.Errorf("error reading config file: %v", err)
	}

	bt := &APIBeat{
		done:   make(chan struct{}),
		config: c,
		logger: logp.NewLogger("apibeat"),
	}
	return bt, nil
}

// Run starts apibeat
func (bt *APIBeat) Run(b *beat.Beat) error {
	bt.logger.Info("apibeat is running! Config: %+v", bt.config)

	var err error
	bt.client, err = b.Publisher.Connect()
	if err != nil {
		return err
	}

	ticker := time.NewTicker(bt.config.Period)
	for {
		select {
		case <-bt.done:
			return nil
		case <-ticker.C:
			for _, endpoint := range bt.config.Endpoints {
				bt.logger.Debugf("Fetching API endpoint: %s", endpoint.URL)
				
				startTime := time.Now()
				
				// Make API request
				req, err := http.NewRequest("GET", endpoint.URL, nil)
				if err != nil {
					bt.logger.Errorf("Error creating request: %v", err)
					continue
				}
				
				// Add headers
				for key, value := range endpoint.Headers {
					req.Header.Add(key, value)
				}
				
				// Execute request
				client := &http.Client{
					Timeout: bt.config.Timeout,
				}
				resp, err := client.Do(req)
				
				responseTime := time.Since(startTime).Milliseconds()
				
				// Create base event
				event := beat.Event{
					Timestamp: time.Now(),
					Fields: common.MapStr{
						"type":          "apibeat",
						"api.name":      endpoint.Name,
						"api.url":       endpoint.URL,
						"api.method":    "GET",
						"api.response_time_ms": responseTime,
					},
				}
				
				// Handle errors
				if err != nil {
					event.Fields["api.status"] = "error"
					event.Fields["api.error"] = err.Error()
					bt.client.Publish(event)
					continue
				}
				
				// Add response details
				event.Fields["api.status"] = "success"
				event.Fields["api.response.status_code"] = resp.StatusCode
				
				// Parse response if JSON and configured to do so
				if endpoint.ParseResponse && resp.Header.Get("Content-Type") == "application/json" {
					var result map[string]interface{}
					if err := json.NewDecoder(resp.Body).Decode(&result); err == nil {
						// Add selected fields from response
						for _, field := range endpoint.IncludeFields {
							if value, ok := result[field]; ok {
								event.Fields["api.response."+field] = value
							}
						}
					}
				}
				
				resp.Body.Close()
				bt.client.Publish(event)
			}
		}
	}
}

// Stop stops apibeat
func (bt *APIBeat) Stop() {
	bt.logger.Info("apibeat is stopping")
	close(bt.done)
	bt.client.Close()
}
```

Configuration:

```yaml
apibeat:
  period: 1m
  timeout: 10s
  endpoints:
    - name: "User API"
      url: "https://api.example.com/users"
      headers:
        Authorization: "Bearer TOKEN"
        Accept: "application/json"
      parse_response: true
      include_fields: ["count", "status", "version"]
    
    - name: "Products API"
      url: "https://api.example.com/products"
      headers:
        Accept: "application/json"
      parse_response: true
      include_fields: ["count", "last_updated"]
    
    - name: "Health Check"
      url: "https://api.example.com/health"
      parse_response: false
```

### Case Study 2: IoT Device Beat

A Beat for collecting data from IoT devices:

```go
// IoTBeat collects metrics from IoT devices via MQTT
package beater

import (
	"encoding/json"
	"fmt"
	"time"

	mqtt "github.com/eclipse/paho.mqtt.golang"
	"github.com/elastic/beats/v7/libbeat/beat"
	"github.com/elastic/beats/v7/libbeat/common"
	"github.com/elastic/beats/v7/libbeat/logp"

	"github.com/yourusername/iotbeat/config"
)

// IoTBeat configuration
type IoTBeat struct {
	done      chan struct{}
	config    config.Config
	client    beat.Client
	mqttClient mqtt.Client
	logger    *logp.Logger
}

// New creates an instance of iotbeat
func New(b *beat.Beat, cfg *common.Config) (beat.Beater, error) {
	c := config.DefaultConfig
	if err := cfg.Unpack(&c); err != nil {
		return nil, fmt.Errorf("error reading config file: %v", err)
	}

	bt := &IoTBeat{
		done:   make(chan struct{}),
		config: c,
		logger: logp.NewLogger("iotbeat"),
	}
	return bt, nil
}

// Run starts iotbeat
func (bt *IoTBeat) Run(b *beat.Beat) error {
	bt.logger.Info("iotbeat is running! Config: %+v", bt.config)

	var err error
	bt.client, err = b.Publisher.Connect()
	if err != nil {
		return err
	}

	// Connect to MQTT broker
	opts := mqtt.NewClientOptions()
	opts.AddBroker(bt.config.Broker)
	opts.SetClientID(bt.config.ClientID)
	
	// Set credentials if provided
	if bt.config.Username != "" {
		opts.SetUsername(bt.config.Username)
		opts.SetPassword(bt.config.Password)
	}
	
	// Set connection handlers
	opts.SetOnConnectHandler(bt.onConnect)
	opts.SetConnectionLostHandler(bt.onConnectionLost)
	
	// Create and connect MQTT client
	bt.mqttClient = mqtt.NewClient(opts)
	if token := bt.mqttClient.Connect(); token.Wait() && token.Error() != nil {
		return fmt.Errorf("error connecting to MQTT broker: %v", token.Error())
	}
	
	<-bt.done
	return nil
}

// onConnect is called when connection to MQTT broker is established
func (bt *IoTBeat) onConnect(client mqtt.Client) {
	bt.logger.Info("Connected to MQTT broker")
	
	// Subscribe to all configured topics
	for _, topic := range bt.config.Topics {
		bt.logger.Infof("Subscribing to topic: %s", topic)
		token := client.Subscribe(topic, 0, func(client mqtt.Client, msg mqtt.Message) {
			bt.processMQTTMessage(topic, msg)
		})
		token.Wait()
	}
}

// onConnectionLost is called when connection to MQTT broker is lost
func (bt *IoTBeat) onConnectionLost(client mqtt.Client, err error) {
	bt.logger.Errorf("Connection to MQTT broker lost: %v", err)
}

// processMQTTMessage handles incoming MQTT messages
func (bt *IoTBeat) processMQTTMessage(topic string, msg mqtt.Message) {
	bt.logger.Debugf("Received message from topic %s", topic)
	
	// Parse message payload
	var data map[string]interface{}
	if err := json.Unmarshal(msg.Payload(), &data); err != nil {
		bt.logger.Errorf("Error parsing message payload: %v", err)
		return
	}
	
	// Extract device ID from topic or message
	deviceID := extractDeviceID(topic, data)
	
	// Create beat event
	event := beat.Event{
		Timestamp: time.Now(),
		Fields: common.MapStr{
			"type":        "iotbeat",
			"topic":       topic,
			"device_id":   deviceID,
			"device_type": getDeviceType(deviceID, bt.config.DeviceTypes),
		},
	}
	
	// Add message data
	for key, value := range data {
		event.Fields[key] = value
	}
	
	// Add device metadata if available
	if metadata, ok := bt.config.DeviceMetadata[deviceID]; ok {
		event.Fields["device"] = metadata
	}
	
	bt.client.Publish(event)
}

// extractDeviceID gets device ID from topic or message
func extractDeviceID(topic string, data map[string]interface{}) string {
	// Try to extract from data
	if id, ok := data["device_id"]; ok {
		if strID, ok := id.(string); ok {
			return strID
		}
	}
	
	// Extract from topic pattern devices/DEVICE_ID/telemetry
	parts := strings.Split(topic, "/")
	if len(parts) > 1 {
		return parts[1]
	}
	
	return "unknown"
}

// getDeviceType returns the device type based on device ID
func getDeviceType(deviceID string, deviceTypes map[string]string) string {
	// Check if we have a specific mapping
	if deviceType, ok := deviceTypes[deviceID]; ok {
		return deviceType
	}
	
	// Check for prefix matches
	for prefix, deviceType := range deviceTypes {
		if strings.HasPrefix(deviceID, prefix) {
			return deviceType
		}
	}
	
	return "unknown"
}

// Stop stops iotbeat
func (bt *IoTBeat) Stop() {
	bt.logger.Info("iotbeat is stopping")
	
	// Disconnect MQTT client if connected
	if bt.mqttClient != nil && bt.mqttClient.IsConnected() {
		bt.mqttClient.Disconnect(250)
	}
	
	close(bt.done)
	bt.client.Close()
}
```

Configuration:

```yaml
iotbeat:
  broker: "tcp://mqtt.example.com:1883"
  client_id: "iotbeat-collector"
  username: "iot_user"
  password: "iot_password"
  topics:
    - "devices/+/telemetry"
    - "devices/+/status"
    - "devices/+/alerts"
  device_types:
    "temp": "temperature_sensor"
    "hum": "humidity_sensor"
    "door": "door_sensor"
    "mot": "motion_detector"
  device_metadata:
    "temp01":
      location: "Building A, Room 101"
      model: "TempSensor-2000"
      installation_date: "2022-01-15"
    "door03":
      location: "Building A, Main Entrance"
      model: "DoorContact-Pro"
      installation_date: "2022-02-20"
```

## Conclusion

Developing custom Beats allows you to extend the Elastic Stack with specialized data collection capabilities tailored to your unique requirements. By following the patterns and practices described in this chapter, you can create reliable, efficient, and maintainable data shippers that seamlessly integrate with the broader Elastic ecosystem.

Whether you're collecting data from custom APIs, IoT devices, proprietary systems, or other specialized sources, custom Beats provide a powerful framework for bringing that data into Elasticsearch for analysis, visualization, and alerting.

In the next chapter, we'll explore high availability and architectural considerations for the ELK Stack, including clustering, resilience, and disaster recovery strategies.