# ELK on Kubernetes Fundamentals

## Introduction to Running ELK on Kubernetes

Kubernetes has become the de facto standard for container orchestration, and running the ELK Stack on Kubernetes offers significant advantages:

- **Scalability**: Easily scale components up or down based on demand
- **Resilience**: Automatic recovery from failures
- **Resource Efficiency**: Better utilization of cluster resources
- **Declarative Configuration**: Infrastructure as code for the entire stack
- **Consistency**: Same deployment across development, testing, and production
- **Portability**: Run on any Kubernetes cluster (on-premises, cloud, or hybrid)

This chapter covers the fundamental concepts and considerations for running Elasticsearch, Logstash, and Kibana on Kubernetes.

## Kubernetes Resources for ELK Stack

When deploying ELK on Kubernetes, you'll work with various Kubernetes resources:

### StatefulSets

Used primarily for Elasticsearch nodes because they:
- Provide stable, persistent storage
- Guarantee stable network identifiers
- Ensure ordered deployment and scaling
- Support predictable pod naming (`elasticsearch-0`, `elasticsearch-1`, etc.)

### Deployments

Used for stateless components like Kibana, Logstash, and ingest nodes:
- Support automatic updates and rollbacks
- Enable replica scaling for high availability
- Don't guarantee stable network identifiers
- Don't require persistent storage (except for Logstash persistent queues)

### Services

Create stable endpoints for each component:
- Cluster IP services for internal communication
- Headless services for StatefulSets (direct pod access)
- LoadBalancer or NodePort services for external access to Kibana

### ConfigMaps and Secrets

Store configuration and sensitive information:
- ConfigMaps for component configuration files
- Secrets for certificates, credentials, and API keys

### PersistentVolumes and PersistentVolumeClaims

Manage data storage for Elasticsearch:
- Define storage requirements in PVCs
- Connect to underlying storage through PVs
- Support various storage classes (SSD, HDD)

### Ingress

Expose Kibana and Elasticsearch API with proper routing and TLS termination:
- Path-based routing
- TLS certificate management
- Authentication integration

## Key Considerations for ELK on Kubernetes

### Resource Requirements

Elasticsearch is resource-intensive. Ensure your Kubernetes cluster has:

- **CPU**: Minimum 2 cores per Elasticsearch pod, 4+ recommended for production
- **Memory**: Minimum 4GB per Elasticsearch pod, 8GB+ recommended
- **Storage**: Fast SSD storage (avoid network storage when possible)
- **Network**: Low-latency network between pods

### Storage Considerations

Elasticsearch performance depends heavily on storage:

- Use local storage when possible for better performance
- If using network storage, choose high-performance options
- Set appropriate storage class for each tier (hot/warm/cold)
- Define proper resource limits and requests

### Network Considerations

Elasticsearch nodes need to communicate efficiently:

- Use headless services for direct pod-to-pod communication
- Configure transport layer settings properly
- Set up proper network policies
- Consider network latency between availability zones

### Security Considerations

Protect your ELK stack on Kubernetes:

- Use TLS for all communications
- Implement proper authentication and authorization
- Use Kubernetes Secrets for credentials
- Configure network policies to restrict traffic
- Consider service meshes for additional security layers

## Deployment Options for ELK on Kubernetes

### Manual Deployment with YAML

Create and manage your own Kubernetes manifests:
- Full control over configuration
- Higher complexity and maintenance burden
- Good for learning and understanding the components
- More work to handle upgrades

### Helm Charts

Use community or official Helm charts:
- Simplified deployment with configurable values
- Standard patterns and practices
- Easier upgrades
- Less customization than manual deployment

### Operators

Use specialized Kubernetes operators like Elastic Cloud on Kubernetes (ECK):
- Automated provisioning and management
- Built-in best practices
- Seamless upgrades and scaling
- Native custom resources for Elastic Stack components
- Simplified security setup

## Architectural Patterns for ELK on Kubernetes

### Single-Cluster Pattern

Simplest approach with all components in one Kubernetes cluster:
- Easier to set up and manage
- Suitable for development and smaller production deployments
- Limited fault isolation
- Single point of failure for the Kubernetes control plane

### Multi-Cluster Pattern

Distribute ELK components across multiple Kubernetes clusters:
- Better fault isolation
- Can span multiple regions or data centers
- More complex setup and management
- Requires cross-cluster networking

### Dedicated ELK Cluster Pattern

Run ELK in a dedicated Kubernetes cluster, separate from applications:
- Better resource isolation
- Dedicated team can manage the ELK infrastructure
- Reduced risk to application workloads
- May increase overall infrastructure costs

## Monitoring Kubernetes with ELK

While running ELK on Kubernetes, you can also use it to monitor Kubernetes itself:

- **Metricbeat**: Collect Kubernetes metrics
- **Filebeat**: Collect container and Kubernetes logs
- **APM Server**: Monitor applications running on Kubernetes
- **Packetbeat**: Monitor network traffic between services

This creates a powerful feedback loop where ELK monitors both your applications and the platform it runs on.

## Kubernetes Concepts Relevant to ELK

### Pod Affinity and Anti-Affinity

Control where Elasticsearch pods are scheduled:
- Keep master nodes on separate physical hosts (anti-affinity)
- Keep data nodes close to related services (affinity)
- Distribute nodes across availability zones

Example:
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
            - key: role
              operator: In
              values:
                - master
        topologyKey: "kubernetes.io/hostname"
```

### Resource Limits and Requests

Set appropriate resource limits and requests:
- CPU and memory requests for guaranteed resources
- CPU and memory limits to prevent overconsumption
- Ensure JVM heap size is set according to container limits

Example:
```yaml
resources:
  requests:
    memory: "4Gi"
    cpu: "1000m"
  limits:
    memory: "4Gi"
    cpu: "2000m"
```

### Readiness and Liveness Probes

Ensure Kubernetes can monitor component health:
- Liveness probes to detect and restart unhealthy containers
- Readiness probes to determine when pods can receive traffic
- Properly configured timeout and threshold values

Example:
```yaml
livenessProbe:
  httpGet:
    path: /_cluster/health
    port: 9200
  initialDelaySeconds: 90
  periodSeconds: 10
  timeoutSeconds: 5
readinessProbe:
  httpGet:
    path: /_cluster/health?wait_for_status=yellow
    port: 9200
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
```

### Init Containers

Use init containers for setup tasks:
- Increase system settings (vm.max_map_count)
- Check for prerequisites
- Prepare storage or configuration
- Wait for dependencies to be ready

Example:
```yaml
initContainers:
  - name: increase-vm-max-map-count
    image: busybox
    command: ["sysctl", "-w", "vm.max_map_count=262144"]
    securityContext:
      privileged: true
  - name: wait-for-dependencies
    image: busybox
    command: ['sh', '-c', 'until ping -c 1 elasticsearch-0.elasticsearch; do echo waiting for elasticsearch-0; sleep 2; done;']
```

### Security Contexts

Set appropriate security settings:
- Non-root users for containers
- Read-only file systems where possible
- Drop unnecessary capabilities
- Apply pod security policies

Example:
```yaml
securityContext:
  runAsUser: 1000
  fsGroup: 1000
  runAsNonRoot: true
```

## Deploying a Basic ELK Stack on Kubernetes

Below is a conceptual overview of deploying a basic ELK stack on Kubernetes. In the next chapters, we'll provide detailed manifests and step-by-step instructions.

### 1. Create Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: elk
```

### 2. Create ConfigMaps

For Elasticsearch:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: elasticsearch-config
  namespace: elk
data:
  elasticsearch.yml: |
    cluster.name: k8s-logs
    node.name: ${HOSTNAME}
    network.host: 0.0.0.0
    discovery.seed_hosts: ["elasticsearch-0.elasticsearch"]
    cluster.initial_master_nodes: ["elasticsearch-0"]
    xpack.security.enabled: true
    xpack.monitoring.collection.enabled: true
```

For Kibana:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-config
  namespace: elk
data:
  kibana.yml: |
    server.name: kibana
    server.host: "0.0.0.0"
    elasticsearch.hosts: ["http://elasticsearch:9200"]
    monitoring.ui.container.elasticsearch.enabled: true
```

### 3. Create Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: elasticsearch-credentials
  namespace: elk
type: Opaque
data:
  username: ZWxhc3RpYw==  # elastic
  password: Y2hhbmdlbWU=  # changeme
```

### 4. Create StorageClass (if needed)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```

### 5. Deploy Elasticsearch

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
  namespace: elk
spec:
  serviceName: elasticsearch
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:8.8.0
        env:
        - name: ES_JAVA_OPTS
          value: "-Xms1g -Xmx1g"
        ports:
        - containerPort: 9200
          name: http
        - containerPort: 9300
          name: transport
        volumeMounts:
        - name: config
          mountPath: /usr/share/elasticsearch/config/elasticsearch.yml
          subPath: elasticsearch.yml
        - name: data
          mountPath: /usr/share/elasticsearch/data
      volumes:
      - name: config
        configMap:
          name: elasticsearch-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast
      resources:
        requests:
          storage: 100Gi
```

### 6. Create Elasticsearch Services

```yaml
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: elk
spec:
  selector:
    app: elasticsearch
  ports:
  - port: 9200
    targetPort: 9200
    name: http
  - port: 9300
    targetPort: 9300
    name: transport
---
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch-headless
  namespace: elk
spec:
  clusterIP: None
  selector:
    app: elasticsearch
  ports:
  - port: 9200
    targetPort: 9200
    name: http
  - port: 9300
    targetPort: 9300
    name: transport
```

### 7. Deploy Kibana

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: elk
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:8.8.0
        ports:
        - containerPort: 5601
          name: http
        volumeMounts:
        - name: config
          mountPath: /usr/share/kibana/config/kibana.yml
          subPath: kibana.yml
      volumes:
      - name: config
        configMap:
          name: kibana-config
```

### 8. Create Kibana Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: elk
spec:
  selector:
    app: kibana
  ports:
  - port: 5601
    targetPort: 5601
  type: LoadBalancer
```

### 9. Deploy Filebeat for Log Collection

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

## Best Practices for ELK on Kubernetes

1. **Resource Allocation**: Properly size resources based on workload
2. **Node Affinity**: Use anti-affinity to distribute nodes across hosts
3. **Persistent Storage**: Use fast local storage when possible
4. **Security**: Enable TLS and proper authentication
5. **Monitoring**: Monitor both ELK and Kubernetes itself
6. **Scaling**: Scale horizontally by adding more nodes rather than making nodes larger
7. **Backup**: Implement regular snapshot backups
8. **Updates**: Have a strategy for rolling updates with minimal downtime
9. **Configuration Management**: Use GitOps principles for configuration
10. **High Availability**: Deploy across multiple availability zones

## Common Challenges and Solutions

### Challenge: Elasticsearch Clustering Issues

**Solution**:
- Use proper discovery settings
- Ensure headless services are correctly configured
- Set up proper network policies
- Use init containers to ensure prerequisites are met

### Challenge: Resource Pressure

**Solution**:
- Set appropriate resource limits and requests
- Monitor resource usage and adjust as needed
- Implement auto-scaling
- Use node selectors to place pods on appropriate nodes

### Challenge: Storage Performance

**Solution**:
- Use local SSDs for hot data
- Configure proper storage classes
- Implement tiered storage (hot/warm/cold)
- Monitor I/O performance

### Challenge: Security Configuration

**Solution**:
- Use Kubernetes secrets for credentials
- Implement TLS for all communications
- Configure network policies
- Use service accounts with minimal permissions

## Conclusion

Running the ELK Stack on Kubernetes provides significant advantages in terms of scalability, resilience, and resource efficiency. By understanding the fundamental concepts outlined in this chapter, you'll be well-prepared to deploy and manage ELK on Kubernetes.

In the next chapters, we'll dive deeper into deploying specific components of the ELK Stack on Kubernetes, including detailed configurations, optimizations, and real-world examples.