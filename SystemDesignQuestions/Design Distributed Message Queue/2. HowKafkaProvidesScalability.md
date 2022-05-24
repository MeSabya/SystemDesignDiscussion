## Partitions are the way that Kafka provides scalability

A Kafka cluster is made of one or more servers. In the Kafka universe, they are called Brokers. Each broker holds a subset of records that belongs to the entire cluster.

Kafka distributes the partitions of a particular topic across multiple brokers. By doing so, we’ll get the following benefits.

1. A topic will never get bigger than the biggest machine in the cluster. By spreading partitions across multiple brokers, a single topic can be scaled horizontally to provide performance far beyond a single broker’s ability.
2. A single topic can be consumed by multiple consumers in parallel. Serving all partitions from a single broker limits the number of consumers it can support. Partitions on multiple brokers enable more consumers.
3. Multiple instances of the same consumer can connect to partitions on different brokers, allowing very high message processing throughput. Each consumer instance will be served by one partition, ensuring that each record has a clear processing owner.
  
