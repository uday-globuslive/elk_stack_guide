# Elastic Cloud on Kubernetes (ECK)

This chapter covers deploying, managing, and operating the Elastic Stack using the Elastic Cloud on Kubernetes (ECK) operator.

## Table of Contents
- [Introduction to ECK](#introduction-to-eck)
- [Installing the ECK Operator](#installing-the-eck-operator)
- [Deploying Elasticsearch](#deploying-elasticsearch)
- [Deploying Kibana](#deploying-kibana)
- [Deploying Logstash](#deploying-logstash)
- [Deploying APM Server](#deploying-apm-server)
- [Deploying Beats](#deploying-beats)
- [Deploying Enterprise Search](#deploying-enterprise-search)
- [Security Configuration](#security-configuration)
- [High Availability Setup](#high-availability-setup)
- [Resource Management](#resource-management)
- [Upgrades and Version Management](#upgrades-and-version-management)
- [Monitoring ECK Deployments](#monitoring-eck-deployments)
- [Backing Up and Restoring](#backing-up-and-restoring)
- [Advanced Configurations](#advanced-configurations)
- [Troubleshooting](#troubleshooting)

## Introduction to ECK

Elastic Cloud on Kubernetes (ECK) is an operator pattern implementation for managing Elastic Stack resources on Kubernetes. ECK automates and simplifies the deployment, provisioning, management, and orchestration of Elasticsearch, Kibana, APM Server, Enterprise Search, Beats, Logstash, and Elastic Maps Server on Kubernetes.

### Key Benefits of ECK

1. **Simplified Operations**: Automates the deployment and management of Elastic Stack components
2. **Kubernetes-Native Experience**: Leverages Kubernetes custom resources for management
3. **Secure by Default**: Automatic TLS configuration and user creation
4. **Enterprise Readiness**: Supports advanced features like hot-warm architectures, snapshots, and more
5. **Version Management**: Simplifies upgrades and version compatibility
6. **High Availability**: Built-in support for deploying resilient clusters
7. **Elastic Stack Integration**: Seamless integration between all Elastic Stack components

### ECK Architecture

ECK follows the Kubernetes operator pattern:

1. **Custom Resource Definitions (CRDs)**: Define Elastic Stack resources (Elasticsearch, Kibana, etc.)
2. **Controller**: Watches for changes to these custom resources and reconciles the actual state with the desired state
3. **Custom Resources (CRs)**: Instance definitions of the custom resource types

```
+----------------------------------+
|          Kubernetes API          |
+----------------------------------+
              |
+----------------------------------+
|     ECK Operator Controller      |
+----------------------------------+
              |
+----------------------------------+
|        Custom Resources          |
|   Elasticsearch, Kibana, etc.    |
+----------------------------------+
              |
+----------------------------------+
|      Deployed Elastic Stack      |
|         Components               |
+----------------------------------+
```

### ECK Components

The ECK operator manages the following Elastic Stack components:

- **Elasticsearch**: The distributed search and analytics engine
- **Kibana**: The user interface for data visualization and management
- **APM Server**: Receives and processes application performance monitoring data
- **Enterprise Search**: Search solutions for websites, applications, and workplaces
- **Beats**: Lightweight data shippers
- **Logstash**: Server-side data processing pipeline
- **Elastic Maps Server**: Self-managed version of the Elastic Maps Service

## Installing the ECK Operator

Installing the ECK operator is the first step to deploying Elastic Stack components on Kubernetes.

### Prerequisites

- Kubernetes cluster (version 1.19+)
- kubectl command-line tool
- Administrator access to the cluster

### Standard Installation

Install ECK using the standard installation manifests:

```bash
# Install custom resource definitions
kubectl create -f https://download.elastic.co/downloads/eck/2.8.0/crds.yaml

# Install the operator
kubectl apply -f https://download.elastic.co/downloads/eck/2.8.0/operator.yaml
```

This installs the operator in the `elastic-system` namespace and creates all necessary CRDs, RBAC rules, and the operator deployment.

### Helm Chart Installation

For more customization options, use the Helm chart:

```bash
# Add the Elastic Helm repository
helm repo add elastic https://helm.elastic.co

# Update your local Helm chart repository cache
helm repo update

# Install ECK using Helm
helm install elastic-operator elastic/eck-operator -n elastic-system --create-namespace
```

### Verifying the Installation

Check that the operator is running correctly:

```bash
kubectl get pods -n elastic-system

# Example output:
# NAME                             READY   STATUS    RESTARTS   AGE
# elastic-operator-0               1/1     Running   0          1m
```

View the operator logs:

```bash
kubectl logs -f -n elastic-system elastic-operator-0
```

### Operator Configuration Options

For advanced installations, you can customize the operator deployment:

```yaml
# operator-config.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elastic-operator
  namespace: elastic-system
spec:
  template:
    spec:
      containers:
      - name: manager
        args:
        - "manager"
        - "--enable-webhook"
        - "--manage-webhook-certs=true"
        - "--enforce-rbac-on-refs"
        - "--operator-roles=global,namespaced"
        - "--expose-metrics=true"
        env:
        - name: CA_CERT_VALIDITY
          value: "8760h"
        - name: CA_CERT_ROTATE_BEFORE
          value: "24h"
        - name: CONTAINER_REGISTRY
          value: "docker.elastic.co"
        - name: WEBHOOK_NAME
          value: "elastic-webhook.k8s.elastic.co"
```

## Deploying Elasticsearch

The ECK operator makes it easy to deploy and manage Elasticsearch clusters on Kubernetes.

### Basic Elasticsearch Deployment

A simple Elasticsearch deployment with a single node:

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: quickstart
  namespace: elastic
spec:
  version: 8.8.0
  nodeSets:
  - name: default
    count: 1
    config:
      node.store.allow_mmap: false
```

### Production Elasticsearch Deployment

A more comprehensive Elasticsearch deployment for production:

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch-production
  namespace: elastic
spec:
  version: 8.8.0
  secureSettings:
  - secretName: elasticsearch-secure-settings
  http:
    service:
      spec:
        type: ClusterIP
    tls:
      selfSignedCertificate:
        subjectAltNames:
        - dns: elasticsearch-production-es-http.elastic.svc.cluster.local
        - dns: elasticsearch-production-es-http.elastic.svc
  nodeSets:
  - name: master
    count: 3
    config:
      node.roles: ["master"]
      node.store.allow_mmap: false
    podTemplate:
      spec:
        affinity:
          podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  elasticsearch.k8s.elastic.co/cluster-name: elasticsearch-production
                  elasticsearch.k8s.elastic.co/node-master: "true"
              topologyKey: kubernetes.io/hostname
        initContainers:
        - name: sysctl
          securityContext:
            privileged: true
          command: ['sh', '-c', 'sysctl -w vm.max_map_count=262144']
        containers:
        - name: elasticsearch
          resources:
            requests:
              memory: 4Gi
              cpu: 1
            limits:
              memory: 4Gi
              cpu: 2
  - name: hot
    count: 3
    config:
      node.roles: ["data_hot", "data_content", "ingest"]
      node.attr.data: hot
      node.store.allow_mmap: false
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 200Gi
        storageClassName: premium-rwo
    podTemplate:
      spec:
        affinity:
          podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  elasticsearch.k8s.elastic.co/cluster-name: elasticsearch-production
                  elasticsearch.k8s.elastic.co/node-data: "true"
              topologyKey: kubernetes.io/hostname
        initContainers:
        - name: sysctl
          securityContext:
            privileged: true
          command: ['sh', '-c', 'sysctl -w vm.max_map_count=262144']
        containers:
        - name: elasticsearch
          resources:
            requests:
              memory: 16Gi
              cpu: 4
            limits:
              memory: 16Gi
  - name: warm
    count: 2
    config:
      node.roles: ["data_warm"]
      node.attr.data: warm
      node.store.allow_mmap: false
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 1000Gi
        storageClassName: standard-rwo
    podTemplate:
      spec:
        affinity:
          podAntiAffinity:
            preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    elasticsearch.k8s.elastic.co/cluster-name: elasticsearch-production
                    elasticsearch.k8s.elastic.co/node-data: "true"
                topologyKey: kubernetes.io/hostname
        containers:
        - name: elasticsearch
          resources:
            requests:
              memory: 16Gi
              cpu: 2
            limits:
              memory: 16Gi
```

### Elasticsearch Configuration Options

ECK provides many configuration options for Elasticsearch:

#### Node Roles

Define specialized node roles:

```yaml
nodeSets:
- name: master
  count: 3
  config:
    node.roles: ["master"]
- name: data
  count: 3
  config:
    node.roles: ["data", "ingest"]
- name: ml
  count: 1
  config:
    node.roles: ["ml", "remote_cluster_client"]
```

#### Persistent Storage

Configure persistent storage for data:

```yaml
volumeClaimTemplates:
- metadata:
    name: elasticsearch-data
  spec:
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 100Gi
    storageClassName: standard
```

#### Custom JVM Options

Set custom JVM options:

```yaml
podTemplate:
  spec:
    containers:
    - name: elasticsearch
      env:
      - name: ES_JAVA_OPTS
        value: "-Xms2g -Xmx2g"
```

#### Additional Configurations

Add custom configurations to elasticsearch.yml:

```yaml
config:
  node.store.allow_mmap: false
  cluster.routing.allocation.disk.threshold_enabled: false
  xpack.ml.enabled: true
  xpack.security.authc.api_key.enabled: true
```

### Accessing Elasticsearch

The ECK operator creates Kubernetes services to access Elasticsearch:

1. **ClusterIP Service**: `<cluster-name>-es-http` for internal access
2. **Transport Service**: `<cluster-name>-es-transport` for internal node-to-node communication

Get the auto-generated elastic user password:

```bash
kubectl get secret quickstart-es-elastic-user -n elastic -o go-template='{{.data.elastic | base64decode}}'
```

Port forward to access Elasticsearch locally:

```bash
kubectl port-forward service/quickstart-es-http 9200:9200 -n elastic
```

Then access Elasticsearch:

```bash
curl -u "elastic:$PASSWORD" -k "https://localhost:9200"
```

## Deploying Kibana

ECK simplifies the deployment and integration of Kibana with Elasticsearch.

### Basic Kibana Deployment

A simple Kibana deployment connected to Elasticsearch:

```yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: quickstart
  namespace: elastic
spec:
  version: 8.8.0
  count: 1
  elasticsearchRef:
    name: quickstart
```

### Production Kibana Deployment

A more comprehensive Kibana deployment for production:

```yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana-production
  namespace: elastic
spec:
  version: 8.8.0
  count: 3
  elasticsearchRef:
    name: elasticsearch-production
  http:
    service:
      spec:
        type: ClusterIP
    tls:
      selfSignedCertificate:
        subjectAltNames:
        - dns: kibana-production-kb-http.elastic.svc.cluster.local
        - dns: kibana-production-kb-http.elastic.svc
  podTemplate:
    spec:
      containers:
      - name: kibana
        resources:
          requests:
            memory: 1Gi
            cpu: 0.5
          limits:
            memory: 2Gi
            cpu: 1
        env:
        - name: SERVER_BASEPATH
          value: "/kibana"
        - name: SERVER_REWRITEBASEPATH
          value: "true"
        - name: XPACK_ENCRYPTEDSAVEDOBJECTS_ENCRYPTIONKEY
          valueFrom:
            secretKeyRef:
              name: kibana-encryption-key
              key: encryptionkey
```

### Kibana Configuration Options

ECK provides various configuration options for Kibana:

#### Custom Configuration

Add custom configurations to kibana.yml:

```yaml
config:
  kibana.autocompleteTimeout: 2000
  kibana.autocompleteTerminateAfter: 100000
  xpack.fleet.agents.elasticsearch.hosts: ["https://elasticsearch-fleet:9200"]
  xpack.fleet.agents.fleet_server.hosts: ["https://fleet-server:8220"]
```

#### Environment Variables

Set environment variables for configuration:

```yaml
podTemplate:
  spec:
    containers:
    - name: kibana
      env:
      - name: NODE_OPTIONS
        value: "--max-old-space-size=2048"
      - name: SERVER_BASEPATH
        value: "/kibana"
```

#### Custom Feature Enablement

Enable or disable specific Kibana features:

```yaml
config:
  xpack.security.enabled: true
  xpack.reporting.enabled: true
  xpack.security.encryptionKey: "${SECURITY_ENCRYPTION_KEY}"
  xpack.reporting.encryptionKey: "${REPORTING_ENCRYPTION_KEY}"
  xpack.encryptedSavedObjects.encryptionKey: "${SAVEDOBJECTS_ENCRYPTION_KEY}"
```

### Accessing Kibana

ECK creates Kubernetes services to access Kibana:

1. **ClusterIP Service**: `<kibana-name>-kb-http` for internal access

Port forward to access Kibana locally:

```bash
kubectl port-forward service/quickstart-kb-http 5601:5601 -n elastic
```

Then access Kibana at `https://localhost:5601` with:
- Username: elastic
- Password: (get from the Elasticsearch secret as shown earlier)

## Deploying Logstash

ECK provides support for deploying Logstash on Kubernetes.

### Basic Logstash Deployment

A simple Logstash deployment connected to Elasticsearch:

```yaml
apiVersion: logstash.k8s.elastic.co/v1alpha1
kind: Logstash
metadata:
  name: quickstart
  namespace: elastic
spec:
  count: 1
  version: 8.8.0
  elasticsearchRefs:
  - name: quickstart
    namespace: elastic
  pipelines:
  - pipeline.id: main
    pipeline.contents: |
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
          hosts => [ "${ES_HOST}" ]
          user => "${ES_USER}"
          password => "${ES_PASSWORD}"
          ssl => true
          ssl_certificate_verification => false
          index => "%{[@metadata][target_index]}"
        }
      }
```

### Production Logstash Deployment

A more comprehensive Logstash deployment for production:

```yaml
apiVersion: logstash.k8s.elastic.co/v1alpha1
kind: Logstash
metadata:
  name: logstash-production
  namespace: elastic
spec:
  count: 3
  version: 8.8.0
  elasticsearchRefs:
  - name: elasticsearch-production
    namespace: elastic
  services:
  - name: beats
    service:
      spec:
        type: ClusterIP
        ports:
        - name: beats
          port: 5044
          targetPort: 5044
          protocol: TCP
  - name: metrics
    service:
      spec:
        type: ClusterIP
        ports:
        - name: metrics
          port: 9600
          targetPort: 9600
          protocol: TCP
  podTemplate:
    spec:
      containers:
      - name: logstash
        resources:
          requests:
            memory: 2Gi
            cpu: 1
          limits:
            memory: 4Gi
            cpu: 2
  config:
    log.level: info
    queue.type: persisted
    queue.max_bytes: 1gb
    config.reload.automatic: true
    config.reload.interval: 30s
  pipelines:
  - pipeline.id: main
    pipeline.contents: |
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
          hosts => [ "${ES_HOST}" ]
          user => "${ES_USER}"
          password => "${ES_PASSWORD}"
          ssl => true
          ssl_certificate_verification => false
          index => "%{[@metadata][target_index]}"
        }
      }
  volumeClaimTemplates:
  - metadata:
      name: logstash-data
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 10Gi
      storageClassName: standard-rwo
```

### Logstash Configuration Options

ECK provides various configuration options for Logstash:

#### Pipeline Configuration

Configure multiple pipelines:

```yaml
pipelines:
- pipeline.id: application-logs
  pipeline.contents: |
    input {
      beats {
        port => 5044
        tags => ["application"]
      }
    }
    
    filter {
      # Application-specific filters
    }
    
    output {
      elasticsearch {
        hosts => [ "${ES_HOST}" ]
        user => "${ES_USER}"
        password => "${ES_PASSWORD}"
        index => "logs-app-%{+YYYY.MM.dd}"
      }
    }
- pipeline.id: system-logs
  pipeline.contents: |
    input {
      beats {
        port => 5045
        tags => ["system"]
      }
    }
    
    filter {
      # System-specific filters
    }
    
    output {
      elasticsearch {
        hosts => [ "${ES_HOST}" ]
        user => "${ES_USER}"
        password => "${ES_PASSWORD}"
        index => "logs-system-%{+YYYY.MM.dd}"
      }
    }
```

#### Persistent Storage

Configure persistent storage for Logstash:

```yaml
volumeClaimTemplates:
- metadata:
    name: logstash-data
  spec:
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 10Gi
    storageClassName: standard
```

#### JVM Settings

Customize JVM settings:

```yaml
podTemplate:
  spec:
    containers:
    - name: logstash
      env:
      - name: LS_JAVA_OPTS
        value: "-Xms1g -Xmx1g -XX:+UseG1GC"
```

## Deploying APM Server

ECK simplifies the deployment of APM Server to collect application performance metrics.

### Basic APM Server Deployment

A simple APM Server deployment connected to Elasticsearch:

```yaml
apiVersion: apm.k8s.elastic.co/v1
kind: ApmServer
metadata:
  name: quickstart
  namespace: elastic
spec:
  version: 8.8.0
  count: 1
  elasticsearchRef:
    name: quickstart
```

### Production APM Server Deployment

A more comprehensive APM Server deployment for production:

```yaml
apiVersion: apm.k8s.elastic.co/v1
kind: ApmServer
metadata:
  name: apm-production
  namespace: elastic
spec:
  version: 8.8.0
  count: 3
  elasticsearchRef:
    name: elasticsearch-production
  kibanaRef:
    name: kibana-production
  http:
    service:
      spec:
        type: ClusterIP
    tls:
      selfSignedCertificate:
        subjectAltNames:
        - dns: apm-production-apm-http.elastic.svc.cluster.local
        - dns: apm-production-apm-http.elastic.svc
  config:
    apm-server:
      rum:
        enabled: true
      kibana:
        enabled: true
  podTemplate:
    spec:
      containers:
      - name: apm-server
        resources:
          requests:
            memory: 512Mi
            cpu: 0.5
          limits:
            memory: 1Gi
            cpu: 1
```

### APM Server Configuration Options

ECK provides various configuration options for APM Server:

#### Custom Configuration

Add custom configurations to apm-server.yml:

```yaml
config:
  apm-server:
    capture_body: errors
    max_event_size: 1000000
    rum:
      enabled: true
      event_rate:
        limit: 300
      source_mapping:
        enabled: true
    secret_token: "${SECRET_TOKEN}"
```

#### Environment Variables

Set environment variables for configuration:

```yaml
podTemplate:
  spec:
    containers:
    - name: apm-server
      env:
      - name: SECRET_TOKEN
        valueFrom:
          secretKeyRef:
            name: apm-server-secret
            key: secret-token
```

### Accessing APM Server

ECK creates Kubernetes services to access APM Server:

1. **ClusterIP Service**: `<apm-name>-apm-http` for internal access

Get the APM Server URL and secret token to configure your applications:

```bash
# Get APM Server URL
kubectl get service quickstart-apm-http -n elastic

# Get secret token (if configured)
kubectl get secret apm-server-secret -n elastic -o go-template='{{.data.secret-token | base64decode}}'
```

## Deploying Beats

ECK enables the deployment of various Beats data shippers.

### Deploying Filebeat

A Filebeat deployment to collect logs:

```yaml
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: filebeat
  namespace: elastic
spec:
  type: filebeat
  version: 8.8.0
  elasticsearchRef:
    name: elasticsearch-production
  kibanaRef:
    name: kibana-production
  config:
    filebeat.inputs:
    - type: container
      paths:
      - /var/log/containers/*.log
    processors:
    - add_kubernetes_metadata:
        host: ${NODE_NAME}
        matchers:
        - logs_path:
            logs_path: "/var/log/containers/"
  daemonSet:
    podTemplate:
      spec:
        serviceAccountName: filebeat
        automountServiceAccountToken: true
        terminationGracePeriodSeconds: 30
        dnsPolicy: ClusterFirstWithHostNet
        hostNetwork: true
        securityContext:
          runAsUser: 0
        containers:
        - name: filebeat
          resources:
            limits:
              memory: 200Mi
            requests:
              cpu: 100m
              memory: 100Mi
          volumeMounts:
          - name: varlogcontainers
            mountPath: /var/log/containers
          - name: varlogpods
            mountPath: /var/log/pods
          - name: varlibdockercontainers
            mountPath: /var/lib/docker/containers
        volumes:
        - name: varlogcontainers
          hostPath:
            path: /var/log/containers
        - name: varlogpods
          hostPath:
            path: /var/log/pods
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
```

### Deploying Metricbeat

A Metricbeat deployment to collect metrics:

```yaml
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: metricbeat
  namespace: elastic
spec:
  type: metricbeat
  version: 8.8.0
  elasticsearchRef:
    name: elasticsearch-production
  kibanaRef:
    name: kibana-production
  config:
    metricbeat.modules:
    - module: kubernetes
      period: 10s
      metricsets:
        - container
        - node
        - pod
        - system
        - volume
      hosts: ["https://${KUBERNETES_NODE_NAME}:10250"]
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      ssl.verification_mode: none
    - module: kubernetes
      period: 10s
      metricsets:
        - apiserver
        - event
      hosts: ["https://kubernetes.default.svc:443"]
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      ssl.verification_mode: none
    processors:
    - add_cloud_metadata: {}
    - add_host_metadata: {}
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
          - name: KUBERNETES_NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
          resources:
            limits:
              memory: 200Mi
            requests:
              cpu: 100m
              memory: 100Mi
```

### Deploying Heartbeat

A Heartbeat deployment to monitor uptime:

```yaml
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: heartbeat
  namespace: elastic
spec:
  type: heartbeat
  version: 8.8.0
  elasticsearchRef:
    name: elasticsearch-production
  kibanaRef:
    name: kibana-production
  config:
    heartbeat.monitors:
    - type: http
      id: service-status
      name: Service Status
      urls: 
      - http://elasticsearch-production-es-http.elastic.svc:9200
      - http://kibana-production-kb-http.elastic.svc:5601/api/status
      schedule: '@every 10s'
      timeout: 5s
    - type: icmp
      id: ping-google
      name: Ping Google
      hosts: ["www.google.com"]
      schedule: '@every 10s'
      timeout: 3s
  deployment:
    podTemplate:
      spec:
        securityContext:
          runAsUser: 0
        containers:
        - name: heartbeat
          resources:
            limits:
              memory: 200Mi
            requests:
              cpu: 100m
              memory: 100Mi
```

### Deploying Packetbeat

A Packetbeat deployment to analyze network traffic:

```yaml
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: packetbeat
  namespace: elastic
spec:
  type: packetbeat
  version: 8.8.0
  elasticsearchRef:
    name: elasticsearch-production
  kibanaRef:
    name: kibana-production
  config:
    packetbeat.interfaces.device: any
    packetbeat.protocols:
    - type: dns
      ports: [53]
    - type: http
      ports: [80, 8080, 8000, 5000, 8002]
    - type: redis
      ports: [6379]
    processors:
    - add_kubernetes_metadata:
        host: ${NODE_NAME}
        indexers:
        - ip_port:
            source_ip: true
    - add_host_metadata: {}
  daemonSet:
    podTemplate:
      spec:
        serviceAccountName: packetbeat
        automountServiceAccountToken: true
        hostNetwork: true
        dnsPolicy: ClusterFirstWithHostNet
        securityContext:
          runAsUser: 0
        containers:
        - name: packetbeat
          securityContext:
            capabilities:
              add:
              - NET_ADMIN
          resources:
            limits:
              memory: 200Mi
            requests:
              cpu: 100m
              memory: 100Mi
```

## Deploying Enterprise Search

ECK enables the deployment of Enterprise Search for website and workplace search capabilities.

### Basic Enterprise Search Deployment

A simple Enterprise Search deployment connected to Elasticsearch:

```yaml
apiVersion: enterprisesearch.k8s.elastic.co/v1
kind: EnterpriseSearch
metadata:
  name: quickstart
  namespace: elastic
spec:
  version: 8.8.0
  count: 1
  elasticsearchRef:
    name: quickstart
```

### Production Enterprise Search Deployment

A more comprehensive Enterprise Search deployment for production:

```yaml
apiVersion: enterprisesearch.k8s.elastic.co/v1
kind: EnterpriseSearch
metadata:
  name: enterprise-search-production
  namespace: elastic
spec:
  version: 8.8.0
  count: 3
  elasticsearchRef:
    name: elasticsearch-production
  kibanaRef:
    name: kibana-production
  config:
    ent_search.listen_host: 0.0.0.0
    ent_search.auth.source: standard
    worker.threads: 14
    kibana.host: https://kibana-production-kb-http.elastic.svc:5601
  podTemplate:
    spec:
      containers:
      - name: enterprise-search
        env:
        - name: ENT_SEARCH_DEFAULT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: ent-search-password
              key: password
        resources:
          requests:
            memory: 4Gi
            cpu: 1
          limits:
            memory: 4Gi
  http:
    service:
      spec:
        type: ClusterIP
    tls:
      selfSignedCertificate:
        subjectAltNames:
        - dns: enterprise-search-production-ent-http.elastic.svc
```

### Enterprise Search Configuration Options

ECK provides various configuration options for Enterprise Search:

#### Custom Configuration

Add custom configurations:

```yaml
config:
  ent_search.listen_host: 0.0.0.0
  ent_search.external_url: https://enterprise-search.example.com
  ent_search.ssl.enabled: true
  ent_search.ssl.verify: false
  ent_search.auth.source: standard
  secret_management.encryption_keys: [${ENCRYPTION_KEY}]
  kibana.host: https://kibana-production-kb-http.elastic.svc:5601
  kibana.ssl.verify: false
```

#### Environment Variables

Set environment variables for configuration:

```yaml
podTemplate:
  spec:
    containers:
    - name: enterprise-search
      env:
      - name: ENT_SEARCH_DEFAULT_PASSWORD
        valueFrom:
          secretKeyRef:
            name: ent-search-password
            key: password
      - name: ENCRYPTION_KEY
        valueFrom:
          secretKeyRef:
            name: ent-search-encryption
            key: key
```

### Accessing Enterprise Search

ECK creates Kubernetes services to access Enterprise Search:

1. **ClusterIP Service**: `<ent-search-name>-ent-http` for internal access

Port forward to access Enterprise Search locally:

```bash
kubectl port-forward service/quickstart-ent-http 3002:3002 -n elastic
```

Then access Enterprise Search at `http://localhost:3002` with:
- Username: enterprise_search
- Password: (get from the configured secret)

## Security Configuration

ECK provides comprehensive security features for Elastic Stack deployments.

### TLS Configuration

Configure TLS for all components:

```yaml
spec:
  http:
    tls:
      selfSignedCertificate:
        subjectAltNames:
        - dns: elasticsearch-production-es-http.elastic.svc.cluster.local
        - dns: elasticsearch-production-es-http.elastic.svc
        - ip: 10.0.0.1
      certificate:
        secretName: elasticsearch-tls-cert
```

For using your own certificates:

```yaml
spec:
  http:
    tls:
      certificate:
        secretName: custom-tls-cert
```

### User Management

ECK automatically creates an elastic user. For additional users:

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch-production
  namespace: elastic
spec:
  # ...
  secureSettings:
  - secretName: elasticsearch-secure-settings
```

Create the secure settings secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: elasticsearch-secure-settings
  namespace: elastic
type: Opaque
data:
  xpack.security.audit.enabled: dHJ1ZQ==  # true in base64
  xpack.security.authc.realms.ldap.ldap1.url: bGRhcDovL2xkYXAuZXhhbXBsZS5jb206Mzg5  # ldap://ldap.example.com:389 in base64
```

### Network Security

Configure network security policies:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: elasticsearch-network-policy
  namespace: elastic
spec:
  podSelector:
    matchLabels:
      elasticsearch.k8s.elastic.co/cluster-name: elasticsearch-production
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: kibana
    ports:
    - protocol: TCP
      port: 9200
  - from:
    - podSelector:
        matchLabels:
          app: logstash
    ports:
    - protocol: TCP
      port: 9200
  egress:
  - to:
    - podSelector:
        matchLabels:
          elasticsearch.k8s.elastic.co/cluster-name: elasticsearch-production
    ports:
    - protocol: TCP
      port: 9300
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

## High Availability Setup

ECK makes it easy to configure high availability for Elastic Stack components.

### Elasticsearch HA Configuration

Configure high availability for Elasticsearch:

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch-ha
  namespace: elastic
spec:
  version: 8.8.0
  nodeSets:
  - name: master
    count: 3
    config:
      node.roles: ["master"]
      node.store.allow_mmap: false
    podTemplate:
      spec:
        affinity:
          podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  elasticsearch.k8s.elastic.co/cluster-name: elasticsearch-ha
                  elasticsearch.k8s.elastic.co/node-master: "true"
              topologyKey: kubernetes.io/hostname
  - name: data
    count: 3
    config:
      node.roles: ["data", "ingest"]
      node.store.allow_mmap: false
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 100Gi
    podTemplate:
      spec:
        affinity:
          podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  elasticsearch.k8s.elastic.co/cluster-name: elasticsearch-ha
                  elasticsearch.k8s.elastic.co/node-data: "true"
              topologyKey: kubernetes.io/hostname
```

### Kibana HA Configuration

Configure high availability for Kibana:

```yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana-ha
  namespace: elastic
spec:
  version: 8.8.0
  count: 3
  elasticsearchRef:
    name: elasticsearch-ha
  podTemplate:
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  kibana.k8s.elastic.co/name: kibana-ha
              topologyKey: kubernetes.io/hostname
```

### APM Server HA Configuration

Configure high availability for APM Server:

```yaml
apiVersion: apm.k8s.elastic.co/v1
kind: ApmServer
metadata:
  name: apm-ha
  namespace: elastic
spec:
  version: 8.8.0
  count: 3
  elasticsearchRef:
    name: elasticsearch-ha
  podTemplate:
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  apm.k8s.elastic.co/name: apm-ha
              topologyKey: kubernetes.io/hostname
```

### Logstash HA Configuration

Configure high availability for Logstash:

```yaml
apiVersion: logstash.k8s.elastic.co/v1alpha1
kind: Logstash
metadata:
  name: logstash-ha
  namespace: elastic
spec:
  count: 3
  version: 8.8.0
  elasticsearchRefs:
  - name: elasticsearch-ha
  podTemplate:
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  logstash.k8s.elastic.co/name: logstash-ha
              topologyKey: kubernetes.io/hostname
```

## Resource Management

Managing resources appropriately for ECK deployments is crucial for stable operation.

### Elasticsearch Resource Management

Configure resources for Elasticsearch nodes:

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch-resources
  namespace: elastic
spec:
  version: 8.8.0
  nodeSets:
  - name: master
    count: 3
    config:
      node.roles: ["master"]
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          resources:
            requests:
              memory: 2Gi
              cpu: 0.5
            limits:
              memory: 2Gi
              cpu: 1
  - name: data
    count: 3
    config:
      node.roles: ["data", "ingest"]
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          resources:
            requests:
              memory: 8Gi
              cpu: 2
            limits:
              memory: 8Gi
```

### JVM Heap Size Configuration

Configure JVM heap size for Elasticsearch:

```yaml
podTemplate:
  spec:
    containers:
    - name: elasticsearch
      env:
      - name: ES_JAVA_OPTS
        value: "-Xms4g -Xmx4g"
```

### Recommended Resource Configurations

| Component | Size | CPU Requests | Memory Requests | Storage |
|-----------|------|--------------|-----------------|---------|
| Elasticsearch Master | Small | 0.5 | 2Gi | 20Gi |
| Elasticsearch Master | Medium | 1 | 4Gi | 20Gi |
| Elasticsearch Master | Large | 2 | 8Gi | 20Gi |
| Elasticsearch Data | Small | 1 | 4Gi | 100Gi |
| Elasticsearch Data | Medium | 2 | 8Gi | 500Gi |
| Elasticsearch Data | Large | 4+ | 16Gi+ | 1000Gi+ |
| Kibana | Small | 0.5 | 1Gi | n/a |
| Kibana | Medium | 1 | 2Gi | n/a |
| Kibana | Large | 2+ | 4Gi+ | n/a |
| APM Server | Small | 0.5 | 512Mi | n/a |
| APM Server | Medium | 1 | 1Gi | n/a |
| APM Server | Large | 2+ | 2Gi+ | n/a |
| Logstash | Small | 0.5 | 1Gi | 10Gi |
| Logstash | Medium | 1 | 2Gi | 20Gi |
| Logstash | Large | 2+ | 4Gi+ | 50Gi+ |

## Upgrades and Version Management

ECK simplifies version upgrades for Elastic Stack components.

### Upgrading Elasticsearch

To upgrade Elasticsearch, simply update the version field:

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch-production
  namespace: elastic
spec:
  version: 8.9.0  # Update from 8.8.0 to 8.9.0
  # ...
```

The ECK operator handles the upgrade process automatically, following Elasticsearch best practices.

### Upgrade Strategy Customization

Customize the upgrade strategy:

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch-production
  namespace: elastic
spec:
  version: 8.9.0
  updateStrategy:
    changeBudget:
      maxSurge: 3
      maxUnavailable: 1
  # ...
```

### Upgrading Other Components

Similarly, upgrade other components by updating their version field:

```yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana-production
  namespace: elastic
spec:
  version: 8.9.0  # Update from 8.8.0 to 8.9.0
  # ...
```

### Version Compatibility Considerations

When upgrading, consider the following:

1. **Order of upgrades**: 
   - Elasticsearch first
   - Kibana, APM Server, and other components after

2. **Compatible versions**:
   - Kibana should be the same version as Elasticsearch
   - APM Server can be the same or one version behind Elasticsearch
   - Beats can be the same or one version behind Elasticsearch

3. **Major version upgrades**:
   - May require additional steps
   - Review the Elastic documentation for specific requirements
   - Consider using a blue/green deployment approach

## Monitoring ECK Deployments

Monitoring ECK deployments ensures optimal performance and reliability.

### Built-in Monitoring

Enable Elastic Stack monitoring:

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch-production
  namespace: elastic
spec:
  version: 8.8.0
  monitoring:
    metrics:
      elasticsearchRefs:
      - name: elasticsearch-monitoring  # Separate monitoring cluster
    logs:
      elasticsearchRefs:
      - name: elasticsearch-monitoring
  # ...
```

### Prometheus Integration

Enable Prometheus metrics for ECK components:

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch-production
  namespace: elastic
spec:
  version: 8.8.0
  http:
    service:
      metadata:
        annotations:
          prometheus.io/scrape: "true"
          prometheus.io/path: "/_prometheus/metrics"
          prometheus.io/port: "9200"
  # ...
```

### Metricbeat for Comprehensive Monitoring

Deploy Metricbeat to monitor all components:

```yaml
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: metricbeat-elastic-stack
  namespace: elastic
spec:
  type: metricbeat
  version: 8.8.0
  elasticsearchRef:
    name: elasticsearch-monitoring
  kibanaRef:
    name: kibana-monitoring
  config:
    metricbeat.modules:
    - module: elasticsearch
      metricsets:
      - node
      - node_stats
      - cluster_stats
      period: 10s
      hosts: ["https://elasticsearch-production-es-http.elastic.svc:9200"]
      username: "${ELASTICSEARCH_USERNAME}"
      password: "${ELASTICSEARCH_PASSWORD}"
      ssl.verification_mode: "certificate"
      xpack.enabled: true
    - module: kibana
      metricsets:
      - stats
      period: 10s
      hosts: ["https://kibana-production-kb-http.elastic.svc:5601"]
      username: "${ELASTICSEARCH_USERNAME}"
      password: "${ELASTICSEARCH_PASSWORD}"
      ssl.verification_mode: "certificate"
      xpack.enabled: true
    - module: apm
      metricsets:
      - server
      period: 10s
      hosts: ["https://apm-production-apm-http.elastic.svc:8200"]
      ssl.verification_mode: "certificate"
    - module: logstash
      metricsets:
      - node
      - node_stats
      period: 10s
      hosts: ["https://logstash-production-metrics.elastic.svc:9600"]
      ssl.verification_mode: "certificate"
      xpack.enabled: true
  deployment:
    podTemplate:
      spec:
        serviceAccountName: metricbeat
        automountServiceAccountToken: true
        containers:
        - name: metricbeat
          env:
          - name: ELASTICSEARCH_USERNAME
            valueFrom:
              secretKeyRef:
                name: elasticsearch-monitoring-es-elastic-user
                key: elastic
          - name: ELASTICSEARCH_PASSWORD
            valueFrom:
              secretKeyRef:
                name: elasticsearch-monitoring-es-elastic-user
                key: elastic
```

## Backing Up and Restoring

Implementing backup and restore strategies for Elasticsearch in ECK.

### Snapshot Repositories

Configure a snapshot repository for backups:

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch-production
  namespace: elastic
spec:
  version: 8.8.0
  # ...
  secureSettings:
  - secretName: elasticsearch-s3-credentials
```

Create the S3 credentials secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: elasticsearch-s3-credentials
  namespace: elastic
type: Opaque
stringData:
  s3.client.default.access_key: "YOUR_ACCESS_KEY"
  s3.client.default.secret_key: "YOUR_SECRET_KEY"
```

### Snapshot Repository Setup

Create a Kubernetes job to set up the repository:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: setup-snapshot-repo
  namespace: elastic
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: setup-snapshot
        image: curlimages/curl:latest
        env:
        - name: ES_URL
          value: https://elasticsearch-production-es-http.elastic.svc:9200
        - name: ES_USERNAME
          valueFrom:
            secretKeyRef:
              name: elasticsearch-production-es-elastic-user
              key: elastic
        - name: ES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: elasticsearch-production-es-elastic-user
              key: elastic
        command:
        - sh
        - -c
        - |
          curl -k -u "${ES_USERNAME}:${ES_PASSWORD}" -X PUT "${ES_URL}/_snapshot/s3_repository" -H 'Content-Type: application/json' -d '
          {
            "type": "s3",
            "settings": {
              "bucket": "elasticsearch-snapshots",
              "region": "us-east-1",
              "compress": true
            }
          }'
```

### Snapshot Creation

Create a Kubernetes CronJob for regular snapshots:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: elasticsearch-snapshot
  namespace: elastic
spec:
  schedule: "0 1 * * *"  # Every day at 1 AM
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: snapshot
            image: curlimages/curl:latest
            env:
            - name: ES_URL
              value: https://elasticsearch-production-es-http.elastic.svc:9200
            - name: ES_USERNAME
              valueFrom:
                secretKeyRef:
                  name: elasticsearch-production-es-elastic-user
                  key: elastic
            - name: ES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: elasticsearch-production-es-elastic-user
                  key: elastic
            command:
            - sh
            - -c
            - |
              SNAPSHOT_NAME="snapshot-$(date +%Y%m%d-%H%M%S)"
              curl -k -u "${ES_USERNAME}:${ES_PASSWORD}" -X PUT "${ES_URL}/_snapshot/s3_repository/${SNAPSHOT_NAME}" -H 'Content-Type: application/json' -d '
              {
                "indices": "*",
                "ignore_unavailable": true,
                "include_global_state": true
              }'
```

### Snapshot Restoration

Create a Kubernetes job for snapshot restoration:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: elasticsearch-restore
  namespace: elastic
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: restore
        image: curlimages/curl:latest
        env:
        - name: ES_URL
          value: https://elasticsearch-production-es-http.elastic.svc:9200
        - name: ES_USERNAME
          valueFrom:
            secretKeyRef:
              name: elasticsearch-production-es-elastic-user
              key: elastic
        - name: ES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: elasticsearch-production-es-elastic-user
              key: elastic
        - name: SNAPSHOT_NAME
          value: "snapshot-20230601-010000"  # Specify the snapshot to restore
        command:
        - sh
        - -c
        - |
          curl -k -u "${ES_USERNAME}:${ES_PASSWORD}" -X POST "${ES_URL}/_snapshot/s3_repository/${SNAPSHOT_NAME}/_restore" -H 'Content-Type: application/json' -d '
          {
            "indices": "*",
            "ignore_unavailable": true,
            "include_global_state": true
          }'
```

## Advanced Configurations

Advanced configuration options for Elastic Stack components in ECK.

### Custom Elasticsearch Plugins

Install custom Elasticsearch plugins:

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch-custom
  namespace: elastic
spec:
  version: 8.8.0
  # ...
  podTemplate:
    spec:
      initContainers:
      - name: install-plugins
        command:
        - sh
        - -c
        - |
          bin/elasticsearch-plugin install --batch repository-gcs
          bin/elasticsearch-plugin install --batch analysis-kuromoji
```

### Custom Kibana Plugins

Install custom Kibana plugins:

```yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana-custom
  namespace: elastic
spec:
  version: 8.8.0
  elasticsearchRef:
    name: elasticsearch-custom
  # ...
  podTemplate:
    spec:
      initContainers:
      - name: install-plugins
        command:
        - sh
        - -c
        - |
          bin/kibana-plugin install https://github.com/example/plugin/releases/download/v1.0.0/plugin.zip
          bin/kibana-plugin install another-plugin-name
```

### Custom Certificates

Use custom certificates instead of self-signed ones:

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch-custom-tls
  namespace: elastic
spec:
  version: 8.8.0
  # ...
  http:
    tls:
      certificate:
        secretName: elasticsearch-custom-cert
  transport:
    tls:
      certificate:
        secretName: elasticsearch-transport-cert
```

Create the certificate secrets:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: elasticsearch-custom-cert
  namespace: elastic
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-certificate>
  tls.key: <base64-encoded-private-key>
  ca.crt: <base64-encoded-ca-certificate>
```

### Advanced Networking

Configure advanced networking options:

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch-network
  namespace: elastic
spec:
  version: 8.8.0
  # ...
  http:
    service:
      spec:
        type: LoadBalancer
        loadBalancerIP: 10.0.0.100
        externalTrafficPolicy: Local
  transport:
    service:
      spec:
        type: ClusterIP
        clusterIP: None
```

### Custom Sidecars

Add sidecar containers for additional functionality:

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch-sidecar
  namespace: elastic
spec:
  version: 8.8.0
  # ...
  podTemplate:
    spec:
      containers:
      - name: elasticsearch
        # Default Elasticsearch container config
      - name: custom-sidecar
        image: custom-image:latest
        command: ["/bin/sh", "-c", "while true; do echo 'Sidecar running'; sleep 60; done"]
        resources:
          requests:
            memory: 64Mi
            cpu: 100m
          limits:
            memory: 128Mi
            cpu: 200m
```

## Troubleshooting

Common issues and troubleshooting techniques for ECK deployments.

### Checking Operator Status

Verify that the ECK operator is running correctly:

```bash
kubectl get pods -n elastic-system
kubectl logs -f -n elastic-system elastic-operator-0
```

### Checking Resource Status

Check the status of Elastic Stack resources:

```bash
# Check Elasticsearch status
kubectl get elasticsearch -n elastic
kubectl describe elasticsearch quickstart -n elastic

# Check Kibana status
kubectl get kibana -n elastic
kubectl describe kibana quickstart -n elastic

# Check all ECK resources
kubectl get all -n elastic
```

### Common Issues and Solutions

1. **Pods Stuck in Pending State**
   - Issue: Insufficient resources
   - Solution: Check node resources and resource requests/limits

   ```bash
   kubectl describe pod <pod-name> -n elastic
   kubectl get nodes -o wide
   ```

2. **Elasticsearch Not Ready**
   - Issue: Volume mounting or network issues
   - Solution: Check pod events and logs

   ```bash
   kubectl describe pod <elasticsearch-pod> -n elastic
   kubectl logs <elasticsearch-pod> -n elastic
   ```

3. **TLS Certificate Issues**
   - Issue: Certificate validation failures
   - Solution: Check certificate configuration

   ```bash
   kubectl get secret <certificate-secret> -n elastic
   kubectl describe secret <certificate-secret> -n elastic
   ```

4. **Memory Issues**
   - Issue: JVM memory configuration
   - Solution: Adjust JVM and container memory settings

   ```bash
   kubectl logs <pod-name> -n elastic | grep "memory"
   ```

### Debugging Tools

Use kubectl to inspect and debug resources:

```bash
# Port forward to access components locally
kubectl port-forward service/quickstart-es-http 9200:9200 -n elastic
kubectl port-forward service/quickstart-kb-http 5601:5601 -n elastic

# Check events
kubectl get events -n elastic --sort-by='.lastTimestamp'

# Access logs
kubectl logs <pod-name> -n elastic

# Execute commands in pods
kubectl exec -it <pod-name> -n elastic -- bash

# Check resource usage
kubectl top pods -n elastic
```

## Summary

Elastic Cloud on Kubernetes (ECK) provides a Kubernetes-native way to deploy, manage, and operate the Elastic Stack. Key benefits include:

1. **Simplified Management**: Automates deployment and management of Elastic Stack components
2. **Security**: Built-in security with TLS certificates and user management
3. **High Availability**: Easy configuration of resilient clusters
4. **Resource Management**: Granular control over resource allocation
5. **Upgrades**: Simplified version management and upgrades
6. **Custom Configuration**: Extensive customization options for all components
7. **Monitoring**: Comprehensive monitoring of all deployed resources
8. **Backup and Restore**: Integrated backup and restore capabilities

By following the patterns and examples in this chapter, you can deploy and manage a production-ready Elastic Stack on Kubernetes using ECK.

## References

- [Elastic Cloud on Kubernetes Documentation](https://www.elastic.co/guide/en/cloud-on-k8s/current/index.html)
- [ECK Operator GitHub Repository](https://github.com/elastic/cloud-on-k8s)
- [Elasticsearch Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Kibana Documentation](https://www.elastic.co/guide/en/kibana/current/index.html)
- [Kubernetes Documentation](https://kubernetes.io/docs/home/)