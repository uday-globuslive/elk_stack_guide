# Monitoring Kubernetes with the ELK Stack

This chapter covers comprehensive strategies and implementations for monitoring Kubernetes clusters using the Elastic Stack.

## Table of Contents
- [Introduction to Kubernetes Monitoring](#introduction-to-kubernetes-monitoring)
- [Monitoring Architecture](#monitoring-architecture)
- [Core Metrics Collection](#core-metrics-collection)
- [Kubernetes Events Monitoring](#kubernetes-events-monitoring)
- [Application Metrics Monitoring](#application-metrics-monitoring)
- [Control Plane Monitoring](#control-plane-monitoring)
- [Alerting and Anomaly Detection](#alerting-and-anomaly-detection)
- [Performance Analysis and Optimization](#performance-analysis-and-optimization)
- [Troubleshooting with Monitoring Data](#troubleshooting-with-monitoring-data)
- [Integration with Other Monitoring Solutions](#integration-with-other-monitoring-solutions)
- [Dashboards and Visualizations](#dashboards-and-visualizations)

## Introduction to Kubernetes Monitoring

Monitoring Kubernetes environments is essential for ensuring optimal performance, reliability, and resource utilization. The Elastic Stack provides a powerful platform for collecting, analyzing, and visualizing metrics from Kubernetes clusters.

### Monitoring Requirements

A comprehensive Kubernetes monitoring solution should address:

1. **Infrastructure Metrics**
   - Node resource utilization (CPU, memory, disk, network)
   - Kubernetes component health
   - Container runtime metrics

2. **Application Metrics**
   - Pod and container resource usage
   - Application-specific metrics
   - Service-level indicators (SLIs)

3. **Control Plane Metrics**
   - API server performance
   - Scheduler and controller manager metrics
   - etcd database metrics

4. **Events and State Changes**
   - Pod lifecycle events
   - Node conditions
   - System events

5. **Service Level Monitoring**
   - Latency and throughput
   - Error rates and availability
   - Performance bottlenecks

### Benefits of Using the ELK Stack

The Elastic Stack offers several advantages for Kubernetes monitoring:

- **Unified Platform**: Collect, store, analyze, and visualize metrics and logs in one place
- **Scalability**: Handle metrics from large and complex Kubernetes environments
- **Flexibility**: Customize metrics collection and visualizations to specific needs
- **Correlation**: Combine metrics, logs, and traces for comprehensive observability
- **Advanced Analytics**: Use Machine Learning for anomaly detection and forecasting

## Monitoring Architecture

### Overall Architecture

A typical Kubernetes monitoring architecture with the Elastic Stack includes:

```
+---------------------+                   +---------------------+
|  Kubernetes Cluster |                   |    Elastic Stack    |
|                     |                   |                     |
|  +---------------+  |                   |  +---------------+  |
|  |     Nodes     |  |                   |  | Elasticsearch |  |
|  |  +---------+  |  |                   |  |               |  |
|  |  |   Pods  |  |  |                   |  | +-----------+ |  |
|  |  +---------+  |  |  +-----------+    |  | |  Indices  | |  |
|  |  |Containers|<------>Metricbeat |--------->|Dashboards| |  |
|  |  +---------+  |  |  +-----------+    |  | +-----------+ |  |
|  +---------------+  |                   |  +---------------+  |
|                     |                   |                     |
|  +---------------+  |                   |  +---------------+  |
|  | Control Plane |  |                   |  |    Kibana     |  |
|  +---------------+  |                   |  +---------------+  |
|                     |                   |                     |
+---------------------+                   +---------------------+
```

### Collection Strategies

Several strategies can be employed for metrics collection:

1. **DaemonSet Approach**: Deploy Metricbeat as a DaemonSet to collect node and container metrics.
2. **Sidecar Approach**: Deploy Metricbeat as a sidecar for application-specific metrics.
3. **Deployment Approach**: Deploy Metricbeat as a Deployment to scrape Kubernetes API and external services.
4. **Hybrid Approach**: Combine multiple deployment patterns for comprehensive coverage.

### Component Roles

- **Metricbeat**: Collects metrics from Kubernetes nodes, pods, and system components
- **Elasticsearch**: Stores and indexes metrics data
- **Kibana**: Provides dashboards and visualizations for metrics analysis
- **APM Server** (optional): Collects application performance data
- **Machine Learning** (optional): Performs anomaly detection and forecasting

## Core Metrics Collection

### Node-level Metrics

Metricbeat can collect detailed node-level metrics using the system module:

```yaml
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: node-metrics
  namespace: monitoring
spec:
  type: metricbeat
  version: 8.8.0
  elasticsearchRef:
    name: elasticsearch
  config:
    metricbeat.modules:
    - module: system
      period: 10s
      metricsets:
        - cpu
        - load
        - memory
        - network
        - process
        - process_summary
        - core
        - diskio
        - filesystem
        - fsstat
      processes: ['.*']
      process.include_top_n:
        by_cpu: 5
        by_memory: 5
    processors:
    - add_kubernetes_metadata:
        host: ${NODE_NAME}
  daemonSet:
    podTemplate:
      spec:
        serviceAccountName: metricbeat
        automountServiceAccountToken: true
        hostNetwork: true
        dnsPolicy: ClusterFirstWithHostNet
        securityContext:
          runAsUser: 0
        containers:
        - name: metricbeat
          env:
          - name: NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
          securityContext:
            runAsUser: 0
          volumeMounts:
          - name: proc
            mountPath: /hostfs/proc
            readOnly: true
          - name: sys
            mountPath: /hostfs/sys
            readOnly: true
          - name: root
            mountPath: /hostfs
            readOnly: true
        volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: sys
          hostPath:
            path: /sys
        - name: root
          hostPath:
            path: /
```

### Container Metrics

Use the Kubernetes module to collect container metrics:

```yaml
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: container-metrics
  namespace: monitoring
spec:
  type: metricbeat
  version: 8.8.0
  elasticsearchRef:
    name: elasticsearch
  config:
    metricbeat.modules:
    - module: kubernetes
      metricsets:
        - container
        - pod
        - system
        - volume
      period: 10s
      host: "${NODE_NAME}"
      hosts: ["https://${NODE_NAME}:10250"]
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      ssl.verification_mode: "none"
    processors:
    - add_kubernetes_metadata:
        host: ${NODE_NAME}
  daemonSet:
    podTemplate:
      spec:
        serviceAccountName: metricbeat
        automountServiceAccountToken: true
        hostNetwork: true
        dnsPolicy: ClusterFirstWithHostNet
        containers:
        - name: metricbeat
          env:
          - name: NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
```

### Kubernetes State Metrics

Collect metrics from kube-state-metrics:

```yaml
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: kube-state-metrics
  namespace: monitoring
spec:
  type: metricbeat
  version: 8.8.0
  elasticsearchRef:
    name: elasticsearch
  config:
    metricbeat.modules:
    - module: kubernetes
      metricsets:
        - state_node
        - state_deployment
        - state_replicaset
        - state_pod
        - state_container
        - state_cronjob
        - state_resourcequota
        - state_statefulset
        - state_service
      period: 30s
      hosts: ["kube-state-metrics:8080"]
  deployment:
    podTemplate:
      spec:
        serviceAccountName: metricbeat
        automountServiceAccountToken: true
```

First, ensure kube-state-metrics is deployed:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-state-metrics
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: kube-state-metrics
  replicas: 1
  template:
    metadata:
      labels:
        app: kube-state-metrics
    spec:
      serviceAccountName: kube-state-metrics
      containers:
      - name: kube-state-metrics
        image: registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.8.0
        ports:
        - name: http-metrics
          containerPort: 8080
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          timeoutSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: kube-state-metrics
  namespace: monitoring
  labels:
    app: kube-state-metrics
spec:
  type: ClusterIP
  ports:
  - name: http-metrics
    port: 8080
    targetPort: 8080
  selector:
    app: kube-state-metrics
```

### Resource Quota Monitoring

Monitor Kubernetes resource quotas and limits:

```yaml
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: quota-metrics
  namespace: monitoring
spec:
  type: metricbeat
  version: 8.8.0
  elasticsearchRef:
    name: elasticsearch
  config:
    metricbeat.modules:
    - module: kubernetes
      metricsets:
        - state_resourcequota
        - state_persistentvolume
        - state_persistentvolumeclaim
      period: 1m
      hosts: ["kube-state-metrics:8080"]
  deployment:
    podTemplate:
      spec:
        serviceAccountName: metricbeat
        automountServiceAccountToken: true
```

## Kubernetes Events Monitoring

### Events Collection

Monitor Kubernetes events using Metricbeat:

```yaml
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: event-metrics
  namespace: monitoring
spec:
  type: metricbeat
  version: 8.8.0
  elasticsearchRef:
    name: elasticsearch
  config:
    metricbeat.modules:
    - module: kubernetes
      metricsets:
        - event
      period: 10s
      hosts: ["kube-state-metrics:8080"]
    processors:
    - add_kubernetes_metadata:
        in_cluster: true
  deployment:
    podTemplate:
      spec:
        serviceAccountName: metricbeat
        automountServiceAccountToken: true
```

### Event Processing and Analysis

Configure Elasticsearch templates to enhance event analysis:

```json
PUT _component_template/kubernetes-events-mappings
{
  "template": {
    "mappings": {
      "properties": {
        "kubernetes.event": {
          "properties": {
            "message": {
              "type": "text"
            },
            "reason": {
              "type": "keyword"
            },
            "type": {
              "type": "keyword"
            },
            "count": {
              "type": "integer"
            },
            "involved_object": {
              "properties": {
                "kind": {
                  "type": "keyword"
                },
                "name": {
                  "type": "keyword"
                },
                "namespace": {
                  "type": "keyword"
                }
              }
            }
          }
        }
      }
    }
  }
}
```

Create a pipeline to categorize events by severity:

```json
PUT _ingest/pipeline/kubernetes-events-pipeline
{
  "description": "Process Kubernetes events",
  "processors": [
    {
      "set": {
        "field": "event.severity",
        "value": "warning",
        "if": "ctx.kubernetes?.event?.type == 'Warning'"
      }
    },
    {
      "set": {
        "field": "event.severity",
        "value": "info",
        "if": "ctx.kubernetes?.event?.type == 'Normal'"
      }
    },
    {
      "set": {
        "field": "event.category",
        "value": "node",
        "if": "ctx.kubernetes?.event?.involved_object?.kind == 'Node'"
      }
    },
    {
      "set": {
        "field": "event.category",
        "value": "pod",
        "if": "ctx.kubernetes?.event?.involved_object?.kind == 'Pod'"
      }
    }
  ]
}
```

## Application Metrics Monitoring

### Prometheus Metrics Collection

Many Kubernetes applications expose Prometheus metrics. Collect these with Metricbeat:

```yaml
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: prometheus-metrics
  namespace: monitoring
spec:
  type: metricbeat
  version: 8.8.0
  elasticsearchRef:
    name: elasticsearch
  config:
    metricbeat.modules:
    - module: prometheus
      period: 10s
      metricsets:
        - collector
      hosts:
        - "prometheus-server:9090"
      metrics_path: /metrics
    - module: prometheus
      period: 10s
      metricsets:
        - collector
      hosts:
        - "app-service:8080"
      metrics_path: /actuator/prometheus
  deployment:
    podTemplate:
      spec:
        serviceAccountName: metricbeat
        automountServiceAccountToken: true
```

### Autodiscovery for Application Metrics

Enable autodiscovery to automatically detect and monitor applications with Prometheus endpoints:

```yaml
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: prometheus-autodiscover
  namespace: monitoring
spec:
  type: metricbeat
  version: 8.8.0
  elasticsearchRef:
    name: elasticsearch
  config:
    metricbeat.autodiscover:
      providers:
        - type: kubernetes
          node: ${NODE_NAME}
          templates:
            - condition:
                contains:
                  kubernetes.annotations.prometheus.io/scrape: "true"
              config:
                - module: prometheus
                  metricsets:
                    - collector
                  hosts:
                    - "${data.host}:${data.port}"
                  metrics_path: ${data.fields.prometheus.metrics_path:/metrics}
    processors:
    - add_kubernetes_metadata:
        host: ${NODE_NAME}
  daemonSet:
    podTemplate:
      spec:
        serviceAccountName: metricbeat
        automountServiceAccountToken: true
        hostNetwork: true
        dnsPolicy: ClusterFirstWithHostNet
        containers:
        - name: metricbeat
          env:
          - name: NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
```

### Application Performance Monitoring (APM)

Deploy the Elastic APM server for detailed application performance monitoring:

```yaml
apiVersion: apm.k8s.elastic.co/v1
kind: ApmServer
metadata:
  name: apm-server
  namespace: monitoring
spec:
  version: 8.8.0
  count: 1
  elasticsearchRef:
    name: elasticsearch
  kibanaRef:
    name: kibana
  http:
    service:
      spec:
        type: ClusterIP
    tls:
      selfSignedCertificate:
        disabled: true
  config:
    apm-server:
      rum:
        enabled: true
```

Instrument your applications with the Elastic APM agents:

For Java applications:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app
spec:
  # ...
  template:
    spec:
      containers:
      - name: java-app
        # ...
        env:
        - name: ELASTIC_APM_SERVER_URL
          value: "http://apm-server-apm-http:8200"
        - name: ELASTIC_APM_SERVICE_NAME
          value: "java-app"
        - name: ELASTIC_APM_ENVIRONMENT
          value: "production"
        - name: ELASTIC_APM_APPLICATION_PACKAGES
          value: "com.example"
        - name: JAVA_TOOL_OPTIONS
          value: "-javaagent:/elastic-apm-agent.jar"
        volumeMounts:
        - name: apm-agent
          mountPath: /elastic-apm-agent.jar
          subPath: elastic-apm-agent.jar
      volumes:
      - name: apm-agent
        configMap:
          name: apm-agent-jar
```

## Control Plane Monitoring

### API Server Metrics

Monitor the Kubernetes API server:

```yaml
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: control-plane-metrics
  namespace: monitoring
spec:
  type: metricbeat
  version: 8.8.0
  elasticsearchRef:
    name: elasticsearch
  config:
    metricbeat.modules:
    - module: kubernetes
      metricsets:
        - apiserver
      period: 30s
      hosts: ["https://kubernetes.default.svc:443"]
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      ssl.verification_mode: "none"
  deployment:
    podTemplate:
      spec:
        serviceAccountName: metricbeat
        automountServiceAccountToken: true
```

### etcd Monitoring

Monitor etcd performance:

```yaml
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: etcd-metrics
  namespace: monitoring
spec:
  type: metricbeat
  version: 8.8.0
  elasticsearchRef:
    name: elasticsearch
  config:
    metricbeat.modules:
    - module: etcd
      metricsets:
        - leader
        - self
        - store
      period: 10s
      hosts: ["https://etcd-0.etcd:2379"]
      ssl.certificate_authorities:
        - /etc/kubernetes/pki/etcd/ca.crt
      ssl.certificate: /etc/kubernetes/pki/etcd/server.crt
      ssl.key: /etc/kubernetes/pki/etcd/server.key
  deployment:
    podTemplate:
      spec:
        serviceAccountName: metricbeat
        automountServiceAccountToken: true
        hostNetwork: true
        volumes:
        - name: etcd-certs
          hostPath:
            path: /etc/kubernetes/pki/etcd
            type: DirectoryOrCreate
        containers:
        - name: metricbeat
          volumeMounts:
          - name: etcd-certs
            mountPath: /etc/kubernetes/pki/etcd
            readOnly: true
```

### Scheduler and Controller Manager

Monitor the scheduler and controller manager:

```yaml
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: kube-scheduler-metrics
  namespace: monitoring
spec:
  type: metricbeat
  version: 8.8.0
  elasticsearchRef:
    name: elasticsearch
  config:
    metricbeat.modules:
    - module: kubernetes
      metricsets:
        - scheduler
      period: 10s
      hosts: ["https://localhost:10259"]
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      ssl.verification_mode: "none"
    - module: kubernetes
      metricsets:
        - controllermanager
      period: 10s
      hosts: ["https://localhost:10257"]
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      ssl.verification_mode: "none"
  daemonSet:
    podTemplate:
      spec:
        serviceAccountName: metricbeat
        automountServiceAccountToken: true
        hostNetwork: true
        nodeSelector:
          node-role.kubernetes.io/control-plane: ""
        tolerations:
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule
```

## Alerting and Anomaly Detection

### Basic Alerting

Create alerts in Kibana for critical Kubernetes metrics:

1. **Node Resource Alerts**

Navigate to Kibana → Stack Management → Rules and Alerts → Create rule

```
Rule name: High Node CPU Usage
Rule type: Threshold
Index pattern: metricbeat-*
Field: system.cpu.total.pct
Conditions: IS ABOVE 0.85 FOR 5 MINUTES
Action: Notify via Slack/Email
```

2. **Pod Restart Alerts**

```
Rule name: Frequent Pod Restarts
Rule type: Threshold
Index pattern: metricbeat-*
Field: kubernetes.container.restarts
Conditions: RATE() IS ABOVE 3 PER 5 MINUTES
Action: Notify on-call team
```

### Anomaly Detection with Machine Learning

Set up Machine Learning jobs to detect anomalies in Kubernetes metrics:

```json
PUT _ml/anomaly_detectors/k8s-node-cpu
{
  "description": "Kubernetes node CPU anomaly detection",
  "analysis_config": {
    "bucket_span": "5m",
    "detectors": [
      {
        "detector_description": "High CPU usage",
        "function": "high_mean",
        "field_name": "system.cpu.total.pct",
        "by_field_name": "kubernetes.node.name"
      }
    ],
    "influencers": [
      "kubernetes.node.name"
    ]
  },
  "data_description": {
    "time_field": "@timestamp"
  },
  "model_plot_config": {
    "enabled": true
  }
}
```

```json
PUT _ml/anomaly_detectors/k8s-memory-usage
{
  "description": "Kubernetes memory usage anomaly detection",
  "analysis_config": {
    "bucket_span": "5m",
    "detectors": [
      {
        "detector_description": "Unusual memory usage patterns",
        "function": "high_mean",
        "field_name": "kubernetes.container.memory.usage.bytes",
        "by_field_name": "kubernetes.pod.name",
        "partition_field_name": "kubernetes.namespace"
      }
    ],
    "influencers": [
      "kubernetes.pod.name",
      "kubernetes.namespace",
      "kubernetes.node.name"
    ]
  },
  "data_description": {
    "time_field": "@timestamp"
  }
}
```

### Forecasting Resource Usage

Use Machine Learning to forecast resource usage trends:

```json
PUT _ml/anomaly_detectors/k8s-pod-forecast
{
  "description": "Kubernetes pod resource forecasting",
  "analysis_config": {
    "bucket_span": "1h",
    "detectors": [
      {
        "detector_description": "CPU usage forecasting",
        "function": "mean",
        "field_name": "kubernetes.pod.cpu.usage.nanocores",
        "by_field_name": "kubernetes.pod.name"
      }
    ],
    "influencers": [
      "kubernetes.pod.name"
    ],
    "model_prune_window": "30d"
  },
  "data_description": {
    "time_field": "@timestamp"
  },
  "model_plot_config": {
    "enabled": true
  },
  "forecast_config": {
    "forecast_interval": "7d"
  }
}
```

## Performance Analysis and Optimization

### Performance Metrics Collection

Set up detailed performance metrics collection:

```yaml
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: performance-metrics
  namespace: monitoring
spec:
  type: metricbeat
  version: 8.8.0
  elasticsearchRef:
    name: elasticsearch
  config:
    metricbeat.modules:
    - module: linux
      metricsets:
        - ksm
        - pageinfo
      period: 10s
      # Additional Linux performance metrics
    - module: system
      metricsets:
        - diskio
        - socket_summary
        - process_socket
      period: 10s
    processors:
    - add_kubernetes_metadata:
        host: ${NODE_NAME}
  daemonSet:
    podTemplate:
      spec:
        serviceAccountName: metricbeat
        automountServiceAccountToken: true
        hostNetwork: true
        dnsPolicy: ClusterFirstWithHostNet
        containers:
        - name: metricbeat
          env:
          - name: NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
```

### Capacity Planning Tools

Create Elasticsearch transforms for capacity planning:

```json
PUT _transform/kubernetes_daily_resources
{
  "source": {
    "index": [
      "metricbeat-*"
    ],
    "query": {
      "bool": {
        "filter": [
          {
            "exists": {
              "field": "kubernetes.pod.name"
            }
          }
        ]
      }
    }
  },
  "dest": {
    "index": "kubernetes_resource_usage"
  },
  "pivot": {
    "group_by": {
      "kubernetes.namespace": {
        "terms": {
          "field": "kubernetes.namespace"
        }
      },
      "day": {
        "date_histogram": {
          "field": "@timestamp",
          "calendar_interval": "day"
        }
      }
    },
    "aggregations": {
      "avg_cpu_usage": {
        "avg": {
          "field": "kubernetes.pod.cpu.usage.nanocores"
        }
      },
      "avg_memory_usage": {
        "avg": {
          "field": "kubernetes.pod.memory.usage.bytes"
        }
      },
      "max_cpu_usage": {
        "max": {
          "field": "kubernetes.pod.cpu.usage.nanocores"
        }
      },
      "max_memory_usage": {
        "max": {
          "field": "kubernetes.pod.memory.usage.bytes"
        }
      },
      "pod_count": {
        "cardinality": {
          "field": "kubernetes.pod.name"
        }
      }
    }
  },
  "frequency": "1h",
  "sync": {
    "time": {
      "field": "@timestamp",
      "delay": "60s"
    }
  },
  "retention_policy": {
    "time": {
      "field": "day",
      "max_age": "90d"
    }
  }
}
```

## Troubleshooting with Monitoring Data

### Correlation with Logs

Create a dashboard that correlates metrics with logs for troubleshooting:

1. Create a Kibana dashboard with:
   - Node resource usage panel
   - Pod resource usage panel
   - Events timeline panel
   - Log viewer panel with linked filters

2. Set up drilldowns from metrics to logs:
   - Click on a pod in metrics view
   - Auto-filter logs for that pod
   - View events for the same time period

### Root Cause Analysis

Use Elastic Observability to identify root causes:

1. **Automated Root Cause Analysis**
   - Machine Learning for anomaly correlation
   - Automated identification of related events
   - Impact analysis across services

2. **Manual Investigation Tools**
   - Timeline analysis with metrics, events, and logs
   - Service maps for dependency analysis
   - Trace analysis for transaction flows

## Integration with Other Monitoring Solutions

### Prometheus Integration

Integrate with existing Prometheus monitoring:

```yaml
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: prometheus-remote-write
  namespace: monitoring
spec:
  type: metricbeat
  version: 8.8.0
  elasticsearchRef:
    name: elasticsearch
  config:
    metricbeat.modules:
    - module: prometheus
      period: 10s
      metricsets:
        - remote_write
      host: "0.0.0.0"
      port: "9201"
  deployment:
    podTemplate:
      spec:
        containers:
        - name: metricbeat
          ports:
          - containerPort: 9201
```

Configure Prometheus to remote_write to Metricbeat:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    
    scrape_configs:
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
        - role: pod
        relabel_configs:
        # Include pods with prometheus.io/scrape=true annotation
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        # Use the pod's name as the instance label
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: instance
    
    remote_write:
      - url: "http://prometheus-remote-write-beat:9201/write"
```

### OpenTelemetry Integration

Collect metrics from OpenTelemetry sources:

```yaml
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: otel-metrics
  namespace: monitoring
spec:
  type: metricbeat
  version: 8.8.0
  elasticsearchRef:
    name: elasticsearch
  config:
    metricbeat.modules:
    - module: opentelemetry
      metricsets:
        - metrics
      host: "0.0.0.0"
      port: "8888"
    processors:
    - add_kubernetes_metadata:
        host: ${NODE_NAME}
  deployment:
    podTemplate:
      spec:
        serviceAccountName: metricbeat
        containers:
        - name: metricbeat
          ports:
          - containerPort: 8888
            name: otel-metrics
```

## Dashboards and Visualizations

### Cluster Overview Dashboard

Create a comprehensive cluster overview dashboard:

1. **Cluster Health**
   - Nodes status (Ready/NotReady)
   - Pod/container count
   - Resource utilization

2. **Node Metrics**
   - CPU/Memory usage per node
   - Disk space and IOPS
   - Network throughput

3. **Workload Overview**
   - Namespace resource usage
   - Deployment status
   - Pod restart counts

### Pod Performance Dashboard

Design a detailed pod performance dashboard:

1. **Pod Resource Usage**
   - CPU usage (absolute and relative to limits)
   - Memory usage and trends
   - Network traffic

2. **Container Metrics**
   - Restart counts
   - Status changes
   - Runtime metrics

3. **Resource Efficiency**
   - CPU throttling events
   - Memory pressure indicators
   - Request vs. usage analysis

### Node Drilldown Dashboard

Create a detailed node analysis dashboard:

1. **Node Resource Usage**
   - System-level CPU/Memory/Disk metrics
   - Kernel metrics
   - Process resource consumption

2. **Pod Distribution**
   - Pods per node
   - Resource allocation per pod
   - DaemonSet coverage

3. **Node Events Timeline**
   - Status changes
   - System events
   - Pod scheduling events

## Summary

Building a comprehensive Kubernetes monitoring solution with the Elastic Stack provides deep visibility into cluster health, performance, and potential issues. By collecting and analyzing metrics from nodes, containers, applications, and Kubernetes components, you can ensure optimal cluster operation and quickly troubleshoot problems.

The key to effective Kubernetes monitoring with the ELK Stack is:
- Comprehensive metrics collection from all layers
- Integration of metrics, logs, and traces
- Automated anomaly detection and alerting
- Insightful dashboards for different user personas
- Performance analysis and capacity planning tools

By implementing these strategies, you can achieve the observability required to run production Kubernetes clusters with confidence.

## References

- [Metricbeat Kubernetes Module](https://www.elastic.co/guide/en/beats/metricbeat/current/metricbeat-module-kubernetes.html)
- [Kubernetes Monitoring Guide](https://www.elastic.co/guide/en/observability/current/monitor-kubernetes.html)
- [Elastic Stack Kubernetes Operator](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-overview.html)
- [Prometheus Remote Write Integration](https://www.elastic.co/guide/en/beats/metricbeat/current/metricbeat-metricset-prometheus-remote_write.html)
- [Kubernetes Dashboard Templates](https://www.elastic.co/guide/en/kibana/current/dashboard-templates.html)