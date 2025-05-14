# Logging Architecture for Kubernetes

## Introduction

Implementing a robust logging architecture for Kubernetes environments requires understanding both Kubernetes-specific logging challenges and how to effectively leverage the ELK Stack to address them. This chapter covers comprehensive logging architecture designs for Kubernetes, from basic setups to advanced enterprise-ready solutions.

## Understanding Kubernetes Logging Challenges

Kubernetes presents unique challenges for logging:

### Container Ephemerality

- Containers start and stop frequently
- Pod logs are lost when pods are deleted
- Containers may crash before writing logs

### Log Output Diversity

- Standard output/error logs (stdout/stderr)
- Application logs written to files
- System and host logs
- Control plane component logs

### Multi-Node Distribution

- Pods distributed across multiple nodes
- Need for node-level and cluster-level collection
- Log aggregation across the entire cluster

### Resource Constraints

- Log collection agents compete for resources
- Need to balance logging detail with resource usage
- Prevent logging components from impacting application performance

## Kubernetes Logging Fundamentals

### Log Types in Kubernetes

1. **Container Logs**: Application logs sent to stdout/stderr
2. **Application Logs**: Logs written to files within containers
3. **Node Logs**: System logs from Kubernetes nodes
4. **Control Plane Logs**: Logs from Kubernetes API server, controller manager, scheduler, etc.
5. **Audit Logs**: Kubernetes API audit trail

### Native Kubernetes Logging

Kubernetes provides basic logging capabilities:

- `kubectl logs pod-name` to view container logs
- Container Runtime Log Rotation
- Log files typically stored in `/var/log/containers/` on nodes

Limitations of native logging:
- No built-in aggregation across nodes
- Limited retention and search capabilities
- No structured parsing or analysis

## ELK Stack Logging Architecture for Kubernetes

### Design Principles

1. **Resilience**: Ensure logging continues despite node failures
2. **Performance**: Minimize impact on application workloads
3. **Scalability**: Handle log volume growth as cluster expands
4. **Security**: Protect log data and comply with security policies
5. **Usability**: Make logs easily accessible and queryable

### Architecture Patterns

#### Pattern 1: Sidecar Container

Deploy a logging agent (Filebeat) as a sidecar in each pod:

![Sidecar Pattern](https://i.imgur.com/5NLxvqO.png)

**Advantages**:
- Fine-grained control over collection
- Independent scaling per application
- Can collect logs from files, not just stdout

**Disadvantages**:
- Increases resource usage per pod
- Requires pod specification modification
- More complex to manage

**Implementation Example**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
spec:
  containers:
  - name: app
    image: my-app:1.0
    volumeMounts:
    - name: logs-volume
      mountPath: /var/log/app
  - name: filebeat-sidecar
    image: docker.elastic.co/beats/filebeat:8.8.0
    volumeMounts:
    - name: logs-volume
      mountPath: /var/log/app
      readOnly: true
    - name: filebeat-config
      mountPath: /usr/share/filebeat/filebeat.yml
      subPath: filebeat.yml
  volumes:
  - name: logs-volume
    emptyDir: {}
  - name: filebeat-config
    configMap:
      name: filebeat-config
```

#### Pattern 2: Node-level DaemonSet

Deploy a logging agent (Filebeat) as a DaemonSet running on each node:

![DaemonSet Pattern](https://i.imgur.com/VCqb4RP.png)

**Advantages**:
- Simplified deployment (one agent per node)
- Lower resource overhead
- No application pod modifications required

**Disadvantages**:
- Less isolation between applications
- May miss logs from short-lived containers
- Node-level resource constraints impact all logging

**Implementation Example**:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: logging
spec:
  selector:
    matchLabels:
      app: filebeat
  template:
    metadata:
      labels:
        app: filebeat
    spec:
      serviceAccountName: filebeat
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:8.8.0
        args: ["-c", "/etc/filebeat.yml", "-e"]
        volumeMounts:
        - name: config
          mountPath: /etc/filebeat.yml
          subPath: filebeat.yml
        - name: varlog
          mountPath: /var/log
          readOnly: true
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: filebeat-config
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

#### Pattern 3: Logstash as Aggregator

Filebeat DaemonSet with Logstash deployment for processing:

![Logstash Aggregator Pattern](https://i.imgur.com/t1vCu8r.png)

**Advantages**:
- Centralizes log processing
- Reduces direct load on Elasticsearch
- Advanced parsing and filtering capabilities

**Disadvantages**:
- Additional component to manage
- Potential bottleneck for high-volume logs
- More complex configuration

**Implementation**:

Filebeat Configuration:
```yaml
filebeat.autodiscover:
  providers:
    - type: kubernetes
      node: ${NODE_NAME}
      hints.enabled: true
output.logstash:
  hosts: ["logstash:5044"]
```

Logstash Deployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash
  namespace: logging
spec:
  replicas: 2
  selector:
    matchLabels:
      app: logstash
  template:
    metadata:
      labels:
        app: logstash
    spec:
      containers:
      - name: logstash
        image: docker.elastic.co/logstash/logstash:8.8.0
        ports:
        - containerPort: 5044
        volumeMounts:
        - name: config-volume
          mountPath: /usr/share/logstash/pipeline
      volumes:
      - name: config-volume
        configMap:
          name: logstash-pipeline
```

#### Pattern 4: Direct to Elasticsearch

Filebeat sends logs directly to Elasticsearch with ingest pipelines for processing:

![Direct to Elasticsearch Pattern](https://i.imgur.com/JBf3Y88.png)

**Advantages**:
- Simpler architecture (no Logstash)
- Lower resource requirements
- Fewer moving parts to manage

**Disadvantages**:
- Limited transformation capabilities
- Pushes processing load to Elasticsearch
- Less buffering during Elasticsearch outages

**Implementation**:

Filebeat Configuration:
```yaml
filebeat.autodiscover:
  providers:
    - type: kubernetes
      node: ${NODE_NAME}
      hints.enabled: true
output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  pipeline: kubernetes-logs
```

Ingest Pipeline:
```
PUT _ingest/pipeline/kubernetes-logs
{
  "description": "Process Kubernetes logs",
  "processors": [
    {
      "json": {
        "field": "message",
        "target_field": "parsed",
        "ignore_failure": true
      }
    },
    {
      "user_agent": {
        "field": "parsed.user_agent",
        "target_field": "user_agent",
        "ignore_missing": true,
        "ignore_failure": true
      }
    },
    {
      "grok": {
        "field": "kubernetes.container.name",
        "patterns": ["%{DATA:service}-%{NOTSPACE}"],
        "ignore_failure": true
      }
    }
  ]
}
```

### Log Collection Strategies

#### Container Log Collection

Collecting logs from container stdout/stderr:

```yaml
filebeat.inputs:
- type: container
  paths:
    - /var/lib/docker/containers/*/*.log
```

With Kubernetes metadata enrichment:

```yaml
processors:
  - add_kubernetes_metadata:
      host: ${NODE_NAME}
      matchers:
      - logs_path:
          logs_path: "/var/log/containers/"
```

#### Application Log Collection

For applications writing logs to files:

```yaml
filebeat.autodiscover:
  providers:
    - type: kubernetes
      node: ${NODE_NAME}
      templates:
        - condition:
            contains:
              kubernetes.labels.app: java-app
          config:
            - type: filestream
              paths:
                - /var/log/containers/${data.kubernetes.container.id}.log
                - /var/log/app/*.log  # Custom log path
```

Using Kubernetes annotations for hints:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: java-app
  annotations:
    co.elastic.logs/enabled: "true"
    co.elastic.logs/fileset.stdout: "access"
    co.elastic.logs/fileset.stderr: "error"
    co.elastic.logs/multiline.pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2}'
    co.elastic.logs/multiline.negate: "true"
    co.elastic.logs/multiline.match: "after"
spec:
  containers:
  - name: java-app
    image: my-java-app:1.0
```

#### Node Log Collection

Collecting system logs with Filebeat system module:

```yaml
filebeat.modules:
- module: system
  syslog:
    enabled: true
  auth:
    enabled: true
```

#### Control Plane Log Collection

Deploy Filebeat to collect Kubernetes control plane logs:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: control-plane-filebeat
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: control-plane-filebeat
  template:
    metadata:
      labels:
        app: control-plane-filebeat
    spec:
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:8.8.0
        volumeMounts:
        - name: config
          mountPath: /usr/share/filebeat/filebeat.yml
          subPath: filebeat.yml
        - name: kube-api-logs
          mountPath: /var/log/kube-apiserver
        - name: kube-scheduler-logs
          mountPath: /var/log/kube-scheduler
        - name: kube-controller-logs
          mountPath: /var/log/kube-controller-manager
      volumes:
      - name: config
        configMap:
          name: control-plane-filebeat-config
      - name: kube-api-logs
        hostPath:
          path: /var/log/kube-apiserver
      - name: kube-scheduler-logs
        hostPath:
          path: /var/log/kube-scheduler
      - name: kube-controller-logs
        hostPath:
          path: /var/log/kube-controller-manager
```

### Log Processing

#### Kubernetes Metadata Enrichment

Enhance logs with Kubernetes metadata:

```
filter {
  if [kubernetes] {
    mutate {
      add_field => {
        "[@metadata][target_index]" => "logs-%{[kubernetes][namespace]}-%{+YYYY.MM.dd}"
      }
    }
  }
}
```

#### Multiline Log Handling

Configure Filebeat for multiline logs:

```yaml
filebeat.autodiscover:
  providers:
    - type: kubernetes
      templates:
        - condition:
            contains:
              kubernetes.labels.app: java-app
          config:
            - type: filestream
              paths:
                - /var/log/containers/*${data.kubernetes.container.id}.log
              multiline:
                pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2}'
                negate: true
                match: after
```

For Logstash:

```
filter {
  if [kubernetes][container][name] == "java-app" {
    multiline {
      pattern => "^[0-9]{4}-[0-9]{2}-[0-9]{2}"
      negate => true
      what => "previous"
    }
  }
}
```

#### JSON Log Parsing

Parse JSON formatted logs:

```
filter {
  if [message] =~ /^\{.*\}$/ {
    json {
      source => "message"
      target => "parsed_json"
    }
    
    # Flatten important fields
    if [parsed_json][level] {
      mutate {
        add_field => { "log_level" => "%{[parsed_json][level]}" }
      }
    }
  }
}
```

#### Log Routing by Namespace

Route logs to different indices based on Kubernetes namespace:

```
output {
  if [kubernetes][namespace] =~ /^prod-/ {
    elasticsearch {
      hosts => ["elasticsearch:9200"]
      index => "logs-prod-%{[kubernetes][namespace]}-%{+YYYY.MM.dd}"
    }
  } else if [kubernetes][namespace] =~ /^dev-/ {
    elasticsearch {
      hosts => ["elasticsearch:9200"]
      index => "logs-dev-%{[kubernetes][namespace]}-%{+YYYY.MM.dd}"
    }
  } else {
    elasticsearch {
      hosts => ["elasticsearch:9200"]
      index => "logs-other-%{+YYYY.MM.dd}"
    }
  }
}
```

### Advanced Log Management

#### Index Lifecycle Management

Create an ILM policy for Kubernetes logs:

```
PUT _ilm/policy/kubernetes-logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_size": "50GB",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "3d",
        "actions": {
          "forcemerge": {
            "max_num_segments": 1
          },
          "allocate": {
            "include": {
              "data": "warm"
            }
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "allocate": {
            "include": {
              "data": "cold"
            }
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

#### Index Templates

Create index templates for Kubernetes logs:

```
PUT _index_template/kubernetes-logs
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1,
      "index.lifecycle.name": "kubernetes-logs-policy",
      "index.lifecycle.rollover_alias": "logs"
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "kubernetes": {
          "properties": {
            "namespace": { "type": "keyword" },
            "pod": {
              "properties": {
                "name": { "type": "keyword" },
                "uid": { "type": "keyword" }
              }
            },
            "container": {
              "properties": {
                "name": { "type": "keyword" },
                "image": { "type": "keyword" }
              }
            },
            "node": {
              "properties": {
                "name": { "type": "keyword" }
              }
            },
            "labels": {
              "type": "object",
              "dynamic": true
            },
            "annotations": {
              "type": "object",
              "dynamic": true
            }
          }
        },
        "log": {
          "properties": {
            "level": { "type": "keyword" },
            "logger": { "type": "keyword" }
          }
        },
        "message": { "type": "text" }
      }
    }
  }
}
```

#### Storage Class Optimization

Use appropriate storage classes for Elasticsearch indices:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: elasticsearch-hot
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: elasticsearch-warm
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
```

Then configure Elasticsearch nodes to use these storage classes.

## Enterprise Architectures

### Multi-Cluster Logging

#### Architecture Overview

![Multi-Cluster Logging](https://i.imgur.com/MVtDpwM.png)

Components:
1. Filebeat on each Kubernetes cluster
2. Central Logstash layer for aggregation
3. Cross-cluster compatible Elasticsearch
4. Single Kibana for visualization

#### Implementation

1. **Cluster Identification**:

```yaml
# filebeat.yml for Cluster A
filebeat.autodiscover:
  providers:
    - type: kubernetes
      node: ${NODE_NAME}
processors:
  - add_fields:
      target: cluster
      fields:
        name: "cluster-a"
        region: "us-east"
```

2. **Central Logstash Configuration**:

```
input {
  beats {
    port => 5044
    ssl => true
    ssl_certificate => "/etc/logstash/certs/logstash.crt"
    ssl_key => "/etc/logstash/certs/logstash.key"
  }
}

filter {
  if [cluster][name] == "cluster-a" {
    mutate {
      add_field => { "[@metadata][target_index]" => "logs-cluster-a-%{[kubernetes][namespace]}-%{+YYYY.MM.dd}" }
    }
  } else if [cluster][name] == "cluster-b" {
    mutate {
      add_field => { "[@metadata][target_index]" => "logs-cluster-b-%{[kubernetes][namespace]}-%{+YYYY.MM.dd}" }
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "%{[@metadata][target_index]}"
    ssl => true
    ssl_certificate_verification => true
  }
}
```

### High-Volume Architectures

#### Buffer Management

For high-volume logging, implement persistent queues in Logstash:

```yaml
# logstash.yml
queue.type: persisted
queue.max_bytes: 4gb
```

#### Scaling Strategies

1. **Horizontal Scaling**: Increase Logstash replicas

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash
spec:
  replicas: 5  # Scale up as needed
```

2. **Vertical Scaling**: Increase resources for Elasticsearch

```yaml
resources:
  requests:
    memory: "8Gi"
    cpu: "2"
  limits:
    memory: "8Gi"
    cpu: "4"
```

3. **Shard Allocation**: Optimize for write throughput

```
PUT logs-*/_settings
{
  "index": {
    "number_of_shards": 3,
    "number_of_replicas": 1
  }
}
```

### High-Security Architectures

#### Network Isolation

Implement network policies for logging components:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: filebeat-policy
  namespace: logging
spec:
  podSelector:
    matchLabels:
      app: filebeat
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: logstash
    ports:
    - protocol: TCP
      port: 5044
```

#### Secure Transport

Enable TLS for all communication:

```yaml
# filebeat.yml
output.logstash:
  hosts: ["logstash:5044"]
  ssl.enabled: true
  ssl.certificate_authorities: ["/etc/filebeat/ca.crt"]
  ssl.certificate: "/etc/filebeat/filebeat.crt"
  ssl.key: "/etc/filebeat/filebeat.key"
```

#### Log Redaction

Redact sensitive information before storing:

```
filter {
  mutate {
    gsub => [
      "message", "\b(?:\d[ -]*?){13,16}\b", "[CREDIT_CARD_REDACTED]",
      "message", "\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,6}\b", "[EMAIL_REDACTED]"
    ]
  }
}
```

## Visualization and Analysis

### Kibana Dashboards

Create dedicated Kubernetes dashboards:

1. **Cluster Overview Dashboard**:
   - Pod log volume by namespace
   - Error rates by application
   - Log levels distribution
   - Top pods by log volume

2. **Application Performance Dashboard**:
   - Response time trends
   - Error rates and types
   - Request volume by service
   - Status code distribution

3. **Troubleshooting Dashboard**:
   - Recent errors and exceptions
   - Pod restarts correlation
   - Resource utilization vs. errors
   - Deployment timeline events

### Saved Searches

Create useful saved searches:

1. **Pods with High Error Rates**:
   ```
   kubernetes.namespace:production AND log.level:ERROR
   ```

2. **Failed Pod Startups**:
   ```
   kubernetes.container.status.reason:CrashLoopBackOff OR kubernetes.container.status.reason:Error
   ```

3. **Authentication Failures**:
   ```
   message:"authentication failed" OR message:"forbidden" OR message:"unauthorized"
   ```

### Alerting

Set up alerts for critical issues:

1. **High Error Rate Alert**:
   - Trigger when error rate exceeds threshold
   - Group by namespace and application
   - Alert via Slack/email

2. **Pod Crash Alert**:
   - Monitor for CrashLoopBackOff events
   - Include recent logs in notification
   - Link to troubleshooting dashboard

3. **Unusual Log Volume Alert**:
   - Detect sudden increases in log volume
   - Could indicate issues or attacks
   - Threshold based on historical patterns

## Operational Best Practices

### Resource Management

#### Resource Limits and Requests

Set appropriate resource limits for logging components:

```yaml
# Filebeat
resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"

# Logstash
resources:
  requests:
    memory: "1Gi"
    cpu: "500m"
  limits:
    memory: "2Gi"
    cpu: "1"

# Elasticsearch
resources:
  requests:
    memory: "4Gi"
    cpu: "1"
  limits:
    memory: "4Gi"
    cpu: "2"
```

#### Log Volume Control

Implement log rate limiting:

```yaml
# filebeat.yml
processors:
  - throttle:
      before_count: 100
      after_count: 1
      period: 5s
      fields: ["kubernetes.pod.name"]
```

### Log Retention and Cleanup

Define appropriate retention periods:

```
PUT _ilm/policy/kubernetes-logs-policy
{
  "policy": {
    "phases": {
      "hot": { ... },
      "warm": { ... },
      "cold": { ... },
      "delete": {
        "min_age": "30d",  # Adjust based on requirements
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

### Monitoring the Logging Infrastructure

Monitor Filebeat with Metricbeat:

```yaml
metricbeat.modules:
- module: beat
  metricsets:
    - stats
    - state
  period: 10s
  hosts: ["filebeat:5066"]
```

Monitor Elasticsearch:

```yaml
metricbeat.modules:
- module: elasticsearch
  metricsets:
    - node
    - node_stats
    - index
    - index_recovery
  period: 10s
  hosts: ["elasticsearch:9200"]
```

## Troubleshooting

### Common Issues and Solutions

#### Missing Logs

**Issue**: Some container logs aren't appearing in Elasticsearch

**Potential causes**:
- Filebeat configuration incorrect
- Short-lived containers terminating before logs are collected
- Resource constraints causing dropped events

**Solutions**:
- Check Filebeat configuration and status
- Implement log shipping with a small delay
- Increase Filebeat resources or buffer size

#### High Latency

**Issue**: Long delay between log generation and availability in Kibana

**Potential causes**:
- Logstash bottleneck
- Elasticsearch indexing pressure
- Network issues between components

**Solutions**:
- Scale Logstash horizontally
- Optimize Elasticsearch for indexing
- Check network connectivity and latency

#### Resource Consumption

**Issue**: Logging infrastructure consuming excessive resources

**Potential causes**:
- Collecting unnecessary logs
- Inefficient processing
- Inadequate resource allocation

**Solutions**:
- Filter logs at source
- Optimize Logstash filters
- Adjust resource limits based on actual usage

### Diagnostic Techniques

#### Filebeat Diagnostics

Enable debug logging:

```yaml
logging.level: debug
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat.log
  keepfiles: 7
```

Check Filebeat status:
```bash
kubectl exec -it filebeat-abc12 -n logging -- filebeat test config
kubectl exec -it filebeat-abc12 -n logging -- filebeat test output
```

#### Logstash Diagnostics

Enable debug logging:

```yaml
# logstash.yml
log.level: debug
```

Test pipeline configuration:
```bash
kubectl exec -it logstash-abc12 -n logging -- logstash -t -f /usr/share/logstash/pipeline/
```

#### Elasticsearch Diagnostics

Check cluster health:
```bash
kubectl exec -it elasticsearch-0 -n logging -- curl -s http://localhost:9200/_cluster/health
```

Check indexing rate:
```bash
kubectl exec -it elasticsearch-0 -n logging -- curl -s http://localhost:9200/_stats?level=indices
```

## Conclusion

Building an effective logging architecture for Kubernetes requires careful planning and understanding of both Kubernetes and ELK Stack capabilities. By following the patterns and practices outlined in this chapter, you can create a robust, scalable, and maintainable logging solution that provides valuable insights into your Kubernetes applications and infrastructure.

Remember that logging requirements evolve as your applications grow, so regularly review and adjust your logging architecture to ensure it continues to meet your observability needs.

In the next chapter, we'll explore how to use the ELK Stack for monitoring Kubernetes clusters themselves.