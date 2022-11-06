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

