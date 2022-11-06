- How is a communication channel created between clients and servers?

- How can the high-level design be scaled to support billions of users?

- How is the user’s data stored?

- How is the receiver identified to whom the message is delivered?

## Connection with a WebSocket server#
In WhatsApp, each active device is connected with a WebSocket server via WebSocket protocol. A WebSocket server keeps the connection open with all the active (online) users. Since one server isn’t enough to handle billions of devices, there should be enough servers to handle billions of users. The responsibility of each of these servers is to provide a port to every online user. The mapping between servers, ports, and users is stored in the WebSocket manager that resides on top of a cluster of the data store. In this case, that’s Redis.

![image](https://user-images.githubusercontent.com/33947539/200178357-46aa4a3b-9832-441f-9650-c239c2a47c49.png)

### Why is WebSocket preferred over HTTP(S) protocol for client-server communication?
HTTP(S) doesn’t keep the connection open for the servers to send frequent data to a client. With HTTP(S) protocol, a client constantly requests updates from the server, commonly called polling, which is resource intensive and causes latency. WebSocket maintains a persistent connection between the client and a server. This protocol transfers data to the client immediately whenever it becomes available. It provides a bidirectional connection used as a common solution to send asynchronous updates from a server to a client.

## Send or receive messages#
The WebSocket manager is responsible for maintaining a mapping between an active user and a port assigned to the user. Whenever a user is connected to another WebSocket server, this information will be updated in the data store.

A WebSocket server also communicates with another service called message service. Message service is a repository of messages on top of the Mnesia database cluster. The message service exposes APIs to receive messages by various filters, such as user ID, message ID, and so on.

![image](https://user-images.githubusercontent.com/33947539/200179231-e209f27b-84d6-4dd7-bd5b-89e6a4d9305e.png)

Now, let’s assume that user A wants to send a message to user B. As shown in the above figure, both users are connected to different WebSocket servers. The system performs the following steps to send messages from user A to user B:

User A communicates with the corresponding WebSocket server to which it is connected.

The WebSocket server associated with user A identifies the WebSocket to which user B is connected via the WebSocket manager. If user B is online, the WebSocket manager responds back to user A’s WebSocket server that user B is connected with its WebSocket server.

Simultaneously, the WebSocket server sends the message to the message service and is stored in the Mnesia database where it gets processed in the first-in-first-out order. As soon as these messages are delivered to the receiver, they are deleted from the database.

Now, user A’s WebSocket server has the information that user B is connected with its own WebSocket server. The communication between user A and user B gets started via their WebSocket servers.

If user B is offline, messages are kept in the Mnesia database. Whenever they become online, all the messages intended for user B are delivered via push notification. Otherwise, these messages are deleted permanently after 30 days.

Both users (sender and receiver) communicate with the WebSocket manager to find each other’s WebSocket server. Consider a case where there can be a continuous conversation between both users. This way, many calls are made to the WebSocket manager. To minimize the latency and reduce the number of these calls to the WebSocket manager, each WebSocket server caches the following information:

If both users are connected to the same server, the call to the WebSocket manager is avoided.

It caches information of recent conversations about which user is connected to which WebSocket server.

## Send or receive media files#

So far, we’ve discussed the communication of text messages. But what happens when a user sends media files? Usually, the WebSocket servers are lightweight and don’t support heavy logic such as handling the sending and receiving of media files. We have another service called the asset service, which is responsible for sending and receiving media files.

Moreover, the sending of media files consists of the following steps:

The media file is compressed and encrypted on the device side.

The compressed and encrypted file is sent to the asset service to store the file on blob storage. The asset service assigns an ID that’s communicated with the sender. The asset service also maintains a hash for each file to avoid duplication of content on the blob storage. For example, if a user wants to upload an image that’s already there in the blob storage, the image won’t be uploaded. Instead, the same ID is forwarded to the receiver.

The asset service sends the ID of media files to the receiver via the message service. The receiver downloads the media file from the blob storage using the ID.

The content is loaded onto a CDN if the asset service receives a large number of requests for some particular content.

The following figure demonstrates the components involved in sharing media files over WhatsApp messenger:

![image](https://user-images.githubusercontent.com/33947539/200179351-441aea61-b6da-4d7d-a92c-1c0df503de0b.png)

## Support for group messages#
WebSocket servers don’t keep track of groups because they only track active users. However, some users could be online and others could be offline in a group. For group messages, the following three main components are responsible for delivering messages to each user in a group:

- Group message handler
- Group message service
- Kafka

Let’s assume that user A wants to send a message to a group with some unique ID—for example, Group/A. The following steps explain the flow of a message sent to a group:

Since user A is connected to a WebSocket server, it sends a message to the message service intended for Group/A.

The message service sends the message to Kafka with other specific information about the group. The message is saved there for further processing. In Kafka terminology, a group can be a topic, and the senders and receivers can be producers and consumers, respectively.

Now, here comes the responsibility of the group service. The group service keeps all information about users in each group in the system. It has all the information about each group, including user IDs, group ID, status, group icon, number of users, and so on. This service resides on top of the MySQL database cluster, with multiple secondary replicas distributed geographically. A Redis cache server also exists to cache data from the MySQL servers. Both geographically distributed replicas and Redis cache aid in reducing latency.

The group message handler communicates with the group service to retrieve data of Group/A users.

In the last step, the group message handler follows the same process as a WebSocket server and delivers the message to each user.

![Uploading image.png…]()
