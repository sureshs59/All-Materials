Writing Kafka Consumers
In this guide, we'll dive into the world of Kafka consumers, focusing on how to write them using the Java API with Gradle and Python. Kafka, a distributed streaming platform, enables you to process and analyze data in real-time. Consumers play a crucial role in reading data from Kafka topics. We'll explore consumer configurations, reading messages, and managing offsets to give you a comprehensive understanding of Kafka consumers.

Understanding Kafka Consumers
A Kafka consumer reads records from a Kafka cluster. Consumers subscribe to one or more topics and process the stream of records produced to them. In Kafka, the consumer is responsible for keeping track of the records it has processed—this is known as managing offsets.

Why Kafka Consumers?
Scalability: Kafka consumers can be scaled horizontally to read from topics in parallel.
Fault Tolerance: They support automatic offset commit capabilities, ensuring no data loss even in case of consumer failures.
Flexibility: Consumers can start reading from a specific offset, allowing for various processing strategies, such as reprocessing historical data.
Setting Up Your Environment
Before writing Kafka consumers, you need to set up your development environment. This involves installing Kafka and setting up a project using Gradle for Java or a suitable environment for Python.

Installing Kafka
Download and unzip Kafka from the official website. Start the Zookeeper and Kafka server:

# Start Zookeeper
bin/zookeeper-server-start.sh config/zookeeper.properties

# Start Kafka Server
bin/kafka-server-start.sh config/server.properties
Setting Up a Gradle Project for Java
Create a new directory for your project and initialize a Gradle project:

mkdir kafka-consumer-java
cd kafka-consumer-java
gradle init --type java-application
Add the Kafka clients dependency to your build.gradle file:

dependencies {
    implementation 'org.apache.kafka:kafka-clients:2.8.0'
}
Setting Up Python Environment
Ensure you have Python installed and create a new virtual environment:

python3 -m venv kafka-consumer-python
source kafka-consumer-python/bin/activate
Install the Kafka Python package:

pip install kafka-python
Writing a Simple Kafka Consumer in Java
Let's write a simple Kafka consumer that subscribes to a topic and prints the messages to the console.

Basic Consumer Configuration
Create a new Java class SimpleConsumer.java. Initialize the consumer with basic configurations:

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import java.util.Arrays;
import java.util.Properties;

public class SimpleConsumer {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "test-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
            consumer.subscribe(Arrays.asList("test-topic"));
            // Consumer logic goes here
        }
    }
}
Reading Messages
Inside the try block, add logic to poll for new messages and print them:

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        System.out.printf("offset = %d, key = %s, value = %s%n", record.offset(), record.key(), record.value());
    }
}
Writing a Simple Kafka Consumer in Python
Now, let's implement a similar consumer in Python.

Basic Consumer Configuration
Create a new Python script simple_consumer.py. Initialize the consumer with basic configurations:

from kafka import KafkaConsumer

consumer = KafkaConsumer(
    'test-topic',
    bootstrap_servers=['localhost:9092'],
    auto_offset_reset='earliest',
    group_id='test-group',
    value_deserializer=lambda x: x.decode('utf-8')
)
Reading Messages
Add logic to continuously read messages and print them:

for message in consumer:
    print(f"offset = {message.offset}, key = {message.key}, value = {message.value}")
Managing Offsets
Kafka consumers use offsets to keep track of the messages they have consumed. By default, offsets are committed automatically, but you can also manage them manually for finer control.

Automatic Offset Committing
Both the Java and Python consumers above use automatic offset committing. This behavior is controlled by the enable.auto.commit configuration (set to true by default).

Manual Offset Committing
To manually commit offsets in Java, set enable.auto.commit to false and use the commitSync method:

props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
// Inside the polling loop
consumer.commitSync();
In Python, use the commit method:

consumer = KafkaConsumer(enable_auto_commit=False)
# Inside the message loop
consumer.commit()
Conclusion
Writing Kafka consumers is a fundamental skill for working with real-time data streams. By understanding consumer configurations, reading messages, and managing offsets, you can build robust applications that process data efficiently. Whether using Java with Gradle or Python, Kafka provides the flexibility and power to meet your streaming needs.


Quiz;

Q: What is the primary role of a Kafka consumer in a Kafka ecosystem?
   To read and process messages from kafka topics
   
Q:    What is a Kafka Consumer Group used for?
   To allow multiple consumers to read messages from a topic in parallel without overlapping
   
Q:   What is the first step in creating a Kafka consumer in Java?
    Configure consumer properties such as bootstrap servers and group Id
	
	Which of the following is a required configuration property when creating a Kafka consumer in Java?
         Group.id
		 
Q: After configuring the properties, which class is used to instantiate a Kafka consumer in Java?		 
     KafkaConsumer
	 
	After configuring the properties, which class is used to instantiate a Kafka consumer in Java? 
	
	Subscribe()
	
Q; 	After configuring the properties, which class is used to instantiate a Kafka consumer in Java?
     Poll()
 Q; After calling the poll() method, what data structure is returned to the consumer containing the records?
 
      ConsumerRecords
	  
Q: 	  To commit the offsets of processed messages manually, which method should be used?
  commitsync()
  
 Q;   
     