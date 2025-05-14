# On-Premises Deployments for the ELK Stack

This chapter covers strategies, configurations, and best practices for deploying the ELK Stack in on-premises environments.

## Table of Contents
- [Introduction to On-Premises Deployments](#introduction-to-on-premises-deployments)
- [Hardware Selection and Sizing](#hardware-selection-and-sizing)
- [Operating System Considerations](#operating-system-considerations)
- [Installation Methods](#installation-methods)
- [Networking and Load Balancing](#networking-and-load-balancing)
- [Storage Configuration](#storage-configuration)
- [Production Deployment Architectures](#production-deployment-architectures)
- [Security Hardening](#security-hardening)
- [Integration with On-Premises Systems](#integration-with-on-premises-systems)
- [Virtualization Considerations](#virtualization-considerations)
- [Containerization in On-Premises Environments](#containerization-in-on-premises-environments)

## Introduction to On-Premises Deployments

On-premises deployments of the ELK Stack provide organizations with complete control over their data, infrastructure, and security policies. While cloud deployments offer convenience and managed services, on-premises deployments are often preferred for:

- Strict data sovereignty requirements
- High security or compliance environments
- Existing infrastructure utilization
- Performance optimization for specific workloads
- Cost management for stable, long-term deployments

This chapter provides a comprehensive guide to planning, deploying, and maintaining an on-premises ELK Stack infrastructure.

## Hardware Selection and Sizing

Selecting appropriate hardware is crucial for the performance and reliability of your ELK Stack deployment.

### Elasticsearch Hardware Requirements

#### CPU Considerations

- **Master Nodes**: 8+ cores, focus on clock speed over core count
- **Data Nodes**: 16+ cores, balance between core count and clock speed
- **Coordinating Nodes**: 8+ cores, higher clock speed preferred
- **Machine Learning Nodes**: 16+ cores, AVX2 instruction set support
- **CPU Architecture**: x86_64 preferred for best compatibility

#### Memory Configurations

- **Master Nodes**: 16-32GB RAM
- **Data Nodes**: 32-64GB RAM minimum (scale based on data volume)
- **Coordinating Nodes**: 32GB RAM
- **Machine Learning Nodes**: 64GB+ RAM
- **JVM Heap Sizing**: 50% of available RAM up to 31GB maximum
- **Memory-to-CPU Ratio**: Aim for 4-8GB per core

#### Storage Requirements

- **Master Nodes**: 200GB+ SSD for operating system and logs
- **Data Nodes**:
  - Hot Nodes: NVMe or SSD storage (1-4TB per node)
  - Warm Nodes: SSD or high-performance HDD (4-12TB per node)
  - Cold Nodes: High-capacity HDD (12TB+ per node)
- **RAID Configurations**: RAID 0 for performance, consider RAID 10 for added redundancy

#### Network Specifications

- **Minimum**: 1 Gbps network interfaces
- **Recommended**: 10 Gbps network for data nodes
- **Dedicated Networks**: Separate networks for client, replication, and storage traffic

### Logstash Hardware Requirements

- **CPU**: 4-8 cores per instance
- **Memory**: 8-16GB RAM
- **Storage**: 100GB+ for operating system, plugins, and buffer files
- **Network**: 1-10 Gbps depending on throughput requirements

### Kibana Hardware Requirements

- **CPU**: 4-8 cores per instance
- **Memory**: 8-16GB RAM
- **Storage**: 100GB+ for operating system and logs
- **Network**: 1 Gbps network interfaces

### Hardware Sizing Examples

#### Small Deployment (100GB - 500GB index size)

```
Elasticsearch Cluster:
- 3 combined master/data nodes
  - 8 cores, 32GB RAM per node
  - 1TB SSD storage per node

Logstash:
- 2 Logstash servers
  - 4 cores, 8GB RAM per server

Kibana:
- 2 Kibana servers
  - 4 cores, 8GB RAM per server

Network: 1 Gbps throughout
```

#### Medium Deployment (500GB - 2TB index size)

```
Elasticsearch Cluster:
- 3 dedicated master nodes
  - 8 cores, 16GB RAM per node
  - 200GB SSD storage per node
- 5 data nodes
  - 16 cores, 64GB RAM per node
  - 2TB NVMe storage per node
- 2 coordinating nodes
  - 8 cores, 32GB RAM per node

Logstash:
- 4 Logstash servers
  - 8 cores, 16GB RAM per server

Kibana:
- 3 Kibana servers behind load balancer
  - 8 cores, 16GB RAM per server

Network: 10 Gbps for data nodes, 1 Gbps for others
```

#### Large Deployment (2TB+ index size)

```
Elasticsearch Cluster:
- 3 dedicated master nodes
  - 16 cores, 32GB RAM per node
  - 200GB SSD storage per node
- 10+ hot data nodes
  - 32 cores, 128GB RAM per node
  - 4TB NVMe storage per node
- 5+ warm data nodes
  - 16 cores, 64GB RAM per node
  - 12TB SSD storage per node
- 3+ cold data nodes
  - 16 cores, 64GB RAM per node
  - 24TB HDD storage per node
- 4 coordinating nodes
  - 16 cores, 64GB RAM per node

Logstash:
- 8+ Logstash servers
  - 16 cores, 32GB RAM per server

Kibana:
- 5+ Kibana servers behind load balancer
  - 16 cores, 32GB RAM per server

Network: 10-25 Gbps for data nodes, 10 Gbps for others
```

## Operating System Considerations

The choice of operating system affects performance, management, and compatibility of your ELK Stack deployment.

### Linux Distributions

Linux is the preferred platform for production ELK Stack deployments.

#### Recommended Distributions

- **Ubuntu Server** (18.04 LTS, 20.04 LTS, 22.04 LTS)
- **CentOS** (7.x, 8.x) / **Rocky Linux** (8.x, 9.x)
- **Red Hat Enterprise Linux** (7.x, 8.x, 9.x)
- **Amazon Linux** (2, 2023)

#### System Configuration Recommendations

1. **Kernel Parameters**

```bash
# Add to /etc/sysctl.conf
# Increase file descriptors
fs.file-max = 2097152

# Increase virtual memory areas
vm.max_map_count = 262144

# Increase network buffers
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# Apply changes
sysctl -p
```

2. **User Limits Configuration**

```bash
# Add to /etc/security/limits.conf
elasticsearch soft nofile 65536
elasticsearch hard nofile 65536
elasticsearch soft nproc 4096
elasticsearch hard nproc 4096
elasticsearch soft memlock unlimited
elasticsearch hard memlock unlimited
```

3. **Swappiness Setting**

```bash
# Add to /etc/sysctl.conf
vm.swappiness = 1

# Apply changes
sysctl -p
```

4. **Disable Transparent Huge Pages**

```bash
# Add to /etc/rc.local
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

5. **Time Synchronization**

```bash
# Install chrony for time synchronization
apt-get install -y chrony    # Ubuntu/Debian
# or
yum install -y chrony        # CentOS/RHEL

# Enable and start the service
systemctl enable chronyd
systemctl start chronyd
```

### Windows Considerations

While Linux is recommended, Elasticsearch, Logstash, and Kibana can run on Windows servers.

#### Configuration Adjustments for Windows

1. **JVM Settings**

```
# Adjust in config/jvm.options
-Xms4g
-Xmx4g
```

2. **Windows Service Installation**

```powershell
# Install Elasticsearch as a service
.\bin\elasticsearch-service.bat install

# Install Logstash as a service
nssm install logstash "C:\logstash\bin\logstash.bat" "-f" "C:\logstash\config\logstash.conf"

# Install Kibana as a service
nssm install kibana "C:\kibana\bin\kibana.bat"
```

## Installation Methods

### Package-Based Installation

Package-based installation is recommended for most production deployments.

#### RPM Installation (CentOS/RHEL/Rocky Linux)

```bash
# Import GPG key
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch

# Create repository file
cat > /etc/yum.repos.d/elasticsearch.repo << 'EOL'
[elasticsearch]
name=Elasticsearch repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOL

# Install Elasticsearch
yum install -y elasticsearch

# Install Logstash
yum install -y logstash

# Install Kibana
yum install -y kibana

# Enable services
systemctl daemon-reload
systemctl enable elasticsearch.service
systemctl enable logstash.service
systemctl enable kibana.service

# Start services
systemctl start elasticsearch.service
systemctl start logstash.service
systemctl start kibana.service
```

#### APT Installation (Ubuntu/Debian)

```bash
# Install prerequisites
apt-get update
apt-get install -y apt-transport-https gnupg2

# Import GPG key
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | apt-key add -

# Add repository
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | tee /etc/apt/sources.list.d/elastic-7.x.list

# Install Elasticsearch
apt-get update
apt-get install -y elasticsearch

# Install Logstash
apt-get install -y logstash

# Install Kibana
apt-get install -y kibana

# Enable services
systemctl daemon-reload
systemctl enable elasticsearch.service
systemctl enable logstash.service
systemctl enable kibana.service

# Start services
systemctl start elasticsearch.service
systemctl start logstash.service
systemctl start kibana.service
```

### Configuration Management Deployment

Using configuration management tools like Ansible, Chef, or Puppet is recommended for larger deployments.

#### Sample Ansible Playbook Excerpt

```yaml
---
- name: Deploy Elasticsearch Cluster
  hosts: elasticsearch_nodes
  become: true
  vars:
    elasticsearch_version: "7.17.8"
    cluster_name: "production-cluster"
    network_host: "0.0.0.0"
    discovery_seed_hosts: ["es-node1", "es-node2", "es-node3"]
    initial_master_nodes: ["es-node1", "es-node2", "es-node3"]
    
  tasks:
    - name: Import Elasticsearch GPG key
      rpm_key:
        key: https://artifacts.elastic.co/GPG-KEY-elasticsearch
        state: present
      when: ansible_os_family == "RedHat"
    
    - name: Add Elasticsearch repository
      yum_repository:
        name: elasticsearch
        description: Elasticsearch repository for 7.x packages
        baseurl: https://artifacts.elastic.co/packages/7.x/yum
        gpgcheck: yes
        gpgkey: https://artifacts.elastic.co/GPG-KEY-elasticsearch
        enabled: yes
      when: ansible_os_family == "RedHat"
    
    - name: Install Elasticsearch
      package:
        name: elasticsearch
        state: present
    
    - name: Configure Elasticsearch
      template:
        src: elasticsearch.yml.j2
        dest: /etc/elasticsearch/elasticsearch.yml
      notify: restart elasticsearch
    
    - name: Configure JVM options
      template:
        src: jvm.options.j2
        dest: /etc/elasticsearch/jvm.options
      notify: restart elasticsearch
    
    - name: Start and enable Elasticsearch service
      systemd:
        name: elasticsearch
        state: started
        enabled: yes
  
  handlers:
    - name: restart elasticsearch
      systemd:
        name: elasticsearch
        state: restarted
```

### Docker-Based On-Premises Deployment

Docker can be used for on-premises deployments, particularly in development or smaller production environments.

#### Docker Compose Example

```yaml
version: '3'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.8
    container_name: elasticsearch
    environment:
      - node.name=elasticsearch
      - cluster.name=docker-cluster
      - discovery.seed_hosts=elasticsearch
      - cluster.initial_master_nodes=elasticsearch
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms4g -Xmx4g"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
      - 9300:9300
    networks:
      - elk

  logstash:
    image: docker.elastic.co/logstash/logstash:7.17.8
    container_name: logstash
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml
    ports:
      - 5044:5044
      - 5000:5000/tcp
      - 5000:5000/udp
      - 9600:9600
    environment:
      LS_JAVA_OPTS: "-Xmx1g -Xms1g"
    networks:
      - elk
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:7.17.8
    container_name: kibana
    volumes:
      - ./kibana/config/kibana.yml:/usr/share/kibana/config/kibana.yml
    ports:
      - 5601:5601
    networks:
      - elk
    depends_on:
      - elasticsearch

networks:
  elk:
    driver: bridge

volumes:
  elasticsearch-data:
    driver: local
```

## Networking and Load Balancing

Proper network configuration is critical for high-performance, resilient ELK Stack deployments.

### Network Architecture

1. **Multi-Tier Network Design**
   - Public network tier for client access
   - Private network tier for inter-node communication
   - Management network tier for administration

2. **Traffic Segregation**
   - Client traffic (HTTP/REST APIs)
   - Node-to-node traffic (transport protocol)
   - Monitoring and management traffic

### Load Balancing

#### HAProxy Configuration Example

```
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode http
    option httplog
    option dontlognull
    timeout connect 5000
    timeout client 50000
    timeout server 50000

frontend elasticsearch_http
    bind *:9200
    default_backend elasticsearch_http_backend
    mode http
    option forwardfor

backend elasticsearch_http_backend
    mode http
    balance roundrobin
    option httpchk GET / HTTP/1.1\r\nHost:\ localhost
    http-check expect status 200
    server es-node1 es-node1:9200 check
    server es-node2 es-node2:9200 check
    server es-node3 es-node3:9200 check

frontend kibana_http
    bind *:5601
    default_backend kibana_http_backend
    mode http
    option forwardfor

backend kibana_http_backend
    mode http
    balance roundrobin
    option httpchk GET /api/status HTTP/1.1\r\nHost:\ localhost
    http-check expect status 200
    server kibana1 kibana1:5601 check
    server kibana2 kibana2:5601 check
```

#### NGINX Configuration Example

```nginx
http {
    upstream elasticsearch {
        server es-node1:9200;
        server es-node2:9200;
        server es-node3:9200;
        keepalive 15;
    }

    upstream kibana {
        server kibana1:5601;
        server kibana2:5601;
        server kibana3:5601;
        keepalive 15;
    }

    server {
        listen 9200;
        server_name elastic.example.com;

        location / {
            proxy_pass http://elasticsearch;
            proxy_http_version 1.1;
            proxy_set_header Connection "Keep-Alive";
            proxy_set_header Proxy-Connection "Keep-Alive";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }

    server {
        listen 5601;
        server_name kibana.example.com;

        location / {
            proxy_pass http://kibana;
            proxy_http_version 1.1;
            proxy_set_header Connection "Keep-Alive";
            proxy_set_header Proxy-Connection "Keep-Alive";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

## Storage Configuration

Proper storage configuration is critical for Elasticsearch performance and reliability.

### File System Recommendations

- **XFS**: Recommended for most deployments
- **Ext4**: Good alternative when XFS is not available
- **JFS/ReiserFS**: Not recommended due to potential stability issues with mmapfs
- **NFS/Network Storage**: Generally not recommended except for snapshots

### File System Mount Options

```bash
# XFS Mount Options in /etc/fstab
UUID=xxx /data xfs noatime,nodiratime,logbufs=8,logbsize=256k,largeio,inode64,swalloc,allocsize=64m 0 0

# Ext4 Mount Options in /etc/fstab
UUID=yyy /data ext4 noatime,nodiratime,data=writeback,barrier=0,nobh,errors=remount-ro 0 0
```

### Local Storage Configuration

1. **RAID Configurations**
   - RAID 0: Highest performance, no redundancy
   - RAID 10: Good performance with redundancy
   - RAID 5/6: Not recommended due to poor write performance

2. **Storage Tiering Example**

```
Hot Tier Nodes:
- NVMe SSDs in RAID 0
- Mount path: /var/lib/elasticsearch/hot-data

Warm Tier Nodes:
- SSDs in RAID 10
- Mount path: /var/lib/elasticsearch/warm-data

Cold Tier Nodes:
- HDDs in RAID 10
- Mount path: /var/lib/elasticsearch/cold-data
```

### SAN/NAS Considerations

When using SAN or NAS storage with Elasticsearch:

- Ensure low latency (< 1ms) for storage operations
- Provision dedicated LUNs for Elasticsearch
- Use dedicated network connectivity for storage
- Avoid thin provisioning
- Use block storage protocols (iSCSI, FC) rather than file-based (NFS)
- Set appropriate queue depths on hosts and storage systems

## Production Deployment Architectures

### Small-Scale Architecture (3-5 nodes)

In smaller environments, multi-role nodes are common.

```
+-------------------------------+
|         Load Balancer         |
+-------------------------------+
             |
             v
+-------------------------------+
|   Combined Master/Data Nodes  |
|             x3               |
+-------------------------------+
             |
             v
+-------------------------------+
|    Logstash & Beats Servers   |
|             x2               |
+-------------------------------+
             |
             v
+-------------------------------+
|        Kibana Servers         |
|             x2               |
+-------------------------------+
```

#### Configuration Example (Combined Nodes)

```yaml
# elasticsearch.yml for combined nodes
cluster.name: small-production
node.name: es-node-${NODE_ID}
node.master: true
node.data: true
node.ingest: true
network.host: 0.0.0.0
discovery.seed_hosts: ["es-node-1", "es-node-2", "es-node-3"]
cluster.initial_master_nodes: ["es-node-1", "es-node-2", "es-node-3"]
bootstrap.memory_lock: true
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
```

### Medium-Scale Architecture (6-10 nodes)

Separate node roles for improved performance and stability.

```
+-------------------------------+
|         Load Balancer         |
+-------------------------------+
             |
             v
+-------------------------------+
|     Dedicated Master Nodes    |
|             x3               |
+-------------------------------+
             |
             v
+-------------------------------+
|        Hot Data Nodes         |
|             x3               |
+-------------------------------+
             |
             v
+-------------------------------+
|     Coordinating Nodes (I/O)  |
|             x2               |
+-------------------------------+
             |
             v
+-------------------------------+
|    Logstash & Beats Servers   |
|             x3               |
+-------------------------------+
             |
             v
+-------------------------------+
|        Kibana Servers         |
|             x2               |
+-------------------------------+
```

#### Configuration Examples (Dedicated Roles)

```yaml
# elasticsearch.yml for master nodes
cluster.name: medium-production
node.name: master-${NODE_ID}
node.roles: [ master ]
network.host: 0.0.0.0
discovery.seed_hosts: ["master-1", "master-2", "master-3"]
cluster.initial_master_nodes: ["master-1", "master-2", "master-3"]
bootstrap.memory_lock: true
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
```

```yaml
# elasticsearch.yml for data nodes
cluster.name: medium-production
node.name: data-${NODE_ID}
node.roles: [ data ]
node.attr.data: hot
network.host: 0.0.0.0
discovery.seed_hosts: ["master-1", "master-2", "master-3"]
bootstrap.memory_lock: true
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
```

```yaml
# elasticsearch.yml for coordinating nodes
cluster.name: medium-production
node.name: coord-${NODE_ID}
node.roles: [ ]
network.host: 0.0.0.0
discovery.seed_hosts: ["master-1", "master-2", "master-3"]
bootstrap.memory_lock: true
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
```

### Large-Scale Architecture (10+ nodes)

Complex deployment with specialized node roles and data tiering.

```
+-------------------------------+
|         Load Balancer         |
+-------------------------------+
             |
             v
+-------------------------------+
|     Dedicated Master Nodes    |
|             x3               |
+-------------------------------+
             |
             v
+------------------------+----------------------+----------------------+
|     Hot Data Nodes     |    Warm Data Nodes   |    Cold Data Nodes   |
|         x6+            |        x4+           |         x2+          |
+------------------------+----------------------+----------------------+
             |
             v
+-------------------------------+
|     Coordinating Nodes (I/O)  |
|             x4               |
+-------------------------------+
             |
             v
+-------------------------------+
|     Ingest/ML/APM Nodes      |
|             x3               |
+-------------------------------+
             |
             v
+------------------------+----------------------+
|    Logstash Cluster    |    Beats Forwarders  |
|         x6+            |        x10+          |
+------------------------+----------------------+
             |
             v
+-------------------------------+
|        Kibana Cluster         |
|             x4+              |
+-------------------------------+
```

#### Data Tiering Configuration Example

```yaml
# elasticsearch.yml for hot data nodes
cluster.name: large-production
node.name: hot-${NODE_ID}
node.roles: [ data_hot, data_content ]
node.attr.data: hot
network.host: 0.0.0.0
discovery.seed_hosts: ["master-1", "master-2", "master-3"]
bootstrap.memory_lock: true
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
```

```yaml
# elasticsearch.yml for warm data nodes
cluster.name: large-production
node.name: warm-${NODE_ID}
node.roles: [ data_warm ]
node.attr.data: warm
network.host: 0.0.0.0
discovery.seed_hosts: ["master-1", "master-2", "master-3"]
bootstrap.memory_lock: true
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
```

```yaml
# elasticsearch.yml for cold data nodes
cluster.name: large-production
node.name: cold-${NODE_ID}
node.roles: [ data_cold ]
node.attr.data: cold
network.host: 0.0.0.0
discovery.seed_hosts: ["master-1", "master-2", "master-3"]
bootstrap.memory_lock: true
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
```

## Security Hardening

On-premises deployments require comprehensive security controls.

### Network-Level Security

1. **Firewall Rules Example**

```bash
# Allow Elasticsearch HTTP API (client) traffic
iptables -A INPUT -p tcp --dport 9200 -s 10.0.0.0/24 -j ACCEPT

# Allow Elasticsearch transport traffic between nodes
iptables -A INPUT -p tcp --dport 9300 -s 10.1.0.0/24 -j ACCEPT

# Allow Kibana traffic
iptables -A INPUT -p tcp --dport 5601 -s 10.0.0.0/24 -j ACCEPT

# Allow Logstash beats input
iptables -A INPUT -p tcp --dport 5044 -s 10.2.0.0/24 -j ACCEPT

# Drop all other traffic to these ports
iptables -A INPUT -p tcp --dport 9200 -j DROP
iptables -A INPUT -p tcp --dport 9300 -j DROP
iptables -A INPUT -p tcp --dport 5601 -j DROP
iptables -A INPUT -p tcp --dport 5044 -j DROP
```

2. **TLS Configuration**

```yaml
# elasticsearch.yml excerpt for TLS
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.key: certs/elasticsearch.key
xpack.security.http.ssl.certificate: certs/elasticsearch.crt
xpack.security.http.ssl.certificate_authorities: certs/ca.crt
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.key: certs/elasticsearch.key
xpack.security.transport.ssl.certificate: certs/elasticsearch.crt
xpack.security.transport.ssl.certificate_authorities: certs/ca.crt
```

### Authentication and Authorization

1. **Native Authentication Setup**

```bash
# Create passwords for built-in users
bin/elasticsearch-setup-passwords interactive
```

2. **Role-Based Access Control Example**

```json
PUT _security/role/logs_viewer
{
  "cluster": [ "monitor" ],
  "indices": [
    {
      "names": [ "logs-*" ],
      "privileges": [ "read", "view_index_metadata" ],
      "field_security": {
        "grant": [ "timestamp", "message", "host.name", "source.ip", "event.*" ]
      },
      "query": {
        "term": {
          "environment": "production"
        }
      }
    }
  ]
}
```

3. **LDAP Integration Example**

```yaml
# elasticsearch.yml excerpt for LDAP
xpack.security.authc.realms.ldap.ldap1:
  order: 0
  url: "ldaps://ldap.example.com:636"
  bind_dn: "cn=elasticsearch,ou=services,dc=example,dc=com"
  bind_password: ${LDAP_BIND_PASSWORD}
  user_search:
    base_dn: "ou=users,dc=example,dc=com"
    filter: "(cn={0})"
  group_search:
    base_dn: "ou=groups,dc=example,dc=com"
    filter: "(member={0})"
  user_dn_templates:
    - "cn={0},ou=users,dc=example,dc=com"
  unmapped_groups_as_roles: false
  files:
    role_mapping: "config/role_mapping.yml"
```

### Encryption and Data Protection

1. **Encrypted Settings Configuration**

```bash
# Create keystore
bin/elasticsearch-keystore create

# Add sensitive settings
bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password
bin/elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password
bin/elasticsearch-keystore add xpack.security.http.ssl.keystore.secure_password
bin/elasticsearch-keystore add xpack.security.http.ssl.truststore.secure_password
```

2. **Encryption at Rest**

```yaml
# elasticsearch.yml excerpt for encryption
xpack.security.encryption.key: ${ENCRYPTION_KEY}
xpack.security.encryptionKey: ${ENCRYPTION_KEY}
```

## Integration with On-Premises Systems

### Directory Services Integration

1. **Active Directory Integration**

```yaml
# elasticsearch.yml excerpt for AD integration
xpack.security.authc.realms.active_directory.ad1:
  order: 0
  domain_name: EXAMPLE.COM
  url: ldaps://ad.example.com:636
  unmapped_groups_as_roles: false
  files:
    role_mapping: "config/role_mapping.yml"
```

2. **Role Mapping Configuration**

```yaml
# role_mapping.yml
EXAMPLE.COM\\elasticsearch_admins:
  - superuser

EXAMPLE.COM\\kibana_users:
  - kibana_user

EXAMPLE.COM\\log_analysts:
  - logs_viewer
```

### SIEM and SOC Integration

1. **Integration with On-Premises SIEM**

```yaml
# logstash.conf excerpt for SIEM integration
output {
  elasticsearch {
    hosts => ["https://elasticsearch:9200"]
    index => "siem-events-%{+YYYY.MM.dd}"
    user => "logstash_internal"
    password => "${LOGSTASH_PASSWORD}"
    ssl => true
    ssl_certificate_verification => true
    cacert => "/etc/logstash/certs/ca.crt"
  }
  
  # Forward to on-premises SIEM
  tcp {
    host => "siem.example.com"
    port => 514
    codec => json
  }
}
```

### Enterprise Monitoring Systems

1. **Integration with Prometheus**

```yaml
# prometheus.yml excerpt
scrape_configs:
  - job_name: 'elasticsearch'
    scrape_interval: 15s
    scheme: https
    tls_config:
      ca_file: /etc/prometheus/certs/ca.crt
    basic_auth:
      username: 'monitor_user'
      password: 'monitor_password'
    metrics_path: '/_prometheus/metrics'
    static_configs:
      - targets: 
        - 'es-node1:9200'
        - 'es-node2:9200'
        - 'es-node3:9200'
```

2. **Integration with Nagios/Icinga**

```bash
#!/bin/bash
# check_elasticsearch.sh
HOST="$1"
PORT="$2"
USER="$3"
PASS="$4"

HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" -u "${USER}:${PASS}" "https://${HOST}:${PORT}/_cluster/health")
STATUS=$(curl -s -u "${USER}:${PASS}" "https://${HOST}:${PORT}/_cluster/health" | jq -r '.status')

if [ $HTTP_CODE -ne 200 ]; then
  echo "CRITICAL - Connection failed with HTTP code ${HTTP_CODE}"
  exit 2
fi

if [ "$STATUS" = "green" ]; then
  echo "OK - Elasticsearch cluster is green"
  exit 0
elif [ "$STATUS" = "yellow" ]; then
  echo "WARNING - Elasticsearch cluster is yellow"
  exit 1
else
  echo "CRITICAL - Elasticsearch cluster is red"
  exit 2
fi
```

## Virtualization Considerations

### VMware/vSphere Deployment

1. **VM Configuration Best Practices**

- Use reservations for CPU and memory resources
- Enable memory overcommitment
- Use paravirtual SCSI controllers for data disks
- Use VMXNET3 network adapters
- Consider vSphere DRS affinity rules for node distribution
- Use host anti-affinity rules to distribute master nodes

2. **Sample vSphere Deployment Parameters**

```
Elasticsearch Data Node VM Configuration:
CPU: 16 vCPUs (reserved)
Memory: 64GB RAM (reserved)
Network: VMXNET3, 10Gbps
Storage:
  - OS Disk: 100GB Thin Provision
  - Data Disk: 2TB Thick Provision Eager Zeroed, Paravirtual SCSI
  - Journal Disk: 100GB Thick Provision Eager Zeroed, Paravirtual SCSI
```

### KVM/RHEV Deployment

1. **VM Configuration Best Practices**

- Use virtio drivers for disk and network devices
- Configure CPU pinning for consistent performance
- Use huge pages for memory allocation
- Configure NUMA topology awareness

2. **Sample Deployment Script Snippet**

```bash
# Create Elasticsearch data node VM
virt-install \
  --name es-data-01 \
  --memory 65536 \
  --memtune hard_limit=68719476736 \
  --cpu host --vcpus=16 \
  --numatune mode=strict \
  --os-variant rhel8.3 \
  --disk path=/var/lib/libvirt/images/es-data-01-os.qcow2,size=100,format=qcow2,bus=virtio \
  --disk path=/var/lib/libvirt/images/es-data-01-data.raw,size=2048,format=raw,bus=virtio \
  --network bridge=br0,model=virtio \
  --graphics none \
  --noautoconsole \
  --location /path/to/rhel8.iso
```

## Containerization in On-Premises Environments

Even in on-premises environments, container orchestration can provide flexibility and automation benefits.

### Kubernetes On-Premises

See the Kubernetes chapter for detailed guidance on Kubernetes deployments. Key considerations for on-premises Kubernetes:

- Storage classes and persistent volumes
- Network policies
- Node affinity and anti-affinity
- Resource quotas and limits
- Container security

### Docker Swarm On-Premises

For smaller containerized deployments, Docker Swarm can be simpler to manage than Kubernetes.

#### Swarm Deployment Example

```yaml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.8
    environment:
      - node.name=elasticsearch-${NODE_ID}
      - cluster.name=swarm-cluster
      - discovery.seed_hosts=elasticsearch-1,elasticsearch-2,elasticsearch-3
      - cluster.initial_master_nodes=elasticsearch-1,elasticsearch-2,elasticsearch-3
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms8g -Xmx8g"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
      - ./config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - ./config/certs:/usr/share/elasticsearch/config/certs
    networks:
      - elk
    deploy:
      mode: global
      placement:
        constraints:
          - node.labels.elasticsearch == true
      resources:
        limits:
          cpus: '8'
          memory: 16G
        reservations:
          cpus: '4'
          memory: 8G

  logstash:
    image: docker.elastic.co/logstash/logstash:7.17.8
    volumes:
      - ./config/logstash/pipeline:/usr/share/logstash/pipeline
      - ./config/logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml
    networks:
      - elk
    deploy:
      mode: replicated
      replicas: 3
      placement:
        constraints:
          - node.labels.logstash == true
      resources:
        limits:
          cpus: '4'
          memory: 8G

  kibana:
    image: docker.elastic.co/kibana/kibana:7.17.8
    volumes:
      - ./config/kibana/kibana.yml:/usr/share/kibana/config/kibana.yml
    networks:
      - elk
    deploy:
      mode: replicated
      replicas: 2
      placement:
        constraints:
          - node.labels.kibana == true
      resources:
        limits:
          cpus: '2'
          memory: 4G

networks:
  elk:
    driver: overlay

volumes:
  elasticsearch-data:
    driver: local
```

## Summary

On-premises deployments of the ELK Stack provide complete control over infrastructure, security, and integration with existing systems. Proper planning for hardware, operating systems, networking, storage, and security is essential for a successful deployment.

Key takeaways for on-premises deployments:

1. **Hardware Selection**: Choose appropriate CPU, memory, and storage configurations based on workload requirements
2. **Operating System**: Optimize Linux systems for Elasticsearch performance
3. **Network Architecture**: Implement proper traffic segregation and load balancing
4. **Storage Configuration**: Use local storage where possible with appropriate file systems
5. **Security**: Implement comprehensive security controls at all levels
6. **Integration**: Connect with existing enterprise systems
7. **Deployment Architecture**: Scale from small to large based on organizational needs

On-premises deployments can be customized to meet specific organizational requirements while maintaining complete control over data and infrastructure. While cloud deployments offer convenience, on-premises deployments remain important for many enterprise use cases.

## References

- [Elasticsearch Production Deployment](https://www.elastic.co/guide/en/elasticsearch/reference/current/deploy-elasticsearch.html)
- [Elasticsearch System Configuration](https://www.elastic.co/guide/en/elasticsearch/reference/current/system-config.html)
- [Elasticsearch Security](https://www.elastic.co/guide/en/elasticsearch/reference/current/secure-cluster.html)
- [Logstash Production Deployment](https://www.elastic.co/guide/en/logstash/current/deploying-and-scaling.html)
- [Kibana Production Deployment](https://www.elastic.co/guide/en/kibana/current/production.html)