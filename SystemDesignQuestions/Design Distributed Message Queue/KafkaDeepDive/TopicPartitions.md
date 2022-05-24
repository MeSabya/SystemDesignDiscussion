## Topic partitions

Essentially, a partition is an ordered sequence of messages. Producers continually append new messages to partitions. 
Kafka guarantees that all messages inside a partition are stored in the sequence they came in. 
**Ordering of messages is maintained at the partition level, not across the topic.**

- A unique sequence ID called an offset gets assigned to every message that enters a partition. These numerical offsets are used to identify every message’s sequential position within a topic’s partition.
- Offset sequences are unique only to each partition. This means, to locate a specific message, we need to know the Topic, Partition, and Offset number.
- Producers can choose to publish a message to any partition. If ordering within a partition is not needed, a round-robin partition strategy can be used, so records get distributed evenly across partitions.
- Placing each partition on separate Kafka brokers enables multiple consumers to read from a topic in parallel. That means, different consumers can concurrently read different partitions present on separate brokers.
- Placing each partition of a topic on a separate broker also enables a topic to hold more data than the capacity of one server.
- Messages once written to partitions are immutable and cannot be updated.
- A producer can add a ‘key’ to any message it publishes. Kafka guarantees that messages with the same key are written to the same partition.
- Each broker manages a set of partitions belonging to different topics.



