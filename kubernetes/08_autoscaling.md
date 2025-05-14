# Autoscaling ELK Stack on Kubernetes

This chapter covers strategies, implementations, and best practices for autoscaling the ELK Stack on Kubernetes.

## Table of Contents
- [Introduction to Autoscaling for ELK](#introduction-to-autoscaling-for-elk)
- [Kubernetes Native Autoscaling](#kubernetes-native-autoscaling)
- [Elasticsearch Autoscaling](#elasticsearch-autoscaling)
- [Kibana Autoscaling](#kibana-autoscaling)
- [Logstash Autoscaling](#logstash-autoscaling)
- [Beats Autoscaling](#beats-autoscaling)
- [Custom Metrics for Autoscaling](#custom-metrics-for-autoscaling)
- [Multi-Component Autoscaling](#multi-component-autoscaling)
- [Autoscaling with the Elastic Cloud on Kubernetes (ECK)](#autoscaling-with-the-elastic-cloud-on-kubernetes-eck)
- [Monitoring and Troubleshooting Autoscaling](#monitoring-and-troubleshooting-autoscaling)
- [Performance Testing for Autoscaling](#performance-testing-for-autoscaling)
- [Cost Optimization Strategies](#cost-optimization-strategies)

## Introduction to Autoscaling for ELK

Autoscaling is a critical capability for maintaining ELK Stack performance and cost efficiency in Kubernetes environments. It enables automatic adjustment of computational resources based on workload demands, ensuring optimal performance during peak loads while minimizing resource usage during low-demand periods.

### Why Autoscale the ELK Stack?

- **Fluctuating Workloads**: Log and event volume often varies significantly throughout the day
- **Cost Optimization**: Scale down during quiet periods to save on infrastructure costs
- **Performance Consistency**: Maintain performance SLAs even during unexpected traffic spikes
- **Operational Efficiency**: Reduce manual scaling operations and human error

### Autoscaling Considerations for ELK

The ELK Stack presents unique autoscaling challenges:

1. **Stateful vs. Stateless Components**:
   - Elasticsearch is stateful and requires careful scaling
   - Kibana and Logstash are generally stateless and easier to scale

2. **Resource-Intensive Processing**:
   - Elasticsearch is memory and CPU intensive
   - Logstash requires sufficient CPU for data processing

3. **Data-Centric Workloads**:
   - Ingestion rates affect resource requirements
   - Query complexity impacts performance

4. **Component Interdependencies**:
   - Components must scale together for balanced performance
   - Bottlenecks in one component affect the entire stack

### Types of Scaling

Two primary types of scaling apply to ELK on Kubernetes:

1. **Horizontal Pod Autoscaling (HPA)**:
   - Adding or removing pods based on metrics
   - Works well for stateless components like Kibana and Logstash
   - More complex for stateful services like Elasticsearch

2. **Vertical Pod Autoscaling (VPA)**:
   - Adjusting CPU and memory resources for existing pods
   - Useful for optimizing resource allocation
   - Often requires pod restarts which impacts availability

## Kubernetes Native Autoscaling

### Horizontal Pod Autoscaler (HPA)

The HPA automatically scales the number of pods based on observed CPU utilization or other custom metrics.

#### Basic HPA for Kibana

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kibana-hpa
  namespace: elastic
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kibana
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 75
```

#### HPA for Logstash

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: logstash-hpa
  namespace: elastic
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: logstash
  minReplicas: 2
  maxReplicas: 15
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 65
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 120
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 20
        periodSeconds: 120
```

### Vertical Pod Autoscaler (VPA)

The VPA automatically adjusts the CPU and memory resource requests and limits for containers.

#### VPA for Elasticsearch

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: elasticsearch-vpa
  namespace: elastic
spec:
  targetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: elasticsearch-data
  updatePolicy:
    updateMode: "Auto"  # Can be "Off", "Initial", "Recreate", or "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: elasticsearch
      minAllowed:
        memory: "2Gi"
        cpu: "500m"
      maxAllowed:
        memory: "16Gi"
        cpu: "4"
      controlledResources: ["cpu", "memory"]
```

#### VPA for Kibana

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: kibana-vpa
  namespace: elastic
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kibana
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: kibana
      minAllowed:
        memory: "512Mi"
        cpu: "200m"
      maxAllowed:
        memory: "4Gi"
        cpu: "2"
      controlledResources: ["cpu", "memory"]
```

### Combined HPA and VPA Approach

For optimal resource utilization, you can combine HPA and VPA:

1. **Configure VPA in recommendation mode**:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: logstash-vpa
  namespace: elastic
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: logstash
  updatePolicy:
    updateMode: "Off"  # Recommendation mode only
  resourcePolicy:
    containerPolicies:
    - containerName: logstash
      minAllowed:
        memory: "1Gi"
        cpu: "500m"
      maxAllowed:
        memory: "8Gi"
        cpu: "4"
```

2. **Use HPA for actual scaling**:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: logstash-hpa
  namespace: elastic
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

3. **Periodically review VPA recommendations to adjust resource requests**

## Elasticsearch Autoscaling

Elasticsearch requires special consideration for autoscaling due to its stateful nature and data distribution model.

### Data Tiering and Autoscaling

Implement different scaling strategies based on node roles:

#### Hot Nodes Autoscaling

Hot nodes handle active writes and recent data, requiring more aggressive scaling:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: elasticsearch-hot-hpa
  namespace: elastic
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: elasticsearch-hot
  minReplicas: 3
  maxReplicas: 8
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 65
  - type: Pods
    pods:
      metric:
        name: elasticsearch_indexing_pressure
      target:
        type: AverageValue
        averageValue: 75
```

#### Warm Nodes Autoscaling

Warm nodes handle less active data with less aggressive scaling:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: elasticsearch-warm-hpa
  namespace: elastic
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: elasticsearch-warm
  minReplicas: 2
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 600
```

### Shard Management During Scaling

When scaling Elasticsearch, you need to manage shard allocation:

1. **Scale-Up Considerations**:
   - New nodes should receive shards for balanced distribution
   - Control shard allocation to prevent excessive rebalancing

2. **Scale-Down Considerations**:
   - Ensure data is properly migrated before node termination
   - Use controlled shard allocation settings

Example configuration to adjust allocation settings during scaling operations:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: elasticsearch-scaling-scripts
  namespace: elastic
data:
  pre-scale-down.sh: |
    #!/bin/bash
    # Script to run before scaling down Elasticsearch
    curl -XPUT "http://elasticsearch:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
    {
      "persistent": {
        "cluster.routing.allocation.enable": "primaries",
        "cluster.routing.allocation.cluster_concurrent_rebalance": "2",
        "cluster.routing.allocation.node_concurrent_recoveries": "2",
        "cluster.routing.allocation.node_initial_primaries_recoveries": "4"
      }
    }'
    
  post-scale-up.sh: |
    #!/bin/bash
    # Script to run after scaling up Elasticsearch
    curl -XPUT "http://elasticsearch:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
    {
      "persistent": {
        "cluster.routing.allocation.enable": "all"
      }
    }'
    # Wait for cluster to stabilize
    until curl -s "http://elasticsearch:9200/_cluster/health" | grep -q '"status":"green"'; do
      echo "Waiting for cluster to stabilize..."
      sleep 30
    done
```

### Scaling Master Nodes

Master nodes require special consideration for scaling:

1. **Always maintain an odd number of master nodes** (typically 3 or 5)
2. **Scale with caution** as master node changes affect cluster stability
3. **Configure `minimum_master_nodes` correctly** (typically `(n/2)+1` where n is the number of master nodes)

Example StatefulSet configuration for master nodes:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch-master
  namespace: elastic
spec:
  replicas: 3  # Should always be an odd number
  selector:
    matchLabels:
      app: elasticsearch
      role: master
  template:
    metadata:
      labels:
        app: elasticsearch
        role: master
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:8.8.0
        env:
        - name: node.roles
          value: master
        - name: cluster.initial_master_nodes
          value: "elasticsearch-master-0,elasticsearch-master-1,elasticsearch-master-2"
        - name: discovery.zen.minimum_master_nodes
          value: "2"  # (3/2)+1 = 2
```

## Kibana Autoscaling

As a stateless frontend application, Kibana is well-suited for horizontal scaling.

### HPA for Kibana Based on Requests

Scale Kibana based on HTTP request rate using Prometheus metrics:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kibana-http-hpa
  namespace: elastic
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kibana
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: 50
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
```

### HPA for Kibana Based on Concurrent Users

Scale Kibana based on concurrent user sessions:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kibana-sessions-hpa
  namespace: elastic
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kibana
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: kibana_concurrent_connections
      target:
        type: AverageValue
        averageValue: 100
```

### Scaling Kibana for Different User Types

Different types of Kibana users have different resource impacts:

1. **Dashboard Viewers**: Many concurrent viewers with lower individual resource usage
2. **Data Analysts**: Fewer users but higher resource consumption due to complex queries
3. **Administrators**: Even fewer users but potentially resource-intensive operations

To handle different user categories, implement scaling notifications from a central scheduling system:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: kibana-predictive-scaling
  namespace: elastic
spec:
  schedule: "0 8 * * 1-5"  # 8 AM on weekdays
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: scale-up
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              kubectl scale deployment kibana -n elastic --replicas=6
          restartPolicy: OnFailure
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: kibana-scale-down
  namespace: elastic
spec:
  schedule: "0 18 * * 1-5"  # 6 PM on weekdays
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: scale-down
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              kubectl scale deployment kibana -n elastic --replicas=2
          restartPolicy: OnFailure
```

## Logstash Autoscaling

Logstash scaling depends on ingestion rates and processing complexity.

### Queue-Based Autoscaling

Scale Logstash based on input queue metrics:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: logstash-queue-hpa
  namespace: elastic
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: logstash
  minReplicas: 2
  maxReplicas: 15
  metrics:
  - type: Pods
    pods:
      metric:
        name: logstash_queue_events_count
      target:
        type: AverageValue
        averageValue: 1000
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 200
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
```

### Processing Time-Based Autoscaling

Scale based on event processing time metrics:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: logstash-processing-hpa
  namespace: elastic
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: logstash
  minReplicas: 2
  maxReplicas: 15
  metrics:
  - type: Pods
    pods:
      metric:
        name: logstash_node_event_processing_time
      target:
        type: AverageValue
        averageValue: 200  # milliseconds
```

### Pipeline-Specific Scaling

For complex Logstash deployments with multiple pipelines, scale based on specific pipeline metrics:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash-apache
  namespace: elastic
spec:
  replicas: 2
  selector:
    matchLabels:
      app: logstash
      pipeline: apache
  template:
    metadata:
      labels:
        app: logstash
        pipeline: apache
    spec:
      containers:
      - name: logstash
        image: docker.elastic.co/logstash/logstash:8.8.0
        # Config for Apache logs pipeline
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logstash-syslog
  namespace: elastic
spec:
  replicas: 3
  selector:
    matchLabels:
      app: logstash
      pipeline: syslog
  template:
    metadata:
      labels:
        app: logstash
        pipeline: syslog
    spec:
      containers:
      - name: logstash
        image: docker.elastic.co/logstash/logstash:8.8.0
        # Config for Syslog pipeline
```

Implement separate HPA for each pipeline:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: logstash-apache-hpa
  namespace: elastic
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: logstash-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: logstash_pipeline_apache_events_in_count
      target:
        type: AverageValue
        averageValue: 1000
```

## Beats Autoscaling

Beats components require careful consideration for scaling.

### Filebeat DaemonSet Scaling

Filebeat typically runs as a DaemonSet, which automatically scales with nodes:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: elastic
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
        image: docker.elastic.co/beats/filebeat:8.8.0
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

### Metricbeat Resource Scaling

Adjust Metricbeat resources based on monitoring needs:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: metricbeat-vpa
  namespace: elastic
spec:
  targetRef:
    apiVersion: apps/v1
    kind: DaemonSet
    name: metricbeat
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: metricbeat
      minAllowed:
        memory: "128Mi"
        cpu: "100m"
      maxAllowed:
        memory: "512Mi"
        cpu: "500m"
```

### Heartbeat Deployment Scaling

Scale Heartbeat based on target endpoints:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: heartbeat-hpa
  namespace: elastic
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: heartbeat
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: External
    external:
      metric:
        name: heartbeat_monitors_count
      target:
        type: AverageValue
        averageValue: 50  # Monitors per instance
```

## Custom Metrics for Autoscaling

### Prometheus Adapter for Custom Metrics

Install and configure the Prometheus Adapter to expose Elasticsearch, Logstash, and Kibana metrics to the Kubernetes metrics API:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-adapter
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus-adapter
  template:
    metadata:
      labels:
        app: prometheus-adapter
    spec:
      containers:
      - name: prometheus-adapter
        image: registry.k8s.io/prometheus-adapter/prometheus-adapter:v0.10.0
        args:
        - --secure-port=6443
        - --cert-dir=/var/run/serving-cert
        - --prometheus-url=http://prometheus-server.monitoring.svc:9090/
        - --metrics-relist-interval=1m
        - --v=4
        - --config=/etc/adapter/config.yaml
        ports:
        - containerPort: 6443
        volumeMounts:
        - name: config
          mountPath: /etc/adapter/
      volumes:
      - name: config
        configMap:
          name: prometheus-adapter-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-adapter-config
  namespace: monitoring
data:
  config.yaml: |
    rules:
    - seriesQuery: 'elasticsearch_indices_docs_count'
      resources:
        overrides:
          namespace: {resource: "namespace"}
          pod: {resource: "pod"}
      name:
        matches: "elasticsearch_indices_docs_count"
        as: "elasticsearch_docs_count"
      metricsQuery: 'sum(elasticsearch_indices_docs_count{<<.LabelMatchers>>})'
    - seriesQuery: 'logstash_node_event_duration_in_millis'
      resources:
        overrides:
          namespace: {resource: "namespace"}
          pod: {resource: "pod"}
      name:
        matches: "logstash_node_event_duration_in_millis"
        as: "logstash_processing_time"
      metricsQuery: 'avg(logstash_node_event_duration_in_millis{<<.LabelMatchers>>}) by (pod)'
    - seriesQuery: 'kibana_concurrent_connections'
      resources:
        overrides:
          namespace: {resource: "namespace"}
          pod: {resource: "pod"}
      name:
        matches: "kibana_concurrent_connections"
        as: "kibana_concurrent_connections"
      metricsQuery: 'sum(kibana_concurrent_connections{<<.LabelMatchers>>}) by (pod)'
```

### Elasticsearch Custom Metrics

Define custom metrics for Elasticsearch scaling:

1. **Indexing Pressure**:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: elasticsearch-indexing-hpa
  namespace: elastic
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: elasticsearch-data
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: elasticsearch_indexing_pressure
      target:
        type: AverageValue
        averageValue: 75
```

2. **Search Latency**:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: elasticsearch-search-hpa
  namespace: elastic
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: elasticsearch-data
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: elasticsearch_search_latency_ms
      target:
        type: AverageValue
        averageValue: 200
```

### Logstash Custom Metrics

Define custom metrics for Logstash scaling:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: logstash-pipeline-hpa
  namespace: elastic
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
        name: logstash_pipeline_events_in_count
      target:
        type: AverageValue
        averageValue: 1000
  - type: Pods
    pods:
      metric:
        name: logstash_pipeline_events_duration_in_millis
      target:
        type: AverageValue
        averageValue: 200
```

## Multi-Component Autoscaling

### Coordinated Scaling Approach

To ensure balanced performance across the stack, coordinate scaling between components:

1. **Implement a Controller** to manage scaling across components:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elk-scaling-controller
  namespace: elastic
spec:
  replicas: 1
  selector:
    matchLabels:
      app: elk-scaling-controller
  template:
    metadata:
      labels:
        app: elk-scaling-controller
    spec:
      serviceAccountName: elk-scaling-controller
      containers:
      - name: controller
        image: custom-elk-controller:latest
        env:
        - name: ELASTICSEARCH_SERVICE
          value: "elasticsearch-master"
        - name: LOGSTASH_DEPLOYMENT
          value: "logstash"
        - name: KIBANA_DEPLOYMENT
          value: "kibana"
```

2. **Define Scaling Ratios** between components:

For every N Elasticsearch nodes, you might need:
- M Logstash instances (typically M = N/2)
- P Kibana instances (typically P = N/4)

### Event-Driven Scaling

Implement event-driven scaling using Kubernetes Events:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: elk-scaling-rules
  namespace: elastic
data:
  rules.yaml: |
    rules:
    - name: "Elasticsearch scale-up"
      condition: "elasticsearch_cpu_usage > 70% for 5m"
      actions:
        - scale:
            component: "elasticsearch-data"
            replicas: "+1"
        - wait: "10m"
        - scale:
            component: "logstash"
            replicas: "+1"
    - name: "Logstash queue growth"
      condition: "logstash_queue_size > 5000 for 2m"
      actions:
        - scale:
            component: "logstash"
            replicas: "+2"
    - name: "Kibana user growth"
      condition: "kibana_active_connections > 200 for 3m"
      actions:
        - scale:
            component: "kibana"
            replicas: "+1"
```

## Autoscaling with the Elastic Cloud on Kubernetes (ECK)

The Elastic Cloud on Kubernetes (ECK) operator provides built-in capabilities for managing Elasticsearch and other components.

### ECK Autoscaling for Elasticsearch

Use the ECK Autoscaling API to manage Elasticsearch resources:

```yaml
apiVersion: autoscaling.k8s.elastic.co/v1alpha1
kind: ElasticsearchAutoscaler
metadata:
  name: elasticsearch-autoscaler
  namespace: elastic
spec:
  elasticsearchRef:
    name: elasticsearch
  policies:
  - name: data-nodes-policy
    roles: ["data", "ingest"]
    resources:
      nodeCount:
        min: 3
        max: 10
        controllerName: resources-controller
    deciders:
      - name: fixed-shards
        settings:
          storage: 500GB
```

### Combining ECK with Custom Scaling Logic

Extend ECK's built-in scaling with custom logic:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: elasticsearch-workload-predictor
  namespace: elastic
spec:
  schedule: "0 * * * *"  # Hourly
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: elastic-job
          containers:
          - name: predict-scaling
            image: custom-scaling-predictor:latest
            env:
            - name: ELASTICSEARCH_URL
              value: "https://elasticsearch-es-http:9200"
            - name: MODEL_ENDPOINT
              value: "http://prediction-service:8080/predict"
            volumeMounts:
            - name: elasticsearch-certs
              mountPath: /etc/elasticsearch/certs
          volumes:
          - name: elasticsearch-certs
            secret:
              secretName: elasticsearch-es-http-certs-public
          restartPolicy: OnFailure
```

### ECK Component Auto-Scaling

Using ECK for managing all Elastic Stack components:

```yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana
  namespace: elastic
spec:
  version: 8.8.0
  count: 2  # Initial replica count
  elasticsearchRef:
    name: elasticsearch
  podTemplate:
    spec:
      containers:
      - name: kibana
        resources:
          requests:
            memory: "1Gi"
            cpu: "0.5"
          limits:
            memory: "2Gi"
            cpu: "1"
```

For auto-scaling Kibana with HPA:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kibana-hpa
  namespace: elastic
spec:
  scaleTargetRef:
    apiVersion: kibana.k8s.elastic.co/v1
    kind: Kibana
    name: kibana
  minReplicas: 2
  maxReplicas: 6
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

## Monitoring and Troubleshooting Autoscaling

### Monitoring Autoscaling Decisions

Track autoscaling actions and behaviors:

```yaml
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: autoscale-metrics
  namespace: elastic
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
    - module: prometheus
      period: 10s
      hosts: ["prometheus-server:9090"]
      metrics_path: '/federate'
      query:
        'match[]': '{__name__=~".*autoscal.*"}'
    processors:
    - add_fields:
        target: ""
        fields:
          type: "autoscaling_metrics"
  deployment:
    podTemplate:
      spec:
        serviceAccountName: metricbeat
```

Create a Kibana dashboard for monitoring autoscaling events:

1. **HPA Status Panel**: Shows current HPA status and recent scaling decisions
2. **Component Resource Utilization**: Shows resource usage vs. scaling thresholds
3. **Scaling Events Timeline**: Shows when scaling actions occurred
4. **Performance Impact**: Shows performance before and after scaling events

### Troubleshooting Scaling Issues

Common autoscaling issues and solutions:

1. **Scaling Too Frequently (Thrashing)**
   - Implement a longer stabilization window
   - Adjust scaling thresholds
   - Use more appropriate metrics

   ```yaml
   behavior:
     scaleDown:
       stabilizationWindowSeconds: 600  # 10 minutes
     scaleUp:
       stabilizationWindowSeconds: 300  # 5 minutes
   ```

2. **Not Scaling When Expected**
   - Verify HPA has access to required metrics
   - Check metric values against thresholds
   - Validate RBAC permissions for HPA controller

   ```bash
   kubectl describe hpa elasticsearch-hpa -n elastic
   kubectl get --raw "/apis/metrics.k8s.io/v1beta1/namespaces/elastic/pods"
   ```

3. **Delayed Scaling Response**
   - Reduce stabilization window (cautiously)
   - Check metrics polling interval
   - Verify metrics are current and accurate

## Performance Testing for Autoscaling

### Simulating Load for Testing

Create load testing jobs to validate autoscaling:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: elasticsearch-load-test
  namespace: elastic
spec:
  template:
    spec:
      containers:
      - name: es-load-generator
        image: custom-es-benchmarking:latest
        env:
        - name: ELASTICSEARCH_URL
          value: "http://elasticsearch:9200"
        - name: TEST_DURATION
          value: "60m"
        - name: RAMP_UP
          value: "10m"
        - name: QUERIES_PER_SECOND
          value: "100"
      restartPolicy: Never
  backoffLimit: 1
```

### Measuring Scaling Effectiveness

Use custom dashboards to evaluate autoscaling behavior:

1. **Response Time vs. Scale**: Chart showing how response time changes with scaling
2. **Cost vs. Performance**: Track resource usage and cost against performance
3. **Scaling Response Time**: Measure time from threshold breach to effective scaling

### Chaos Testing Autoscaling

Implement chaos testing to verify resilient autoscaling:

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: elasticsearch-pod-failure
  namespace: elastic
spec:
  action: pod-failure
  mode: one
  selector:
    namespaces:
      - elastic
    labelSelectors:
      app: elasticsearch
      role: data
  duration: "5m"
  scheduler:
    cron: "@every 30m"
```

## Cost Optimization Strategies

### Time-Based Scaling

Implement time-based scaling for predictable workload patterns:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nighttime-scale-down
  namespace: elastic
spec:
  schedule: "0 20 * * *"  # 8 PM every day
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: scaling-job
          containers:
          - name: scale-down
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              kubectl patch elasticsearch elasticsearch -n elastic --type merge --patch '{"spec":{"nodeSets":[{"name":"data","count":3}]}}'
              kubectl scale deployment logstash -n elastic --replicas=2
              kubectl scale deployment kibana -n elastic --replicas=1
          restartPolicy: OnFailure
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: morning-scale-up
  namespace: elastic
spec:
  schedule: "0 6 * * *"  # 6 AM every day
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: scaling-job
          containers:
          - name: scale-up
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              kubectl patch elasticsearch elasticsearch -n elastic --type merge --patch '{"spec":{"nodeSets":[{"name":"data","count":6}]}}'
              kubectl scale deployment logstash -n elastic --replicas=4
              kubectl scale deployment kibana -n elastic --replicas=3
          restartPolicy: OnFailure
```

### Node Affinity for Cost Saving

Leverage node affinity to use spot/preemptible instances for suitable components:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: elastic
spec:
  # ...
  template:
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: cloud.google.com/gke-spot
                operator: In
                values:
                - "true"
```

For stateful workloads like Elasticsearch, use a mixed approach:

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch
  namespace: elastic
spec:
  version: 8.8.0
  nodeSets:
  - name: masters
    count: 3
    config:
      node.roles: ["master"]
    podTemplate:
      spec:
        affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: cloud.google.com/gke-spot
                  operator: NotIn
                  values:
                  - "true"
  - name: hot-data
    count: 4
    config:
      node.roles: ["data", "ingest"]
    podTemplate:
      spec:
        affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: cloud.google.com/gke-spot
                  operator: NotIn
                  values:
                  - "true"
  - name: warm-data
    count: 3
    config:
      node.roles: ["data"]
      node.attr.data: warm
    podTemplate:
      spec:
        affinity:
          nodeAffinity:
            preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                - key: cloud.google.com/gke-spot
                  operator: In
                  values:
                  - "true"
```

### Resource Optimization

Implement periodic resource optimization reviews:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: resource-optimization-review
  namespace: elastic
spec:
  schedule: "0 1 * * 0"  # Weekly at 1 AM on Sunday
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: optimizer
            image: custom-resource-optimizer:latest
            env:
            - name: ELASTICSEARCH_URL
              value: "http://elasticsearch:9200"
            - name: OPTIMIZATION_WINDOW_DAYS
              value: "14"
            - name: OUTPUT_FORMAT
              value: "json"
            - name: NOTIFICATION_WEBHOOK
              value: "http://slack-webhook-service/alerts"
          restartPolicy: OnFailure
```

## Summary

Autoscaling the ELK Stack on Kubernetes is essential for handling variable workloads efficiently while optimizing costs. By leveraging Kubernetes' native autoscaling capabilities, custom metrics, and component-specific approaches, you can build a highly responsive and cost-effective ELK deployment.

Key takeaways:

1. **Component-Specific Scaling**: Each ELK component requires a unique autoscaling approach:
   - Elasticsearch needs careful scaling due to its stateful nature
   - Kibana benefits from horizontal scaling based on user load
   - Logstash scales well based on queue and processing metrics
   - Beats typically scale automatically as DaemonSets

2. **Custom Metrics Matter**: Standard CPU/memory metrics are insufficient for optimal ELK scaling; implement custom metrics based on workload characteristics.

3. **Coordinated Scaling**: Coordinate scaling across components to maintain balanced performance.

4. **Cost Optimization**: Implement time-based scaling, node affinity, and resource optimization to reduce costs.

5. **Monitoring Autoscaling**: Create dedicated dashboards to monitor autoscaling behavior and effectiveness.

With these strategies in place, your ELK Stack can automatically adapt to changing workloads, maintaining performance while optimizing resource usage.

## References

- [Kubernetes Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
- [Elastic Cloud on Kubernetes Documentation](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-overview.html)
- [Elasticsearch Autoscaling API](https://www.elastic.co/guide/en/elasticsearch/reference/current/autoscaling-apis.html)
- [Prometheus Adapter for Kubernetes Metrics](https://github.com/kubernetes-sigs/prometheus-adapter)