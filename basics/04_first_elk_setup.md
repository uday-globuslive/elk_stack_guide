# Setting Up Your First ELK Stack

This chapter guides you through setting up a basic ELK Stack on a single machine for development or testing purposes. We'll install Elasticsearch, Logstash, Kibana, and Filebeat to create a complete logging system.

## Prerequisites

Before starting, ensure your system meets the following requirements:

- Linux-based operating system (Ubuntu 20.04+ recommended)
- At least 4GB RAM
- 2+ CPU cores
- 20GB+ free disk space
- Java 11 or higher installed
- Internet connection for downloads

## Installing Java

Elasticsearch requires Java. If you don't have it installed, run:

```bash
# For Ubuntu/Debian
sudo apt update
sudo apt install openjdk-11-jdk

# For CentOS/RHEL
sudo yum install java-11-openjdk

# Verify installation
java -version
```

## Installing Elasticsearch

### Step 1: Add Elastic Repository

```bash
# Download and install the public signing key
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

# Add repository
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
```

### Step 2: Install Elasticsearch

```bash
sudo apt update
sudo apt install elasticsearch
```

### Step 3: Configure Elasticsearch

Edit the Elasticsearch configuration file:

```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

Make the following changes:

```yaml
# Set the cluster name
cluster.name: my-first-elk-cluster

# Set the node name
node.name: node-1

# Use a single node for development
discovery.type: single-node

# Listen on all interfaces
network.host: 0.0.0.0

# Default REST API port
http.port: 9200

# Disable security for simplicity in development
xpack.security.enabled: false
```

### Step 4: Start Elasticsearch

```bash
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch
```

### Step 5: Verify Elasticsearch Installation

```bash
curl http://localhost:9200
```

You should see a JSON response with cluster information.

## Installing Kibana

### Step 1: Install Kibana

```bash
sudo apt install kibana
```

### Step 2: Configure Kibana

Edit the Kibana configuration file:

```bash
sudo nano /etc/kibana/kibana.yml
```

Make the following changes:

```yaml
# Server host and port
server.host: "0.0.0.0"
server.port: 5601

# Elasticsearch connection
elasticsearch.hosts: ["http://localhost:9200"]
```

### Step 3: Start Kibana

```bash
sudo systemctl daemon-reload
sudo systemctl enable kibana
sudo systemctl start kibana
```

### Step 4: Verify Kibana Installation

Open a web browser and navigate to `http://your_server_ip:5601`. You should see the Kibana interface.

## Installing Logstash

### Step 1: Install Logstash

```bash
sudo apt install logstash
```

### Step 2: Create a Simple Logstash Pipeline

Create a basic pipeline configuration file:

```bash
sudo nano /etc/logstash/conf.d/01-syslog.conf
```

Add the following configuration:

```
input {
  file {
    path => "/var/log/syslog"
    start_position => "beginning"
    type => "syslog"
  }
}

filter {
  if [type] == "syslog" {
    grok {
      match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}" }
    }
    date {
      match => [ "syslog_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
    }
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "syslog-%{+YYYY.MM.dd}"
  }
  stdout { codec => rubydebug }
}
```

### Step 3: Start Logstash

```bash
sudo systemctl daemon-reload
sudo systemctl enable logstash
sudo systemctl start logstash
```

## Installing Filebeat

### Step 1: Install Filebeat

```bash
sudo apt install filebeat
```

### Step 2: Configure Filebeat

```bash
sudo nano /etc/filebeat/filebeat.yml
```

Update the configuration:

```yaml
filebeat.inputs:
- type: filestream
  id: my-filestream-id
  enabled: true
  paths:
    - /var/log/*.log

setup.kibana:
  host: "localhost:5601"

output.elasticsearch:
  hosts: ["localhost:9200"]
```

### Step 3: Enable and Start Filebeat

```bash
sudo filebeat modules enable system
sudo filebeat setup
sudo systemctl enable filebeat
sudo systemctl start filebeat
```

## Configuring Kibana for Logs Viewing

### Step 1: Access Kibana

Open your browser and navigate to `http://your_server_ip:5601`

### Step 2: Create Index Pattern

1. Navigate to Stack Management → Kibana → Index Patterns
2. Click "Create index pattern"
3. Enter `filebeat-*` as the index pattern name
4. Select `@timestamp` as the time field
5. Click "Create index pattern"

### Step 3: View Logs in Discover

1. Navigate to Discover in the main menu
2. Select the `filebeat-*` index pattern
3. You should now see logs flowing in

## Testing the Stack

Let's generate some logs to test our setup:

```bash
logger "This is a test message for ELK stack"
```

Within a minute or two, this message should appear in Kibana's Discover view.

## Creating Your First Visualization

1. Go to Visualize in Kibana
2. Click "Create visualization"
3. Choose "Line" visualization
4. Select the `filebeat-*` index pattern
5. For Y-Axis, select Count aggregation
6. For X-Axis, select Date Histogram with @timestamp field
7. Click "Update" to see your visualization
8. Save your visualization

## Creating Your First Dashboard

1. Go to Dashboard in Kibana
2. Click "Create dashboard"
3. Click "Add visualization"
4. Select the visualization you just created
5. Save the dashboard

## Next Steps

Congratulations! You've successfully set up a basic ELK Stack. Here are some next steps to consider:

1. Secure your ELK Stack by enabling authentication and encryption
2. Configure Filebeat to collect logs from specific applications
3. Create more visualizations and dashboards
4. Set up centralized logging for multiple servers
5. Explore more advanced Logstash configurations

In the upcoming chapters, we'll dive deeper into each component and explore advanced configurations, including security, scalability, and best practices.

## Troubleshooting Common Issues

### Elasticsearch Won't Start

Check the logs:
```bash
sudo journalctl -u elasticsearch
```

Common issues:
- Insufficient memory (increase or enable swap)
- Permission problems (ensure proper ownership of config and data directories)
- Network binding issues (check network.host setting)

### Kibana Can't Connect to Elasticsearch

Check the logs:
```bash
sudo journalctl -u kibana
```

Verify:
- Elasticsearch is running (`curl localhost:9200`)
- Kibana configuration points to the correct Elasticsearch URL
- No network issues between Kibana and Elasticsearch

### Logstash Not Processing Logs

Check the logs:
```bash
sudo journalctl -u logstash
```

Verify:
- Configuration syntax is correct
- Logstash has permission to read input files
- Output is correctly configured

### Filebeat Not Sending Data

Check the logs:
```bash
sudo journalctl -u filebeat
```

Verify:
- Filebeat has permission to read the log files
- Output configuration points to the correct Elasticsearch instance
- No network issues between Filebeat and Elasticsearch