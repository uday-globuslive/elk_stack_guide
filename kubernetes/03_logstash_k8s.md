# Logstash on Kubernetes

This chapter covers deploying, configuring, and managing Logstash on Kubernetes environments.

## Table of Contents
- [Introduction to Logstash on Kubernetes](#introduction-to-logstash-on-kubernetes)
- [Deployment Strategies](#deployment-strategies)
- [Resource Configuration](#resource-configuration)
- [Pipeline Management](#pipeline-management)
- [ConfigMaps and Secrets](#configmaps-and-secrets)
- [StatefulSets vs Deployments](#statefulsets-vs-deployments)
- [Scaling and High Availability](#scaling-and-high-availability)
- [Persistent Storage](#persistent-storage)
- [Monitoring and Logging](#monitoring-and-logging)
- [Security Considerations](#security-considerations)
- [Performance Tuning](#performance-tuning)
- [Integration with Elasticsearch and Kibana](#integration-with-elasticsearch-and-kibana)
- [Handling Upgrades](#handling-upgrades)

## Introduction to Logstash on Kubernetes

Logstash serves as a critical component in the ELK Stack for data processing, transformation, and enrichment. Deploying Logstash on Kubernetes brings benefits including high availability, scalability, and simplified management - but also introduces unique challenges related to stateful processing and configuration management.

### Key Considerations for Logstash on Kubernetes

1. **Stateful Processing**: Logstash maintains in-memory queues and can leverage persistent queues on disk
2. **Resource Requirements**: Logstash is JVM-based and requires appropriate memory allocation
3. **Configuration Management**: Pipeline configurations need to be managed and updated efficiently
4. **Input/Output Connectivity**: Logstash needs to connect to various inputs and outputs, requiring proper network configuration
5. **Scaling Considerations**: Different pipelines have different scaling requirements

### Logstash Architecture in Kubernetes

In a Kubernetes environment, Logstash can be deployed in various topologies:

1. **Centralized Logstash**: All logs flow to a central Logstash deployment
   - Simpler management
   - Potential single point of failure
   - Requires more resources in a single deployment

2. **Distributed Logstash**: Multiple purpose-specific Logstash deployments
   - Better isolation and scalability
   - More complex management
   - Can align with specific data sources or use cases

3. **Hybrid Approach**: Filebeat for collection, multiple Logstash deployments for processing
   - Balances scalability and manageability
   - Recommended for most production deployments

## Deployment Strategies

### Basic Deployment with ConfigMap

A simple Logstash deployment using ConfigMaps for configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-config
  namespace: elk
data:
  logstash.yml: |
    http.host: "0.0.0.0"
    xpack.monitoring.elasticsearch.hosts: [ "http://elasticsearch-master:9200" ]
    
  pipelines.yml: |
    - pipeline.id: main
      path.config: "/usr/share/logstash/pipeline"
      
  logstash.conf: |
    input {
      beats {
        port => 5044
      }
    }
    
    filter {
      if [kubernetes] {
        mutate {
          add_field => { "[@metadata][target_index]" => "logs-k8s-%{+YYYY.MM.dd}" }
        }
      } else {
        mutate {
          add_field => { "[@metadata][target_index]" => "logs-default-%{+YYYY.MM.dd}" }
        }
      }
    }
    
    output {
      elasticsearch {
        hosts => ["elasticsearch-master:9200"]
        index => "%{[@metadata][target_index]}"
      }
    }

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash
  namespace: elk
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
        image: docker.elastic.co/logstash/logstash:7.17.8
        ports:
        - containerPort: 5044
          name: beats
        - containerPort: 9600
          name: monitoring
        volumeMounts:
        - name: config-volume
          mountPath: /usr/share/logstash/config/logstash.yml
          subPath: logstash.yml
        - name: config-volume
          mountPath: /usr/share/logstash/config/pipelines.yml
          subPath: pipelines.yml
        - name: config-volume
          mountPath: /usr/share/logstash/pipeline/logstash.conf
          subPath: logstash.conf
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "2"
            memory: "2Gi"
        env:
        - name: LS_JAVA_OPTS
          value: "-Xmx1g -Xms1g"
      volumes:
      - name: config-volume
        configMap:
          name: logstash-config

---
apiVersion: v1
kind: Service
metadata:
  name: logstash
  namespace: elk
spec:
  selector:
    app: logstash
  ports:
  - port: 5044
    name: beats
    targetPort: 5044
  - port: 9600
    name: monitoring
    targetPort: 9600
```

### Advanced Deployment with Multiple Pipelines

For more complex scenarios, multiple pipelines can be deployed:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-config
  namespace: elk
data:
  logstash.yml: |
    http.host: "0.0.0.0"
    xpack.monitoring.elasticsearch.hosts: [ "http://elasticsearch-master:9200" ]
    
  pipelines.yml: |
    - pipeline.id: application-logs
      path.config: "/usr/share/logstash/pipeline/application.conf"
      pipeline.workers: 2
    - pipeline.id: system-logs
      path.config: "/usr/share/logstash/pipeline/system.conf"
      pipeline.workers: 1
    - pipeline.id: security-logs
      path.config: "/usr/share/logstash/pipeline/security.conf"
      pipeline.workers: 3
      
  application.conf: |
    input {
      beats {
        port => 5044
        tags => ["application"]
      }
    }
    
    filter {
      # Application-specific filters
      if [log][file][path] =~ "app" {
        json {
          source => "message"
        }
        # Additional application-specific filters
      }
    }
    
    output {
      elasticsearch {
        hosts => ["elasticsearch-master:9200"]
        index => "logs-app-%{+YYYY.MM.dd}"
      }
    }
    
  system.conf: |
    input {
      beats {
        port => 5045
        tags => ["system"]
      }
    }
    
    filter {
      # System log specific filters
      if [system] {
        # System-specific filters
      }
    }
    
    output {
      elasticsearch {
        hosts => ["elasticsearch-master:9200"]
        index => "logs-system-%{+YYYY.MM.dd}"
      }
    }
    
  security.conf: |
    input {
      beats {
        port => 5046
        tags => ["security"]
      }
      tcp {
        port => 5047
        codec => "json"
        tags => ["firewall"]
      }
    }
    
    filter {
      # Security log specific filters
      if [tags] == "firewall" {
        # Firewall-specific filters
      }
    }
    
    output {
      elasticsearch {
        hosts => ["elasticsearch-master:9200"]
        index => "logs-security-%{+YYYY.MM.dd}"
      }
    }
```

### Deployment with Multiple Pipeline-Specific Deployments

For large-scale environments, deploying separate Logstash instances for different pipelines:

```yaml
# Application Logstash Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash-app
  namespace: elk
spec:
  replicas: 3
  selector:
    matchLabels:
      app: logstash
      pipeline: application
  template:
    metadata:
      labels:
        app: logstash
        pipeline: application
    spec:
      containers:
      - name: logstash
        image: docker.elastic.co/logstash/logstash:7.17.8
        ports:
        - containerPort: 5044
          name: app-input
        volumeMounts:
        - name: app-config-volume
          mountPath: /usr/share/logstash/config/logstash.yml
          subPath: logstash.yml
        - name: app-config-volume
          mountPath: /usr/share/logstash/config/pipelines.yml
          subPath: pipelines.yml
        - name: app-config-volume
          mountPath: /usr/share/logstash/pipeline/application.conf
          subPath: application.conf
        resources:
          requests:
            cpu: "1"
            memory: "2Gi"
          limits:
            cpu: "2"
            memory: "4Gi"
        env:
        - name: LS_JAVA_OPTS
          value: "-Xmx2g -Xms2g"
      volumes:
      - name: app-config-volume
        configMap:
          name: logstash-app-config

---
# System Logstash Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash-system
  namespace: elk
spec:
  replicas: 2
  selector:
    matchLabels:
      app: logstash
      pipeline: system
  template:
    metadata:
      labels:
        app: logstash
        pipeline: system
    spec:
      containers:
      - name: logstash
        image: docker.elastic.co/logstash/logstash:7.17.8
        # System-specific configuration
        # ...

---
# Security Logstash Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash-security
  namespace: elk
spec:
  replicas: 3
  selector:
    matchLabels:
      app: logstash
      pipeline: security
  template:
    metadata:
      labels:
        app: logstash
        pipeline: security
    spec:
      containers:
      - name: logstash
        image: docker.elastic.co/logstash/logstash:7.17.8
        # Security-specific configuration
        # ...
```

## Resource Configuration

Properly configuring resources is critical for Logstash performance in Kubernetes.

### JVM and Container Resource Alignment

Logstash in Kubernetes requires careful alignment between JVM heap and container resources:

```yaml
spec:
  containers:
  - name: logstash
    image: docker.elastic.co/logstash/logstash:7.17.8
    resources:
      requests:
        cpu: "2"
        memory: "4Gi"
      limits:
        cpu: "4"
        memory: "6Gi"
    env:
    - name: LS_JAVA_OPTS
      value: "-Xmx4g -Xms4g -XX:+UseG1GC -XX:G1ReservePercent=25 -XX:InitiatingHeapOccupancyPercent=75"
```

### Resource Guidelines

1. **JVM Heap Size**:
   - Set Xms and Xmx to the same value to prevent heap resizing
   - Allocate ~50-75% of container memory to JVM heap
   - Never exceed container memory limits

2. **CPU Allocation**:
   - Request enough CPU for consistent performance
   - Set CPU limits higher than requests to handle spikes
   - Consider one CPU core per worker
   
3. **Memory Allocation**:
   - Request enough memory to handle expected event volume
   - Include overhead for operating system, JVM, and off-heap data
   - Higher memory for complex pipelines with stateful filters like aggregation

4. **Recommended Starting Points**:

| Event Volume | CPU Request | Memory Request | JVM Heap | Workers |
|--------------|------------|----------------|----------|---------|
| Low (<5K/s)  | 0.5 - 1    | 1Gi - 2Gi      | 1g       | 2       |
| Medium (5K-15K/s) | 2 - 4 | 4Gi - 8Gi      | 4g       | 4-8     |
| High (>15K/s)| 4+         | 8Gi+           | 8g+      | 8+      |

### Customizing Worker Configuration

The number of workers can significantly impact performance:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-config
  namespace: elk
data:
  pipelines.yml: |
    - pipeline.id: main
      path.config: "/usr/share/logstash/pipeline"
      pipeline.workers: 4
      pipeline.batch.size: 250
```

## Pipeline Management

Efficient pipeline management is essential for Logstash on Kubernetes.

### ConfigMap-based Pipeline Management

Using ConfigMaps is the simplest approach:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-pipelines
  namespace: elk
data:
  main.conf: |
    input { ... }
    filter { ... }
    output { ... }
  
  secondary.conf: |
    input { ... }
    filter { ... }
    output { ... }

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash
  namespace: elk
spec:
  # ...
  template:
    spec:
      containers:
      - name: logstash
        # ...
        volumeMounts:
        - name: pipeline-volume
          mountPath: /usr/share/logstash/pipeline/
      volumes:
      - name: pipeline-volume
        configMap:
          name: logstash-pipelines
```

### ConfigMap Updates and Reloading

When using ConfigMaps, pipeline updates require proper handling:

1. **Manual Reload**:
   Update the ConfigMap and restart pods (using rolling updates)

2. **Automatic Reload**:
   Configure Logstash to watch for pipeline changes:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-config
  namespace: elk
data:
  logstash.yml: |
    http.host: "0.0.0.0"
    config.reload.automatic: true
    config.reload.interval: 30s
    path.config: /usr/share/logstash/pipeline/*.conf
```

### Git-Sync for Pipeline Management

For more advanced pipeline management, Git-Sync can be used:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash
  namespace: elk
spec:
  # ...
  template:
    spec:
      containers:
      - name: logstash
        # ...
        volumeMounts:
        - name: pipeline-volume
          mountPath: /usr/share/logstash/pipeline/
      - name: git-sync
        image: k8s.gcr.io/git-sync:v3.1.6
        args:
        - "--repo=https://github.com/example/logstash-pipelines.git"
        - "--branch=main"
        - "--root=/tmp/git"
        - "--dest=pipelines"
        - "--wait=60"
        volumeMounts:
        - name: pipeline-volume
          mountPath: /tmp/git
        env:
        - name: GIT_SYNC_USERNAME
          valueFrom:
            secretKeyRef:
              name: git-credentials
              key: username
        - name: GIT_SYNC_PASSWORD
          valueFrom:
            secretKeyRef:
              name: git-credentials
              key: password
      volumes:
      - name: pipeline-volume
        emptyDir: {}
```

## ConfigMaps and Secrets

Managing configuration and secrets securely is crucial for Logstash deployments.

### Configuration Layers

A typical Logstash deployment uses multiple configuration layers:

1. **Base Configuration (logstash.yml)**:
   - System settings
   - Monitoring configuration
   - Reload settings

2. **Pipeline Management (pipelines.yml)**:
   - Pipeline definitions
   - Worker settings
   - Pipeline-specific configurations

3. **Pipeline Definitions (.conf files)**:
   - Input, filter, and output configurations
   - Data transformation logic

### Sensitive Configuration with Secrets

For sensitive data like credentials, use Kubernetes Secrets:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: logstash-elasticsearch-credentials
  namespace: elk
type: Opaque
data:
  username: ZWxhc3RpYw==  # base64 encoded "elastic"
  password: Y2hhbmdlbWU=  # base64 encoded "changeme"

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-pipeline
  namespace: elk
data:
  logstash.conf: |
    input {
      # Input configuration
    }
    
    filter {
      # Filter configuration
    }
    
    output {
      elasticsearch {
        hosts => ["elasticsearch-master:9200"]
        user => "${ES_USERNAME}"
        password => "${ES_PASSWORD}"
        index => "logs-%{+YYYY.MM.dd}"
      }
    }

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash
  namespace: elk
spec:
  # ...
  template:
    spec:
      containers:
      - name: logstash
        # ...
        env:
        - name: ES_USERNAME
          valueFrom:
            secretKeyRef:
              name: logstash-elasticsearch-credentials
              key: username
        - name: ES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: logstash-elasticsearch-credentials
              key: password
```

### Managing Large Configurations

For large configurations, consider splitting them into multiple ConfigMaps:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-system-config
  namespace: elk
data:
  logstash.yml: |
    # System configuration
  
  pipelines.yml: |
    # Pipeline definitions

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-main-pipeline
  namespace: elk
data:
  main.conf: |
    # Main pipeline configuration

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-secondary-pipeline
  namespace: elk
data:
  secondary.conf: |
    # Secondary pipeline configuration
```

## StatefulSets vs Deployments

Choosing between StatefulSets and Deployments depends on your Logstash workload characteristics.

### Deployment-based Logstash

Suitable for most scenarios where Logstash processes events without needing to maintain state:

**Advantages**:
- Simpler to manage
- Easier scaling
- Automatic load balancing

**Example**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash
  namespace: elk
spec:
  replicas: 3
  selector:
    matchLabels:
      app: logstash
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: logstash
    spec:
      containers:
      - name: logstash
        image: docker.elastic.co/logstash/logstash:7.17.8
        # ...
```

### StatefulSet-based Logstash

Necessary when Logstash requires persistent storage or stable network identities:

**Advantages**:
- Stable, predictable pod names and DNS entries
- Ordered, graceful deployment and scaling
- Persistent storage for each pod

**Example**:
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: logstash
  namespace: elk
spec:
  serviceName: "logstash-headless"
  replicas: 3
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
        image: docker.elastic.co/logstash/logstash:7.17.8
        volumeMounts:
        - name: data
          mountPath: /usr/share/logstash/data
        # ...
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "standard"
      resources:
        requests:
          storage: 10Gi

---
apiVersion: v1
kind: Service
metadata:
  name: logstash-headless
  namespace: elk
spec:
  clusterIP: None
  selector:
    app: logstash
  ports:
  - port: 5044
    name: beats
```

### Choosing Between Deployment and StatefulSet

Use StatefulSets when:
- Using persistent queues
- Requiring ordered startup/shutdown
- Needing stable network identities
- Using stateful plugins that require persistent storage

Use Deployments when:
- Running stateless pipelines
- Focusing on horizontal scaling
- Implementing rolling updates
- Requiring faster recovery and scaling

## Scaling and High Availability

Properly scaling Logstash ensures optimal performance under varying loads.

### Horizontal Pod Autoscaler for Logstash

Auto-scaling based on CPU utilization:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: logstash
  namespace: elk
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: logstash
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### Custom Metrics Autoscaling

For more advanced scaling based on Logstash-specific metrics:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: logstash
  namespace: elk
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: logstash
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: logstash_events_in_count
      target:
        type: AverageValue
        averageValue: 5000
```

This requires setting up a custom metrics pipeline with Prometheus and the Prometheus Adapter.

### Affinity and Anti-Affinity Rules

Distribute Logstash pods across nodes for high availability:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash
  namespace: elk
spec:
  # ...
  template:
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - logstash
              topologyKey: kubernetes.io/hostname
```

### Pod Disruption Budget

Ensure high availability during cluster operations:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: logstash-pdb
  namespace: elk
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: logstash
```

## Persistent Storage

For Logstash workloads requiring persistent storage, proper configuration is essential.

### Persistent Queue Configuration

Enabling persistent queues for Logstash:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-config
  namespace: elk
data:
  logstash.yml: |
    http.host: "0.0.0.0"
    queue.type: persisted
    queue.max_bytes: 1gb
    path.queue: /usr/share/logstash/data/queue

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: logstash
  namespace: elk
spec:
  serviceName: "logstash-headless"
  replicas: 3
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
        image: docker.elastic.co/logstash/logstash:7.17.8
        volumeMounts:
        - name: queue-data
          mountPath: /usr/share/logstash/data
  volumeClaimTemplates:
  - metadata:
      name: queue-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "standard"
      resources:
        requests:
          storage: 10Gi
```

### Storage Class Selection

Choose appropriate storage classes based on performance requirements:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iopsPerGB: "3000"
  throughput: "125"
```

### Local Storage for High Performance

For high-performance requirements, local storage might be necessary:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: logstash-local-pv-1
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - worker-node-1
```

## Monitoring and Logging

Monitoring Logstash in Kubernetes requires specific configuration.

### Prometheus and Grafana Monitoring

Using the Prometheus exporter for Logstash:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-config
  namespace: elk
data:
  logstash.yml: |
    http.host: "0.0.0.0"
    xpack.monitoring.enabled: false
    
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash
  namespace: elk
spec:
  # ...
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9600"
        prometheus.io/path: "/_node/stats"
    spec:
      containers:
      - name: logstash
        # ...
```

### Elasticsearch Monitoring

Using Elasticsearch for monitoring:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-config
  namespace: elk
data:
  logstash.yml: |
    http.host: "0.0.0.0"
    xpack.monitoring.enabled: true
    xpack.monitoring.elasticsearch.hosts: ["http://elasticsearch-master:9200"]
```

### Metricbeat for Advanced Monitoring

Deploying Metricbeat for comprehensive Logstash monitoring:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: metricbeat
  namespace: elk
spec:
  selector:
    matchLabels:
      app: metricbeat
  template:
    metadata:
      labels:
        app: metricbeat
    spec:
      containers:
      - name: metricbeat
        image: docker.elastic.co/beats/metricbeat:7.17.8
        args: [
          "-c", "/etc/metricbeat.yml",
          "-e",
        ]
        volumeMounts:
        - name: config
          mountPath: /etc/metricbeat.yml
          subPath: metricbeat.yml
      volumes:
      - name: config
        configMap:
          name: metricbeat-config
          items:
          - key: metricbeat.yml
            path: metricbeat.yml

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: metricbeat-config
  namespace: elk
data:
  metricbeat.yml: |
    metricbeat.modules:
    - module: logstash
      metricsets: ["node", "node_stats"]
      period: 10s
      hosts: ["logstash:9600"]
      xpack.enabled: true
    
    output.elasticsearch:
      hosts: ["elasticsearch-master:9200"]
```

### Log Collection from Logstash Pods

Collecting Logstash logs with Filebeat:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: elk
spec:
  selector:
    matchLabels:
      app: filebeat
  template:
    metadata:
      labels:
        app: filebeat
    spec:
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:7.17.8
        volumeMounts:
        - name: config
          mountPath: /usr/share/filebeat/filebeat.yml
          subPath: filebeat.yml
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: filebeat-config
          items:
          - key: filebeat.yml
            path: filebeat.yml
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: elk
data:
  filebeat.yml: |
    filebeat.inputs:
    - type: container
      paths:
        - /var/log/containers/logstash-*.log
      processors:
        - add_kubernetes_metadata:
            host: ${NODE_NAME}
            matchers:
            - logs_path:
                logs_path: "/var/log/containers/"
    
    output.elasticsearch:
      hosts: ["elasticsearch-master:9200"]
      index: "filebeat-logstash-%{+yyyy.MM.dd}"
```

## Security Considerations

Securing Logstash in Kubernetes environments requires specific configurations.

### Network Policies

Restricting traffic to and from Logstash pods:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: logstash-network-policy
  namespace: elk
spec:
  podSelector:
    matchLabels:
      app: logstash
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: filebeat
    ports:
    - protocol: TCP
      port: 5044
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 9600
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: elasticsearch
    ports:
    - protocol: TCP
      port: 9200
```

### TLS Configuration

Enabling TLS for inputs and outputs:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: logstash-certificates
  namespace: elk
type: Opaque
data:
  logstash.keystore.jks: <base64-encoded-keystore>
  logstash.truststore.jks: <base64-encoded-truststore>
  keystore-password: <base64-encoded-password>
  truststore-password: <base64-encoded-password>

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-config
  namespace: elk
data:
  logstash.conf: |
    input {
      beats {
        port => 5044
        ssl => true
        ssl_certificate => "/etc/logstash/certificates/logstash.crt"
        ssl_key => "/etc/logstash/certificates/logstash.key"
        ssl_certificate_authorities => ["/etc/logstash/certificates/ca.crt"]
        ssl_verify_mode => "force_peer"
      }
    }
    
    output {
      elasticsearch {
        hosts => ["https://elasticsearch-master:9200"]
        user => "${ES_USERNAME}"
        password => "${ES_PASSWORD}"
        index => "logs-%{+YYYY.MM.dd}"
        ssl => true
        ssl_certificate_verification => true
        cacert => "/etc/logstash/certificates/ca.crt"
      }
    }

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash
  namespace: elk
spec:
  # ...
  template:
    spec:
      containers:
      - name: logstash
        # ...
        volumeMounts:
        - name: certificates
          mountPath: /etc/logstash/certificates
          readOnly: true
      volumes:
      - name: certificates
        secret:
          secretName: logstash-certificates
```

### Service Account Configuration

Using dedicated service accounts with limited permissions:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: logstash-service-account
  namespace: elk

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: logstash-role
  namespace: elk
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "watch", "list"]
  resourceNames: ["logstash-config", "logstash-pipelines"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: logstash-role-binding
  namespace: elk
subjects:
- kind: ServiceAccount
  name: logstash-service-account
  namespace: elk
roleRef:
  kind: Role
  name: logstash-role
  apiGroup: rbac.authorization.k8s.io

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash
  namespace: elk
spec:
  # ...
  template:
    spec:
      serviceAccountName: logstash-service-account
      # ...
```

## Performance Tuning

Optimizing Logstash performance in Kubernetes environments requires consideration of various factors.

### Tuning JVM Options

Optimizing JVM settings for Logstash:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash
  namespace: elk
spec:
  # ...
  template:
    spec:
      containers:
      - name: logstash
        # ...
        env:
        - name: LS_JAVA_OPTS
          value: >-
            -Xms4g
            -Xmx4g
            -XX:+UseG1GC
            -XX:G1ReservePercent=25
            -XX:InitiatingHeapOccupancyPercent=75
            -XX:+HeapDumpOnOutOfMemoryError
            -XX:HeapDumpPath=/usr/share/logstash/logs/heapdump.hprof
            -Xlog:gc*,gc+age=trace,safepoint:file=/usr/share/logstash/logs/gc.log:utctime,pid,tags:filecount=5,filesize=100m
```

### Pipeline Tuning

Optimizing pipeline settings for better performance:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-config
  namespace: elk
data:
  pipelines.yml: |
    - pipeline.id: main
      path.config: "/usr/share/logstash/pipeline/main.conf"
      pipeline.workers: 8
      pipeline.batch.size: 2000
      pipeline.batch.delay: 50
```

### CPU and Memory Tuning

Guidelines for CPU and memory allocation:

1. **CPU Allocation**:
   - Rule of thumb: 1 CPU core per 2-4 pipeline workers
   - Event complexity increases CPU requirements
   - Set requests at 50-75% of limits to allow for scaling

2. **Memory Allocation**:
   - JVM heap: ~50-75% of container memory
   - Memory needs scale with batch size and filter complexity
   - Complex filters like aggregate or translate require more memory
   - Reserve memory for off-heap operations

### Network Optimization

Optimizing network settings for high throughput:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-config
  namespace: elk
data:
  logstash.yml: |
    http.host: "0.0.0.0"
    pipeline.output.workers: 8  # Increased output workers for Elasticsearch
    queue.type: persisted       # Persistent queue for reliability
    queue.max_bytes: 4gb        # Larger queue for burst handling
```

### Performance Testing with Different Configurations

Example performance test matrix:

| Workers | Batch Size | Batch Delay | Queue Type | Events/sec | Latency (ms) | CPU Use | Mem Use |
|---------|------------|-------------|------------|------------|--------------|---------|---------|
| 4       | 125        | 50          | memory     | 10,000     | 150          | 70%     | 2GB     |
| 4       | 250        | 50          | memory     | 15,000     | 200          | 75%     | 2.2GB   |
| 8       | 250        | 50          | memory     | 25,000     | 250          | 85%     | 2.5GB   |
| 8       | 500        | 50          | persisted  | 22,000     | 350          | 80%     | 3GB     |

## Integration with Elasticsearch and Kibana

Properly integrating Logstash with Elasticsearch and Kibana in Kubernetes.

### Service Discovery

Using Kubernetes service discovery for Elasticsearch:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-config
  namespace: elk
data:
  logstash.conf: |
    input {
      # Input configuration
    }
    
    filter {
      # Filter configuration
    }
    
    output {
      elasticsearch {
        hosts => ["${ELASTICSEARCH_HOST}:${ELASTICSEARCH_PORT}"]
        user => "${ELASTICSEARCH_USER}"
        password => "${ELASTICSEARCH_PASSWORD}"
        index => "logs-%{+YYYY.MM.dd}"
      }
    }

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash
  namespace: elk
spec:
  # ...
  template:
    spec:
      containers:
      - name: logstash
        # ...
        env:
        - name: ELASTICSEARCH_HOST
          value: "elasticsearch-master"
        - name: ELASTICSEARCH_PORT
          value: "9200"
        - name: ELASTICSEARCH_USER
          valueFrom:
            secretKeyRef:
              name: elasticsearch-credentials
              key: username
        - name: ELASTICSEARCH_PASSWORD
          valueFrom:
            secretKeyRef:
              name: elasticsearch-credentials
              key: password
```

### Index Templates and Lifecycle Policies

Creating index templates for Logstash-generated indices:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: logstash-index-setup
  namespace: elk
spec:
  template:
    spec:
      containers:
      - name: index-setup
        image: curlimages/curl:latest
        command:
        - sh
        - -c
        - |
          # Create ILM policy
          curl -X PUT -u "${ES_USER}:${ES_PASSWORD}" "${ES_HOST}:9200/_ilm/policy/logs-policy" -H 'Content-Type: application/json' -d '
          {
            "policy": {
              "phases": {
                "hot": {
                  "min_age": "0ms",
                  "actions": {
                    "rollover": {
                      "max_age": "1d",
                      "max_size": "50gb"
                    },
                    "set_priority": {
                      "priority": 100
                    }
                  }
                },
                "warm": {
                  "min_age": "3d",
                  "actions": {
                    "shrink": {
                      "number_of_shards": 1
                    },
                    "forcemerge": {
                      "max_num_segments": 1
                    },
                    "set_priority": {
                      "priority": 50
                    }
                  }
                },
                "delete": {
                  "min_age": "30d",
                  "actions": {
                    "delete": {}
                  }
                }
              }
            }
          }'
          
          # Create index template
          curl -X PUT -u "${ES_USER}:${ES_PASSWORD}" "${ES_HOST}:9200/_index_template/logs-template" -H 'Content-Type: application/json' -d '
          {
            "index_patterns": ["logs-*"],
            "template": {
              "settings": {
                "number_of_shards": 3,
                "number_of_replicas": 1,
                "index.lifecycle.name": "logs-policy",
                "index.lifecycle.rollover_alias": "logs"
              },
              "mappings": {
                "properties": {
                  "@timestamp": { "type": "date" },
                  "message": { "type": "text" },
                  "kubernetes": {
                    "properties": {
                      "namespace_name": { "type": "keyword" },
                      "pod_name": { "type": "keyword" },
                      "container_name": { "type": "keyword" }
                    }
                  },
                  "host": { "type": "keyword" },
                  "severity": { "type": "keyword" }
                }
              }
            }
          }'
        env:
        - name: ES_HOST
          value: "elasticsearch-master"
        - name: ES_USER
          valueFrom:
            secretKeyRef:
              name: elasticsearch-credentials
              key: username
        - name: ES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: elasticsearch-credentials
              key: password
      restartPolicy: OnFailure
```

### Kibana Dashboards Setup

Setting up Kibana dashboards for Logstash data:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: logstash-kibana-setup
  namespace: elk
spec:
  template:
    spec:
      containers:
      - name: kibana-setup
        image: docker.elastic.co/kibana/kibana:7.17.8
        command:
        - sh
        - -c
        - |
          kibana-setup-init && \
          kibana-keystore add elasticsearch.username --stdin <<< "${ELASTICSEARCH_USERNAME}" && \
          kibana-keystore add elasticsearch.password --stdin <<< "${ELASTICSEARCH_PASSWORD}" && \
          kibana --optimize && \
          KIBANA_INDEX_URL="${ELASTICSEARCH_HOST}:${ELASTICSEARCH_PORT}/.kibana" && \
          
          # Wait for Kibana index to be ready
          echo "Waiting for Kibana index..." && \
          until $(curl --output /dev/null --silent --head --fail -u "${ELASTICSEARCH_USERNAME}:${ELASTICSEARCH_PASSWORD}" "$KIBANA_INDEX_URL"); do
            printf '.'
            sleep 5
          done && \
          
          # Create index pattern
          curl -X POST -H "kbn-xsrf: true" \
            -H "Content-Type: application/json" \
            "http://${KIBANA_HOST}:5601/api/saved_objects/index-pattern/logs-*" \
            -u "${ELASTICSEARCH_USERNAME}:${ELASTICSEARCH_PASSWORD}" \
            -d '{"attributes": {"title": "logs-*", "timeFieldName": "@timestamp"}}'
          
          # Import dashboards
          curl -X POST -H "kbn-xsrf: true" \
            -u "${ELASTICSEARCH_USERNAME}:${ELASTICSEARCH_PASSWORD}" \
            "http://${KIBANA_HOST}:5601/api/kibana/dashboards/import" \
            -d @/tmp/dashboards/logstash-dashboard.json
        env:
        - name: ELASTICSEARCH_HOST
          value: "elasticsearch-master"
        - name: ELASTICSEARCH_PORT
          value: "9200"
        - name: KIBANA_HOST
          value: "kibana"
        - name: ELASTICSEARCH_USERNAME
          valueFrom:
            secretKeyRef:
              name: elasticsearch-credentials
              key: username
        - name: ELASTICSEARCH_PASSWORD
          valueFrom:
            secretKeyRef:
              name: elasticsearch-credentials
              key: password
        volumeMounts:
        - name: dashboards
          mountPath: /tmp/dashboards
      volumes:
      - name: dashboards
        configMap:
          name: logstash-dashboards
      restartPolicy: OnFailure
```

## Handling Upgrades

Managing Logstash upgrades in Kubernetes environments requires careful planning.

### Rolling Upgrade Strategy

Using Kubernetes rolling updates for Logstash upgrades:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash
  namespace: elk
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  # ...
```

### Blue-Green Deployment for Major Upgrades

For major version upgrades, consider a blue-green approach:

```yaml
# Blue deployment (current version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash-blue
  namespace: elk
spec:
  replicas: 3
  selector:
    matchLabels:
      app: logstash
      deployment: blue
  template:
    metadata:
      labels:
        app: logstash
        deployment: blue
    spec:
      containers:
      - name: logstash
        image: docker.elastic.co/logstash/logstash:7.14.0
        # ...

---
# Green deployment (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash-green
  namespace: elk
spec:
  replicas: 0  # Start with 0 replicas
  selector:
    matchLabels:
      app: logstash
      deployment: green
  template:
    metadata:
      labels:
        app: logstash
        deployment: green
    spec:
      containers:
      - name: logstash
        image: docker.elastic.co/logstash/logstash:7.17.8
        # ...

---
# Service that will switch between blue and green
apiVersion: v1
kind: Service
metadata:
  name: logstash
  namespace: elk
spec:
  selector:
    app: logstash
    deployment: blue  # Initially points to blue
  ports:
  - port: 5044
    name: beats
    targetPort: 5044
```

For the transition:

1. Scale up the green deployment
2. Test the green deployment
3. Update the service selector to point to green
4. Scale down the blue deployment once traffic is successfully flowing through green

### Plugin Compatibility

Managing plugin compatibility during upgrades:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-init-scripts
  namespace: elk
data:
  plugin-setup.sh: |
    #!/bin/bash
    # Install or update plugins
    bin/logstash-plugin install logstash-filter-geoip
    bin/logstash-plugin install logstash-filter-translate
    bin/logstash-plugin install logstash-output-lumberjack
    
    # Check for plugin compatibility with current Logstash version
    bin/logstash-plugin list --verbose

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash
  namespace: elk
spec:
  # ...
  template:
    spec:
      initContainers:
      - name: plugin-setup
        image: docker.elastic.co/logstash/logstash:7.17.8
        command:
        - sh
        - -c
        - /tmp/scripts/plugin-setup.sh
        volumeMounts:
        - name: scripts
          mountPath: /tmp/scripts
        - name: logstash-data
          mountPath: /usr/share/logstash/data
      volumes:
      - name: scripts
        configMap:
          name: logstash-init-scripts
          defaultMode: 0755
```

## Summary

Deploying Logstash on Kubernetes provides powerful capabilities for scalable, resilient data processing pipelines. Key considerations include:

1. **Deployment Strategy**: Choose between Deployments and StatefulSets based on state requirements
2. **Configuration Management**: Use ConfigMaps and Secrets for configuration, with Git-Sync for advanced use cases
3. **Resource Allocation**: Properly configure CPU, memory, and JVM settings for optimal performance
4. **Pipeline Management**: Efficiently manage pipelines with appropriate reload strategies
5. **High Availability**: Implement anti-affinity rules, PDBs, and proper scaling configurations
6. **Persistent Storage**: Use appropriate storage classes and consider local storage for high performance
7. **Security**: Implement network policies, TLS configuration, and proper service accounts
8. **Monitoring**: Set up comprehensive monitoring with Prometheus or Elasticsearch Monitoring
9. **Performance Tuning**: Optimize JVM settings, pipeline workers, batch size, and resource allocation
10. **Upgrade Management**: Implement proper strategies for smooth upgrades

By following these best practices, you can build a robust, scalable, and maintainable Logstash deployment on Kubernetes that integrates seamlessly with the rest of your ELK Stack.

## References

- [Logstash Reference](https://www.elastic.co/guide/en/logstash/current/index.html)
- [Kubernetes StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Kubernetes ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Logstash Kubernetes Deployment Examples](https://github.com/elastic/logstash/tree/main/docs/static/kubernetes)