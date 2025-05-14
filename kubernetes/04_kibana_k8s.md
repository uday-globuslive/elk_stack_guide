# Kibana on Kubernetes

This chapter covers deploying, configuring, and managing Kibana on Kubernetes environments.

## Table of Contents
- [Introduction to Kibana on Kubernetes](#introduction-to-kibana-on-kubernetes)
- [Deployment Strategies](#deployment-strategies)
- [Configuration Management](#configuration-management)
- [Security Configuration](#security-configuration)
- [Networking and Ingress](#networking-and-ingress)
- [Resource Management](#resource-management)
- [High Availability](#high-availability)
- [Scaling Kibana](#scaling-kibana)
- [Monitoring and Logging](#monitoring-and-logging)
- [Integration with Elasticsearch](#integration-with-elasticsearch)
- [Plugin Management](#plugin-management)
- [Customization and Branding](#customization-and-branding)
- [Handling Upgrades](#handling-upgrades)
- [Troubleshooting](#troubleshooting)

## Introduction to Kibana on Kubernetes

Kibana serves as the visualization and user interface component of the Elastic Stack. Deploying Kibana on Kubernetes offers benefits such as simplified deployment, high availability, and integration with existing Kubernetes infrastructure.

### Key Considerations for Kibana on Kubernetes

1. **Stateless Nature**: Kibana is primarily stateless, making it well-suited for Kubernetes deployments
2. **Elasticsearch Connection**: Kibana requires a stable connection to Elasticsearch
3. **User Interface Access**: End users need reliable access to the Kibana interface
4. **Security**: Authentication, authorization, and TLS configuration are essential
5. **Configuration Management**: Managing Kibana configuration in a Kubernetes-native way

### Kibana Architecture in Kubernetes

In a Kubernetes environment, a typical Kibana deployment includes:

1. **Kibana Deployment**: One or more Kibana pods serving the web interface
2. **ConfigMaps**: For Kibana configuration
3. **Secrets**: For sensitive configuration like Elasticsearch credentials
4. **Services**: For internal and external access
5. **Ingress**: For exposing Kibana to external users
6. **NetworkPolicies**: For controlling traffic to and from Kibana

This architecture integrates with Elasticsearch running in the same Kubernetes cluster or externally.

## Deployment Strategies

### Basic Deployment

A simple Kibana deployment using a Deployment and Service:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-config
  namespace: elk
data:
  kibana.yml: |
    server.host: "0.0.0.0"
    server.name: "kibana"
    elasticsearch.hosts: ["http://elasticsearch-master:9200"]
    monitoring.ui.container.elasticsearch.enabled: true

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: elk
  labels:
    app: kibana
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
        image: docker.elastic.co/kibana/kibana:7.17.8
        ports:
        - containerPort: 5601
          name: http
        volumeMounts:
        - name: config
          mountPath: /usr/share/kibana/config/kibana.yml
          subPath: kibana.yml
          readOnly: true
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1"
        env:
        - name: ELASTICSEARCH_URL
          value: http://elasticsearch-master:9200
        readinessProbe:
          httpGet:
            path: /api/status
            port: 5601
          initialDelaySeconds: 60
          timeoutSeconds: 10
        livenessProbe:
          httpGet:
            path: /api/status
            port: 5601
          initialDelaySeconds: 120
          timeoutSeconds: 10
          failureThreshold: 3
      volumes:
      - name: config
        configMap:
          name: kibana-config

---
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: elk
  labels:
    app: kibana
spec:
  ports:
  - port: 5601
    name: http
    targetPort: 5601
  selector:
    app: kibana
```

### Production Deployment with Security

A more comprehensive deployment for production:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-config
  namespace: elk
data:
  kibana.yml: |
    server.host: "0.0.0.0"
    server.name: "kibana"
    
    elasticsearch.hosts: ["https://elasticsearch-master:9200"]
    elasticsearch.ssl.verificationMode: certificate
    elasticsearch.ssl.certificateAuthorities: ["/usr/share/kibana/config/certs/ca.crt"]
    
    server.ssl.enabled: true
    server.ssl.certificate: "/usr/share/kibana/config/certs/tls.crt"
    server.ssl.key: "/usr/share/kibana/config/certs/tls.key"
    
    xpack.security.enabled: true
    xpack.reporting.encryptionKey: "${REPORTING_ENCRYPTION_KEY}"
    xpack.security.encryptionKey: "${SECURITY_ENCRYPTION_KEY}"
    
    elasticsearch.serviceAccountToken: "${ELASTICSEARCH_SERVICE_ACCOUNT_TOKEN}"
    
    logging.root.level: info

---
apiVersion: v1
kind: Secret
metadata:
  name: kibana-secrets
  namespace: elk
type: Opaque
data:
  REPORTING_ENCRYPTION_KEY: VGhpc0lzQVNlY3VyZUVuY3J5cHRpb25LZXk=  # base64 encoded
  SECURITY_ENCRYPTION_KEY: QW5vdGhlclNlY3VyZUVuY3J5cHRpb25LZXk=  # base64 encoded
  ELASTICSEARCH_SERVICE_ACCOUNT_TOKEN: <base64-encoded-token>  # Elasticsearch service account token

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: elk
  labels:
    app: kibana
spec:
  replicas: 2
  selector:
    matchLabels:
      app: kibana
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: kibana
    spec:
      securityContext:
        fsGroup: 1000
        runAsUser: 1000
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:7.17.8
        ports:
        - containerPort: 5601
          name: https
        volumeMounts:
        - name: config
          mountPath: /usr/share/kibana/config/kibana.yml
          subPath: kibana.yml
          readOnly: true
        - name: certificates
          mountPath: /usr/share/kibana/config/certs
          readOnly: true
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1"
        envFrom:
        - secretRef:
            name: kibana-secrets
        readinessProbe:
          httpGet:
            path: /api/status
            port: 5601
            scheme: HTTPS
          initialDelaySeconds: 60
          timeoutSeconds: 10
        livenessProbe:
          httpGet:
            path: /api/status
            port: 5601
            scheme: HTTPS
          initialDelaySeconds: 120
          timeoutSeconds: 10
          failureThreshold: 3
      volumes:
      - name: config
        configMap:
          name: kibana-config
      - name: certificates
        secret:
          secretName: elasticsearch-certificates
```

## Configuration Management

### ConfigMap-based Configuration

Managing Kibana configuration through Kubernetes ConfigMaps:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-config
  namespace: elk
data:
  kibana.yml: |
    server.host: "0.0.0.0"
    server.name: "kibana"
    elasticsearch.hosts: ["http://elasticsearch-master:9200"]
    
    # Kibana settings
    server.basePath: "/kibana"
    server.rewriteBasePath: true
    
    # Feature settings
    xpack.apm.enabled: true
    xpack.grokdebugger.enabled: true
    xpack.searchprofiler.enabled: true
    
    # Monitoring settings
    monitoring.kibana.collection.enabled: true
    monitoring.ui.container.elasticsearch.enabled: true
    
    # Default dashboards
    kibana.defaultAppId: "dashboard/722b74f0-b882-11e8-a6d9-e546fe2bba5f"
    
    # Custom settings
    kibana.autocompleteTimeout: 2000
    kibana.autocompleteTerminateAfter: 100000
```

### Overriding Configuration with Environment Variables

Using environment variables to override configuration:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: elk
spec:
  # ...
  template:
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:7.17.8
        env:
        - name: SERVER_HOST
          value: "0.0.0.0"
        - name: ELASTICSEARCH_HOSTS
          value: "http://elasticsearch-master:9200"
        - name: LOGGING_VERBOSE
          value: "true"
        - name: SERVER_BASEPATH
          value: "/kibana"
        - name: SERVER_REWRITEBASEPATH
          value: "true"
        - name: XPACK_APM_ENABLED
          value: "true"
        - name: XPACK_REPORTING_ENABLED 
          value: "true"
```

### Managing Sensitive Configuration with Secrets

Secure handling of sensitive configuration:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: kibana-secrets
  namespace: elk
type: Opaque
data:
  elasticsearch.username: ZWxhc3RpYw==  # "elastic" base64 encoded
  elasticsearch.password: Y2hhbmdlbWU=  # "changeme" base64 encoded
  encryption.key: RGZKcU5uYVZiS1ZPazZkeDFEeXM1dz09  # base64 encoded encryption key

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: elk
spec:
  # ...
  template:
    spec:
      containers:
      - name: kibana
        # ...
        env:
        - name: ELASTICSEARCH_USERNAME
          valueFrom:
            secretKeyRef:
              name: kibana-secrets
              key: elasticsearch.username
        - name: ELASTICSEARCH_PASSWORD
          valueFrom:
            secretKeyRef:
              name: kibana-secrets
              key: elasticsearch.password
        - name: XPACK_SECURITY_ENCRYPTIONKEY
          valueFrom:
            secretKeyRef:
              name: kibana-secrets
              key: encryption.key
        - name: XPACK_REPORTING_ENCRYPTIONKEY
          valueFrom:
            secretKeyRef:
              name: kibana-secrets
              key: encryption.key
```

### Multiple Configuration Techniques

Using a combination of ConfigMaps, Secrets, and environment variables:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: elk
spec:
  # ...
  template:
    spec:
      containers:
      - name: kibana
        # ...
        volumeMounts:
        - name: config-volume
          mountPath: /usr/share/kibana/config/kibana.yml
          subPath: kibana.yml
        env:
        # Override specific settings with environment variables
        - name: ELASTICSEARCH_HOSTS
          value: "http://elasticsearch-master:9200"
        # Use secrets for sensitive data
        - name: ELASTICSEARCH_USERNAME
          valueFrom:
            secretKeyRef:
              name: kibana-secrets
              key: elasticsearch.username
        - name: ELASTICSEARCH_PASSWORD
          valueFrom:
            secretKeyRef:
              name: kibana-secrets
              key: elasticsearch.password
      volumes:
      - name: config-volume
        configMap:
          name: kibana-config
```

## Security Configuration

### Enabling Authentication

Configuring Kibana to use Elasticsearch's built-in security:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-config
  namespace: elk
data:
  kibana.yml: |
    server.host: "0.0.0.0"
    elasticsearch.hosts: ["https://elasticsearch-master:9200"]
    
    # Security Settings
    xpack.security.enabled: true
    elasticsearch.ssl.verificationMode: certificate
    elasticsearch.ssl.certificateAuthorities: ["/usr/share/kibana/config/certs/ca.crt"]
```

### SAML Authentication

Configuring SAML-based SSO for Kibana:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-config
  namespace: elk
data:
  kibana.yml: |
    server.host: "0.0.0.0"
    elasticsearch.hosts: ["https://elasticsearch-master:9200"]
    
    # Basic Security Settings
    xpack.security.enabled: true
    elasticsearch.ssl.verificationMode: certificate
    elasticsearch.ssl.certificateAuthorities: ["/usr/share/kibana/config/certs/ca.crt"]
    
    # SAML Configuration
    xpack.security.authc.providers:
      saml.saml1:
        order: 0
        realm: saml1
        description: "Log in with SSO"
      basic.basic1:
        order: 1
        icon: "logoElasticsearch"
        description: "Log in with Elasticsearch"
    
    xpack.security.authc.saml:
      realm: saml1
      idp.metadata.path: "/usr/share/kibana/config/saml/idp-metadata.xml"
      idp.entityId: "https://idp.example.com/saml2/metadata"
      sp.entityId: "https://kibana.example.com"
      sp.asc: "https://kibana.example.com/api/security/v1/saml"
      signature.algorithm: "rsa-sha256"
      decryption.certificates: ["/usr/share/kibana/config/saml/sp-cert.crt"]
      decryption.privateKeys: ["/usr/share/kibana/config/saml/sp-key.pem"]
```

### OIDC Authentication

Configuring OpenID Connect for Kibana:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-config
  namespace: elk
data:
  kibana.yml: |
    server.host: "0.0.0.0"
    elasticsearch.hosts: ["https://elasticsearch-master:9200"]
    
    # Basic Security Settings
    xpack.security.enabled: true
    elasticsearch.ssl.verificationMode: certificate
    elasticsearch.ssl.certificateAuthorities: ["/usr/share/kibana/config/certs/ca.crt"]
    
    # OIDC Configuration
    xpack.security.authc.providers:
      oidc.oidc1:
        order: 0
        realm: oidc1
        description: "Log in with SSO"
      basic.basic1:
        order: 1
        icon: "logoElasticsearch"
        description: "Log in with Elasticsearch"
    
    xpack.security.authc.oidc:
      realm: oidc1
      client.id: "kibana-client"
      client.secret: "${OIDC_CLIENT_SECRET}"
      redirect_uri: "https://kibana.example.com/api/security/v1/oidc/callback"
      issuer: "https://auth.example.com/oidc"
      authorization_endpoint: "https://auth.example.com/oidc/authorize"
      token_endpoint: "https://auth.example.com/oidc/token"
      userinfo_endpoint: "https://auth.example.com/oidc/userinfo"
      jwkset_path: "/usr/share/kibana/config/oidc/jwks.json"
      claims:
        principal: "sub"
        groups: "groups"
        mail: "email"
        name: "name"
```

### TLS Configuration

Enabling HTTPS for Kibana:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-config
  namespace: elk
data:
  kibana.yml: |
    server.host: "0.0.0.0"
    elasticsearch.hosts: ["https://elasticsearch-master:9200"]
    
    # TLS Configuration
    server.ssl.enabled: true
    server.ssl.certificate: "/usr/share/kibana/config/certs/kibana.crt"
    server.ssl.key: "/usr/share/kibana/config/certs/kibana.key"
    server.ssl.certificateAuthorities: ["/usr/share/kibana/config/certs/ca.crt"]
    server.ssl.supportedProtocols: ["TLSv1.2", "TLSv1.3"]
    server.ssl.cipherSuites: [
      "TLS_AES_256_GCM_SHA384",
      "TLS_CHACHA20_POLY1305_SHA256",
      "TLS_AES_128_GCM_SHA256",
      "ECDHE-RSA-AES256-GCM-SHA384",
      "ECDHE-ECDSA-AES256-GCM-SHA384"
    ]
```

### Network Security

Configuring NetworkPolicy for Kibana:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: kibana-network-policy
  namespace: elk
spec:
  podSelector:
    matchLabels:
      app: kibana
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    - podSelector:
        matchLabels:
          app.kubernetes.io/name: ingress-nginx
    ports:
    - protocol: TCP
      port: 5601
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: elasticsearch
    ports:
    - protocol: TCP
      port: 9200
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

## Networking and Ingress

### Basic Service

Creating a basic Kubernetes Service for Kibana:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: elk
  labels:
    app: kibana
spec:
  ports:
  - port: 5601
    name: http
    targetPort: 5601
  selector:
    app: kibana
```

### NodePort Service

Exposing Kibana via NodePort:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kibana-nodeport
  namespace: elk
  labels:
    app: kibana
spec:
  type: NodePort
  ports:
  - port: 5601
    targetPort: 5601
    nodePort: 30601
    name: http
  selector:
    app: kibana
```

### LoadBalancer Service

Exposing Kibana via LoadBalancer:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kibana-lb
  namespace: elk
  labels:
    app: kibana
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer
  ports:
  - port: 443
    targetPort: 5601
    name: https
  selector:
    app: kibana
```

### Ingress Configuration

Setting up Kibana with Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kibana-ingress
  namespace: elk
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - kibana.example.com
    secretName: kibana-tls-secret
  rules:
  - host: kibana.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kibana
            port:
              name: https
```

### Ingress with Path-based Routing

Multiple apps sharing a single domain:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: elk-ingress
  namespace: elk
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - elk.example.com
    secretName: elk-tls-secret
  rules:
  - host: elk.example.com
    http:
      paths:
      - path: /kibana(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: kibana
            port:
              name: https
      - path: /apm(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: apm-server
            port:
              name: https
```

### Configuring Kibana with Base Path

When using path-based routing, configure Kibana's base path:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-config
  namespace: elk
data:
  kibana.yml: |
    server.host: "0.0.0.0"
    elasticsearch.hosts: ["https://elasticsearch-master:9200"]
    
    # Base path configuration
    server.basePath: "/kibana"
    server.rewriteBasePath: true
    
    # Other configurations...
```

## Resource Management

### Basic Resource Configuration

Setting CPU and memory requests and limits:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: elk
spec:
  # ...
  template:
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:7.17.8
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "1"
            memory: "2Gi"
```

### Resource Guidelines

| Deployment Size | CPU Requests | Memory Requests | CPU Limits | Memory Limits | Concurrent Users |
|-----------------|--------------|-----------------|------------|---------------|------------------|
| Small           | 500m         | 1Gi             | 1          | 2Gi           | ~20              |
| Medium          | 1            | 2Gi             | 2          | 4Gi           | ~50              |
| Large           | 2            | 4Gi             | 4          | 8Gi           | 100+             |

### Node Selection

Placing Kibana pods on specific nodes:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: elk
spec:
  # ...
  template:
    spec:
      nodeSelector:
        kubernetes.io/role: frontend
      containers:
      - name: kibana
        # ...
```

### Tolerations

Allowing Kibana to run on tainted nodes:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: elk
spec:
  # ...
  template:
    spec:
      tolerations:
      - key: "dedicated"
        operator: "Equal"
        value: "frontend"
        effect: "NoSchedule"
      containers:
      - name: kibana
        # ...
```

### Pod Disruption Budget

Ensuring availability during cluster operations:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: kibana-pdb
  namespace: elk
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: kibana
```

## High Availability

### Multi-Pod Deployment

Running multiple Kibana pods for high availability:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: elk
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kibana
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  # ...
```

### Affinity Rules

Using pod anti-affinity to distribute Kibana pods:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
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
                  - kibana
              topologyKey: kubernetes.io/hostname
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/role
                operator: In
                values:
                - frontend
      containers:
      - name: kibana
        # ...
```

### Health Probes

Configuring readiness and liveness probes:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: elk
spec:
  # ...
  template:
    spec:
      containers:
      - name: kibana
        # ...
        readinessProbe:
          httpGet:
            path: /api/status
            port: 5601
            scheme: HTTPS
          initialDelaySeconds: 60
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /api/status
            port: 5601
            scheme: HTTPS
          initialDelaySeconds: 120
          periodSeconds: 20
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 3
```

### Startup Probe

Adding startup probe for slow-starting Kibana instances:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: elk
spec:
  # ...
  template:
    spec:
      containers:
      - name: kibana
        # ...
        startupProbe:
          httpGet:
            path: /api/status
            port: 5601
            scheme: HTTPS
          failureThreshold: 30
          periodSeconds: 10
```

## Scaling Kibana

### Horizontal Pod Autoscaler

Automatically scaling Kibana based on CPU usage:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kibana
  namespace: elk
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
```

### Vertical Pod Autoscaler

Using Vertical Pod Autoscaler for automatic resource adjustments:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: kibana-vpa
  namespace: elk
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: kibana
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: kibana
      minAllowed:
        memory: "1Gi"
        cpu: "500m"
      maxAllowed:
        memory: "4Gi"
        cpu: "2"
```

### Manual Scaling Considerations

Guidelines for manual scaling decisions:

1. **Concurrent Users**: Scale with ~50 concurrent users per pod
2. **Response Time**: Add pods if response time exceeds acceptable thresholds
3. **Memory Usage**: Scale if memory utilization consistently exceeds 80%
4. **CPU Usage**: Scale if CPU utilization consistently exceeds 70%
5. **Geographic Distribution**: Deploy in multiple regions for global user bases

## Monitoring and Logging

### Prometheus Metrics

Monitoring Kibana with Prometheus:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kibana-metrics
  namespace: elk
  labels:
    app: kibana
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "5601"
    prometheus.io/path: "/api/reporting/stats"
spec:
  ports:
  - port: 5601
    targetPort: 5601
    name: metrics
  selector:
    app: kibana
```

### Metricbeat Monitoring

Using Metricbeat to monitor Kibana:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: metricbeat-kibana
  namespace: elk
data:
  metricbeat.yml: |
    metricbeat.modules:
    - module: kibana
      metricsets: ["status"]
      period: 10s
      hosts: ["https://kibana:5601"]
      basepath: ""
      username: "${ELASTICSEARCH_USERNAME}"
      password: "${ELASTICSEARCH_PASSWORD}"
      ssl.verification_mode: "certificate"
      ssl.certificate_authorities:
        - /usr/share/metricbeat/config/certs/ca.crt
      xpack.enabled: true
    
    output.elasticsearch:
      hosts: ["https://elasticsearch-master:9200"]
      username: "${ELASTICSEARCH_USERNAME}"
      password: "${ELASTICSEARCH_PASSWORD}"
      ssl.certificate_authorities:
        - /usr/share/metricbeat/config/certs/ca.crt
```

### Logging Configuration

Configuring Kibana logging:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-config
  namespace: elk
data:
  kibana.yml: |
    # Other configurations...
    
    # Logging Configuration
    logging:
      appenders:
        file:
          type: file
          fileName: /var/log/kibana/kibana.log
          layout:
            type: json
      root:
        appenders: [file]
        level: info
      loggers:
        - name: plugins
          level: info
        - name: optimization
          level: error
```

### Filebeat for Log Collection

Collecting Kibana logs with Filebeat:

```yaml
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
        - /var/log/containers/kibana-*.log
      processors:
        - add_kubernetes_metadata:
            host: ${NODE_NAME}
            matchers:
            - logs_path:
                logs_path: "/var/log/containers/"
    
    output.elasticsearch:
      hosts: ["https://elasticsearch-master:9200"]
      username: "${ELASTICSEARCH_USERNAME}"
      password: "${ELASTICSEARCH_PASSWORD}"
      ssl.certificate_authorities:
        - /usr/share/filebeat/config/certs/ca.crt
      index: "filebeat-kibana-%{+yyyy.MM.dd}"
```

## Integration with Elasticsearch

### Basic Elasticsearch Connection

Simple Elasticsearch connection configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-config
  namespace: elk
data:
  kibana.yml: |
    server.host: "0.0.0.0"
    server.name: "kibana"
    elasticsearch.hosts: ["http://elasticsearch-master:9200"]
```

### Secure Elasticsearch Connection

Configuring Kibana to connect to secured Elasticsearch:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-config
  namespace: elk
data:
  kibana.yml: |
    server.host: "0.0.0.0"
    server.name: "kibana"
    elasticsearch.hosts: ["https://elasticsearch-master:9200"]
    elasticsearch.ssl.verificationMode: certificate
    elasticsearch.ssl.certificateAuthorities: ["/usr/share/kibana/config/certs/ca.crt"]
    elasticsearch.username: "${ELASTICSEARCH_USERNAME}"
    elasticsearch.password: "${ELASTICSEARCH_PASSWORD}"
```

### Service Account Authentication

Using Elasticsearch service accounts for Kibana:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-config
  namespace: elk
data:
  kibana.yml: |
    server.host: "0.0.0.0"
    server.name: "kibana"
    elasticsearch.hosts: ["https://elasticsearch-master:9200"]
    elasticsearch.ssl.verificationMode: certificate
    elasticsearch.ssl.certificateAuthorities: ["/usr/share/kibana/config/certs/ca.crt"]
    elasticsearch.serviceAccountToken: "${ELASTICSEARCH_SERVICE_ACCOUNT_TOKEN}"
```

### Cross-Cluster Configuration

Configuring Kibana to work with multiple Elasticsearch clusters:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-config
  namespace: elk
data:
  kibana.yml: |
    server.host: "0.0.0.0"
    server.name: "kibana"
    
    elasticsearch.hosts: ["https://elasticsearch-master:9200"]
    elasticsearch.ssl.verificationMode: certificate
    elasticsearch.ssl.certificateAuthorities: ["/usr/share/kibana/config/certs/ca.crt"]
    elasticsearch.username: "${ELASTICSEARCH_USERNAME}"
    elasticsearch.password: "${ELASTICSEARCH_PASSWORD}"
    
    # Cross-cluster configuration
    elasticsearch.requestHeadersWhitelist: ["Authorization", "X-Security-User", "securitytenant"]
    elasticsearch.customHeaders: { remote-cluster: true }
    monitoring.ui.container.elasticsearch.enabled: true
```

## Plugin Management

### Pre-Installing Plugins

Installing plugins as part of the Kibana deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: elk
spec:
  # ...
  template:
    spec:
      initContainers:
      - name: install-plugins
        image: docker.elastic.co/kibana/kibana:7.17.8
        command:
        - /bin/bash
        - -c
        - |
          bin/kibana-plugin install https://github.com/example/example-plugin/releases/download/v1.0.0/example-plugin-1.0.0.zip
          chown -R kibana:kibana /usr/share/kibana/plugins
        volumeMounts:
        - name: plugins
          mountPath: /usr/share/kibana/plugins
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:7.17.8
        # ...
        volumeMounts:
        - name: plugins
          mountPath: /usr/share/kibana/plugins
      volumes:
      - name: plugins
        emptyDir: {}
```

### Custom Docker Image with Plugins

Creating a custom Kibana image with pre-installed plugins:

```Dockerfile
FROM docker.elastic.co/kibana/kibana:7.17.8

USER kibana
RUN /usr/share/kibana/bin/kibana-plugin install https://github.com/example/example-plugin/releases/download/v1.0.0/example-plugin-1.0.0.zip

USER root
RUN chown -R kibana:kibana /usr/share/kibana/plugins

USER kibana
```

### Plugin Configuration

Configuring installed plugins:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-config
  namespace: elk
data:
  kibana.yml: |
    server.host: "0.0.0.0"
    elasticsearch.hosts: ["http://elasticsearch-master:9200"]
    
    # Plugin Configuration
    example-plugin:
      enabled: true
      setting1: "value1"
      setting2: "value2"
    
    # Enable/disable built-in plugins
    xpack.reporting.enabled: true
    xpack.graph.enabled: false
    xpack.ml.enabled: true
```

## Customization and Branding

### Custom Branding

Customizing Kibana branding:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-config
  namespace: elk
data:
  kibana.yml: |
    server.host: "0.0.0.0"
    elasticsearch.hosts: ["http://elasticsearch-master:9200"]
    
    # Branding configuration
    server.customBranding.logo.defaultUrl: "/assets/custom_logo.svg"
    server.customBranding.favicon.defaultUrl: "/assets/custom_favicon.ico"
    server.customBranding.mark.defaultUrl: "/assets/custom_mark.svg"
    server.customBranding.loadingLogo.defaultUrl: "/assets/custom_loading_logo.svg"
    server.customBranding.background.defaultUrl: "/assets/custom_background.svg"
    server.customBranding.useFullColorLogo: true
    server.customBranding.useFullColorMark: true
    server.customBranding.pageTitle: "My Custom Kibana"
    server.customBranding.applicationTitle: "My Organization Analytics"
    server.customBranding.linkToTextContent: "My Organization"
```

### Custom Assets

Adding custom branding assets:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: elk
spec:
  # ...
  template:
    spec:
      containers:
      - name: kibana
        # ...
        volumeMounts:
        - name: config
          mountPath: /usr/share/kibana/config/kibana.yml
          subPath: kibana.yml
        - name: custom-branding
          mountPath: /usr/share/kibana/assets
      volumes:
      - name: config
        configMap:
          name: kibana-config
      - name: custom-branding
        configMap:
          name: kibana-branding-assets
```

### ConfigMap for Branding Assets

Storing branding assets in a ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-branding-assets
  namespace: elk
binaryData:
  # These would be base64 encoded binary files
  custom_logo.svg: <base64-encoded-svg>
  custom_favicon.ico: <base64-encoded-ico>
  custom_mark.svg: <base64-encoded-svg>
  custom_loading_logo.svg: <base64-encoded-svg>
  custom_background.svg: <base64-encoded-svg>
```

### Default Dashboard Configuration

Setting default dashboards and visualizations:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-config
  namespace: elk
data:
  kibana.yml: |
    server.host: "0.0.0.0"
    elasticsearch.hosts: ["http://elasticsearch-master:9200"]
    
    # Default app settings
    kibana.defaultAppId: "dashboard/my-custom-dashboard"
    
    # Dashboard mode (removes editing capabilities)
    kibana.savedobjects.reindexOnStartup: false
```

## Handling Upgrades

### Rolling Upgrade Strategy

Using Kubernetes rolling updates for Kibana upgrades:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
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
  name: kibana-blue
  namespace: elk
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kibana
      deployment: blue
  template:
    metadata:
      labels:
        app: kibana
        deployment: blue
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:7.14.0
        # ...

---
# Green deployment (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana-green
  namespace: elk
spec:
  replicas: 0  # Start with 0 replicas
  selector:
    matchLabels:
      app: kibana
      deployment: green
  template:
    metadata:
      labels:
        app: kibana
        deployment: green
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:7.17.8
        # ...

---
# Service that will switch between blue and green
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: elk
spec:
  selector:
    app: kibana
    deployment: blue  # Initially points to blue
  ports:
  - port: 5601
    name: http
```

For the transition:

1. Scale up the green deployment
2. Test the green deployment
3. Update the service selector to point to green
4. Scale down the blue deployment once traffic is successfully flowing through green

### Version-Specific Configuration

Managing configuration changes during upgrades:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-config-7-17
  namespace: elk
data:
  kibana.yml: |
    server.host: "0.0.0.0"
    elasticsearch.hosts: ["http://elasticsearch-master:9200"]
    
    # Settings specific to 7.17+
    kibana.autocompleteTerminateAfter: 100000
    kibana.autocompleteTimeout: 1000
    
    # Other configurations...
```

### Backing Up Saved Objects

Backing up Kibana saved objects before an upgrade:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: kibana-backup
  namespace: elk
spec:
  template:
    spec:
      containers:
      - name: kibana-backup
        image: curlimages/curl:latest
        command:
        - sh
        - -c
        - |
          curl -X POST "https://kibana:5601/api/saved_objects/_export" \
            -H "kbn-xsrf: true" \
            -H "Content-Type: application/json" \
            -u "${KIBANA_USER}:${KIBANA_PASSWORD}" \
            -d'{"objects":[{"type":"dashboard","id":"*"},{"type":"visualization","id":"*"},{"type":"search","id":"*"},{"type":"index-pattern","id":"*"}]}' \
            -o /backup/kibana-backup-$(date +%Y%m%d).ndjson
        volumeMounts:
        - name: backup-volume
          mountPath: /backup
      volumes:
      - name: backup-volume
        persistentVolumeClaim:
          claimName: kibana-backup-pvc
      restartPolicy: OnFailure
```

## Troubleshooting

### Common Issues and Solutions

1. **Connection Issues with Elasticsearch**
   - Verify Elasticsearch service and endpoints
   - Check credentials and TLS configuration
   - Ensure network policies allow communication

2. **Memory Problems**
   - Adjust resource limits
   - Monitor memory usage with Prometheus or Metricbeat
   - Check for memory leaks through the Kibana status API

3. **Performance Issues**
   - Scale Kibana horizontally for more users
   - Optimize Elasticsearch indices that Kibana queries
   - Check browser network performance

4. **Plugin Compatibility**
   - Ensure plugins are compatible with Kibana version
   - Remove incompatible plugins during upgrades
   - Check plugin logs for errors

### Debug Configuration

Adding debug logging for troubleshooting:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-debug-config
  namespace: elk
data:
  kibana.yml: |
    server.host: "0.0.0.0"
    elasticsearch.hosts: ["http://elasticsearch-master:9200"]
    
    # Debug logging
    logging.root.level: debug
    logging.appenders.default:
      type: console
      layout:
        type: json
    
    # Component-specific logging
    logging.loggers:
      - name: "elasticsearch"
        level: debug
      - name: "http"
        level: debug
      - name: "plugins"
        level: debug
      - name: "reporting"
        level: debug
```

### Status Checking Scripts

Init container to verify Elasticsearch availability:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: elk
spec:
  # ...
  template:
    spec:
      initContainers:
      - name: elasticsearch-ready
        image: curlimages/curl:latest
        command:
        - sh
        - -c
        - |
          until curl -s -k https://elasticsearch-master:9200 -u "${ELASTICSEARCH_USERNAME}:${ELASTICSEARCH_PASSWORD}" | grep -q '"cluster_name"'; do
            echo "Waiting for Elasticsearch to be ready..."
            sleep 5
          done
          echo "Elasticsearch is ready!"
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
      containers:
      - name: kibana
        # ...
```

### Diagnostic Information Script

ConfigMap with diagnostic script:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-diagnostic-script
  namespace: elk
data:
  collect-diagnostics.sh: |
    #!/bin/bash
    
    # Create diagnostic directory
    DIAG_DIR="/tmp/kibana-diagnostics-$(date +%Y%m%d-%H%M%S)"
    mkdir -p $DIAG_DIR
    
    # Collect Kibana status
    curl -s http://localhost:5601/api/status > $DIAG_DIR/kibana-status.json
    
    # Collect Kibana metrics
    curl -s http://localhost:5601/api/stats?extended > $DIAG_DIR/kibana-stats.json
    
    # Collect logs
    cp /var/log/kibana/kibana.log $DIAG_DIR/
    
    # Collect configuration (without sensitive data)
    grep -v "password\|secret\|key" /usr/share/kibana/config/kibana.yml > $DIAG_DIR/kibana-sanitized.yml
    
    # Collect Kubernetes metadata
    kubectl get pods -l app=kibana -o yaml > $DIAG_DIR/kibana-pods.yaml
    kubectl describe pods -l app=kibana > $DIAG_DIR/kibana-pods-describe.txt
    
    # Package diagnostics
    tar -czf /tmp/kibana-diagnostics.tar.gz -C /tmp $(basename $DIAG_DIR)
    echo "Diagnostics collected at /tmp/kibana-diagnostics.tar.gz"
```

## Summary

Deploying Kibana on Kubernetes provides a powerful, scalable, and manageable visualization platform for your Elasticsearch data. Key considerations include:

1. **Deployment Strategy**: Choose appropriate deployment patterns for your use case
2. **Configuration Management**: Use ConfigMaps and Secrets for efficient configuration
3. **Security**: Implement authentication, TLS, and network policies
4. **Networking**: Set up proper ingress and service configurations
5. **Resource Management**: Allocate appropriate CPU and memory resources
6. **High Availability**: Deploy multiple pods with anti-affinity for redundancy
7. **Scaling**: Implement horizontal scaling with HPA or manual scaling
8. **Monitoring**: Set up comprehensive monitoring for Kibana performance
9. **Integration**: Configure proper connections to Elasticsearch
10. **Upgrades**: Plan for smooth upgrades with minimal disruption

By following these best practices, you can create a robust, scalable, and maintainable Kibana deployment on Kubernetes that integrates seamlessly with the rest of your ELK Stack.

## References

- [Kibana Reference](https://www.elastic.co/guide/en/kibana/current/index.html)
- [Kibana Kubernetes Examples](https://github.com/elastic/kibana/tree/main/docs/settings)
- [Elasticsearch Kubernetes Operator](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-kibana-es.html)
- [Elastic Cloud on Kubernetes](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-overview.html)