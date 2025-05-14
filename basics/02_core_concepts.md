# Core Concepts and Terminology

Understanding the fundamental concepts and terminology is crucial for working effectively with the ELK Stack. This chapter covers the essential terms and concepts you'll encounter.

## Elasticsearch Concepts

### Documents and Indices

- **Document**: The basic unit of information in Elasticsearch, stored in JSON format. A document might be a log entry, a product, a customer record, etc.

- **Index**: A collection of documents with similar characteristics. Conceptually similar to a database table, an index stores and provides search capabilities for documents.

- **Mapping**: Defines the schema for documents within an index, specifying field types and how they should be indexed.

- **Type**: A logical category/partition within an index (deprecated in recent versions).

### Elasticsearch Distribution and Organization

- **Node**: A single instance of Elasticsearch running on a machine.

- **Cluster**: A collection of connected Elasticsearch nodes that work together.

- **Shard**: An index is divided into shards, which are distributed across nodes in a cluster. Shards allow horizontal scaling and parallel operations.
  - **Primary Shard**: Original shard that receives all indexing operations.
  - **Replica Shard**: Copy of a primary shard for redundancy and read scaling.

- **Index Lifecycle Management (ILM)**: Policies to manage indices as they age, including rollover, shrink, freeze, and delete actions.

### Search and Analytics Concepts

- **Query**: Request for information from Elasticsearch, using the Query DSL (Domain Specific Language).

- **Filter**: A query that determines whether a document matches certain criteria without scoring.

- **Aggregation**: Analytical operations on data, similar to GROUP BY in SQL.

- **Analyzer**: Process of converting text into tokens for indexing and searching, including tokenization, token filtering, and character filtering.

## Logstash Concepts

- **Pipeline**: The processing path of Logstash, consisting of inputs, filters, and outputs.

- **Input Plugin**: Collects data from a source (e.g., files, Beats, HTTP).

- **Filter Plugin**: Modifies, transforms, or enriches data (e.g., grok, mutate, geoip).

- **Output Plugin**: Sends data to a destination (e.g., Elasticsearch, S3, email).

- **Codec**: Format of the data being processed (e.g., JSON, plain).

- **Event**: A single unit of data processed by Logstash.

- **Grok**: Pattern matching syntax used to parse unstructured data.

## Kibana Concepts

- **Dashboard**: Collection of visualizations, saved searches, and other content.

- **Visualization**: Graphical representation of your data (e.g., charts, tables, metrics).

- **Saved Search**: Stored search queries and filters.

- **Index Pattern**: Way to identify the Elasticsearch indices you want to explore in Kibana.

- **Canvas**: Feature for creating custom presentations of data.

- **Lens**: Drag-and-drop interface for building visualizations.

- **Spaces**: Organizational feature to separate dashboards and saved objects.

## Beats Concepts

- **Beat**: Lightweight data shipper installed as an agent on servers.

- **Module**: Pre-packaged configurations for specific data sources.

- **Field**: Individual piece of data collected by a Beat.

- **Metricset**: Group of metrics collected together by Metricbeat.

## Security Concepts

- **Role**: Set of privileges assigned to users.

- **User**: Identity that can access the Elastic Stack.

- **API Key**: Secret used to authenticate API requests.

- **TLS/SSL**: Protocol for securing communications.

- **Encrypted communications**: Data encrypted during transport.

- **Node-to-node encryption**: Encryption between Elasticsearch nodes.

- **Field-level security**: Controls which document fields a user can see.

## Common Architecture Concepts

- **Ingest Node**: Elasticsearch node that pre-processes documents before indexing.

- **Coordinator Node**: Elasticsearch node that routes requests and aggregates results.

- **Master Node**: Elasticsearch node responsible for cluster-wide actions.

- **Data Node**: Elasticsearch node that stores data and executes queries.

- **Machine Learning Node**: Specialized node for machine learning jobs.

- **Hot-Warm-Cold Architecture**: Tiered storage approach based on data age and access patterns.

## Monitoring Concepts

- **Metricbeat**: Collects system and service metrics.

- **Monitoring**: Built-in feature to track the health of your Elastic Stack.

- **Uptime Monitoring**: Tracks availability of services using Heartbeat.

- **APM (Application Performance Monitoring)**: Tracks application performance and errors.

## Understanding these core concepts will provide a solid foundation as we explore each component of the ELK Stack in more detail in the following chapters.