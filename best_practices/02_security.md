# Security Best Practices

Securing your ELK Stack deployment is critical to protect both the infrastructure and the data it contains. This chapter covers comprehensive security best practices for each component of the ELK Stack, from authentication and encryption to network security and compliance requirements.

## Elasticsearch Security

### Authentication and Authorization

#### Enabling Security

Security is enabled by default in recent Elasticsearch versions. If using an older version, explicitly enable it:

```yaml
# elasticsearch.yml
xpack.security.enabled: true
```

#### User Authentication Methods

Elasticsearch supports multiple authentication methods:

1. **Native Realm**: Built-in user database within Elasticsearch
   ```
   POST /_security/user/admin
   {
     "password" : "strong-password",
     "roles" : ["superuser"],
     "full_name" : "Administrator"
   }
   ```

2. **LDAP/Active Directory**: Integrate with existing directory services
   ```yaml
   # elasticsearch.yml
   xpack.security.authc.realms.ldap.ldap1:
     order: 1
     url: "ldaps://ldap.example.com:636"
     bind_dn: "cn=elasticsearch,ou=services,dc=example,dc=com"
     bind_password: ${LDAP_BIND_PASSWORD}
     user_search:
       base_dn: "ou=users,dc=example,dc=com"
       filter: "(cn={0})"
     group_search:
       base_dn: "ou=groups,dc=example,dc=com"
     unmapped_groups_as_roles: false
   ```

3. **SAML**: Single sign-on using SAML providers (Okta, Auth0, etc.)
   ```yaml
   # elasticsearch.yml
   xpack.security.authc.realms.saml.saml1:
     order: 2
     idp.metadata.path: saml/idp-metadata.xml
     idp.entity_id: "https://idp.example.com"
     sp.entity_id: "https://kibana.example.com"
     sp.acs: "https://kibana.example.com:5601/api/security/v1/saml"
     attributes.principal: "nameid"
     attributes.groups: "groups"
   ```

4. **OpenID Connect**: Authentication using OpenID providers (Google, Azure AD, etc.)
   ```yaml
   # elasticsearch.yml
   xpack.security.authc.realms.oidc.oidc1:
     order: 3
     rp.client_id: "elasticsearch"
     rp.response_type: "code"
     rp.redirect_uri: "https://kibana.example.com:5601/api/security/v1/oidc"
     op.issuer: "https://accounts.google.com"
     op.authorization_endpoint: "https://accounts.google.com/o/oauth2/v2/auth"
     op.token_endpoint: "https://oauth2.googleapis.com/token"
     op.jwkset_path: "https://www.googleapis.com/oauth2/v3/certs"
     claims.principal: sub
   ```

5. **API Keys**: For service account authentication
   ```
   POST /_security/api_key
   {
     "name": "app1-key",
     "expiration": "30d",
     "role_descriptors": {
       "app1": {
         "cluster": ["monitor"],
         "indices": [
           {
             "names": ["app-logs-*"],
             "privileges": ["read"]
           }
         ]
       }
     }
   }
   ```

#### Role-Based Access Control (RBAC)

Define roles with specific privileges:

```
POST /_security/role/logs_reader
{
  "cluster": ["monitor"],
  "indices": [
    {
      "names": ["logs-*"],
      "privileges": ["read", "view_index_metadata"],
      "field_security": {
        "grant": ["timestamp", "message", "host", "severity"]
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

Create role mappings for LDAP/SAML groups:

```
POST /_security/role_mapping/ops_team
{
  "roles": ["logs_reader"],
  "rules": {
    "field": {
      "groups": "cn=ops,ou=groups,dc=example,dc=com"
    }
  },
  "enabled": true
}
```

#### Field and Document Level Security

Restrict access to specific fields:

```
POST /_security/role/pci_auditor
{
  "indices": [
    {
      "names": ["payment-*"],
      "privileges": ["read"],
      "field_security": {
        "grant": ["timestamp", "status", "amount"],
        "except": ["credit_card", "ssn"]
      }
    }
  ]
}
```

Restrict access to specific documents:

```
POST /_security/role/region_europe
{
  "indices": [
    {
      "names": ["customer-*"],
      "privileges": ["read"],
      "query": {
        "term": {
          "region": "europe"
        }
      }
    }
  ]
}
```

### Encryption

#### Transport Layer Encryption

Secure node-to-node communication:

```yaml
# elasticsearch.yml
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
```

Generate certificates with elasticsearch-certutil:

```bash
bin/elasticsearch-certutil ca
bin/elasticsearch-certutil cert --ca elastic-ca.p12
```

#### REST API Encryption

Secure client-to-node communication:

```yaml
# elasticsearch.yml
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: http.p12
```

#### Encryption at Rest

Use encrypted file systems or Elasticsearch's built-in encryption:

```yaml
# elasticsearch.yml
xpack.security.encryption_key: "a12345678901234567890123456789012"
```

For self-managed encryption keys:

```yaml
xpack.security.encryptionKey: ${ENCRYPTION_KEY}
```

### Audit Logging

Enable comprehensive audit logging:

```yaml
# elasticsearch.yml
xpack.security.audit.enabled: true
```

Configure specific events to audit:

```yaml
xpack.security.audit.logfile.events.include: ["authentication_success", "authentication_failed", "access_denied", "connection_denied", "tampered_request", "run_as_denied", "security_config_change"]
```

### Network Security

#### IP Filtering

Restrict which clients can connect:

```yaml
# elasticsearch.yml
network.host: 0.0.0.0
discovery.seed_hosts: ["10.0.0.1", "10.0.0.2", "10.0.0.3"]
transport.bind_host: ["10.0.0.1", "10.0.0.2", "10.0.0.3"]
http.host: ["10.0.0.1", "10.0.0.2", "10.0.0.3", "192.168.0.1"]
```

#### Firewall Configuration

Configure firewall rules to protect Elasticsearch:

```bash
# Allow internal communication between nodes
sudo iptables -A INPUT -p tcp -s 10.0.0.0/24 --dport 9300 -j ACCEPT

# Allow client communication only from specific IPs
sudo iptables -A INPUT -p tcp -s 192.168.0.0/24 --dport 9200 -j ACCEPT

# Drop all other traffic to these ports
sudo iptables -A INPUT -p tcp --dport 9200 -j DROP
sudo iptables -A INPUT -p tcp --dport 9300 -j DROP
```

#### Proxy Configuration

Use a reverse proxy (e.g., Nginx) for additional security:

```nginx
server {
    listen 443 ssl;
    server_name elastic.example.com;

    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;

    location / {
        proxy_pass https://localhost:9200;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Only allow GET, POST, and HEAD requests
        limit_except GET POST HEAD {
            deny all;
        }
    }
}
```

## Kibana Security

### Authentication

#### Basic Authentication

Enable basic authentication in Kibana:

```yaml
# kibana.yml
elasticsearch.username: "kibana_system"
elasticsearch.password: "${KIBANA_PASSWORD}"
```

#### SSO Integration

Configure SAML authentication:

```yaml
# kibana.yml
xpack.security.authc.providers:
  saml.saml1:
    order: 0
    realm: saml1
    description: "Log in with SSO"
    icon: "logoSecurity"
  basic.basic1:
    order: 1
```

Configure OpenID Connect:

```yaml
# kibana.yml
xpack.security.authc.providers:
  oidc.oidc1:
    order: 0
    realm: oidc1
    description: "Log in with Google"
    icon: "logoGoogle"
  basic.basic1:
    order: 1
```

### Authorization

#### Kibana Spaces

Create isolated spaces for different teams:

```
POST /api/spaces/space
{
  "id": "marketing",
  "name": "Marketing",
  "description": "Marketing team dashboards",
  "color": "#00bfb3",
  "initials": "MK"
}
```

#### Feature Controls

Restrict access to specific Kibana features:

```
POST /_security/role/kibana_editor
{
  "elasticsearch": {
    "cluster": ["monitor"],
    "indices": [
      {
        "names": ["logs-*"],
        "privileges": ["read", "view_index_metadata"]
      }
    ]
  },
  "kibana": [
    {
      "base": ["all"],
      "feature": {
        "dashboard": ["all"],
        "discover": ["all"],
        "visualize": ["all"],
        "dev_tools": ["none"],
        "advancedSettings": ["none"],
        "indexPatternManagement": ["read"]
      },
      "spaces": ["marketing"]
    }
  ]
}
```

### Secure Headers

Enable security headers in Kibana:

```yaml
# kibana.yml
server.xsrf.disableProtection: false
server.customResponseHeaders:
  X-Content-Type-Options: nosniff
  X-Frame-Options: SAMEORIGIN
  Strict-Transport-Security: "max-age=31536000; includeSubDomains"
```

### Session Management

Configure session timeout and other settings:

```yaml
# kibana.yml
xpack.security.session.idleTimeout: "1h"
xpack.security.session.lifespan: "24h"
```

## Logstash Security

### Input Security

#### Secure Beats Input

Configure SSL/TLS for Beats input:

```ruby
input {
  beats {
    port => 5044
    ssl => true
    ssl_certificate => "/etc/logstash/certs/logstash.crt"
    ssl_key => "/etc/logstash/certs/logstash.key"
    ssl_certificate_authorities => ["/etc/logstash/certs/ca.crt"]
    ssl_verify_mode => "force_peer"
  }
}
```

#### Client Authentication

Implement client certificate authentication:

```ruby
input {
  beats {
    port => 5044
    ssl => true
    ssl_certificate => "/etc/logstash/certs/logstash.crt"
    ssl_key => "/etc/logstash/certs/logstash.key"
    ssl_certificate_authorities => ["/etc/logstash/certs/ca.crt"]
    ssl_verify_mode => "force_peer"
    ssl_client_authentication => "required"
  }
}
```

### Pipeline Security

#### Sensitive Information Handling

Handle sensitive data using filter plugins:

```ruby
filter {
  mutate {
    gsub => ["message", "\d{4}-\d{4}-\d{4}-\d{4}", "[REDACTED]"]
  }
}
```

Use fingerprint filter for PII:

```ruby
filter {
  fingerprint {
    source => ["customer_ssn"]
    target => "anonymized_ssn"
    method => "SHA256"
    key => "${FINGERPRINT_KEY}"
    base64encode => true
  }
  mutate {
    remove_field => ["customer_ssn"]
  }
}
```

#### Secure Environment Variables

Use environment variables for sensitive configuration:

```ruby
output {
  elasticsearch {
    hosts => ["https://elasticsearch:9200"]
    user => "${ES_USER}"
    password => "${ES_PASSWORD}"
    ssl => true
    ssl_certificate_verification => true
  }
}
```

### Output Security

#### Elasticsearch Authentication

Configure secure connection to Elasticsearch:

```ruby
output {
  elasticsearch {
    hosts => ["https://elasticsearch:9200"]
    user => "${ES_USER}"
    password => "${ES_PASSWORD}"
    ssl => true
    ssl_certificate_verification => true
    cacert => "/etc/logstash/certs/ca.crt"
  }
}
```

#### Secure Keystores

Store sensitive information in Logstash keystore:

```bash
bin/logstash-keystore create
bin/logstash-keystore add ES_USER
bin/logstash-keystore add ES_PASSWORD
```

## Beats Security

### Secure Communication

#### TLS Configuration

Enable TLS for Filebeat outputs:

```yaml
# filebeat.yml
output.logstash:
  hosts: ["logstash:5044"]
  ssl.enabled: true
  ssl.certificate_authorities: ["/etc/filebeat/ca.crt"]
  ssl.certificate: "/etc/filebeat/filebeat.crt"
  ssl.key: "/etc/filebeat/filebeat.key"
```

#### Elasticsearch Authentication

Configure secure connection to Elasticsearch:

```yaml
# filebeat.yml
output.elasticsearch:
  hosts: ["https://elasticsearch:9200"]
  username: "${ES_USER}"
  password: "${ES_PASSWORD}"
  ssl.certificate_authorities: ["/etc/filebeat/ca.crt"]
```

### Secure Keystores

Store sensitive information in Beats keystore:

```bash
filebeat keystore create
filebeat keystore add ES_USER
filebeat keystore add ES_PASSWORD
```

## Container Security

### Docker Deployment

#### Secure Docker Configurations

Run Elasticsearch securely in Docker:

```bash
docker run \
  --name elasticsearch \
  --net elastic \
  -p 9200:9200 \
  -e "discovery.type=single-node" \
  -e "xpack.security.enabled=true" \
  -e "ELASTIC_PASSWORD=secure_password" \
  -e "bootstrap.memory_lock=true" \
  --ulimit memlock=-1:-1 \
  -v elasticsearch-data:/usr/share/elasticsearch/data \
  -v elasticsearch-certs:/usr/share/elasticsearch/config/certs \
  docker.elastic.co/elasticsearch/elasticsearch:8.8.0
```

#### Network Isolation

Create dedicated Docker networks:

```bash
docker network create --driver bridge elastic
```

### Kubernetes Deployment

#### Security Contexts

Configure security contexts for Elasticsearch pods:

```yaml
securityContext:
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
  runAsNonRoot: true
  capabilities:
    drop: ["ALL"]
```

#### Secret Management

Store sensitive information in Kubernetes Secrets:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: elasticsearch-credentials
type: Opaque
data:
  username: ZWxhc3RpYw==  # elastic
  password: c2VjdXJlX3Bhc3N3b3Jk  # secure_password
```

#### Network Policies

Implement Kubernetes network policies:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: elasticsearch-network-policy
spec:
  podSelector:
    matchLabels:
      app: elasticsearch
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
          app: elasticsearch
    ports:
    - protocol: TCP
      port: 9300
```

## Compliance Best Practices

### Data Protection and Privacy

#### GDPR Compliance

Implement pseudonymization:

```ruby
filter {
  mutate {
    gsub => [
      "message", "\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}\b", "[EMAIL_REDACTED]",
      "message", "\b(?:\+\d{1,2}\s)?\(?\d{3}\)?[\s.-]?\d{3}[\s.-]?\d{4}\b", "[PHONE_REDACTED]"
    ]
  }
}
```

Enable data deletion capabilities:

```
POST /customer-data/_delete_by_query
{
  "query": {
    "term": {
      "user_id": "user123"
    }
  }
}
```

#### PCI-DSS Requirements

Implement security controls for payment card data:

- Field level encryption
- Field level security
- Audit logging
- Strong authentication

### Audit Trail

#### Comprehensive Logging

Enable detailed audit logs:

```yaml
# elasticsearch.yml
xpack.security.audit.enabled: true
xpack.security.audit.logfile.events.include: ["access_denied", "authentication_failed", "connection_denied", "tampered_request", "run_as_denied", "authentication_success", "realm_authentication"]
```

#### Log Retention

Implement proper log retention policies:

```
PUT _ilm/policy/audit_logs_policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_size": "5GB",
            "max_age": "7d"
          },
          "set_priority": {
            "priority": 100
          }
        }
      },
      "warm": {
        "min_age": "30d",
        "actions": {
          "forcemerge": {
            "max_num_segments": 1
          },
          "readonly": {},
          "set_priority": {
            "priority": 50
          }
        }
      },
      "cold": {
        "min_age": "60d",
        "actions": {
          "set_priority": {
            "priority": 0
          }
        }
      },
      "delete": {
        "min_age": "365d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

## Security Monitoring

### Intrusion Detection

#### Watcher for Security Events

Create watches for security event detection:

```
PUT _watcher/watch/failed_login_attempts
{
  "trigger": {
    "schedule": {
      "interval": "1m"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": [".security-*"],
        "body": {
          "query": {
            "bool": {
              "must": [
                {
                  "term": {
                    "event.action": "authentication_failed"
                  }
                },
                {
                  "range": {
                    "@timestamp": {
                      "gte": "now-5m"
                    }
                  }
                }
              ]
            }
          },
          "aggs": {
            "ips": {
              "terms": {
                "field": "source.ip",
                "size": 10
              },
              "aggs": {
                "attempts": {
                  "value_count": {
                    "field": "_id"
                  }
                }
              }
            }
          },
          "size": 0
        }
      }
    }
  },
  "condition": {
    "script": {
      "source": "for (ip in ctx.payload.aggregations.ips.buckets) { if (ip.attempts.value >= 5) { return true; } } return false;"
    }
  },
  "actions": {
    "email_admin": {
      "email": {
        "to": "security@example.com",
        "subject": "Multiple Failed Login Attempts",
        "body": {
          "html": "<p>Multiple failed login attempts detected:</p><ul>{{#ctx.payload.aggregations.ips.buckets}}<li>IP: {{key}} - {{attempts.value}} attempts</li>{{/ctx.payload.aggregations.ips.buckets}}</ul>"
        }
      }
    }
  }
}
```

#### Real-time Monitoring

Create security monitoring dashboards in Kibana:

1. Index pattern: `.security-*`
2. Visualizations:
   - Failed authentication attempts over time
   - Top users with failed login attempts
   - Geographic distribution of connections
   - Authentication success vs. failure ratio

### Anomaly Detection

#### Machine Learning Jobs

Create machine learning jobs to detect anomalies:

```
PUT _ml/anomaly_detectors/auth_anomalies
{
  "description": "Unusual authentication patterns",
  "analysis_config": {
    "bucket_span": "15m",
    "detectors": [
      {
        "detector_description": "Unusual authentication count",
        "function": "high_count",
        "over_field_name": "user.name"
      },
      {
        "detector_description": "Unusual source IPs per user",
        "function": "rare",
        "by_field_name": "user.name",
        "over_field_name": "source.ip"
      }
    ],
    "influencers": [
      "user.name",
      "source.ip",
      "event.action"
    ]
  },
  "data_description": {
    "time_field": "@timestamp"
  }
}
```

## Operational Security

### Secrets Management

#### Secure Configuration

Use environment variables instead of hardcoded values:

```yaml
# elasticsearch.yml
xpack.security.authc.realms.ldap.ldap1:
  url: "${LDAP_URL}"
  bind_dn: "${LDAP_BIND_DN}"
  bind_password: "${LDAP_BIND_PASSWORD}"
```

#### External Secret Stores

Integrate with external secret management tools:

1. HashiCorp Vault
2. Azure Key Vault
3. AWS Secrets Manager
4. Google Secret Manager

### Vulnerability Management

#### Regular Updates

Keep all ELK components updated:

```bash
# Update Elasticsearch
curl -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.8.0-linux-x86_64.tar.gz
curl -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.8.0-linux-x86_64.tar.gz.sha512
shasum -a 512 -c elasticsearch-8.8.0-linux-x86_64.tar.gz.sha512
tar -xzf elasticsearch-8.8.0-linux-x86_64.tar.gz
# Stop the service, replace binaries, start the service
```

#### Security Scanning

Implement container security scanning:

```bash
# Scan Elasticsearch Docker image
docker scan docker.elastic.co/elasticsearch/elasticsearch:8.8.0
```

### Backup and Recovery

#### Secure Snapshots

Create encrypted snapshots:

```
PUT _snapshot/secure_backup
{
  "type": "fs",
  "settings": {
    "location": "/mnt/backup",
    "compress": true,
    "max_snapshot_bytes_per_sec": "50mb",
    "chunk_size": "100m",
    "readonly": false
  }
}
```

#### Access Control for Backups

Implement strict access controls for backup repositories:

```
POST /_security/role/snapshot_admin
{
  "cluster": ["manage_slm", "cluster:admin/snapshot/*"],
  "indices": [
    {
      "names": ["*"],
      "privileges": ["create_index", "monitor", "read"]
    }
  ]
}
```

## Conclusion

Securing the ELK Stack requires a layered approach covering authentication, authorization, encryption, network security, and operational security. Following these best practices will help protect your ELK Stack deployment from most security threats while maintaining compliance with industry regulations.

Remember that security is an ongoing process, requiring regular reviews, updates, and tests to ensure that your ELK Stack remains secure as your environment evolves and new threats emerge.

In the next chapter, we'll explore data modeling best practices to ensure your ELK Stack is not only secure but also optimally structured for performance and analysis.