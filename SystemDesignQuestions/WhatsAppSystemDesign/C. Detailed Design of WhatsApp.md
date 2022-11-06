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

![Uploading image.png…]()

