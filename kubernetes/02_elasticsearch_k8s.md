# Elasticsearch on Kubernetes

## Introduction

Deploying Elasticsearch on Kubernetes requires careful planning due to its stateful nature and resource requirements. This chapter provides a comprehensive guide to deploying, configuring, and managing Elasticsearch clusters on Kubernetes.

## Prerequisites

Before deploying Elasticsearch on Kubernetes, ensure your environment meets these requirements:

- Kubernetes cluster running version 1.19+
- Sufficient resources (CPU, memory, storage)
- Kubernetes permissions to create StatefulSets, Services, ConfigMaps, etc.
- StorageClass that supports the performance requirements of Elasticsearch

## Planning Your Elasticsearch Deployment

### Node Types and Roles

In Kubernetes, you typically deploy different StatefulSets for different Elasticsearch node roles:

1. **Master Nodes**: Control the cluster
2. **Data Nodes**: Store and search data
3. **Ingest Nodes**: Pre-process documents
4. **Coordinating Nodes**: Load balancers for search requests

For smaller deployments, you can combine roles, but production systems benefit from dedicated role separation.

### Resource Sizing

Start with these minimum resource recommendations per node:

| Node Type | CPU Request | Memory Request | Storage |
|-----------|------------|----------------|---------|
| Master    | 2 CPU      | 4GB            | 20GB    |
| Data      | 4 CPU      | 8GB            | 100GB+  |
| Ingest    | 2 CPU      | 4GB            | 20GB    |
| Coordinating | 2 CPU    | 4GB           | N/A     |

Remember to set JVM heap size to about 50% of the container's memory limit, but no more than 31GB to avoid inefficient memory management.

### Storage Considerations

Elasticsearch performance depends heavily on storage performance:

- Use SSDs for all node types in production
- Consider local storage (via local-path provisioner or local PVs) for best performance
- Set up storage classes with appropriate performance characteristics
- For tiered architecture, use different storage classes for hot, warm, and cold tiers

## Basic Elasticsearch Deployment

Let's start with a basic three-node Elasticsearch cluster on Kubernetes.

### Create Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: elasticsearch
```

### Create ConfigMap for Elasticsearch Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: elasticsearch-config
  namespace: elasticsearch
data:
  elasticsearch.yml: |
    cluster.name: elasticsearch-cluster
    network.host: 0.0.0.0
    
    # Minimum nodes that must be available to form a cluster
    discovery.zen.minimum_master_nodes: 2
    
    # Node discovery settings
    discovery.seed_hosts: ["elasticsearch-0.elasticsearch-headless", "elasticsearch-1.elasticsearch-headless", "elasticsearch-2.elasticsearch-headless"]
    cluster.initial_master_nodes: ["elasticsearch-0", "elasticsearch-1", "elasticsearch-2"]
    
    # Enable X-Pack security
    xpack.security.enabled: true
    
    # TLS configuration
    xpack.security.transport.ssl.enabled: true
    xpack.security.transport.ssl.verification_mode: certificate
    xpack.security.transport.ssl.keystore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12
    xpack.security.transport.ssl.truststore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12
    
    # Memory settings
    bootstrap.memory_lock: true
    
    # Set node roles (dynamic based on environment variables)
    node.master: ${NODE_MASTER}
    node.data: ${NODE_DATA}
    node.ingest: ${NODE_INGEST}
    
    # Path settings
    path.data: /usr/share/elasticsearch/data
    path.logs: /usr/share/elasticsearch/logs
```

### Create Secrets for Credentials and Certificates

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: elasticsearch-credentials
  namespace: elasticsearch
type: Opaque
data:
  username: ZWxhc3RpYw==  # elastic (base64 encoded)
  password: UGFzc3dvcmQxMjM=  # Password123 (base64 encoded)

---
apiVersion: v1
kind: Secret
metadata:
  name: elasticsearch-certificates
  namespace: elasticsearch
type: Opaque
data:
  elastic-certificates.p12: ""  # This would contain your actual certificate
  elastic-certificates.password: ""  # Certificate password
```

### Create Headless Service for Elasticsearch

```yaml
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch-headless
  namespace: elasticsearch
  labels:
    app: elasticsearch
spec:
  clusterIP: None  # Headless service
  selector:
    app: elasticsearch
  ports:
  - name: http
    port: 9200
    targetPort: 9200
  - name: transport
    port: 9300
    targetPort: 9300
```

### Create Service for External Access

```yaml
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: elasticsearch
  labels:
    app: elasticsearch
spec:
  selector:
    app: elasticsearch
  ports:
  - name: http
    port: 9200
    targetPort: 9200
  - name: transport
    port: 9300
    targetPort: 9300
  type: ClusterIP  # Change to LoadBalancer for external access
```

### Create StatefulSet for Elasticsearch Nodes

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
  namespace: elasticsearch
spec:
  serviceName: elasticsearch-headless
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      terminationGracePeriodSeconds: 120
      initContainers:
      - name: fix-permissions
        image: busybox
        command: ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
        securityContext:
          runAsUser: 0
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
      - name: increase-vm-max-map
        image: busybox
        command: ["sysctl", "-w", "vm.max_map_count=262144"]
        securityContext:
          privileged: true
      - name: increase-fd-ulimit
        image: busybox
        command: ["sh", "-c", "ulimit -n 65536"]
        securityContext:
          privileged: true
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:8.8.0
        resources:
          requests:
            cpu: "1"
            memory: "2Gi"
          limits:
            cpu: "2"
            memory: "2Gi"
        ports:
        - containerPort: 9200
          name: http
        - containerPort: 9300
          name: transport
        env:
        - name: cluster.name
          value: elasticsearch-cluster
        - name: node.name
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: discovery.seed_hosts
          value: "elasticsearch-0.elasticsearch-headless,elasticsearch-1.elasticsearch-headless,elasticsearch-2.elasticsearch-headless"
        - name: cluster.initial_master_nodes
          value: "elasticsearch-0,elasticsearch-1,elasticsearch-2"
        - name: bootstrap.memory_lock
          value: "true"
        - name: ES_JAVA_OPTS
          value: "-Xms1g -Xmx1g"
        - name: NODE_MASTER
          value: "true"
        - name: NODE_DATA
          value: "true"
        - name: NODE_INGEST
          value: "true"
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
        - name: config
          mountPath: /usr/share/elasticsearch/config/elasticsearch.yml
          subPath: elasticsearch.yml
        - name: certificates
          mountPath: /usr/share/elasticsearch/config/certs
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: elasticsearch-config
      - name: certificates
        secret:
          secretName: elasticsearch-certificates
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "standard"  # Change to your fast storage class
      resources:
        requests:
          storage: 100Gi
```

## Advanced Elasticsearch Configurations

### Role-Based Deployment

For larger production environments, separate node roles into different StatefulSets:

#### Master Nodes StatefulSet (3 nodes)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch-master
  namespace: elasticsearch
spec:
  serviceName: elasticsearch-master-headless
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch-master
  template:
    metadata:
      labels:
        app: elasticsearch-master
        role: master
    spec:
      # Similar to previous StatefulSet but with:
      containers:
      - name: elasticsearch
        # Same as before, but with:
        env:
        - name: NODE_MASTER
          value: "true"
        - name: NODE_DATA
          value: "false"
        - name: NODE_INGEST
          value: "false"
        # Resources suited for master nodes
        resources:
          requests:
            cpu: "1"
            memory: "2Gi"
          limits:
            cpu: "2"
            memory: "2Gi"
```

#### Data Nodes StatefulSet (minimum 3 nodes)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch-data
  namespace: elasticsearch
spec:
  serviceName: elasticsearch-data-headless
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch-data
  template:
    metadata:
      labels:
        app: elasticsearch-data
        role: data
    spec:
      # Similar to previous StatefulSet but with:
      containers:
      - name: elasticsearch
        # Same as before, but with:
        env:
        - name: NODE_MASTER
          value: "false"
        - name: NODE_DATA
          value: "true"
        - name: NODE_INGEST
          value: "false"
        # Resources suited for data nodes
        resources:
          requests:
            cpu: "4"
            memory: "8Gi"
          limits:
            cpu: "4"
            memory: "8Gi"
```

#### Ingest Nodes Deployment (stateless, can use a Deployment)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch-ingest
  namespace: elasticsearch
spec:
  replicas: 2
  selector:
    matchLabels:
      app: elasticsearch-ingest
  template:
    metadata:
      labels:
        app: elasticsearch-ingest
        role: ingest
    spec:
      # Similar to previous StatefulSet but with:
      containers:
      - name: elasticsearch
        # Same as before, but with:
        env:
        - name: NODE_MASTER
          value: "false"
        - name: NODE_DATA
          value: "false"
        - name: NODE_INGEST
          value: "true"
        # Resources suited for ingest nodes
        resources:
          requests:
            cpu: "2"
            memory: "4Gi"
          limits:
            cpu: "2"
            memory: "4Gi"
```

### Hot-Warm-Cold Architecture

For time-series data like logs, a hot-warm-cold architecture can optimize storage costs:

#### Hot Nodes (SSDs for active data)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch-hot
  namespace: elasticsearch
spec:
  # Similar to data nodes, but with:
  template:
    spec:
      containers:
      - name: elasticsearch
        env:
        - name: node.attr.data
          value: "hot"
        # Additional settings
```

#### Warm Nodes (HDDs for older data)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch-warm
  namespace: elasticsearch
spec:
  # Similar to data nodes, but with different storage class and:
  template:
    spec:
      containers:
      - name: elasticsearch
        env:
        - name: node.attr.data
          value: "warm"
        # Additional settings
```

### Configuration for Index Lifecycle Management

Create an ILM policy for transitioning indices between tiers:

```
PUT _ilm/policy/logs_policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_size": "50GB",
            "max_age": "1d"
          },
          "set_priority": {
            "priority": 100
          }
        }
      },
      "warm": {
        "min_age": "3d",
        "actions": {
          "allocate": {
            "include": {
              "data": "warm"
            }
          },
          "set_priority": {
            "priority": 50
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
          },
          "set_priority": {
            "priority": 0
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

## Security Configurations

### Enabling Transport Layer Security (TLS)

Generate certificates for Elasticsearch:

```bash
# Create a certificate authority
elasticsearch-certutil ca --out /tmp/elastic-stack-ca.p12 --pass ""

# Generate certificates for nodes
elasticsearch-certutil cert --ca /tmp/elastic-stack-ca.p12 --pass "" --out /tmp/elastic-certificates.p12 --pass elastic
```

Create a Secret containing these certificates:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: elasticsearch-certificates
  namespace: elasticsearch
type: Opaque
data:
  elastic-certificates.p12: <base64-encoded-cert>
  elastic-certificates.password: ZWxhc3RpYw==  # "elastic" in base64
```

Update Elasticsearch configuration to use TLS:

```yaml
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12
```

### Setting Up Users and Roles

Create a script to set up built-in users after Elasticsearch starts:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: elasticsearch-setup-users
  namespace: elasticsearch
data:
  setup-users.sh: |
    #!/bin/bash
    # Wait for Elasticsearch to start
    until curl -s http://localhost:9200 -u elastic:${ELASTIC_PASSWORD}; do
      sleep 10
    done
    
    # Set built-in user passwords
    elasticsearch-setup-passwords auto -b
```

Create a Job to execute this script:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: elasticsearch-setup-users
  namespace: elasticsearch
spec:
  template:
    spec:
      containers:
      - name: setup-users
        image: docker.elastic.co/elasticsearch/elasticsearch:8.8.0
        command: ["/bin/bash", "/scripts/setup-users.sh"]
        env:
        - name: ELASTIC_PASSWORD
          valueFrom:
            secretKeyRef:
              name: elasticsearch-credentials
              key: password
        volumeMounts:
        - name: setup-script
          mountPath: /scripts
      volumes:
      - name: setup-script
        configMap:
          name: elasticsearch-setup-users
          defaultMode: 0755
      restartPolicy: OnFailure
```

## Resource Management and Pod Distribution

### Node Affinity and Anti-Affinity

Use pod anti-affinity to distribute Elasticsearch nodes across Kubernetes nodes:

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values:
          - elasticsearch
      topologyKey: "kubernetes.io/hostname"
```

For multi-zone clusters, distribute across availability zones:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: failure-domain.beta.kubernetes.io/zone
          operator: In
          values:
          - us-east-1a
          - us-east-1b
          - us-east-1c
```

### Resource Limits and Requests

Set appropriate resource limits and requests:

```yaml
resources:
  requests:
    memory: "4Gi"
    cpu: "1000m"
  limits:
    memory: "4Gi"
    cpu: "2000m"
```

Always ensure:
- Memory limits and requests are equal to prevent OOM kills
- CPU limits are set higher than requests to allow for burst capacity
- JVM heap size is set to about 50% of container memory

### PodDisruptionBudget

Create a PodDisruptionBudget to ensure availability during node maintenance:

```yaml
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: elasticsearch-pdb
  namespace: elasticsearch
spec:
  minAvailable: 2  # Or maxUnavailable: 1
  selector:
    matchLabels:
      app: elasticsearch
```

## Monitoring Elasticsearch on Kubernetes

### Prometheus and Grafana

Install the Elasticsearch Prometheus exporter:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch-exporter
  namespace: elasticsearch
spec:
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch-exporter
  template:
    metadata:
      labels:
        app: elasticsearch-exporter
    spec:
      containers:
      - name: elasticsearch-exporter
        image: justwatch/elasticsearch_exporter:1.1.0
        args:
        - "--es.uri=http://elasticsearch:9200"
        - "--es.all=true"
        - "--es.indices=true"
        - "--es.timeout=30s"
        env:
        - name: ES_USERNAME
          valueFrom:
            secretKeyRef:
              name: elasticsearch-credentials
              key: username
        - name: ES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: elasticsearch-credentials
              key: password
        ports:
        - containerPort: 9114
          name: http
```

Create a service for the exporter:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch-exporter
  namespace: elasticsearch
  labels:
    app: elasticsearch-exporter
spec:
  selector:
    app: elasticsearch-exporter
  ports:
  - port: 9114
    targetPort: 9114
```

Configure Prometheus to scrape this service.

### Metricbeat for Elasticsearch Monitoring

Deploy Metricbeat to monitor Elasticsearch:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: metricbeat
  namespace: elasticsearch
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
        image: docker.elastic.co/beats/metricbeat:8.8.0
        args: ["-c", "/etc/metricbeat.yml", "-e"]
        env:
        - name: ELASTICSEARCH_HOST
          value: elasticsearch
        - name: ELASTICSEARCH_PORT
          value: "9200"
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
        - name: config
          mountPath: /etc/metricbeat.yml
          subPath: metricbeat.yml
      volumes:
      - name: config
        configMap:
          name: metricbeat-config
```

## Backup and Restore

### Setting Up Snapshots

Create a persistent volume for snapshots:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: elasticsearch-snapshots
  namespace: elasticsearch
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
```

Register the snapshot repository:

```
PUT /_snapshot/backup
{
  "type": "fs",
  "settings": {
    "location": "/snapshots"
  }
}
```

Create a CronJob for regular snapshots:

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: elasticsearch-snapshot
  namespace: elasticsearch
spec:
  schedule: "0 1 * * *"  # Daily at 1 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: snapshot
            image: curlimages/curl
            command:
            - /bin/sh
            - -c
            - >
              curl -X PUT "elasticsearch:9200/_snapshot/backup/snapshot_$(date +%Y%m%d)?wait_for_completion=true" 
              -H "Content-Type: application/json" 
              -u "${ELASTICSEARCH_USERNAME}:${ELASTICSEARCH_PASSWORD}" 
              -d '{"indices": "*", "ignore_unavailable": true}'
            env:
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
          restartPolicy: OnFailure
```

## Scaling Elasticsearch

### Horizontal Scaling

To scale the cluster horizontally, increase the replica count:

```bash
kubectl scale statefulset elasticsearch-data --replicas=5 -n elasticsearch
```

With Horizontal Pod Autoscaler (HPA):

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: elasticsearch-data
  namespace: elasticsearch
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: elasticsearch-data
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 75
```

### Vertical Scaling

To scale vertically, update the resource requirements in the StatefulSet:

```yaml
resources:
  requests:
    memory: "8Gi"  # Increased from 4Gi
    cpu: "2000m"   # Increased from 1000m
  limits:
    memory: "8Gi"  # Increased from 4Gi
    cpu: "4000m"   # Increased from 2000m
```

Remember to adjust the JVM heap size accordingly.

## Upgrading Elasticsearch

### Minor Version Upgrade

1. Update the image version in the StatefulSet:

```yaml
containers:
- name: elasticsearch
  image: docker.elastic.co/elasticsearch/elasticsearch:8.8.1  # Updated from 8.8.0
```

2. Apply the change:

```bash
kubectl apply -f elasticsearch-statefulset.yaml
```

3. Monitor the rolling update:

```bash
kubectl rollout status statefulset/elasticsearch -n elasticsearch
```

### Major Version Upgrade

For major version upgrades, follow a more careful approach:

1. Take a snapshot of all indices
2. Deploy a new cluster with the new version
3. Restore data from the snapshot
4. Validate the new cluster
5. Switch traffic to the new cluster
6. Decommission the old cluster

## Troubleshooting

### Common Issues and Solutions

#### Elasticsearch Pods Stuck in Pending

**Cause**: Insufficient resources or PVs not provisioning

**Solution**:
- Check node resources: `kubectl describe node`
- Check PV provisioning: `kubectl get pv,pvc -n elasticsearch`
- Check storage class: `kubectl get storageclass`

#### Elasticsearch Not Forming Cluster

**Cause**: Discovery or network issues

**Solution**:
- Check logs: `kubectl logs -f elasticsearch-0 -n elasticsearch`
- Verify headless service: `kubectl get svc -n elasticsearch`
- Check network policies: `kubectl get networkpolicy -n elasticsearch`

#### Elasticsearch OOM Killed

**Cause**: Memory pressure, often due to JVM heap too large or container limits too small

**Solution**:
- Check JVM settings: `kubectl exec -it elasticsearch-0 -n elasticsearch -- env | grep ES_JAVA_OPTS`
- Adjust memory limits and JVM heap size
- Consider enabling swap

#### Performance Issues

**Cause**: Resource constraints, slow storage, or configuration issues

**Solution**:
- Check CPU/memory usage: `kubectl top pods -n elasticsearch`
- Monitor disk I/O (using Prometheus metrics)
- Optimize shard count and replication
- Review query performance with slow logs

## Conclusion

Deploying Elasticsearch on Kubernetes requires careful planning and configuration, but the benefits of automated management, scalability, and resilience make it worthwhile for production environments. This chapter covered the key aspects of running Elasticsearch on Kubernetes, from basic deployment to advanced configurations, security, monitoring, and troubleshooting.

In the next chapter, we'll explore running Logstash on Kubernetes to complete the ELK stack deployment.