## Fulfill the requirements#
Our non-functional requirements for the proposed WhatsApp design are low latency, consistency, availability, and security. Let’s discuss how we have achieved these requirements in our system:

### Low latency: 
We can minimize the latency of the system at various levels:

We can do this through geographically distributed WebSocket servers and the cache associated with them.

We can use Redis cache clusters on top of MySQL database clusters.

We can use CDNs for frequently sharing documents and media content.

### Consistency: 
The system also provides high consistency in messages with the help of a FIFO messaging queue with strict ordering. However, the ordering of messages would require the Sequencer to provide ID with appropriate causality inference mechanisms to each message. For offline users, the Mnesia database stores messages in a queue. The messages are sent later in a sequence after the user goes online.

### Availability: 
The system can be made highly available if we have enough WebSocket servers and replicate data across multiple servers. When a user gets disconnected due to some fault in the WebSocket server, the session is re-created via a load balancer with a different server. Moreover, the messages are stored on the Mnesia cluster following the primary-secondary replication model, which provides high availability and durability.

### Security: 
The system also provides an end-to-end encryption mechanism that secures the chat between users.

![image](https://user-images.githubusercontent.com/33947539/200180968-cadf3b32-f69c-49eb-9a03-cd62ec584172.png)

