# chat-backend

## Functional Requirement

1. start group chats with multiple participants (limit 100).
2. send/receive messages.
3. receive messages sent while they are not online (up to 30 days).
4. send/receive media in their messages.

## NFR

1. highly available, eventual consistency
2. guarantee message deliverability
3. fault tolerant
4. msg delivery latency < 500ms
5. Messages should be stored on centralized servers no longer than necessary

## Entity
- User
- Chat
- Message
- Client

## API
WebSocket
1. -> create chat
2. -> modifyChatParticipants
3. -> send msg
4. -> create attachment
5. <- new msg
6. <- chat update

## High Level Design

1. start group chats with multiple participants (limit 100): TODO
2. send/receive messages: TODO
3. receive messages sent while they are not online (up to 30 days): TODO
4. send/receive media in their messages: TODO

## Deep Dive

