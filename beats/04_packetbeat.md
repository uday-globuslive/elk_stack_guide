# Packetbeat

This chapter covers Packetbeat, the real-time network packet analyzer that provides visibility into the communication between applications in your infrastructure.

## Table of Contents
- [Packetbeat Overview](#packetbeat-overview)
- [Architecture and Concepts](#architecture-and-concepts)
- [Installation and Setup](#installation-and-setup)
- [Basic Configuration](#basic-configuration)
- [Protocol Configuration](#protocol-configuration)
- [Flow Monitoring](#flow-monitoring)
- [Processors and Enrichment](#processors-and-enrichment)
- [Output Configuration](#output-configuration)
- [Advanced Features](#advanced-features)
- [Performance Tuning](#performance-tuning)
- [Monitoring Packetbeat](#monitoring-packetbeat)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)
- [Use Cases](#use-cases)

## Packetbeat Overview

Packetbeat is a lightweight network packet analyzer that captures network traffic and decodes the application-layer protocols, providing real-time application performance monitoring and transaction monitoring capabilities.

### Key Features

- **Protocol Decoding**: Understands application protocols like HTTP, MySQL, Redis
- **Transaction Timing**: Measures request/response times
- **Real-time Monitoring**: Captures and analyzes data in real-time
- **Low Overhead**: Minimal impact on network and host performance
- **Security Monitoring**: Identifies unusual network patterns or security threats
- **Distributed Tracing**: Tracks transactions across services
- **Topology Mapping**: Visualizes communication between applications
- **Out-of-the-box Visualizations**: Pre-built dashboards for common protocols

### Common Use Cases

1. **Application Performance Monitoring**: Track response times and error rates
2. **Network Troubleshooting**: Identify network issues and bottlenecks
3. **Security Monitoring**: Detect unusual traffic patterns or potential breaches
4. **Transaction Tracing**: Follow requests through distributed systems
5. **API Monitoring**: Monitor API usage, performance, and errors
6. **Database Query Performance**: Analyze database query execution times
7. **Service Dependency Mapping**: Discover service relationships
8. **Network Flow Analysis**: Understand network traffic patterns

### Supported Protocols

Packetbeat can decode and analyze traffic for numerous protocols:

- **Web**: HTTP, HTTPS (TLS metadata only)
- **Databases**: MySQL, PostgreSQL, MongoDB, Cassandra
- **Caching**: Redis, Memcached
- **RPC**: Thrift
- **Messaging**: AMQP, Kafka
- **Network**: DNS, TLS, NFS
- **Generic**: TCP, UDP, ICMP
- **Raw Traffic**: PCAP files

## Architecture and Concepts

Packetbeat uses a multi-stage pipeline to capture, process, and ship network data.

### Core Components

1. **Sniffer**: Captures network packets (libpcap or AF_PACKET)
2. **Decoder**: Decodes network and transport layer protocols
3. **Transaction Aggregation**: Groups packets into application-level transactions
4. **Protocol Parsers**: Protocol-specific modules that extract relevant information
5. **Processors**: Transforms and enriches events
6. **Output**: Sends processed events to configured destinations

### Data Flow

```
Network Traffic → Sniffer → Decoder → Protocol Parsers → Transaction Assembly → Processors → Output
```

1. **Packet Capture**: Raw packets are captured from network interfaces
2. **IP/TCP Decoding**: Network and transport layers are decoded
3. **Protocol Decoding**: Application layer protocol is identified and decoded
4. **Transaction Assembly**: Related packets are grouped into transactions
5. **Event Creation**: Transactions are converted to events
6. **Processing**: Events are enriched and transformed
7. **Output**: Events are sent to the configured destination

### Transaction and Flow Concepts

- **Transaction**: Complete request-response cycle in an application protocol
- **Flow**: Network connection between two endpoints (IP addresses and ports)
- **Request**: Initial part of a transaction from client to server
- **Response**: Server's reply to a client request
- **RTT (Round-Trip Time)**: Time between request and response

## Installation and Setup

### System Requirements

- **Memory**: 100MB minimum (varies with traffic volume)
- **CPU**: 1 core minimum (more for high-volume networks)
- **Disk**: 200MB free space for installation
- **Network**: Access to network interfaces in promiscuous mode
- **Operating Systems**:
  - Linux (most distributions)
  - Windows Server 2012 R2 or later
  - macOS 10.13 or later
  - FreeBSD

### Prerequisites

Packetbeat requires special capabilities to capture network traffic:

- **Linux**: `libpcap-dev` package or `AF_PACKET` support
- **Windows**: WinPcap or Npcap drivers
- **macOS**: libpcap is included by default
- **Privileges**: Root/Administrator access or `CAP_NET_RAW` capability on Linux

### Installation Methods

1. **Package Repositories (Linux)**:
   ```bash
   # Debian/Ubuntu
   curl -L -O https://artifacts.elastic.co/downloads/beats/packetbeat/packetbeat-8.1.0-amd64.deb
   sudo dpkg -i packetbeat-8.1.0-amd64.deb
   
   # RHEL/CentOS
   curl -L -O https://artifacts.elastic.co/downloads/beats/packetbeat/packetbeat-8.1.0-x86_64.rpm
   sudo rpm -vi packetbeat-8.1.0-x86_64.rpm
   ```

2. **Windows**:
   - Download and install Npcap from https://nmap.org/npcap/
   - Download Packetbeat installer from Elastic website
   - Install using PowerShell as administrator:
     ```powershell
     # Install as a Windows service
     PS> .\packetbeat.exe install
     ```

3. **Docker**:
   ```bash
   docker pull docker.elastic.co/beats/packetbeat:8.1.0
   docker run --cap-add=NET_RAW --cap-add=NET_ADMIN docker.elastic.co/beats/packetbeat:8.1.0
   ```

4. **Kubernetes**:
   - Use Helm chart or Elastic Cloud on Kubernetes operator
   - Apply Packetbeat DaemonSet with appropriate privileges

### Initial Setup

1. **Configuration File**: Edit `packetbeat.yml` in the configuration directory
2. **Configure Network Interfaces**: Specify which interfaces to monitor
3. **Enable Protocols**: Choose which protocols to analyze
4. **Set Up Assets**: Load index templates and dashboards:
   ```bash
   packetbeat setup -e
   ```
5. **Start Packetbeat**:
   ```bash
   # Systemd (Linux)
   sudo systemctl start packetbeat
   
   # Windows
   Start-Service packetbeat
   ```

### Verification

Confirm Packetbeat is running correctly:

```bash
# Check service status (Linux)
sudo systemctl status packetbeat

# View logs
sudo journalctl -u packetbeat

# Test configuration
packetbeat test config -c /etc/packetbeat/packetbeat.yml

# Test output connectivity
packetbeat test output -c /etc/packetbeat/packetbeat.yml
```

## Basic Configuration

The `packetbeat.yml` file is the main configuration file, typically located in `/etc/packetbeat/` on Linux or `C:\Program Files\Packetbeat\` on Windows.

### Configuration Structure

```yaml
packetbeat.interfaces:          # Network interface configuration
packetbeat.flows:               # Network flow configuration
packetbeat.protocols:           # Protocol analyzers configuration
processors:                     # Event processors
output.elasticsearch:           # Output configuration
setup:                          # Setup configurations
logging:                        # Logging configurations
```

### Minimal Configuration Example

```yaml
packetbeat.interfaces:
  device: any
  snaplen: 1514
  type: pcap
  buffer_size_mb: 100

packetbeat.flows:
  timeout: 30s
  period: 10s

packetbeat.protocols:
  http:
    ports: [80, 8080, 8000, 5000, 8002]
  mysql:
    ports: [3306]
  redis:
    ports: [6379]

output.elasticsearch:
  hosts: ["localhost:9200"]
  username: "elastic"
  password: "changeme"
```

### Interface Configuration

Configure which network interfaces Packetbeat should monitor:

```yaml
packetbeat.interfaces:
  # Capture from specific device
  device: eth0
  
  # Or capture from all interfaces
  # device: any
  
  # Maximum packet size to capture
  snaplen: 1514
  
  # Capture method (pcap, af_packet, pf_ring)
  type: pcap
  
  # Buffer size for the sniffer
  buffer_size_mb: 100
  
  # Include loopback traffic
  with_loopback: true
  
  # For Linux af_packet sniffer
  # packetbeat.interfaces.type: af_packet
  # packetbeat.interfaces.buffer_size_mb: 100
  # packetbeat.interfaces.with_vlans: true
  # packetbeat.interfaces.bpf_filter: "port 80 or port 3306"
```

### Common Settings

1. **General Settings**:
   ```yaml
   name: "my-packetbeat"        # Custom name for the Beat
   tags: ["production", "web"]  # Tags for all events
   fields:                      # Custom fields for all events
     env: production
     dc: us-east-1
   fields_under_root: true      # Places custom fields at root level
   ```

2. **Logging Settings**:
   ```yaml
   logging.level: info
   logging.to_files: true
   logging.files:
     path: /var/log/packetbeat
     name: packetbeat
     keepfiles: 7
     permissions: 0644
   ```

3. **BPF Filter**:
   Berkeley Packet Filter for efficient packet filtering:
   ```yaml
   packetbeat.interfaces.bpf_filter: "port 80 or port 3306 or port 6379"
   ```

## Protocol Configuration

Packetbeat can decode and analyze various application protocols. Each protocol has its own configuration section.

### HTTP Protocol

```yaml
packetbeat.protocols:
  http:
    # List of ports where to find HTTP traffic
    ports: [80, 8080, 8000, 8002, 5000, 8080, 3000]
    
    # Include request and response body in events
    include_body_for: 
      - text/plain
      - text/html
      - application/json
      - application/x-www-form-urlencoded
      - multipart/form-data
    
    # Maximum body size to include
    max_body_size: 2kb
    
    # Hide sensitive data
    redact_authorization: true
    redact_headers: ["Cookie", "Set-Cookie"]
    
    # Split cookies into separate key-value pairs
    split_cookie: true
    
    # Keep original header keys
    preserve_original_event: true
    
    # Decode JSON responses
    json_decode_body: true
    
    # Maximum message size
    max_message_size: 10mb
    
    # Transaction timeout
    transaction_timeout: 10s
```

### Database Protocols

1. **MySQL**:
   ```yaml
   packetbeat.protocols:
     mysql:
       # List of ports where to find MySQL traffic
       ports: [3306]
       
       # Maximum query length
       max_query_length: 1024
       
       # Maximum message size
       max_message_size: 10mb
       
       # Transaction timeout
       transaction_timeout: 10s
       
       # Include examples of SELECT, UPDATE, INSERT, etc.
       query_examples: true
       
       # Number of random examples per query type
       max_examples: 10
   ```

2. **PostgreSQL**:
   ```yaml
   packetbeat.protocols:
     postgresql:
       # List of ports where to find PostgreSQL traffic
       ports: [5432]
       
       # Maximum query length
       max_query_length: 1024
       
       # Maximum message size
       max_message_size: 10mb
       
       # Transaction timeout
       transaction_timeout: 10s
   ```

3. **MongoDB**:
   ```yaml
   packetbeat.protocols:
     mongodb:
       # List of ports where to find MongoDB traffic
       ports: [27017]
       
       # Maximum document length
       max_doc_length: 5000
       
       # Maximum message size
       max_message_size: 10mb
       
       # Transaction timeout
       transaction_timeout: 10s
   ```

### Caching and Queuing Protocols

1. **Redis**:
   ```yaml
   packetbeat.protocols:
     redis:
       # List of ports where to find Redis traffic
       ports: [6379]
       
       # Hide sensitive commands
       hide_password: true
       
       # Maximum body size to include
       max_body_size: 1kb
       
       # Transaction timeout
       transaction_timeout: 5s
   ```

2. **Memcached**:
   ```yaml
   packetbeat.protocols:
     memcache:
       # List of ports where to find Memcached traffic
       ports: [11211]
       
       # Maximum value length
       max_value_length: 100
       
       # Split commands
       parse_keys_max: 5
       
       # Transaction timeout
       transaction_timeout: 5s
   ```

3. **AMQP**:
   ```yaml
   packetbeat.protocols:
     amqp:
       # List of ports where to find AMQP traffic
       ports: [5672]
       
       # Hide sensitive data
       hide_connection_information: true
       
       # Maximum body size
       max_body_length: 1000
       
       # Transaction timeout
       transaction_timeout: 5s
   ```

### DNS Protocol

```yaml
packetbeat.protocols:
  dns:
    # List of ports where to find DNS traffic
    ports: [53]
    
    # Include full DNS payload
    include_authorities: true
    include_additionals: true
    
    # Send events at the end of transaction
    send_request: false
    send_response: false
    
    # Transaction timeout
    transaction_timeout: 2s
```

### TLS Protocol

```yaml
packetbeat.protocols:
  tls:
    # List of ports where to find TLS traffic
    ports: [443, 8443, 9443]
    
    # Send events at the end of transaction
    send_certificates: false
    
    # Include server certificates in events
    include_raw_certificates: false
    
    # Include certificate details
    fingerprints: ["sha1", "sha256"]
    
    # Transaction timeout
    transaction_timeout: 30s
```

### Multiple Protocol Configuration Example

```yaml
packetbeat.protocols:
  # Web
  http:
    ports: [80, 8080, 8000, 5000, 8002]
    
  # Databases
  mysql:
    ports: [3306]
  postgresql:
    ports: [5432]
  mongodb:
    ports: [27017]
    
  # Caching
  redis:
    ports: [6379]
  memcache:
    ports: [11211]
    
  # Messaging
  amqp:
    ports: [5672]
  kafka:
    ports: [9092]
    
  # Network services
  dns:
    ports: [53]
  tls:
    ports: [443, 8443]
    
  # Generic TCP monitoring
  tcp:
    ports: [9000-9999]
    send_request: false
    send_response: false
```

## Flow Monitoring

Packetbeat can monitor network flows (connections between pairs of IP addresses and ports), providing insights into network traffic patterns.

### Basic Flow Configuration

```yaml
packetbeat.flows:
  # Enable flow monitoring
  enabled: true
  
  # Report flows every N seconds
  period: 10s
  
  # Expire flows after N seconds of inactivity
  timeout: 30s
```

### Advanced Flow Settings

```yaml
packetbeat.flows:
  enabled: true
  period: 10s
  timeout: 30s
  
  # Keep flow information in memory
  keep_null: false
  
  # Keep metrics after a flow expired
  keep_expired: true
  
  # Export flows on shutdown
  report_on_shutdown: true
```

### Flow Metrics

Packetbeat collects these flow metrics:

- **Basic Information**:
  - Source and destination IP addresses
  - Source and destination ports
  - Transport protocol (TCP, UDP)
  - Direction (in, out)
  
- **Volume Metrics**:
  - Bytes sent and received
  - Packets sent and received
  
- **TCP Metrics**:
  - TCP flags observed (SYN, ACK, FIN, RST)
  - Connection state and duration
  
- **Timing**:
  - Flow start and end times
  - Flow duration

### Example Flow Event

```json
{
  "@timestamp": "2023-05-15T12:34:56.789Z",
  "flow": {
    "id": "jL5fDHcBEIKF9WKShOLj",
    "final": true,
    "start": "2023-05-15T12:33:56.789Z",
    "duration": 60000000000,
    "source": {
      "ip": "192.168.1.10",
      "port": 59213,
      "bytes": 3240,
      "packets": 12
    },
    "destination": {
      "ip": "192.168.1.20",
      "port": 80,
      "bytes": 12580,
      "packets": 8
    },
    "transport": "tcp",
    "tcp": {
      "flags": ["SYN", "ACK", "FIN"]
    }
  },
  "network": {
    "type": "ipv4",
    "direction": "outbound",
    "bytes": 15820,
    "packets": 20
  }
}
```

## Processors and Enrichment

Processors transform events before they are sent to the output, allowing for data enrichment, filtering, and transformation.

### Common Processors

1. **Add Fields**:
   ```yaml
   processors:
     - add_fields:
         target: ''
         fields:
           environment: production
           datacenter: us-east-1
   ```

2. **Drop Fields**:
   ```yaml
   processors:
     - drop_fields:
         fields: ["agent.ephemeral_id", "ecs.version"]
         ignore_missing: true
   ```

3. **Rename Fields**:
   ```yaml
   processors:
     - rename:
         fields:
           - from: "source.ip"
             to: "client.ip"
         ignore_missing: true
         fail_on_error: false
   ```

4. **Add Host Metadata**:
   ```yaml
   processors:
     - add_host_metadata:
         netinfo.enabled: true
         geo:
           name: nyc-dc-1
           location: 40.7128, -74.0060
           continent_name: North America
           country_iso_code: US
   ```

5. **Add Network Direction**:
   ```yaml
   processors:
     - add_network_direction:
         internal_networks: 
           - private
           - "192.168.0.0/16"
           - "10.0.0.0/8"
           - "172.16.0.0/12"
   ```

### IP Enrichment

Add geographic and ASN information for IP addresses:

```yaml
processors:
  - add_geoip:
      field: source.ip
      target_field: source.geo
      database_file: /path/to/GeoLite2-City.mmdb
      ignore_missing: true
  
  - add_geoip:
      field: destination.ip
      target_field: destination.geo
      database_file: /path/to/GeoLite2-City.mmdb
      ignore_missing: true
  
  - add_asn:
      field: source.ip
      target_field: source.as
      database_file: /path/to/GeoLite2-ASN.mmdb
      ignore_missing: true
  
  - add_asn:
      field: destination.ip
      target_field: destination.as
      database_file: /path/to/GeoLite2-ASN.mmdb
      ignore_missing: true
```

### Conditional Processing

Apply processors based on conditions:

```yaml
processors:
  - if:
      or:
        - equals:
            http.response.status_code: 500
        - range:
            http.response.status_code:
              gt: 399
              lt: 499
    then:
      - add_tag:
          tags: ["http_error"]
      - add_fields:
          target: ''
          fields:
            alert_level: warning
    else:
      - add_tag:
          tags: ["success"]
```

### DNS Enrichment

Perform DNS lookups for IP addresses:

```yaml
processors:
  - dns:
      type: reverse
      field: source.ip
      target_field: source.domain
      ignore_missing: true
      timeout: 500ms
  
  - dns:
      type: reverse
      field: destination.ip
      target_field: destination.domain
      ignore_missing: true
      timeout: 500ms
```

### Packet-Specific Processors

1. **Drop Events Based on Network Direction**:
   ```yaml
   processors:
     - drop_event:
         when:
           equals:
             network.direction: internal
   ```

2. **Truncate Large Fields**:
   ```yaml
   processors:
     - truncate_fields:
         fields:
           - http.request.body.content
           - http.response.body.content
         max_bytes: 1024
   ```

3. **Community ID for Network Flow Correlation**:
   ```yaml
   processors:
     - community_id: {}
   ```

## Output Configuration

Packetbeat can send data to various outputs, with Elasticsearch and Logstash being the most common.

### Elasticsearch Output

Send data directly to Elasticsearch:

```yaml
output.elasticsearch:
  hosts: ["https://elasticsearch:9200"]
  index: "packetbeat-%{[agent.version]}-%{+yyyy.MM.dd}"
  username: "elastic"
  password: "changeme"
  ssl:
    certificate_authorities: ["/etc/pki/root/ca.pem"]
    certificate: "/etc/pki/client/cert.pem"
    key: "/etc/pki/client/cert.key"
    verification_mode: full
  worker: 3
  bulk_max_size: 50
  compression_level: 3
  loadbalance: true
  max_retries: 5
  backoff.init: 1s
  backoff.max: 60s
```

### Logstash Output

Send data to Logstash for additional processing:

```yaml
output.logstash:
  hosts: ["logstash:5044"]
  loadbalance: true
  ttl: 30s
  index: "packetbeat"
  ssl:
    enabled: true
    verification_mode: none
    certificate_authorities: ["/etc/pki/root/ca.pem"]
    certificate: "/etc/pki/client/cert.pem"]
    key: "/etc/pki/client/cert.key"]
  worker: 2
```

### Kafka Output

Send data to Apache Kafka:

```yaml
output.kafka:
  hosts: ["kafka1:9092", "kafka2:9092"]
  topic: "packetbeat"
  partition.round_robin:
    reachable_only: true
  compression: gzip
  required_acks: 1
  max_message_bytes: 1000000
  client_id: packetbeat
```

### Multiple Outputs

Send data to multiple destinations:

```yaml
output.elasticsearch:
  enabled: true
  hosts: ["elasticsearch:9200"]
  index: "packetbeat-%{[agent.version]}-%{+yyyy.MM.dd}"

output.logstash:
  enabled: true
  hosts: ["logstash:5044"]
```

## Advanced Features

### PCAP File Analysis

Analyze traffic from captured PCAP files:

```yaml
packetbeat.interfaces.type: pcap
packetbeat.interfaces.file: /path/to/traffic.pcap

# Disable real-time processing
packetbeat.interfaces.no_realtime: true

# Process at maximum speed
packetbeat.interfaces.topspeed: true

# Define ports to monitor
packetbeat.protocols.http.ports: [80, 8080]
packetbeat.protocols.mysql.ports: [3306]
```

### Traffic Sampling

Sample only a percentage of packets to reduce load:

```yaml
packetbeat.interfaces:
  device: eth0
  type: af_packet
  buffer_size_mb: 100
  
  # Sample only 50% of packets
  sample_rate: 0.5
```

### Custom Protocol Parsing

Define custom protocol parsers for non-standard ports:

```yaml
packetbeat.protocols:
  http:
    ports: [80, 8080]
    
    # Parse HTTP on custom ports
    send_all_headers: true
    include_all_headers: true
    
    # Custom parser settings
    parse_body: true
    max_message_size: 10mb
    
    # Parse HTTP on additional ports
    ports: [80, 8080, 8000, 9000, 8888, 3000]
```

### SSL/TLS Decryption

Decrypt SSL/TLS traffic for inspection (requires private keys):

```yaml
packetbeat.protocols.tls:
  ports: [443, 8443]
  
  # To enable decryption (EXPERIMENTAL)
  # ssl_key_log_file: /path/to/ssl_keylog.txt
```

> Note: SSL/TLS decryption requires access to private keys or session keys and may have legal implications. Only use this in environments where you have explicit permission to inspect encrypted traffic.

### HTTP Body Capture

Capture HTTP request and response bodies:

```yaml
packetbeat.protocols.http:
  ports: [80, 8080, 8000]
  include_body_for: 
    - application/json
    - application/x-www-form-urlencoded
    - text/plain
  max_body_size: 5kb
  redact_authorization: true
  redact_headers: ["Cookie", "Set-Cookie", "Authorization"]
```

## Performance Tuning

### System Configuration

Optimize system settings for Packetbeat:

1. **Network Buffer Sizes**:
   ```bash
   # Increase kernel ring buffer size
   sudo sysctl -w net.core.rmem_max=33554432
   sudo sysctl -w net.core.rmem_default=262144
   
   # For high-traffic environments
   sudo sysctl -w net.core.netdev_max_backlog=10000
   ```

2. **Memory and CPU**:
   - Allocate sufficient resources based on network volume
   - Monitor memory usage and adjust settings if needed
   - Consider dedicated CPU cores for packet processing

### Packetbeat Configuration Tuning

Optimize Packetbeat configuration for performance:

1. **Sniffer Settings**:
   ```yaml
   packetbeat.interfaces:
     device: eth0
     type: af_packet       # Faster than pcap on Linux
     buffer_size_mb: 200   # Larger buffer for burst traffic
     with_vlans: true      # Enable if VLAN tags are used
   ```

2. **Protocol Settings**:
   ```yaml
   packetbeat.protocols:
     # Limit HTTP parsing to reduce overhead
     http:
       send_headers: true
       send_all_headers: false
       include_body_for: []
     
     # Only monitor specific database queries
     mysql:
       max_query_length: 500
       max_message_size: 5mb
   ```

3. **Flow Settings**:
   ```yaml
   packetbeat.flows:
     enabled: true
     period: 30s       # Reduce reporting frequency
     timeout: 60s      # Longer timeout for established flows
   ```

4. **Output Optimization**:
   ```yaml
   output.elasticsearch:
     worker: 4
     bulk_max_size: 50
     compression_level: 3
   ```

### Filtering Techniques

Reduce processing overhead by filtering traffic:

1. **BPF Filters**:
   ```yaml
   # Only capture HTTP and database traffic
   packetbeat.interfaces.bpf_filter: "port 80 or port 3306 or port 5432 or port 6379"
   
   # Exclude high-volume services
   packetbeat.interfaces.bpf_filter: "not port 5044 and not port 9200"
   
   # Only monitor traffic to/from specific subnets
   packetbeat.interfaces.bpf_filter: "host 10.0.0.0/8"
   ```

2. **Selective Protocol Monitoring**:
   ```yaml
   # Only enable necessary protocols
   packetbeat.protocols:
     http:
       enabled: true
       ports: [80, 443, 8080]
     mysql:
       enabled: true
       ports: [3306]
     # Disable other protocols
     redis:
       enabled: false
     mongodb:
       enabled: false
   ```

### Resource Impact Considerations

Understand resource impact of different features:

1. **High CPU Impact**:
   - Full packet capture on high-volume networks
   - Complex BPF filters
   - HTTP body parsing
   - DNS, TLS protocol analysis
   - GeoIP and ASN enrichment

2. **High Memory Impact**:
   - Large buffer sizes
   - High transaction timeouts
   - Many concurrent transactions
   - Large maximum message sizes

## Monitoring Packetbeat

### Self Monitoring

Enable Packetbeat to report its own metrics:

```yaml
monitoring.enabled: true
monitoring.elasticsearch:
  hosts: ["https://monitoring-es:9200"]
  username: "beats_system"
  password: "changeme"
```

Or use Metricbeat to monitor Packetbeat:

```yaml
# In metricbeat.yml
- module: beat
  metricsets: ["stats", "state"]
  period: 10s
  hosts: ["http://localhost:5066"]
  xpack.enabled: true
```

### Health Metrics

Key metrics to monitor:

1. **System Metrics**:
   - CPU usage
   - Memory usage
   - Network interface statistics
   - Dropped packets

2. **Packetbeat Metrics**:
   - Packets processed/dropped
   - Transactions analyzed
   - Output success/failure rates
   - Protocol-specific metrics

3. **Network Metrics**:
   - Interface utilization
   - Packet capture statistics
   - Traffic volume by protocol
   - Error rates

### Logging Configuration

Configure Packetbeat's own logs:

```yaml
logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/packetbeat
  name: packetbeat
  keepfiles: 7
  permissions: 0644
logging.metrics.enabled: true
logging.metrics.period: 30s
```

## Troubleshooting

### Common Issues

1. **No Packets Captured**:
   - Incorrect interface configuration
   - Insufficient permissions
   - Missing packet capture libraries
   - Interface not in promiscuous mode
   - Incorrect BPF filter

2. **High Packet Drop Rate**:
   - Buffer size too small
   - System overloaded
   - Network traffic volume too high
   - Complex processing overhead

3. **Protocol Parsing Issues**:
   - Non-standard protocol implementations
   - Encrypted traffic
   - Fragmented packets
   - Protocol on non-standard ports

4. **Output Issues**:
   - Network connectivity problems
   - Authentication failures
   - Index mapping errors
   - Elasticsearch cluster health

### Diagnostic Techniques

1. **Test Configuration**:
   ```bash
   packetbeat test config -c /etc/packetbeat/packetbeat.yml
   ```

2. **Test Output Connectivity**:
   ```bash
   packetbeat test output -c /etc/packetbeat/packetbeat.yml
   ```

3. **Debug Mode**:
   ```bash
   packetbeat -e -c /etc/packetbeat/packetbeat.yml -d "publish,http,mysql"
   ```

4. **Packet Capture Verification**:
   ```bash
   # Check if interface can capture traffic
   sudo tcpdump -i eth0 -n
   
   # Verify BPF filter
   sudo tcpdump -i eth0 -n "port 80 or port 3306"
   ```

5. **Check System Logs**:
   ```bash
   journalctl -u packetbeat
   ```

### Solving Common Problems

1. **Permission Issues**:
   ```bash
   # Grant capabilities instead of running as root
   sudo setcap cap_net_raw,cap_net_admin=eip /usr/share/packetbeat/bin/packetbeat
   
   # Verify capabilities
   getcap /usr/share/packetbeat/bin/packetbeat
   ```

2. **Packet Drops**:
   ```bash
   # Increase buffer size
   sudo packetbeat -e -c /etc/packetbeat/packetbeat.yml \
     -E "packetbeat.interfaces.buffer_size_mb=256"
   
   # Simplify processing
   sudo packetbeat -e -c /etc/packetbeat/packetbeat.yml \
     -E "packetbeat.protocols.http.include_body_for=[]"
   ```

3. **Protocol Detection Issues**:
   ```bash
   # Add custom ports for protocols
   sudo packetbeat -e -c /etc/packetbeat/packetbeat.yml \
     -E "packetbeat.protocols.http.ports=[80, 8080, 8000, 9000]"
   ```

## Best Practices

### Deployment Strategies

1. **Installation**:
   - Use official packages when possible
   - Keep Packetbeat updated to latest minor version
   - Automate deployment with configuration management
   - Consider containerization with appropriate privileges

2. **Configuration Management**:
   - Use version control for configurations
   - Implement CI/CD for config validation
   - Use templates with environment-specific variables
   - Document all custom configurations

3. **Scaling**:
   - Deploy on network aggregation points
   - Use multiple instances for high-volume networks
   - Consider distributed traffic analysis
   - Implement load balancing for outputs

### Security Recommendations

1. **Encryption**:
   - Enable TLS for all communications
   - Verify certificates properly
   - Use secure key storage
   - Implement proper certificate management

2. **Authentication**:
   - Use dedicated user accounts with minimal privileges
   - Rotate credentials regularly
   - Use environment variables for sensitive information
   - Store credentials in secure locations

3. **Privacy Considerations**:
   - Redact sensitive data (passwords, cookies, tokens)
   - Implement data minimization principles
   - Be aware of regulatory requirements (GDPR, HIPAA)
   - Document data collection policies

### Operational Guidelines

1. **Monitoring**:
   - Set up alerts for Packetbeat failures
   - Monitor packet drop rates
   - Track CPU and memory usage
   - Set up dashboards for network traffic

2. **Capacity Planning**:
   - Estimate traffic volume
   - Allocate appropriate resources
   - Plan for traffic growth
   - Consider peak traffic periods

3. **Upgrades**:
   - Test upgrades in non-production first
   - Plan for breaking changes
   - Consider rolling upgrades for high-availability
   - Archive old data if index templates change

### Configuration Tips

1. **General Best Practices**:
   - Keep configurations simple
   - Use appropriate BPF filters
   - Implement proper logging
   - Validate configurations before deployment

2. **Performance Optimization**:
   - Only capture necessary traffic
   - Use efficient capture methods (af_packet vs pcap)
   - Balance detail level and overhead
   - Monitor and adjust resource usage

3. **Resilience**:
   - Configure proper retry behavior
   - Implement output load balancing
   - Consider secondary outputs for critical data
   - Implement health monitoring

## Use Cases

### Application Performance Monitoring

Monitor web application performance:

```yaml
packetbeat.protocols:
  http:
    ports: [80, 443, 8080, 3000, 5000]
    send_headers: true
    send_all_headers: false
    include_body_for: []
    
processors:
  - add_fields:
      target: ''
      fields:
        use_case: apm
  - drop_event:
      when:
        or:
          - equals:
              http.request.method: OPTIONS
          - contains:
              http.request.uri: "/health"
```

Key metrics to track:
- Response times by endpoint
- HTTP status code distribution
- Error rates
- Request volume

### Database Performance Monitoring

Monitor database query performance:

```yaml
packetbeat.protocols:
  mysql:
    ports: [3306]
    max_query_length: 1000
    query_examples: true
  postgresql:
    ports: [5432]
    
processors:
  - add_fields:
      target: ''
      fields:
        use_case: database_monitoring
  - drop_fields:
      fields: ["mysql.query"]
      when:
        regexp:
          mysql.query: "^SELECT .*FOR UPDATE$"
```

Key metrics to track:
- Query response times
- Failed queries
- Query types (SELECT, INSERT, UPDATE)
- Slow queries

### Network Security Monitoring

Monitor for suspicious network activity:

```yaml
packetbeat.protocols:
  # Enable all relevant protocols
  http:
    ports: [80, 443, 8000-9000]
  dns:
    ports: [53]
  tls:
    ports: [443, 8443]
  
packetbeat.flows:
  enabled: true
  period: 10s
  timeout: 30s
    
processors:
  - add_network_direction:
      internal_networks: ["private", "10.0.0.0/8", "192.168.0.0/16"]
  - add_fields:
      target: ''
      fields:
        use_case: security
  # Alert on suspicious HTTP user agents
  - add_tag:
      tags: ["suspicious_useragent"]
      when:
        regexp:
          http.request.headers.user-agent: "(curl|wget|nikto|gobuster|sqlmap)"
```

Key metrics to track:
- Unusual connection patterns
- Suspicious HTTP requests
- DNS queries to malicious domains
- Connection attempts to unusual ports
- Failed TLS handshakes

### API Monitoring

Track API usage and performance:

```yaml
packetbeat.protocols:
  http:
    ports: [80, 443, 8080, 3000]
    
processors:
  - add_fields:
      target: ''
      fields:
        use_case: api_monitoring
  # Only keep API traffic
  - drop_event:
      when:
        not:
          regexp:
            http.request.uri: "^/api/.*"
  # Add API version tag
  - add_tag:
      tags: ["api_v1"]
      when:
        contains:
          http.request.uri: "/api/v1/"
  - add_tag:
      tags: ["api_v2"]
      when:
        contains:
          http.request.uri: "/api/v2/"
```

Key metrics to track:
- API response times by endpoint
- API error rates
- Request volume by client
- API version usage

### Network Troubleshooting

Diagnose network issues:

```yaml
packetbeat.protocols:
  # Enable relevant protocols
  http:
    ports: [80, 443]
  dns:
    ports: [53]
  icmp:
    enabled: true
  
packetbeat.flows:
  enabled: true
  period: 10s
  timeout: 30s
    
processors:
  - add_fields:
      target: ''
      fields:
        use_case: troubleshooting
  # Add latency ranges
  - add_tag:
      tags: ["high_latency"]
      when:
        range:
          http.response.time_us:
            gt: 500000
  - add_tag:
      tags: ["very_high_latency"]
      when:
        range:
          http.response.time_us:
            gt: 1000000
```

Key metrics to track:
- TCP retransmissions
- Connection timeouts
- Response time outliers
- DNS resolution failures
- Network latency between services

## Conclusion

Packetbeat provides deep visibility into network traffic, enabling real-time monitoring of application communications, performance metrics, and security insights. By properly configuring and optimizing Packetbeat, you can unlock valuable information about your infrastructure without the overhead of traditional network monitoring tools.

In the next chapter, we'll explore other Beats in the Elastic Stack, including Heartbeat for uptime monitoring, and specialized Beats for other use cases.