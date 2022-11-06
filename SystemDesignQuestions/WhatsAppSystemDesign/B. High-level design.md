## High-level design

At an abstract level, the high-level design consists of a chat server responsible for communication between the sender and the receiver. When a user wants to send a message to another user, both connect to the chat server. Both users send their messages to the chat server. The chat server then sends the message to the other intended user and also stores the message in the database.

![image](https://user-images.githubusercontent.com/33947539/200177640-2c74c54d-5eab-429f-8e9a-8877dda9f3ac.png)

**The following steps describe the communication between both clients:**

- User A and user B create a communication channel with the chat server.
- User A sends a message to the chat server.
- Upon receiving the message, the chat server acknowledges back to user A.
- The chat server sends the message to user B and stores the message in the database if the receiver’s status is offline.
- User B sends an acknowledgment to the chat server.
- The chat server notifies user A that the message has been successfully delivered.
- When user B reads the message, the application notifies the chat server.
- The chat server notifies user A that user B has read the message.

## API design#
WhatsApp provides a vast amount of features to its users via different APIs. Some features are mentioned below:

- Send message
- Get message or receive message
- Upload a media file or document
- Download document or media file
- Send a location
- Send a contact
- Create a status

