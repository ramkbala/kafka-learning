# Spring Kafka Monitoring & Observability: Part 3 - Best Practices & Production Guide

Final part of the comprehensive Spring Kafka Monitoring & Observability guide covering comparisons, trade-offs, best practices, common pitfalls, and version highlights for production deployment.

---

## 📊 Comparisons & Trade-offs

### Monitoring Approaches Comparison

| Approach | Setup Complexity | Performance Impact | Observability Depth | Cost | Best For |
|----------|-----------------|-------------------|-------------------|------|----------|
| **Basic Metrics Only** | ⭐⭐⭐⭐⭐ Simple | ⭐⭐⭐⭐⭐ Minimal | ⭐⭐ Basic | 💰 Low | Small applications |
| **Micrometer + Prometheus** | ⭐⭐⭐⭐ Easy | ⭐⭐⭐⭐ Low | ⭐⭐⭐⭐ Good | 💰💰 Medium | **Production apps** |
| **Full Observability Stack** | ⭐⭐ Complex | ⭐⭐⭐ Medium | ⭐⭐⭐⭐⭐ Excellent | 💰💰💰 High | **Enterprise systems** |
| **Custom Monitoring** | ⭐ Very Complex | ⭐⭐ High | ⭐⭐⭐⭐⭐ Customizable | 💰💰💰💰 Very High | Specialized requirements |

### Health Check Strategies Comparison

| Strategy | Response Time | Accuracy | Resource Usage | Maintenance | Use Case |
|----------|--------------|----------|---------------|-------------|----------|
| **Simple Ping** | ⭐⭐⭐⭐⭐ <10ms | ⭐⭐ Limited | ⭐⭐⭐⭐⭐ Minimal | ⭐⭐⭐⭐⭐ Low | Load balancer checks |
| **Broker Connectivity** | ⭐⭐⭐⭐ <100ms | ⭐⭐⭐⭐ Good | ⭐⭐⭐⭐ Low | ⭐⭐⭐⭐ Medium | **Standard health checks** |
| **Deep Health Check** | ⭐⭐ <1000ms | ⭐⭐⭐⭐⭐ Excellent | ⭐⭐⭐ Medium | ⭐⭐⭐ High | Critical applications |
| **End-to-End Test** | ⭐ <5000ms | ⭐⭐⭐⭐⭐ Perfect | ⭐⭐ High | ⭐⭐ High | Mission-critical systems |

### Distributed Tracing Solutions Comparison

| Solution | Integration Effort | Performance Overhead | Feature Set | Ecosystem | Best For |
|----------|-------------------|---------------------|-------------|-----------|----------|
| **Spring Cloud Sleuth** | ⭐⭐⭐⭐⭐ Minimal | ⭐⭐⭐⭐ Low | ⭐⭐⭐ Good | ⭐⭐⭐⭐⭐ Spring | **Spring Boot apps** |
| **OpenTelemetry** | ⭐⭐⭐ Medium | ⭐⭐⭐ Medium | ⭐⭐⭐⭐⭐ Excellent | ⭐⭐⭐⭐⭐ Universal | **Multi-language systems** |
| **Jaeger** | ⭐⭐⭐⭐ Easy | ⭐⭐⭐⭐ Low | ⭐⭐⭐⭐ Very Good | ⭐⭐⭐⭐ Good | Microservices |
| **Zipkin** | ⭐⭐⭐⭐ Easy | ⭐⭐⭐⭐ Low | ⭐⭐⭐ Good | ⭐⭐⭐ Medium | Simple setups |

### Performance Impact Analysis

```
Monitoring Overhead Analysis (relative to baseline):

No Monitoring (Baseline):    ████████████████████████████████████████ 100%
Micrometer Only:             ████████████████████████████████████ 95%
Health Checks:               ███████████████████████████████████ 93%
Basic Tracing:               ██████████████████████████████ 85%
Full Observability:          ████████████████████████ 75%
Detailed Tracing:            ██████████████████████ 70%

Throughput Impact:
┌─────────────────┬─────────────┬─────────────┬─────────────┐
│ Monitoring      │ Throughput  │ Latency     │ Memory      │
├─────────────────┼─────────────┼─────────────┼─────────────┤
│ None            │ 100,000/sec │ 5ms         │ 512MB       │
│ Micrometer      │ 95,000/sec  │ 5.5ms       │ 530MB       │
│ Health Checks   │ 98,000/sec  │ 5.2ms       │ 520MB       │
│ Basic Tracing   │ 85,000/sec  │ 6ms         │ 580MB       │
│ Full Stack      │ 75,000/sec  │ 7ms         │ 650MB       │
└─────────────────┴─────────────┴─────────────┴─────────────┘
```

---

## 🚨 Common Pitfalls & Best Practices

### Critical Monitoring Anti-Patterns

#### ❌ Metrics Collection Mistakes

```java
// DON'T - Creating too many high-cardinality metrics
@Service
public class BadMetricsService {
    
    private final MeterRegistry meterRegistry;
    
    // BAD: High cardinality tags will overwhelm metrics system
    public void sendMessage(String topic, String userId, Object message) {
        meterRegistry.counter("kafka.messages.sent",
            Tags.of(
                "topic", topic,
                "user.id", userId,           // HIGH CARDINALITY!
                "message.content", message.toString(), // VERY HIGH CARDINALITY!
                "timestamp", String.valueOf(System.currentTimeMillis()) // INFINITE CARDINALITY!
            )).increment();
    }
    
    // BAD: Not using metric naming conventions
    meterRegistry.counter("kafkaMessagesSent").increment(); // Should be kafka.messages.sent
    
    // BAD: Creating metrics on every method call
    public void processMessage(String messageId) {
        Timer timer = Timer.builder("processing.time." + messageId) // Creates new metric per message!
            .register(meterRegistry);
        // This will create memory leaks
    }
}

// DON'T - Ignoring metric performance
@Component
public class BadHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        try {
            // BAD: Synchronous blocking operations in health check
            Thread.sleep(5000); // Blocks health check thread
            
            // BAD: Heavy operations without timeout
            adminClient.listTopics().names().get(); // No timeout!
            
            return Health.up().build();
            
        } catch (Exception e) {
            return Health.down().build();
        }
    }
}
```

#### ❌ Tracing Anti-Patterns

```java
// DON'T - Manual tracing without proper cleanup
@Service
public class BadTracingService {
    
    @Autowired
    private Tracer tracer;
    
    public void processMessage(String message) {
        
        // BAD: Creating spans without proper cleanup
        Span span = tracer.nextSpan().name("process.message").start();
        // No try-with-resources or finally block - span may leak!
        
        processBusinessLogic(message);
        
        // span.end() might not be called if exception occurs
    }
    
    // BAD: Adding too much high-cardinality data to spans
    public void processUserEvent(UserEvent event) {
        Span span = tracer.nextSpan().name("process.user.event")
            .tag("user.id", event.getUserId()) // Potentially high cardinality
            .tag("user.email", event.getEmail()) // PII data in traces!
            .tag("event.content", event.toString()) // Large data in span
            .start();
        
        try {
            // Processing logic
        } finally {
            span.end();
        }
    }
}
```

### Production Best Practices

#### ✅ Optimal Metrics Configuration

```java
/**
 * ✅ GOOD - Production-ready metrics configuration
 */
@Configuration
@lombok.extern.slf4j.Slf4j
public class ProductionMetricsConfiguration {
    
    @Bean
    public MeterRegistryCustomizer<PrometheusMeterRegistry> metricsCommonTags() {
        return registry -> registry.config()
            .commonTags(
                "application", "kafka-service",
                "environment", System.getProperty("env", "unknown"),
                "instance", InetAddress.getLocalHost().getHostName(),
                "version", getClass().getPackage().getImplementationVersion()
            )
            .meterFilter(
                // Filter out high-cardinality metrics
                MeterFilter.deny(id -> {
                    String name = id.getName();
                    return name.contains("user.id") || 
                           name.contains("message.content") ||
                           name.contains("timestamp");
                })
            )
            .meterFilter(
                // Limit histogram buckets for better performance
                MeterFilter.maximumExpectedValue("kafka.consumer.processing.time", Duration.ofSeconds(30))
            );
    }
    
    @Bean
    public KafkaTemplate<String, Object> productionKafkaTemplate(ProducerFactory<String, Object> producerFactory,
                                                                MeterRegistry meterRegistry) {
        
        KafkaTemplate<String, Object> template = new KafkaTemplate<>(producerFactory);
        
        // Enable observations with careful configuration
        template.setObservationEnabled(true);
        template.setObservationConvention(new ProductionKafkaTemplateObservationConvention());
        
        log.info("Configured production KafkaTemplate with optimized metrics");
        
        return template;
    }
    
    /**
     * ✅ GOOD - Low-cardinality observation convention
     */
    public static class ProductionKafkaTemplateObservationConvention implements KafkaTemplateObservationConvention {
        
        @Override
        public KeyValues getLowCardinalityKeyValues(KafkaRecordSenderContext context) {
            return KeyValues.of(
                "topic", context.getDestination(),
                "client.id", extractClientId(context),
                "message.type", getGeneralMessageType(context.getRecord().value())
            );
        }
        
        @Override
        public KeyValues getHighCardinalityKeyValues(KafkaRecordSenderContext context) {
            // Minimize high-cardinality tags
            return KeyValues.of(
                "partition", String.valueOf(context.getRecord().partition() != null ? 
                    context.getRecord().partition() : "unassigned")
            );
        }
        
        private String extractClientId(KafkaRecordSenderContext context) {
            // Extract client ID safely
            return context.getProducerConfig() != null ? 
                String.valueOf(context.getProducerConfig().get(ProducerConfig.CLIENT_ID_CONFIG)) : "unknown";
        }
        
        private String getGeneralMessageType(Object value) {
            // Group message types to reduce cardinality
            if (value == null) return "null";
            
            String className = value.getClass().getSimpleName();
            
            // Group similar message types
            if (className.contains("Event")) return "Event";
            if (className.contains("Command")) return "Command";
            if (className.contains("Query")) return "Query";
            
            return "Other";
        }
    }
    
    /**
     * ✅ GOOD - Metrics with proper sampling
     */
    @Service
    public static class OptimizedMetricsService {
        
        private final MeterRegistry meterRegistry;
        private final Random sampler = new Random();
        
        // Pre-create metrics for better performance
        private final Counter messagesProduced;
        private final Timer processingTimer;
        private final DistributionSummary messageSize;
        
        public OptimizedMetricsService(MeterRegistry meterRegistry) {
            this.meterRegistry = meterRegistry;
            
            this.messagesProduced = Counter.builder("kafka.messages.produced")
                .description("Total messages produced")
                .register(meterRegistry);
            
            this.processingTimer = Timer.builder("kafka.message.processing.time")
                .description("Message processing time")
                .minimumExpectedValue(Duration.ofMillis(1))
                .maximumExpectedValue(Duration.ofSeconds(30))
                .serviceLevelObjectives(
                    Duration.ofMillis(10),
                    Duration.ofMillis(50),
                    Duration.ofMillis(100),
                    Duration.ofMillis(500)
                )
                .register(meterRegistry);
            
            this.messageSize = DistributionSummary.builder("kafka.message.size")
                .description("Message size in bytes")
                .baseUnit("bytes")
                .minimumExpectedValue(1.0)
                .maximumExpectedValue(1_000_000.0)
                .serviceLevelObjectives(1_000, 10_000, 100_000)
                .register(meterRegistry);
        }
        
        public void recordMessageProduced(String topic, Object message) {
            
            // Use sampling for high-frequency events
            if (sampler.nextDouble() > 0.1) return; // Sample 10%
            
            messagesProduced.increment(Tags.of("topic", topic));
            messageSize.record(estimateMessageSize(message), Tags.of("topic", topic));
        }
        
        public void recordProcessingTime(String topic, Duration duration) {
            processingTimer.record(duration, Tags.of("topic", topic));
        }
        
        private double estimateMessageSize(Object message) {
            if (message == null) return 0;
            return message.toString().length() * 2; // Rough estimate
        }
    }
}

/**
 * ✅ GOOD - Robust health check implementation
 */
@Component
@lombok.extern.slf4j.Slf4j
public class ProductionKafkaHealthIndicator implements HealthIndicator {
    
    private final AdminClient adminClient;
    private final MeterRegistry meterRegistry;
    
    // Health check configuration
    private final long healthCheckTimeoutMs = 5000;
    private final int minRequiredBrokers = 1;
    private final Duration cacheTimeout = Duration.ofSeconds(30);
    
    // Cached health result to prevent excessive checks
    private volatile CachedHealth cachedHealth;
    
    // Health check metrics
    private final Counter healthCheckSuccess;
    private final Counter healthCheckFailure;
    private final Timer healthCheckDuration;
    
    public ProductionKafkaHealthIndicator(AdminClient adminClient, MeterRegistry meterRegistry) {
        this.adminClient = adminClient;
        this.meterRegistry = meterRegistry;
        
        this.healthCheckSuccess = Counter.builder("kafka.health.check.success")
            .register(meterRegistry);
        this.healthCheckFailure = Counter.builder("kafka.health.check.failure")
            .register(meterRegistry);
        this.healthCheckDuration = Timer.builder("kafka.health.check.duration")
            .register(meterRegistry);
    }
    
    @Override
    public Health health() {
        
        // Return cached result if still valid
        if (cachedHealth != null && cachedHealth.isValid()) {
            return cachedHealth.getHealth();
        }
        
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            Health health = performHealthCheck();
            cachedHealth = new CachedHealth(health, Instant.now(), cacheTimeout);
            
            healthCheckSuccess.increment();
            
            return health;
            
        } catch (Exception e) {
            healthCheckFailure.increment();
            
            log.error("Kafka health check failed", e);
            
            Health downHealth = Health.down()
                .withDetail("error", e.getMessage())
                .withDetail("timestamp", Instant.now())
                .build();
            
            // Cache failure result with shorter timeout
            cachedHealth = new CachedHealth(downHealth, Instant.now(), Duration.ofSeconds(10));
            
            return downHealth;
            
        } finally {
            sample.stop(healthCheckDuration);
        }
    }
    
    private Health performHealthCheck() throws Exception {
        
        // Use timeout to prevent blocking
        CompletableFuture<Health> healthFuture = CompletableFuture.supplyAsync(() -> {
            try {
                DescribeClusterResult clusterResult = adminClient.describeCluster();
                
                String clusterId = clusterResult.clusterId().get(healthCheckTimeoutMs, TimeUnit.MILLISECONDS);
                Collection<Node> nodes = clusterResult.nodes().get(healthCheckTimeoutMs, TimeUnit.MILLISECONDS);
                
                if (nodes.size() < minRequiredBrokers) {
                    return Health.down()
                        .withDetail("error", "Insufficient brokers")
                        .withDetail("available", nodes.size())
                        .withDetail("required", minRequiredBrokers)
                        .build();
                }
                
                return Health.up()
                    .withDetail("clusterId", clusterId)
                    .withDetail("brokers", nodes.size())
                    .withDetail("timestamp", Instant.now())
                    .build();
                
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        });
        
        return healthFuture.get(healthCheckTimeoutMs, TimeUnit.MILLISECONDS);
    }
    
    @lombok.Data
    @lombok.AllArgsConstructor
    private static class CachedHealth {
        private Health health;
        private Instant timestamp;
        private Duration timeout;
        
        public boolean isValid() {
            return Instant.now().isBefore(timestamp.plus(timeout));
        }
    }
}

/**
 * ✅ GOOD - Proper distributed tracing implementation
 */
@Service
@lombok.extern.slf4j.Slf4j
public class ProductionTracingService {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @Autowired
    private Tracer tracer;
    
    /**
     * ✅ GOOD - Proper span lifecycle management
     */
    public void sendTracedMessage(String topic, String key, Object message) {
        
        Span span = tracer.nextSpan()
            .name("kafka.message.send")
            .tag("messaging.system", "kafka")
            .tag("messaging.destination", topic)
            .tag("messaging.operation", "send")
            .start();
        
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
            
            log.info("Sending traced message: topic={}, key={}", topic, key);
            
            // Add safe, low-cardinality tags
            span.tag("message.type", getMessageType(message));
            span.tag("client.id", "kafka-service");
            
            // Perform the send operation
            ListenableFuture<SendResult<String, Object>> future = kafkaTemplate.send(topic, key, message);
            
            // Handle result asynchronously
            future.addCallback(
                result -> {
                    span.tag("kafka.partition", String.valueOf(result.getRecordMetadata().partition()));
                    span.tag("kafka.offset", String.valueOf(result.getRecordMetadata().offset()));
                    span.tag("success", "true");
                },
                failure -> {
                    span.tag("success", "false");
                    span.tag("error.type", failure.getClass().getSimpleName());
                    // Don't include full error message to avoid high cardinality
                }
            );
            
        } catch (Exception e) {
            span.tag("error", e.getClass().getSimpleName());
            throw e;
        } finally {
            span.end();
        }
    }
    
    /**
     * ✅ GOOD - Traced consumer with proper context propagation
     */
    @KafkaListener(topics = "traced-events", groupId = "production-consumer")
    public void consumeTracedMessage(@Payload ProductionEvent event,
                                   @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
                                   @Header(KafkaHeaders.OFFSET) long offset) {
        
        Span span = tracer.nextSpan()
            .name("kafka.message.process")
            .tag("messaging.system", "kafka")
            .tag("messaging.source", topic)
            .tag("messaging.operation", "process")
            .tag("kafka.offset", String.valueOf(offset))
            .tag("event.type", event.getEventType())
            .start();
        
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
            
            log.info("Processing traced event: eventId={}, topic={}, offset={}", 
                event.getEventId(), topic, offset);
            
            // Process with child spans if needed
            processEventWithTracing(event);
            
            span.tag("processing.successful", "true");
            
        } catch (Exception e) {
            span.tag("processing.successful", "false");
            span.tag("error.type", e.getClass().getSimpleName());
            throw e;
        } finally {
            span.end();
        }
    }
    
    @NewSpan("business.event.processing")
    private void processEventWithTracing(ProductionEvent event) {
        
        // Add span annotations for important business events
        if (tracer.currentSpan() != null) {
            tracer.currentSpan().annotate("processing.started");
        }
        
        try {
            // Simulate processing
            Thread.sleep(50);
            
            if (tracer.currentSpan() != null) {
                tracer.currentSpan().annotate("processing.completed");
            }
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            if (tracer.currentSpan() != null) {
                tracer.currentSpan().annotate("processing.interrupted");
            }
        }
    }
    
    private String getMessageType(Object message) {
        if (message == null) return "null";
        
        // Return general type to avoid high cardinality
        String className = message.getClass().getSimpleName();
        if (className.endsWith("Event")) return "Event";
        if (className.endsWith("Command")) return "Command";
        if (className.endsWith("Query")) return "Query";
        return "Message";
    }
}

// Supporting data classes
@lombok.Data
@lombok.AllArgsConstructor
@lombok.NoArgsConstructor
class ProductionEvent {
    private String eventId;
    private String eventType;
    private String userId;
    private Instant timestamp;
}
```

---

## 📈 Version Highlights

### Spring Kafka Monitoring Evolution

| Version | Release | Key Monitoring Features |
|---------|---------|------------------------|
| **Spring Kafka 3.3.x** | 2024 | **Enhanced Micrometer integration**, improved observation API |
| **Spring Kafka 3.2.x** | 2024 | **Native Micrometer support**, better health indicators |
| **Spring Kafka 3.1.x** | 2023 | **OpenTelemetry integration**, enhanced tracing capabilities |
| **Spring Kafka 3.0.x** | 2022 | **Observation API**, modern metrics framework |
| **Spring Kafka 2.9.x** | 2022 | **Micrometer 1.9 support**, improved consumer metrics |
| **Spring Kafka 2.8.x** | 2022 | **Enhanced health checks**, better error metrics |
| **Spring Kafka 2.7.x** | 2021 | **Producer/Consumer listeners**, JMX metrics improvements |
| **Spring Kafka 2.6.x** | 2021 | **Initial Micrometer integration**, basic observability |

### Spring Boot Actuator Timeline

**Spring Boot 3.x Series (2022-2025)**:
- **Enhanced Health Indicators**: More granular health checking capabilities
- **Improved Metrics Endpoints**: Better performance and customization options
- **Native Compilation Support**: GraalVM compatibility for health endpoints
- **OpenTelemetry Integration**: Built-in support for modern observability

**Spring Boot 2.x Series (2018-2022)**:
- **Actuator 2.0**: Complete redesign with endpoint customization
- **Micrometer Integration**: Vendor-neutral metrics collection
- **Health Indicator Improvements**: Conditional health checks and custom indicators
- **Security Enhancements**: Better endpoint security and access control

### Distributed Tracing Evolution

**OpenTelemetry Era (2020-2025)**:
- **Industry Standardization**: Unified observability framework
- **Better Performance**: Optimized data collection and export
- **Rich Instrumentation**: Automatic and manual instrumentation options
- **Cloud-Native Support**: Kubernetes and service mesh integration

**Spring Cloud Sleuth Era (2016-2022)**:
- **Spring Ecosystem Integration**: Seamless Spring Boot integration  
- **Zipkin Integration**: Built-in trace export capabilities
- **Sampling Strategies**: Configurable trace sampling for performance
- **Correlation ID Management**: Automatic trace context propagation

### Version Compatibility Matrix

```
Monitoring Stack Compatibility:

Spring Boot 3.2.x ────┬─── Spring Kafka 3.2.x
                      ├─── Micrometer 1.12.x
                      ├─── OpenTelemetry 1.32.x
                      └─── Actuator 3.2.x

Spring Boot 3.1.x ────┬─── Spring Kafka 3.1.x  
                      ├─── Micrometer 1.11.x
                      ├─── OpenTelemetry 1.29.x
                      └─── Actuator 3.1.x

Spring Boot 2.7.x ────┬─── Spring Kafka 2.9.x
                      ├─── Micrometer 1.9.x
                      ├─── Sleuth 3.1.x
                      └─── Actuator 2.7.x

Recommended Combinations:
✅ Spring Boot 3.2 + Spring Kafka 3.2 + OpenTelemetry
✅ Spring Boot 2.7 + Spring Kafka 2.9 + Sleuth
⚠️  Mixed versions may have compatibility issues
```

---

## 💡 Production Deployment Checklist

### Essential Monitoring Setup

#### **Metrics Configuration**
- ✅ **Configure low-cardinality tags** to prevent metric explosion
- ✅ **Set appropriate sampling rates** for high-frequency events
- ✅ **Use metric filtering** to exclude unnecessary metrics
- ✅ **Implement metric retention policies** for long-term storage
- ✅ **Set up alerting rules** for critical metrics
- ✅ **Configure dashboard templates** for visualization

#### **Health Check Configuration**
- ✅ **Set appropriate timeouts** for health checks (5-10 seconds)
- ✅ **Configure caching** to prevent excessive broker connections
- ✅ **Implement circuit breakers** for external dependencies
- ✅ **Set up health check endpoints** for different environments
- ✅ **Configure health check logging** for debugging
- ✅ **Integrate with load balancers** and orchestrators

#### **Distributed Tracing Setup**
- ✅ **Configure sampling rates** (1% for high-traffic, 100% for debugging)
- ✅ **Set up trace export** to appropriate backend (Jaeger, Zipkin)
- ✅ **Implement trace correlation** across service boundaries
- ✅ **Configure span duration limits** to prevent memory issues
- ✅ **Set up trace-based alerting** for performance issues
- ✅ **Implement trace context propagation** in async operations

### Performance Considerations

#### **Resource Planning**
- **CPU Overhead**: 2-5% additional CPU usage for full observability stack
- **Memory Overhead**: 50-100MB additional memory for metrics and tracing
- **Network Overhead**: 1-3% additional network traffic for telemetry data
- **Storage Requirements**: Plan for metric and trace data retention policies

#### **Optimization Strategies**
- **Batch metric collection** to reduce overhead
- **Use asynchronous tracing** to minimize latency impact
- **Implement metric aggregation** at application level
- **Configure appropriate buffer sizes** for trace and metric export
- **Use sampling strategies** to balance observability and performance

### Operational Excellence

#### **Alerting Strategy**
```yaml
# Example alerting rules
groups:
  - name: kafka.rules
    rules:
      - alert: KafkaHighConsumerLag
        expr: kafka_consumer_lag_sum > 10000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High consumer lag detected"
          description: "Consumer lag is {{ $value }} messages"
      
      - alert: KafkaProducerErrors
        expr: rate(kafka_producer_errors_total[5m]) > 0.1
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High producer error rate"
          description: "Producer error rate is {{ $value }} errors/sec"
      
      - alert: KafkaHealthDown
        expr: kafka_health_status != 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Kafka health check failing"
          description: "Kafka health indicator reports DOWN status"
```

#### **Dashboard Examples**
- **Producer Dashboard**: Message rates, batch sizes, error rates, latency percentiles
- **Consumer Dashboard**: Consumption rates, lag monitoring, processing times, rebalancing events
- **Health Dashboard**: Broker connectivity, cluster status, partition leadership
- **Business Dashboard**: Domain-specific metrics, SLA compliance, user impact

### Security and Compliance

#### **Sensitive Data Handling**
- ✅ **Avoid logging PII** in metrics and traces
- ✅ **Implement data masking** for sensitive message content
- ✅ **Use secure transport** for telemetry data (TLS)
- ✅ **Configure access controls** for monitoring endpoints
- ✅ **Implement audit logging** for configuration changes
- ✅ **Set up data retention policies** for compliance

#### **Best Practices Summary**
1. **Start Simple**: Begin with basic metrics and health checks
2. **Incremental Enhancement**: Add tracing and advanced monitoring gradually
3. **Monitor the Monitors**: Ensure your monitoring systems are reliable
4. **Regular Review**: Periodically review and optimize monitoring configuration
5. **Documentation**: Maintain runbooks and troubleshooting guides
6. **Testing**: Validate monitoring in non-production environments first

**Last Updated**: September 2025  
**Spring Kafka Version Coverage**: 3.3.x  
**Spring Boot Version**: 3.2.x  
**Micrometer Version**: 1.12.x  
**OpenTelemetry Version**: 1.32.x

This comprehensive Spring Kafka Monitoring & Observability guide provides production-ready patterns for implementing comprehensive observability in Kafka applications, from basic metrics collection to advanced distributed tracing, ensuring operational excellence and proactive issue detection in distributed systems.

[670] [671] [672]