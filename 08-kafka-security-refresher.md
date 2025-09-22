# Kafka Security: Comprehensive Developer Refresher

This README.md covers authentication, authorization, and encryption in Apache Kafka, with simple explanations, Java examples, best practices, and visual aids.

---

## 8.1 Authentication

### Simple Explanation
Kafka authentication makes sure only trusted users, applications, or systems can connect to the Kafka cluster. Without authentication, anyone could produce or consume messages!

#### Problems It Solves
- Prevents unauthorized access
- Helps track and audit users
- Essential for secure, multi-tenant deployments

#### Why It Exists in Kafka
Kafka is often the backbone data pipeline for organizations, so access security is vital. Authentication helps ensure only approved users or apps interact with the system.

### Supported Mechanisms & How They Work
| Mechanism        | Protocol      | Description                                                            | Java Support              |
|------------------|--------------|------------------------------------------------------------------------|---------------------------|
| SSL/TLS          | SSL/TLS      | Certificate-based auth, encrypts connection.                           | Yes (key/trust store)     |
| SASL/PLAIN       | SASL over TLS| Username/password, needs SSL for safe transport.                       | Yes (jaas.conf + prop)    |
| SASL/SCRAM       | SASL over TLS| Advanced username/pass with challenge-response (SCRAM-SHA-256/512).     | Yes                       |
| SASL/GSSAPI      | Kerberos     | Kerberos-enabled, best for corporate AD, SSO.                           | Yes                       |
| SASL/OAUTHBEARER | OAuth 2.0    | OAuth tokens, best for cloud/modern SSO architectures.                  | Yes (with provider)       |

#### Where They’re Used
- SSL/TLS: Internal encryption and optional client certificates
- PLAIN/SCRAM: Most SaaS, cloud, and on-premise clusters
- Kerberos: Large enterprises, Active Directory
- OAuth: Centralized, cloud-native identity

---

### Java Example – Enabling SASL/SCRAM Authentication
```java
Properties props = new Properties();
props.put("bootstrap.servers", "kafka:9093");
props.put("security.protocol", "SASL_SSL");
props.put("sasl.mechanism", "SCRAM-SHA-512");
props.put("sasl.jaas.config",
    "org.apache.kafka.common.security.scram.ScramLoginModule required " +
    "username=\"user1\" password=\"pass1\";");
// Add SSL keystore info as needed
KafkaProducer<String, String> producer = new KafkaProducer<>(props);
```

---

### CLI Example: Enabling SSL for Broker (`server.properties`)
```properties
listeners=SSL://:9093
ssl.keystore.location=/etc/kafka/server.keystore.jks
ssl.keystore.password=changeit
ssl.truststore.location=/etc/kafka/server.truststore.jks
ssl.truststore.password=changeit
ssl.client.auth=required
```

### CLI Example: SASL/PLAIN User Setup (jaas.conf)
```
KafkaServer {
  org.apache.kafka.common.security.plain.PlainLoginModule required
  username="admin"
  password="adminpass"
  user_admin="adminpass"
  user_alice="alicepass";
};
```

### Internal Architecture Diagram
```
Client ↔ [SSL/TLS handshake] ↔ Broker
                     ↓
            [SASL Mechanism: PLAIN/SCRAM/Kerberos/OAuth]
                     ↓
  [Authentication: Username/Token/Principal validated]
```

### Common Pitfalls
- Not enabling SSL with SASL/PLAIN (credentials in plaintext!)
- Certificate or truststore misconfigurations (handshake failures)
- Missing/expired Kerberos tickets
- OAuth token clock skew or refresh errors

### Best Practices
- Prefer SCRAM or Kerberos over PLAIN for password-based auth
- Rotate passwords, tokens, and certificates regularly
- Automate certificate renewals (cron jobs/monitoring)

---

## 8.2 Authorization
### Simple Explanation
After authentication, authorization decides *what* each user can do—like read, write, or create topics.

#### What Problem It Solves
- Prevents data leaks or unwanted data modification
- Enforces least privilege
- Enables secure multi-team/multi-tenant use

#### Why Kafka Has ACLs/RBAC
Kafka needs fine-grained security, not just simple "yes/no" per user. ACLs and RBAC implement this flexibility.

### Architecture/Mechanisms
- **ACLs**: Specify what each user can do on which resource (topic, group, cluster).
- **RBAC (Role-based)**: Assign permissions to roles (admin/producer/consumer), then assign users to roles (supported in Confluent Platform).
- **Authorizer**: ZooKeeper-based or KRaft-based depending on Kafka mode.

| Feature         | Standard Kafka     | Confluent RBAC         |
|-----------------|-------------------|------------------------|
| ACL Pattern     | User/Resource/Op  | Role/Resource          |
| Management      | kafka-acls.sh     | confluent iam rbac     |
| Storage         | ZooKeeper or KRaft| Cluster Metadata       |

### Java Example: Setting Up Producer ACL
```shell
kafka-acls.sh --add --allow-principal User:alice --producer --topic secure_topic
```

### Java Example: Granting Read for Consumers
```shell
kafka-acls.sh --add --allow-principal User:bob --consumer --topic secure_topic --group group1
```

### ACL CLI Syntax
```
Principal P is [Allowed/Denied] Operation O From Host H on Resource R
```

### Best Practices
- Start with least privilege
- Use wildcards only if necessary
- Regularly audit/rotate ACLs
- Prefer group/role bindings for large teams

### Common Pitfalls
- Overpermissive ACLs (User:* = bad in prod)
- Not updating ACLs after employee/team changes
- Race conditions with ACL changes and client connections

### ASCII Diagram: ACL Evaluation
```
User:alice → [authenticate] → [ACLs checked] → [read/write topic] (or error)
User:bob   → [authenticate] → [ACLs checked] → denied (no permission)
```

---

## 8.3 Encryption (In-flight & At-rest)

### Simple Explanation
Encryption ensures that even if data is intercepted or disks stolen, the content cannot be read without the key.

#### Why Encrypt?
- Prevents eavesdropping on the network (in-flight)
- Secures data on brokers/disks (at-rest)
- Regulatory compliance (GDPR, PCI, etc.)

#### Kafka Mechanisms
- **In-Flight Encryption**: Enable SSL/TLS for all listeners/clients between brokers, producers, and consumers
- **At-Rest Encryption**: Not directly provided by Kafka; use disk, volume, or cloud KMS encryption

### Config: Enabling In-Flight Encryption
```properties
listeners=SSL://broker:9093
ssl.keystore.location=/path/keystore.jks
ssl.keystore.password=changeit
ssl.truststore.location=/path/truststore.jks
ssl.truststore.password=changeit
ssl.client.auth=required
```

### Example: At-Rest Encryption (Cloud KMS)
```json
{
  "EncryptionAtRest": {
    "DataVolumeKMSKeyId": "arn:aws:kms:..."
  },
  "EncryptionInTransit": {
    "InCluster": true,
    "ClientBroker": "TLS"
  }
}
```

### Architecture Diagram Reference
- See attached image: Kafka encryption at-rest KMS


### Java Example (Producer with SSL)
```java
props.put("security.protocol", "SSL");
props.put("ssl.keystore.location", "/path/to/keystore.jks");
props.put("ssl.keystore.password", "changeit");
props.put("ssl.truststore.location", "/path/to/truststore.jks");
props.put("ssl.truststore.password", "changeit");
KafkaProducer<?, ?> p = new KafkaProducer<>(props);
```

### Trade-offs Table
| Security Feature         | Pros                         | Cons                       |
|-------------------------|------------------------------|----------------------------|
| SSL/TLS in flight       | Data safe on wire, easy      | CPU overhead, cert mgmt    |
| At-rest (disk/KMS)      | Protects from disk theft     | Must manage keys, setup    |
| Client Auth SSL         | Full trust, mutual auth      | Cert lifecycle management  |

### Real-world Use Cases
- Multi-tenant SaaS (role, group, individual ACLs)
- Regulated industries (TLS, at-rest, audit logs)
- Public cloud: leverage cloud KMS for disk encryption
- Enterprise SSO integration (Kerberos, OAuth)

### Version Highlights Table
| Version    | Feature                       |
|------------|-------------------------------|
| 0.9        | SSL support (in-flight)       |
| 0.10       | SASL/PLAIN & SCRAM, ACLs      |
| 1.0–2.0    | OAuth & Kerberos improvements |
| 2.8+       | KRaft authorizer, RBAC        |
| 3.0+       | Cloud KMS integrations        |


---

## Visual Aids

### ASCII: Authentication/ACL Flow
```
Client ↔ SSL/TLS ↔ Broker
  |           |-- ACL AuthZ --|
 Producer/Consumer (SASL/OAuth/PLAIN) -> [Principal] → [ACL] → [Authorize?]
```

<!-- See attached architecture diagrams below for encryption at-rest-->

---

## Mindmaps & Links
- [Kafka Security Deep Dive (Confluent)](https://developer.confluent.io/courses/security/)
- [Kafka CLI Reference](https://kafka.apache.org/documentation/#security_tools)
- [Kafka Security with AWS (Best Practices)](https://docs.aws.amazon.com/msk/latest/developerguide/msk-working-with-encryption.html)

---

# End of README