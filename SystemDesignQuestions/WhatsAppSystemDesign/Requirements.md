## Functional requirements

**Conversation**: The system should support one-on-one and group conversations between users.

**Acknowledgment**: The system should support message delivery acknowledgment, such as sent, delivered, and read.

**Sharing**: The system should support sharing of media files, such as images, videos, and audio.

**Chat storage**: The system must support the persistent storage of chat messages when a user is offline until the successful delivery of messages.

**Push notifications**: The system should be able to notify offline users of new messages once their status becomes online.

## Non-functional requirements

**Low latency**: Users should be able to receive messages with low latency.

**Consistency**: Messages should be delivered in the order they were sent. Moreover, users must see the same chat history on all of their devices.

**Availability**: The system should be highly available. However, the availability can be compromised in the interest of consistency.

**Security**: The system must be secure via end-to-end encryption. The end-to-end encryption ensures that only the two communicating parties can see the content of messages. Nobody in between, not even WhatsApp, should have access.

## Resource estimation

### Storage estimation
As there are more than 100 billion messages shared per day over WhatsApp, let’s estimate the storage capacity based on this figure. Assume that each message takes 100 Bytes on average. Moreover, the WhatsApp servers keep the messages only for 30 days. So, if the user doesn’t get connected to the server within these days, the messages will be permanently deleted from the server.

100 billion/day∗100 Bytes=10 TB/day

For 30 days, the storage capacity would become the following:

30∗10 TB/day=300 TB/month

### Bandwidth estimation
According to the storage capacity estimation, our service will get 10TB of data each day, giving us a bandwidth of 926 Mb/s.

10 TB/86400sec≈926Mb/s

![image](https://user-images.githubusercontent.com/33947539/200176507-dfe4dedf-c078-4b03-a2dc-91cb1c7e7e40.png)

### Number of servers estimation#
WhatsApp handles around 10 million connections on a single server.

Let’s move to the estimation of the number of servers:

No. of servers=Total connections per day/No. of connections per server=2 billion/10 million=200 servers

![image](https://user-images.githubusercontent.com/33947539/200176597-3e37fca3-0bec-49b4-af92-afdd1a0f43f0.png)

## Building blocks we will use#
The design of WhatsApp utilizes the following building blocks that have also been discussed in the initial chapters:

![image](https://user-images.githubusercontent.com/33947539/200176708-b0fb91eb-ab88-4fbd-8708-de6949592caa.png)
