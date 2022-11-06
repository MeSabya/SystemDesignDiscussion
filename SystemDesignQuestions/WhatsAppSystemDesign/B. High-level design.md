## High-level design

At an abstract level, the high-level design consists of a chat server responsible for communication between the sender and the receiver. When a user wants to send a message to another user, both connect to the chat server. Both users send their messages to the chat server. The chat server then sends the message to the other intended user and also stores the message in the database.

![image](https://user-images.githubusercontent.com/33947539/200177640-2c74c54d-5eab-429f-8e9a-8877dda9f3ac.png)

