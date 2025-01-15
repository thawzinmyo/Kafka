# Kafka Connect with MySQL Connector using Debezium

This document describes how to configure Kafka Connect with a MySQL source connector using Debezium and connect it to a managed Kafka cluster. The configuration includes SSL settings and schema history management.

## Prerequisites

1. **Docker Installed**: Ensure Docker is installed and running.
2. **Kafka Cluster**: Access to a managed Kafka cluster (e.g., DigitalOcean Managed Kafka).
3. **MySQL Database**: Ensure the MySQL database is accessible.
4. **Certificates and Keys**: 
   - `ca-certificate.crt`
   - `user-access-certificate.crt`
   - `user-access-key.key`

## Steps to Set Up Kafka Connect and MySQL Connector

### 1. Prepare Keystore and Truststore

Convert your `.crt` and `.key` files to `.jks` format for Kafka.

```bash
# Create Truststore
keytool -importcert -file ca-certificate.crt -alias ca-cert \
 -keystore kafka.server.truststore.jks -storepass <truststore-password>

# Create Keystore
openssl pkcs12 -export -in user-access-certificate.crt -inkey user-access-key.key \
 -name user-cert -out kafka.server.keystore.p12 -password pass:<keystore-password>
```

### 2. Run Kafka Connect

Run the Kafka Connect container with the following configuration:

```bash
docker run --rm --name kafka-connect \
 -p 8083:8083 \
 -e BOOTSTRAP_SERVERS=********************************.m.db.ondigitalocean.com:25062 \
 -e GROUP_ID=connect-cluster \
 -e CONFIG_STORAGE_TOPIC=connect-configs \
 -e OFFSET_STORAGE_TOPIC=connect-offsets \
 -e STATUS_STORAGE_TOPIC=connect-statuses \
 -e CONNECT_SECURITY_PROTOCOL=SSL \
 -e CONNECT_SSL_MECHANISM=PKCS12 \
 -e CONNECT_SSL_TRUSTSTORE_LOCATION=/etc/kafka/secrets/user-client.truststore.p12 \
 -e CONNECT_SSL_TRUSTSTORE_PASSWORD=<truststore-password> \
 -e CONNECT_SSL_KEYSTORE_TYPE=PKCS12 \
 -e CONNECT_SSL_KEYSTORE_LOCATION=/etc/kafka/secrets/user-client.keystore.p12 \
 -e CONNECT_SSL_KEYSTORE_PASSWORD=<keystore-password> \
 -e CONNECT_SSL_KEY_PASSWORD=<key-password> \
 -e CONNECT_SSL_ENDPOINT_IDENTIFICATION_ALGORITHM=https \
 -e CONNECT_KEY_CONVERTER=org.apache.kafka.connect.json.JsonConverter \
 -e CONNECT_VALUE_CONVERTER=org.apache.kafka.connect.json.JsonConverter \
 -e CONNECT_KEY_CONVERTER_SCHEMAS_ENABLE=false \
 -e CONNECT_VALUE_CONVERTER_SCHEMAS_ENABLE=false \
 -v /path/to/secrets:/etc/kafka/secrets \
 quay.io/debezium/connect:latest
```

### 3. Configure the MySQL Connector

Use the following JSON configuration to set up the MySQL connector:

```json
{
  "name": "inventory-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "tasks.max": "1",
    "database.hostname": "*************************.h.db.ondigitalocean.com",
    "database.port": "25060",
    "database.user": "<your-db-user>",
    "database.password": "<your-db-password>",
    "database.server.id": "1",
    "database.server.name": "defaultdb",
    "database.include.list": "defaultdb",
    "topic.prefix": "inventory",
    "schema.history.internal.kafka.bootstrap.servers": "***************************.m.db.ondigitalocean.com:25062",
    "schema.history.internal.kafka.topic": "inventory.dbhistory",
    "schema.history.kafka.security.protocol": "SSL",
    "schema.history.kafka.sasl.mechanism": "PLAIN",
    "schema.history.kafka.sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule required username='<kafka-user>' password='<kafka-password>';",
    "schema.history.internal.producer.security.protocol": "SSL",
    "schema.history.internal.producer.ssl.endpoint.identification.algorithm": "https",
    "schema.history.internal.consumer.security.protocol": "SSL",
    "include.schema.changes": "false"
  }
}
```

Deploy the connector:

```bash
curl -i -X POST -H "Accept:application/json" \
 -H "Content-Type:application/json" \
 localhost:8083/connectors/ \
 -d @connector-config.json
```

## Lessons Learned

1. **SSL Configuration**:
   - Ensure the keystore and truststore files are correctly configured and accessible in the container.
   - Use `PKCS12` for compatibility if `.jks` is unavailable.

2. **Kafka Topics**:
   - Create required topics (`inventory.dbhistory`, etc.) before starting the connector.
   - Ensure topics have appropriate cleanup policies (`compact`).

3. **Error Handling**:
   - Compacted topics require messages with keys; configure connectors to provide appropriate keys.
   - Validate the schema history topic configuration thoroughly.

4. **Testing Connectivity**:
   - Use tools like `kcat` to test broker connectivity and verify SSL settings.

5. **Connector Configurations**:
   - Differentiate between producer and consumer configurations for finer control.
   - Include `schema.history` settings explicitly in the connector configuration.

6. **Debugging**:
   - Logs are critical. Check both Kafka Connect and MySQL logs for root cause analysis.
   - Validate JSON configurations with tools like `jq`.

## References
- [Debezium Documentation](https://debezium.io/documentation/)
- [Kafka Connect Documentation](https://kafka.apache.org/documentation/#connect)
- [DigitalOcean Managed Kafka](https://www.digitalocean.com/products/managed-kafka/)
