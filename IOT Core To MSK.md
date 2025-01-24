# AWS IoT Core to Amazon MSK Integration

This document provides steps to configure AWS IoT Core to send IoT data to Amazon MSK (Managed Streaming for Apache Kafka) with partitioning from the `DT/Fryer` topic, which contains two partitions: **Single Vat** and **Double Vat**.. It also includes guidance on consuming messages from the Kafka topics.

---

## 1. **Data Structure**

### **Single Vat Example:**
```json
{
  "partition": "Single Vat",
  "sensors": {
    "temperature": {"sensor_id": "sensor_1", "value": 23.43, "timestamp": 1737352762506},
    "humidity": {"sensor_id": "sensor_2", "value": 50.21, "timestamp": 1737352762506},
    "electricity": {"sensor_id": "sensor_3", "value": 120.89, "timestamp": 1737352762506},
    "oil_level": {"sensor_id": "sensor_4", "value": 300.63, "timestamp": 1737352762506}
  }
}
```

### **Double Vat Example:**
```json
{
  "partition": "Double Vat",
  "sensors": {
    "temperature": {"sensor_id": "sensor_1", "value": 25.12, "timestamp": 1737352762506},
    "humidity": {"sensor_id": "sensor_2", "value": 55.80, "timestamp": 1737352762506},
    "electricity": {"sensor_id": "sensor_3", "value": 125.78, "timestamp": 1737352762506},
    "oil_level": {"sensor_id": "sensor_4", "value": 310.45, "timestamp": 1737352762506},
    "vibration": {"sensor_id": "sensor_5", "value": 0.45, "timestamp": 1737352762506}
  }
}
```

---


## 1. AWS IoT Core Rule Configuration

To send IoT data to MSK, create an IoT Core Rule with the following configuration:

### IoT Core Rule JSON
```json
{
  "sql": "SELECT * FROM 'DT/Fryer' WHERE partition = 'Single Vat'",
  "ruleDisabled": false,
  "actions": [
    {
      "kafka": {
        "destinationArn": "arn:aws:kafka:region:account-id:cluster/ClusterName/UUID",
        "topic": "iot-sensors-data",
        "key": "${partition}",
        "partition": "${partition}",
        "clientProperties": {
          "bootstrap.servers": "b-1.kafka-cluster.amazonaws.com:9092,b-2.kafka-cluster.amazonaws.com:9092",
          "sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule required username='USERNAME' password='PASSWORD';",
          "security.protocol": "SASL_SSL",
          "sasl.mechanism": "PLAIN"
        }
      }
    }
  ]
}
```

### Key Components
- **SQL Statement**: Filters messages to route only data with `partition = 'Single Vat'`.
- **Destination ARN**: Replace `region`, `account-id`, `ClusterName`, and `UUID` with your MSK cluster details.
- **Topic**: Set the Kafka topic to `iot-sensors-data` or your desired topic name.
- **Partitioning**: Uses the `partition` field from the message to determine the Kafka partition.
- **Client Properties**:
  - `bootstrap.servers`: Add your MSK brokers.
  - `sasl.jaas.config`: Provide your MSK credentials (username and password).
  - `security.protocol` and `sasl.mechanism`: Ensure secure Kafka connection.

---

## 2. Steps to Deploy
1. **Create the IoT Rule**:
   - Log in to the AWS Management Console.
   - Navigate to **IoT Core** > **Rules**.
   - Create a new rule and paste the above JSON into the rule's configuration.
2. **Prepare MSK**:
   - Ensure your Kafka topic (`iot-sensors-data`) is created.
   - Verify MSK cluster security settings and network access.
3. **Test the Rule**:
   - Send sample MQTT messages to the IoT topic `DT/Fryer` and verify that data appears in the Kafka topic.

---

## 3. Consuming Data from Kafka

To consume messages from the Kafka topic `iot-sensors-data`, use the following steps.

### Example Kafka Consumer Code (Python)

Install the required library:
```bash
pip install confluent-kafka
```

Create a Kafka consumer script:

```python
from confluent_kafka import Consumer, KafkaError

# Kafka consumer configuration
consumer_config = {
    'bootstrap.servers': 'b-1.kafka-cluster.amazonaws.com:9092,b-2.kafka-cluster.amazonaws.com:9092',
    'group.id': 'iot-consumer-group',
    'auto.offset.reset': 'earliest',
    'security.protocol': 'SASL_SSL',
    'sasl.mechanism': 'PLAIN',
    'sasl.username': 'USERNAME',
    'sasl.password': 'PASSWORD'
}

# Create a Kafka consumer
consumer = Consumer(consumer_config)

# Subscribe to the topic
consumer.subscribe(['iot-sensors-data'])

print("\n=== Kafka Consumer Initialized ===")
print("Subscribed to topic: iot-sensors-data\n")

print("Consuming messages from Kafka topic 'iot-sensors-data'...")

try:
    while True:
        msg = consumer.poll(1.0)  # Poll for messages

        if msg is None:
            continue

        if msg.error():
            if msg.error().code() == KafkaError._PARTITION_EOF:
                print(f"End of partition reached {msg.topic()} [{msg.partition()}] at offset {msg.offset()}")
            elif msg.error():
                print(f"Error: {msg.error()}")
            continue

        # Process the message
        print(f"\nReceived message:\nTopic: {msg.topic()}\nPartition: {msg.partition()}\nOffset: {msg.offset()}\nKey: {msg.key()}\nValue: {msg.value().decode('utf-8')}\n")

except KeyboardInterrupt:
    print("\nStopping consumer...")
finally:
    # Close the consumer
    consumer.close()
    print("\nConsumer closed.")
```

### Key Points
- **`group.id`**: Specify a unique consumer group ID.
- **`bootstrap.servers`**: Add your MSK brokers.
- **SASL Configuration**: Update `username` and `password` with your MSK credentials.
- **Topic Subscription**: Replace `iot-sensors-data` with your Kafka topic name if different.

### Run the Consumer
Execute the consumer script:
```bash
python kafka_consumer.py
```

---

## 4. Verifying Data Flow
1. Publish MQTT messages to the IoT Core topic `DT/Fryer`.
2. Verify that the messages are routed to the Kafka topic `iot-sensors-data`.
3. Run the Kafka consumer script to confirm that messages are being consumed.

---

## 5. Additional Notes
- Ensure proper IAM permissions for IoT Core to write to MSK.
- If using MSK with IAM authentication, adjust the `clientProperties` accordingly.
- Use monitoring tools like CloudWatch and MSK metrics to troubleshoot issues.

---

This setup enables real-time routing of IoT data from AWS IoT Core to MSK, ensuring seamless integration for downstream processing.
