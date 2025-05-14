# Security and Access Control

## Introduction to Elasticsearch Security

Security is a critical aspect of any production Elasticsearch deployment. As Elasticsearch often stores sensitive data and provides powerful analytics capabilities, proper security controls are essential to protect your data and infrastructure.

This chapter covers comprehensive security features in Elasticsearch, including authentication, authorization, encryption, auditing, and security best practices.

## Authentication and User Management

Authentication verifies the identity of users and systems accessing Elasticsearch.

### Built-in Security

Enable X-Pack security in `elasticsearch.yml`:

```yaml
xpack.security.enabled: true
```

### Built-in Users

Elasticsearch comes with predefined built-in users:

- **elastic**: Superuser with full cluster privileges
- **kibana_system**: Used by Kibana to connect to Elasticsearch
- **logstash_system**: Used by Logstash to write monitoring information
- **beats_system**: Used by Beats to write monitoring information
- **apm_system**: Used by APM server to write monitoring information
- **remote_monitoring_user**: Used by Metricbeat to collect monitoring data

Set or reset built-in user passwords:

```bash
# Interactive password setting
bin/elasticsearch-setup-passwords interactive

# Auto-generated passwords
bin/elasticsearch-setup-passwords auto
```

### Creating Users

Create a user with the user API:

```json
POST /_security/user/john_smith
{
  "password" : "secure_password",
  "roles" : [ "admin", "read_logs" ],
  "full_name" : "John Smith",
  "email" : "john.smith@example.com",
  "metadata" : {
    "department" : "IT",
    "employee_id" : "12345"
  }
}
```

List existing users:

```
GET /_security/user
```

Get a specific user:

```
GET /_security/user/john_smith
```

Delete a user:

```
DELETE /_security/user/john_smith
```

### Authentication Methods

#### Basic Authentication

The simplest form, using username and password:

```bash
curl -u elastic:password https://localhost:9200
```

#### API Keys

Create API keys for service accounts:

```json
POST /_security/api_key
{
  "name": "application-1-key",
  "expiration": "30d",
  "role_descriptors": {
    "role-a": {
      "cluster": ["monitor"],
      "indices": [
        {
          "names": ["logs-*"],
          "privileges": ["read"]
        }
      ]
    }
  }
}
```

Response:

```json
{
  "id": "VuaCfGcBCdbkQm-e5aOx",
  "name": "application-1-key",
  "expiration": 1623887999999,
  "api_key": "ui2lp2axTNmsyakw9tvNnw"
}
```

Use an API key:

```bash
curl -H "Authorization: ApiKey VnVhQ2ZHY0JDZGJrUW0tZTVhT3g6dWkybHAyYXhUTm1zeWFrdzl0dk5udw==" https://localhost:9200/_cluster/health
```

#### PKI Authentication

Use client certificates for authentication:

Configure Elasticsearch to accept client certificates:

```yaml
# elasticsearch.yml
xpack.security.authc.realms.pki.pki1:
  type: pki
  order: 1
```

#### LDAP Authentication

Integrate with your LDAP directory:

```yaml
# elasticsearch.yml
xpack.security.authc.realms.ldap.ldap1:
  type: ldap
  order: 2
  url: "ldaps://ldap.example.com:636"
  bind_dn: "cn=elasticsearch,ou=services,dc=example,dc=com"
  bind_password: "${LDAP_BIND_PASSWORD}"
  user_search:
    base_dn: "ou=users,dc=example,dc=com"
    filter: "(cn={0})"
  group_search:
    base_dn: "ou=groups,dc=example,dc=com"
  unmapped_groups_as_roles: false
```

#### Active Directory Authentication

Connect to Active Directory:

```yaml
# elasticsearch.yml
xpack.security.authc.realms.active_directory.ad1:
  type: active_directory
  order: 3
  domain_name: example.com
  url: "ldaps://ad.example.com:636"
  unmapped_groups_as_roles: false
```

#### SAML Authentication

Enable Single Sign-On with SAML:

```yaml
# elasticsearch.yml
xpack.security.authc.realms.saml.saml1:
  order: 4
  idp.metadata.path: saml/idp-metadata.xml
  idp.entity_id: "https://idp.example.com/saml2/idp"
  sp.entity_id: "https://elasticsearch.example.com"
  sp.acs: "https://elasticsearch.example.com:9200/api/security/v1/saml"
  attributes.principal: "NameID"
  attributes.groups: "groups"
  attributes.mail: "email"
  attributes.name: "name"
```

#### OpenID Connect Authentication

Authenticate with OpenID Connect providers:

```yaml
# elasticsearch.yml
xpack.security.authc.realms.oidc.oidc1:
  order: 5
  rp.client_id: "elasticsearch"
  rp.response_type: "code"
  rp.redirect_uri: "https://elasticsearch.example.com:5601/api/security/v1/oidc"
  op.issuer: "https://accounts.google.com"
  op.authorization_endpoint: "https://accounts.google.com/o/oauth2/v2/auth"
  op.token_endpoint: "https://oauth2.googleapis.com/token"
  op.jwkset_path: "https://www.googleapis.com/oauth2/v3/certs"
  claims.principal: sub
  claims.groups: "groups"
```

### Authentication Service

Control the authentication chain with `xpack.security.authc.token.enabled`:

```yaml
# elasticsearch.yml
xpack.security.authc.token.enabled: true
```

Generate a token:

```json
POST /_security/oauth2/token
{
  "grant_type" : "password",
  "username" : "john_smith",
  "password" : "secure_password"
}
```

## Authorization and Role-Based Access Control

Authorization determines what authenticated users can do in Elasticsearch.

### Role-Based Access Control

Create roles with specific permissions:

```json
POST /_security/role/logs_reader
{
  "cluster": ["monitor"],
  "indices": [
    {
      "names": ["logs-*"],
      "privileges": ["read", "view_index_metadata"],
      "field_security" : {
        "grant" : ["category", "message", "@timestamp", "level"]
      },
      "query": {
        "term": {
          "level": "ERROR"
        }
      }
    }
  ],
  "applications": [
    {
      "application": "kibana-.kibana",
      "privileges": ["read"],
      "resources": ["*"]
    }
  ],
  "run_as": ["logstash_writer"],
  "metadata": {
    "version": 1
  }
}
```

Key components of a role:
- **Cluster privileges**: Actions that can be performed against the cluster API
- **Index privileges**: Actions that can be performed against specific indices
- **Field-level security**: Control which document fields users can see
- **Document-level security**: Control which documents users can see using queries
- **Application privileges**: Control access to application features
- **Run as privileges**: Allow impersonating other users

### Cluster Privileges

Common cluster privileges include:

- **all**: All cluster operations
- **monitor**: Cluster monitoring operations
- **manage**: Cluster management operations
- **create_snapshot**: Create snapshots
- **manage_ilm**: Manage index lifecycle policies
- **manage_index_templates**: Manage index templates
- **manage_security**: Manage users and roles

### Index Privileges

Common index privileges include:

- **all**: All operations on indices
- **read**: Read operations (search, get)
- **write**: Write operations (index, update, delete)
- **index**: Index documents
- **delete**: Delete documents
- **create_index**: Create indices
- **view_index_metadata**: View index metadata
- **manage**: Index management operations

### Role Management

List existing roles:

```
GET /_security/role
```

Get a specific role:

```
GET /_security/role/logs_reader
```

Delete a role:

```
DELETE /_security/role/logs_reader
```

### Role Mappings

Map roles to external identities (LDAP groups, SAML attributes, etc.):

```json
POST /_security/role_mapping/ldap_admins
{
  "roles": [ "admin" ],
  "rules": {
    "field": {
      "groups": "cn=admins,ou=groups,dc=example,dc=com"
    }
  },
  "enabled": true,
  "metadata": {
    "version": 1
  }
}
```

Rules can be complex with AND, OR, and NOT conditions:

```json
"rules": {
  "all": [
    {
      "field": {
        "groups": "cn=admins,ou=groups,dc=example,dc=com"
      }
    },
    {
      "any": [
        {
          "field": {
            "realm.name": "ldap1"
          }
        },
        {
          "field": {
            "realm.name": "ad1"
          }
        }
      ]
    }
  ]
}
```

### Field-Level Security

Restrict access to specific document fields:

```json
"indices": [
  {
    "names": ["customer-*"],
    "privileges": ["read"],
    "field_security": {
      "grant": ["name", "address.city", "order_count"],
      "except": ["ssn", "credit_card"]
    }
  }
]
```

### Document-Level Security

Restrict access to specific documents:

```json
"indices": [
  {
    "names": ["logs-*"],
    "privileges": ["read"],
    "query": {
      "terms": {
        "level": ["INFO", "WARN"]
      }
    }
  }
]
```

## Encryption and TLS

Secure communications with Transport Layer Security (TLS).

### Node-to-Node Encryption

Encrypt traffic between Elasticsearch nodes:

```yaml
# elasticsearch.yml
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
```

Generate certificates for node-to-node encryption:

```bash
bin/elasticsearch-certutil ca
bin/elasticsearch-certutil cert --ca elastic-ca.p12
```

### HTTP Encryption

Encrypt client-to-node communication:

```yaml
# elasticsearch.yml
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: http.p12
```

Generate certificates for HTTP encryption:

```bash
bin/elasticsearch-certutil http
```

### Certificate Settings

Configure certificate validation:

```yaml
# elasticsearch.yml
xpack.security.transport.ssl.verification_mode: full
xpack.security.transport.ssl.client_authentication: required
```

Verification modes:
- **full**: Verify certificate and hostname
- **certificate**: Verify certificate only
- **none**: No certificate verification

Client authentication options:
- **required**: Client must provide a certificate
- **optional**: Client can provide a certificate
- **none**: Client certificate not required

## IP Filtering

Restrict which hosts can connect to Elasticsearch:

```yaml
# elasticsearch.yml
xpack.security.transport.filter.allow: ["192.168.1.0/24", "10.0.0.0/8"]
xpack.security.transport.filter.deny: ["10.0.0.5"]

xpack.security.http.filter.allow: ["192.168.1.0/24", "10.0.0.0/8"]
xpack.security.http.filter.deny: ["10.0.0.5"]
```

## Audit Logging

Track security-related events with audit logging:

```yaml
# elasticsearch.yml
xpack.security.audit.enabled: true
```

Configure which events to audit:

```yaml
xpack.security.audit.logfile.events.include: ["authentication_success", "authentication_failed", "access_denied", "connection_denied", "anonymous_access_denied"]
```

Excludes can be defined with:

```yaml
xpack.security.audit.logfile.events.exclude: ["realm_authentication"]
```

### Audit Logging Destinations

Configure where audit logs are stored:

```yaml
# Log to a file
xpack.security.audit.logfile.path: /var/log/elasticsearch/audit.json

# Log to index
xpack.security.audit.index.settings.index.number_of_shards: 3
xpack.security.audit.index.settings.index.number_of_replicas: 1
```

### Filtering Audit Events

Filter audit events for specific users or indices:

```yaml
xpack.security.audit.logfile.events.ignore_filters:
  - description: "Ignore kibana system user"
    users: ["kibana_system"]
  - description: "Ignore monitoring calls"
    indices: [".monitoring-*"]
    actions: ["indices:data/read/*"]
```

## Secure Settings

Protect sensitive configuration values:

```bash
# Add a setting to the keystore
bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password

# List settings in the keystore
bin/elasticsearch-keystore list

# Remove a setting from the keystore
bin/elasticsearch-keystore remove xpack.security.transport.ssl.keystore.secure_password
```

Access secure settings in configuration:

```yaml
# elasticsearch.yml
xpack.security.transport.ssl.keystore.password: ${xpack.security.transport.ssl.keystore.secure_password}
```

## API Key Management

### Create API Keys

Create API keys with specific privileges:

```json
POST /_security/api_key
{
  "name": "log_reader_key",
  "expiration": "30d",
  "role_descriptors": {
    "logs_reader": {
      "cluster": ["monitor"],
      "indices": [
        {
          "names": ["logs-*"],
          "privileges": ["read"]
        }
      ]
    }
  },
  "metadata": {
    "application": "log_analyzer",
    "environment": "production"
  }
}
```

### Invalidate API Keys

Invalidate (revoke) API keys:

```json
DELETE /_security/api_key
{
  "ids": ["VuaCfGcBCdbkQm-e5aOx", "WuaCfGcBCdbkQm-e5aOy"],
  "owner": true
}
```

Invalidate by name:

```json
DELETE /_security/api_key
{
  "name": "log_reader_key"
}
```

### Query API Keys

Get information about API keys:

```json
GET /_security/api_key
{
  "id": "VuaCfGcBCdbkQm-e5aOx"
}
```

Get your own API keys:

```json
GET /_security/api_key?owner=true
```

## Application Privileges

Define custom privilege types for applications:

```json
POST /_security/privilege
{
  "my_application": {
    "read": {
      "application": "my_application",
      "name": "read",
      "actions": ["data:read/*", "action:login"],
      "metadata": {
        "description": "Read access to my_application"
      }
    },
    "write": {
      "application": "my_application",
      "name": "write",
      "actions": ["data:write/*"],
      "metadata": {
        "description": "Write access to my_application"
      }
    }
  }
}
```

Use application privileges in roles:

```json
POST /_security/role/my_app_user
{
  "applications": [
    {
      "application": "my_application",
      "privileges": ["read"],
      "resources": ["*"]
    }
  ]
}
```

Check application privileges:

```json
POST /_security/has_privileges
{
  "application": [
    {
      "application": "my_application",
      "privileges": ["read", "write"],
      "resources": ["resource1", "resource2"]
    }
  ]
}
```

## Cross-Cluster Security

Configure security for cross-cluster search and replication:

### Configure Remote Clusters

```yaml
# elasticsearch.yml
cluster.remote.cluster_two.seeds: ["192.168.1.10:9300", "192.168.1.11:9300"]
```

### Remote Cluster Authentication

```yaml
# elasticsearch.yml
cluster.remote.cluster_two.credentials: ${cluster_two_credentials}
```

Add credentials to the keystore:

```bash
echo "elastic:password" | bin/elasticsearch-keystore add --stdin cluster_two_credentials
```

### Cross-Cluster Roles

Create roles for cross-cluster operations:

```json
POST /_security/role/remote_search
{
  "cluster": ["remote_cluster_client"],
  "indices": [
    {
      "names": ["remote_index*"],
      "privileges": ["read", "view_index_metadata"],
      "allow_restricted_indices": false
    }
  ]
}
```

## Security Best Practices

### General Security Recommendations

1. **Use HTTPS for all communications**
   - Enable TLS for HTTP and transport layers
   - Use strong ciphers and TLS 1.2+
   - Regularly update certificates

2. **Implement proper authentication**
   - Use SSO when possible
   - Require strong passwords
   - Implement MFA for sensitive deployments
   - Rotate credentials regularly

3. **Follow least privilege principle**
   - Create specific roles with minimal privileges
   - Avoid using the elastic superuser for regular operations
   - Implement field and document level security where appropriate

4. **Network security**
   - Use firewalls to restrict access
   - Use VPNs or private networks for Elasticsearch clusters
   - Implement IP filtering for all endpoints

5. **Regular auditing**
   - Enable audit logging
   - Review logs regularly
   - Set up alerts for suspicious activity

### User Management Best Practices

1. **Create dedicated users for services**
   - Use separate users for Kibana, Logstash, and Beats
   - Use API keys for application integrations

2. **Implement user lifecycle management**
   - Remove unused accounts
   - Disable accounts for departing users
   - Regular access reviews

3. **Secure the superuser account**
   - Change default passwords
   - Restrict superuser access to administrators
   - Use multi-factor authentication if available

### System Configuration Hardening

1. **Filesystem permissions**
   - Restrict access to Elasticsearch directories
   - Secure keystores and certificate files
   - Use appropriate file permissions

2. **Operating system hardening**
   - Keep the operating system updated
   - Remove unnecessary services
   - Use host-based firewalls

3. **JVM security**
   - Keep the JVM updated
   - Configure appropriate security options
   - Limit JVM permissions when possible

### Monitoring and Alerting

1. **Monitor security events**
   - Set up alerts for authentication failures
   - Monitor for unusual access patterns
   - Track privilege escalation

2. **Regular security audits**
   - Schedule regular security reviews
   - Test authentication and authorization
   - Validate TLS configuration

## Appendix: Security Migration Guide

### Migrating from Open to Secured Cluster

1. **Enable security on a test cluster first**
2. **Back up your data**
3. **Update configuration**:
   ```yaml
   xpack.security.enabled: true
   ```
4. **Generate certificates**:
   ```bash
   bin/elasticsearch-certutil ca
   bin/elasticsearch-certutil cert --ca elastic-ca.p12
   ```
5. **Configure TLS**:
   ```yaml
   xpack.security.transport.ssl.enabled: true
   xpack.security.transport.ssl.verification_mode: certificate
   xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
   xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
   ```
6. **Set passwords**:
   ```bash
   bin/elasticsearch-setup-passwords interactive
   ```
7. **Update clients** to use authentication

### Migrating Between Authentication Methods

1. **Set up the new authentication method alongside existing one**:
   ```yaml
   xpack.security.authc.realms.native.native1:
     order: 0
   xpack.security.authc.realms.ldap.ldap1:
     order: 1
     # LDAP configuration
   ```
2. **Create role mappings** for the new authentication method
3. **Test authentication** with the new method
4. **Migrate users** to the new authentication method
5. **Update configuration** to remove or reprioritize the old method

## Conclusion

Security is a critical aspect of any Elasticsearch deployment. This chapter covered the comprehensive security features available in Elasticsearch, including authentication, authorization, encryption, and auditing. By implementing these security features and following best practices, you can ensure your Elasticsearch cluster remains secure and compliant with organizational and regulatory requirements.

The next chapter will cover monitoring and alerting to help you maintain the health and performance of your Elasticsearch cluster.