6 Things You Need to Know About Kafka Before Using it in a System Design Interview

I quite often ran into system design discussions in which people seemed to have the tendency to use Kafka as a magic box, as if drawing out that horizontal cylinder would just magically make problems go away. It may. After all, “80% of all fortune 100 trust and use Kafka” [1]. It’s such a versatile tool that I think we should give it the credit. Personally, I like it very much as well.

When it comes to a system design interview however, I think we should do better than the superficial name dropping. Knowing a specific product is good. But to succeed in a system design interview, we need to understand things on a more abstract level. Usually that’s what differentiates a senior engineer from an intermediate one. What problems are we solving? How does Kafka allow us to solve it? You could still mention Kafka. Nothing wrong with that at all. But you should be ready for the interviewer’s next question. He/she is going to poke to see if you really understand the design patterns that Kafka embodies, and why they are a good fit for the use cases.

So to help you better prepare, I’ve gathered 6 scenarios/areas that Kafka is commonly referenced in system design interviews. I’ll walk through them one by one. The goal is not to enumerate all the possible use case scenarios, but to build the depth in comprehension by working through a few typical ones. It’s also recommended that you’ve at least heard about Kafka before reading this blog post. While it could serve as an introduction, there are other more newbie-friendly resources online. Now without further ado, let’s dive in.

## 1. Async Processing and Decoupling

Kafka models a distributed messaging queue with message producers on one end and message consumers on the other. It’s a form of asynchronous processing. The producers need not wait for the messages to be consumed. That’s the first design pattern of Kafka we should recognize in a system design interview. It’d be awkward to use Kafka in a synchronous setting where the producers need to block-wait for the consumers’ responses.

In theory, we could achieve the same async effect if we have the producers send a RPC directly to the consumers, expecting only an ACK as the response; or have the consumers fetch directly via an endpoint exposed by the producers. The advantage of using kafka is that it decouples the producers and consumers so that they can be developed, deployed, and managed separately. Once a common message contract is agreed upon, the producers will keep generating the messages and send them to Kafka. Interested consumers will pull from Kafka to retrieve the messages. Producers and consumers don’t need to know about each other’s address. They both only talk to a logically centralized service — kafka. Neither do they need to care about each other’s capacity. They can be monitored and scaled separately. In a system design review, in addition to development, it’s often a bonus point to call out operation and maintenance, which many candidates neglect.

## 2. Persistent Message Store

Now that the producers and consumers are out of sync, it’s easy for the producers to create an excessive amount of messages that the consumers can not process in time. This is another design pattern we need to highlight for Kafka in a system design interview. It’s effectively a durable cache that buffers the unprocessed messages, providing a cushion for our system to handle bursty load or consumer failure. The message retention in Kafka is configurable, making it adaptive to a wide variety of requirements.

Kafka’s persistent message store is also highly efficient. It embraces a log-based structure and only appends messages to the end of a file. In case the interviewer questions its efficiency, load tests have shown that it can be as fast as the network [2]. In addition, Kafka employs a standardized binary message format for both communication and storage, which reduces the processing overhead, and often enables using the sendfile [3] system call to transfer bytes between the network and disk directly.

A desired side-effect of the log-based store is that it preserves ordering of the messages (see the fine print in the next section). The state of consumption can be captured in a simple offset variable that points to the next to-be-consumed message. Consumers advance the offset as they consume messages and can even rewind it to replay the history. This largely simplifies the retry logic, which provides a higher level of primitives for us to answer questions in the system design interview.

## 3. Message Routing and Load Sharding

Kafka supports topic-based message routing. Both producers’ and consumers’ interactions with Kafka pertain to specific topics. Topics logically separate and categorize messages. Kafka makes sure that the right messages are delivered to the consumers which subscribe to the respective topics. In a system design interview, Kafka topics can be used as a routing mechanism. For example, all the user click activities go to one topic and all the system logs go to another. It simplifies our system design diagrams because the upstream systems only need to talk to a unified messaging endpoint. Kafka takes care of multiplexing the messages to the appropriate downstream systems.

Kafka also supports partitions inside a topic. Producers send messages directly to the corresponding topic partitions. A message’s partition is determined by the message partition key. Messages in the same topic partition are stored together and in the same order as they’re sent in. Messages from one topic partition can only be consumed by one consumer instance at any given time. A consumer instance is allowed to consume messages from multiple topic partitions in parallel. If a consumer instance dies, a different one will need to stand in. This can be done manually or automatically via consumer groups. The concept of partition effectively shards the load inside a topic because different partitions inside a topic operate in parallel.

The combination of topic and partitions can also be used as a shuffling mechanism. This system design interview post [4] uses Kafka to organize and count streaming updates.

## 4. Replication and Resilience

So far, we’ve referred to Kafka as a centralized service. The interviewer may ask if that creates a single point of failure. It doesn’t. But in order to answer the question well, we need to know what safeguards Kafka has to defend itself in the event of failure.

The typical deployment of Kafka involves multiple machines. Clients are provided with multiple Kafka server addresses in configuration as a bootstrap, through which they’ll discover all the Kafka servers. Clients can switch to a different server if a particular one fails. All Kafka servers have the ability to provide clients with the latest metadata so that clients know which servers to talk to for their intended functionality and data requests.

Internally, Kafka uses Zookeeper to coordinate controller election and store information such as cluster membership, access control, and topic configs. Zookeeper itself is a distributed system that’s resilient to partial failures. Of course we’ll need to deploy Zookeeper in a distributed fashion. The naive single Zookeeper instance setup is not going to withstand failures.

Each topic partition is replicated across Kafka servers. One server will be the leader of that topic partition. It can also lead other topic partitions at the same time. All reads/writes of the topic partition go through the leader. A set of followers passively replicate the leader’s copy of the topic partition. The followers of this topic partition can be leaders of other topic partitions. Some number of the followers can be configured to run in sync mode, which means that a message is only committed when safely replicated in all sync followers. If the leader fails, a sync follower will pick up the duty.

The interviewer probably won’t ask you to explain the full solution to a distributed log replication problem, as it’s very complex and too domain specific. But if you do want to be fully prepared, you can check out this blog post series [5] that goes in depth about the area.

## 5. Client Failure and Message Delivery Semantics

System design interviewers love to ask about the failure scenarios. A producer could fail before or after the message is committed. It has no way of knowing but to retry, which generates duplicates if the message is already committed. To fence off duplicates, the producer includes a Kafka-assigned ID and a monotonically increasing sequence number when sending messages. Kafka rejects the message if there is already a committed message from the same producer (identified by the Kafka-assigned ID) that has an equal or higher sequence number. Obviously, it’s the producer’s responsibility to keep track of the ID and sequence number.

A consumer could fail after processing the message but before persisting the offset, in which case retry reprocesses the message. If it chooses to persist the offset first, it could fail after persisting the offset but before processing the message, in which case retry leads to a skipped message. So it looks like it’s either at-least-once or at-most-once. What about the widely acclaimed exactly-once? Well, it turns out exactly-once is only possible in a very limited scenario, i.e, the message processing and offset storage need to happen in the same transaction. The transaction can be a traditional database transaction that stores both the output of the message and the updated offset in the same commit. Kafka also has a transaction semantics in publishing to multiple topics, which allows consumers to store the output and offset atomically in two recipient Kafka topics. This blog post [6] has a more elaborate explanation about how Kafka transactions work, though it’s highly unlikely that the interviewer would require those specific details.

## 6. Scalability Characteristics

Another common Kafka gotcha in system design interviews is that people don’t pay attention to its scalability characteristics. Even though Kafka does not impose any hard limit on the number of topics and partitions, there are some internal constraints. Kafka stores the topics and partitions information in Zookeeper. Zookeeper’s availability can be enhanced by adding more instances, but its capacity is bottlenecked by individual nodes. In addition, Kafka assigns one server to act as the controller to manage the topics and partitions metadata. The controller needs to keep track of the partition leaders, and handle leader changes. And when the controller itself fails, the cluster needs to elect a new controller and transfer the metadata management to the newly elected controller. The controller role is crucial in a Kafka cluster. Increasing the cardinality of topics and partitions leads to higher overhead that may overwhelm the controller. Another aspect to take into consideration is that each partition is a physical file folder, within which there are multiple data files and index files for various log segments. So there is also the filesystem overhead in managing a large number of partitions. Finally, don’t forget that all the partitions are replicated, which multiplies the overhead.

Thousands of topics and tens of thousands of partitions are definitely on the large end of the spectrum. The typical Kafka paradigm is fewer and larger topics with a reasonable amount of partitions. So the design of one Kafka topic per user and even one partition per user in a system design interview may be frowned upon. If you find yourselves heading to that rabbit hole, you may want to step back and consider whether a distributed key-value store like Cassendra is more appropriate.

References

[1] https://kafka.apache.org/

[2] https://engineering.linkedin.com/kafka/benchmarking-apache-kafka-2-million-writes-second-three-cheap-machines

[3] https://man7.org/linux/man-pages/man2/sendfile.2.html

[4] https://levelup.gitconnected.com/system-design-interview-distributed-top-k-frequent-elements-in-stream-2e92d63d777e

[5] https://levelup.gitconnected.com/raft-consensus-protocol-made-simpler-922c38675181

[6] https://www.confluent.io/blog/transactions-apache-kafka/



