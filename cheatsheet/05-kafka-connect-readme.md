# Kafka Connect - Master Level Cheat Sheet

A comprehensive reference guide for Kafka Connect covering basics, common connectors, and customization for master-level proficiency.

## Table of Contents
- [Definition](#definition)
- [Key Highlights](#key-highlights)
- [Responsibility / Role](#responsibility--role)
- [Underlying Data Structures / Mechanism](#underlying-data-structures--mechanism)
- [Advantages](#advantages)
- [Disadvantages / Trade-offs](#disadvantages--trade-offs)
- [Corner Cases](#corner-cases)
- [Limits / Boundaries](#limits--boundaries)
- [Default Values](#default-values)
- [Best Practices](#best-practices)

---

## Definition

Kafka Connect is a scalable and reliable streaming data integration framework for Apache Kafka that standardizes the integration of Kafka with external systems through pre-built and custom connectors. It provides a REST API-driven platform for configuring, deploying, and managing data pipelines that move data between Kafka and external databases, file systems, key-value stores, search indexes, and application systems. Connect abstracts the complexity of building reliable, fault-tolerant data integration solutions by providing common infrastructure for connector development, automatic offset management, error handling, and exactly-once delivery semantics.

## Key Highlights

- **Source Connectors** import data from external systems into Kafka topics, while **Sink Connectors** export data from Kafka topics to external systems, enabling bidirectional data flow
- **Distributed Mode** runs Connect as a scalable, fault-tolerant service across multiple worker nodes with automatic load balancing, while **Standalone Mode** runs on a single worker for development and simple use cases
- **REST API Management** enables dynamic connector configuration, deployment, monitoring, and lifecycle management without requiring restarts or downtime
- **Common Enterprise Connectors** include JDBC for database integration, Debezium for change data capture, Elasticsearch for search indexing, and S3 for object storage, covering most integration scenarios
- **Single Message Transforms (SMTs)** provide lightweight, real-time data transformation capabilities during the data pipeline flow without requiring separate stream processing applications

## Responsibility / Role

Kafka Connect serves as the universal data integration layer between Kafka and external systems, handling all aspects of reliable data movement including connection management, offset tracking, error handling, and delivery guarantees. It manages the lifecycle of connector instances across distributed workers, automatically balancing connector tasks, handling failures through restarts and retries, and providing monitoring capabilities through JMX metrics and REST endpoints. Connect is responsible for maintaining exactly-once or at-least-once delivery semantics depending on configuration, managing schema evolution through integration with Schema Registry, and providing a plugin system for extending functionality through custom connectors and transforms.

## Underlying Data Structures / Mechanism

Connect uses **Tasks** as the fundamental unit of work that actually move data, with each connector creating one or more tasks that run on worker nodes and maintain their own offset state. **Worker Processes** form a Connect cluster that coordinates task distribution, configuration storage, and offset management through internal Kafka topics (__connect-config, __connect-offsets, __connect-status) that provide distributed coordination and fault tolerance. **Converter Plugins** handle serialization and deserialization between Connect's internal data format and Kafka message formats, supporting JSON, Avro, and Protobuf with automatic schema handling. The **Plugin System** uses Java's ServiceLoader mechanism to discover and load connector and transform implementations at runtime, enabling modular architecture and easy extensibility.

## Advantages

- **Simplified Integration** eliminates the need to write custom producer/consumer applications for common integration patterns, reducing development time and complexity significantly
- **Operational Excellence** provides built-in fault tolerance, automatic restart capabilities, distributed scaling, and comprehensive monitoring through standardized metrics and REST API endpoints
- **Schema Evolution Support** integrates seamlessly with Confluent Schema Registry to handle schema changes, backward compatibility, and data format evolution automatically during data pipeline operations
- **Exactly-Once Semantics** ensures data consistency and prevents duplicates through connector-level transaction support and offset management, critical for financial and regulatory compliance scenarios
- **Enterprise Connector Ecosystem** offers hundreds of pre-built, tested, and supported connectors for popular databases, cloud services, and enterprise systems, eliminating custom integration development

## Disadvantages / Trade-offs

- **Limited Transformation Capabilities** in Single Message Transforms restrict complex data processing to simple field manipulations, requiring separate stream processing for advanced analytics or complex business logic
- **Memory and Resource Overhead** of the Connect framework and JVM can be significant compared to lightweight custom applications, especially for simple integration scenarios with minimal throughput requirements
- **Configuration Complexity** increases with advanced features like custom converters, transforms, and error handling policies, requiring deep understanding of Kafka internals and connector-specific configurations
- **Dependency Management** challenges arise with connector plugin versioning, compatibility matrices between Connect runtime, Kafka versions, and external system drivers or APIs
- **Debugging Difficulties** when troubleshooting distributed connector failures, offset management issues, or schema evolution problems across multiple worker nodes and external systems

## Corner Cases

- **Schema Evolution Conflicts** occur when source system schema changes are incompatible with existing Kafka topic schemas, potentially causing connector failures or data corruption requiring manual intervention
- **Offset Management Issues** arise during connector restarts, task rebalancing, or external system outages, potentially leading to data loss, duplication, or inconsistent state requiring manual offset reset procedures
- **External System Connectivity** problems during network partitions, database failovers, or cloud service outages can cause connector tasks to fail repeatedly, requiring careful retry and backoff configuration
- **Large Object Handling** for binary data, large JSON documents, or blob fields can exceed Kafka message size limits or cause memory pressure in Connect workers, requiring streaming or chunking strategies
- **Cross-Cluster Replication** scenarios with multiple Kafka clusters and shared external systems can create data consistency challenges, requiring careful coordination and conflict resolution strategies

## Limits / Boundaries

- **Maximum Message Size** limited by Kafka's message.max.bytes (default 1MB) affects large document or binary data ingestion, requiring chunking or external storage references for oversized content
- **Task Scalability** bounded by connector design and external system capabilities, with most connectors supporting 1-100 tasks per connector depending on source/sink partitioning capabilities
- **Memory Usage** per worker node typically requires 2-8GB heap depending on connector types, task count, and message throughput, with additional overhead for schema caching and connection pooling
- **Throughput Limitations** vary by connector implementation and external system performance, typically ranging from thousands to millions of records per second depending on record size and transformation complexity
- **Configuration Size** limits exist for connector configurations stored in internal topics, typically several KB per connector, affecting complex transformation chains and large property sets

## Default Values

- **Worker Memory** allocation defaults to 1GB heap (-Xmx1G) which is often insufficient for production workloads requiring 4-8GB for optimal performance
- **Task Maximum** defaults to 1 task per connector, requiring explicit configuration of tasks.max property to achieve parallelism and higher throughput
- **Offset Flush Interval** defaults to 60 seconds (offset.flush.interval.ms=60000) for committing processed offsets to internal topics, balancing performance and durability
- **Consumer Group ID** defaults to connect-[connector-name] for sink connectors, ensuring proper consumer group isolation and offset management
- **Connect Internal Topics** default to 25 partitions (config.storage.topic=__connect-config) and 3x replication factor for high availability of connector metadata

## Best Practices

- **Resource Planning** requires sizing worker nodes with sufficient memory (4-8GB heap), CPU cores (4-8), and network bandwidth based on expected throughput and connector count to prevent performance bottlenecks
- **Connector Configuration** should specify appropriate task count (tasks.max) based on source system partitioning capabilities and desired parallelism, typically 1 task per database table or file partition
- **Error Handling Strategy** must be configured with appropriate policies (errors.tolerance=all/none, errors.deadletterqueue.topic.name) and retry mechanisms to handle transient failures gracefully
- **Monitoring Implementation** should track connector status, task health, lag metrics, and error rates through JMX metrics, REST API monitoring, and integration with enterprise monitoring systems
- **Schema Management** requires careful coordination between source systems, Schema Registry, and target systems to handle schema evolution without breaking existing data pipelines or downstream consumers

---

## Additional Resources

### Connector Types Overview

**Source Connectors:**
- Import data from external systems → Kafka topics
- Examples: JDBC Source, Debezium CDC, File Source, S3 Source

**Sink Connectors:**
- Export data from Kafka topics → external systems  
- Examples: JDBC Sink, Elasticsearch Sink, S3 Sink, HDFS Sink

### Deployment Modes

**Standalone Mode:**
- Single worker process
- Configuration via properties files
- Suitable for development and simple deployments

**Distributed Mode:**
- Multiple worker processes in a cluster
- Configuration via REST API
- Automatic load balancing and fault tolerance
- Production-ready deployment model

### Common Enterprise Connectors

**JDBC Connectors:**
- Source: Database → Kafka (polling-based)
- Sink: Kafka → Database (batch inserts/updates)

**Debezium CDC:**
- Real-time change data capture
- Database transaction log streaming
- Supports MySQL, PostgreSQL, SQL Server, Oracle, MongoDB

**Cloud Storage:**
- S3, GCS, Azure Blob connectors
- Parquet, JSON, Avro format support
- Partitioning and time-based organization

### Single Message Transforms (SMTs)

**Built-in SMTs:**
- Cast: Type conversion
- ExtractField: Field extraction  
- InsertField: Add metadata fields
- MaskField: Data masking
- ReplaceField: Field renaming
- TimestampRouter: Topic routing based on timestamp

**Custom SMTs:**
- Implement Transform interface
- Lightweight transformation logic
- Applied during data flow without separate processing

---

*Last Updated: September 2025*
*Kafka Connect Version: 3.x compatible*