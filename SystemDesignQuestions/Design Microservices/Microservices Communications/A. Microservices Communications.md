## Microservices Communications
When we are talking about Monolithic applications, we said that the communication in Monolithic applications are inter-process communication. So that means it is working on single process that invoke one to another by using method calls. You just create class and call the method inside of target module. All running the same process.

One of the biggest challenge when moving to microservices-based application is changing the communication mechanism. Because microservices are distributed and microservices communicate with each other by inter-service communication on network level. Each microservice has its own instance and process. Therefore, services must interact using an inter-service communication protocols like HTTP, gRPC or message brokers AMQP protocol.

## Microservices Communication Types — Sync or Async Communication

### What is Synchronous communication ?
Basically, we can say that Synchronous communication is using HTTP or gRPC protocol for returning sync response. The client sends a request and waits for a response from the service. So that means client code block their thread, until the response reach from the server.

### What is Asynchronous communication ?
Basically, In Asynchronous communication, the client sends a request but it doesn’t wait for a response from the service. So the key point here is that, the client should not have blocked a thread while waiting for a response.

The most popular protocol for this Asynchronous communications is AMQP (Advanced Message Queuing Protocol). So with using AMQP protocols, the client sends the message with using message broker systems like Kafka and RabbitMQ queue. The message producer usually does not wait for a response. This message consume from the subscriber systems in async way, and no one waiting for response suddenly.

### Microservices Synchronous Communication and Practices

**There are 2 type of APIs when designing sync communication in microservices architecture.**

👉 1- Public APIs which is APIs calls from the client applications.

👉 2- Backend APIs which is used for inter-service communication between backend microservices.

For Public APIs, should be align with client request. Clients can be web browser or mobile application requests. 
So that means the public API should use RESTful APIs over HTTP protocol. 
So RESTful APIs should use JSON payloads for request-response, this will easy to check payloads and easy agreement with clients.

For the backend APIs, We need to consider network performance instead of easy readable JSON payloads. 
**Inter-service communication can result in a lot of network traffic.**
For that reason, serialization speed and payload size become more important. 
So for the backend APIs, These protocols support binary serialization should implement. 
The protocol alternatives is using **gRPC or other binary protocols are mandatory.**

***Let me compare the REST and gRPC protocols***

👉 REST is using HTTP protocol, and request-response structured JSON objects. API interfaces design based on HTTP verbs like GET-PUT-POST and DELETE.

👉 gRPC is basically Remote Procedure Call, that basically invoke external system method over the binary network protocols. Payloads are not readable but its faster that REST APIs.

So lets elaborate these 2 approaches.

- RESTful API Design over HTTP using JSON
- gRPC binary protocol API Design

#### RESTful API design for Microservices
In synchronous communication, when making request/response communication, we should use REST when designing our APIs. Its also called Restful APIs. REST approach is following the HTTP protocol, and implementing HTTP verbs like GET, POST, and PUT.

##### Features of REST
When we look at the constraints of the REST architecture, we come across six items:

- Stateless
- Uniform Interface
- Cacheable
- Client-Server
- Layered System
- Code on Demand

##### How we can design Restful APIs for microservices ?
The best practice is the resource URIs should be based on nouns (the resource) and not verbs.

```
For example :
https://eshop.com/orders // Correct

https://eshop.com/create-order // Wrong
```
#### grpc design for Microservices

##### gRPC 
It is focused on high performance and uses the HTTP/2 protocol to transport binary messages. It is relies on the Protocol Buffers language to define service contracts. Protocol Buffers, also known as Protobuf, allow you to define the interface to be used in service to service communication regardless of the programming language.

##### How gRPC works ?

In GRPC, a client application can directly call a method on a server application on a different machine like it were a local object, making it easy for you to build distributed applications and services.

![image](https://user-images.githubusercontent.com/33947539/192764480-43ec0147-0f67-4712-bee6-e751eae47b0b.png)

*As with many RPC systems, gRPC is based on the idea of defining a service that specifies methods that can be called remotely with their parameters and return types. On the server side, the server implements this interface and runs a gRPC server to handle client calls. On the client side, the client has a stub that provides the same methods as the server.*

##### Working with Protocol Buffers
gRPC uses Protocol Buffers by Default. Protocol Buffers are Google’s open source mechanism for serializing structured data.
When working with protocol buffers, the first step is to define the structure of the data you want to serialize in a proto file: this is an ordinary text file with the extension .proto. The protocol buffer data is structured as messages where each message is a small logical information record containing a series of name-value pairs called fields.

##### gRPC Method Types — RPC life cycles
gRPC lets you define four kinds of service method:

- **Unary RPCs**: where the client sends a single request to the server and returns a single response back, just like a normal function call.

- **Server streaming RPCs**:  where the client sends a request to the server and gets a stream to read a sequence of messages back.
- **Client streaming RPCs**:
 where the client writes a sequence of messages and sends them to the server, again using a provided stream. Once the client has finished writing the messages, it waits for the server to read them and return its response. Again gRPC guarantees message ordering within an individual RPC call.
- **Bidirectional streaming RPCs**: where both sides send a sequence of messages using a read-write stream.
 
##### Advantages of gRPC

General advantages of gRPC:

- Using HTTP / 2
These differences of HTTP / 2 provide 30–40% more performance. In addition, since gRPC uses binary serialization, it needs both more performance and less bandwidth than json serialization.

- Higher performance and less bandwidth usage than json with binary serialization
- Supporting a wide audience with multi-language / platform support
- Open Source and the powerful community behind it
- Supports Bi-directional Streaming operations
- Support SSL / TLS usage
- Supports many Authentication methods









