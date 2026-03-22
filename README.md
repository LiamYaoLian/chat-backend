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

### 1. Create a Group Chat
POST `/v1/chats`

req header: `Authorization: Bearer <token>`

req body: `{ "name": "string", "participants": ["uid1", "uid2"] }` (max 100 participants including caller per functional requirements)

res body: `{ "chat_id": "uuid", "created_at": "iso8601" }`

### 2. Modify Chat Participants
PATCH `/v1/chats/{chat_id}/participants`

req header: `Authorization: Bearer <token>`

req body: `{ "add": ["uid"], "remove": ["uid"] }`

res body: `{ "chat_id": "uuid", "participants": ["uid", "..."] }`

### 3. List Chats for the Current User
GET `/v1/chats`

req header: `Authorization: Bearer <token>`

res body: `{ "chats": [{ "chat_id": "uuid", "name": "string", "participants": ["uid"], "updated_at": "iso8601" }] }`

### 4. Get Message History (catch-up / offline)
GET `/v1/chats/{chat_id}/messages?cursor=<opaque>&limit=50`

req header: `Authorization: Bearer <token>`

res body: `{ "messages": [{ "message_id": "uuid", "chat_id": "uuid", "sender_id": "uid", "content": "text", "attachments": [{ "attachment_id": "uuid", "content_type": "string", "byte_size": 0, "url": "https://..." }], "created_at": "iso8601" }], "next_cursor": "opaque|null" }` (server only returns messages within the retention window, e.g. last 30 days)

### 5. Create Attachment (register upload)

**5a. Small uploads (multipart, server stores or forwards the object)**

POST `/v1/attachments`

req header: `Authorization: Bearer <token>`, `Content-Type: multipart/form-data`

req body: multipart field `file` (and optional `content_type`)

req body: `{}` (or `{ "etag": "string" }` if the store returns one)

TODO Liam: presigned link

### WebSocket

Connect to `wss://<host>/v1/ws` with authentication (e.g. `?access_token=<jwt>`).

Client → server frames are JSON with a `type` and `payload` (shape below uses the same payload object for clarity).

* `-> CREATE_CHAT`: `{ "name": "string", "participants": ["uid1", "uid2"] }`
* `-> MODIFY_CHAT_PARTICIPANTS`: `{ "chat_id": "uuid", "add": ["uid"], "remove": ["uid"] }`
* `-> SEND_MSG`: `{ "chat_id": "uuid", "content": "text", "attachments": [{ "attachment_id": "uuid" }], "client_message_id": "uuid", "client_created_at": "iso8601" }` (`client_message_id` required for idempotency; `client_created_at` optional display hint — **authoritative time is server `created_at` on `<- NEW_MSG`**)
* `-> CREATE_ATTACHMENT`: `{ "filename": "string", "content_type": "string", "byte_size": 0 }` — server responds with upload instructions (e.g. presigned URL) TODO Liam: REST?

Server → client:

* `<- NEW_MSG`: `{ "message_id": "uuid", "chat_id": "uuid", "sender_id": "uid", "content": "text", "attachments": [{ "attachment_id": "uuid", "content_type": "string", "byte_size": 0, "url": "https://..." }], "created_at": "iso8601" }`
* `<- CHAT_UPDATE`: `{ "chat_id": "uuid", "change": "participants|metadata", "participants": ["uid"], "name": "string|null", "updated_at": "iso8601" }`

Ack / error pattern: for each `->` command, reply with `<- ACK` or `<- ERROR` whose payload includes `request_id` (if sent) and/or `client_message_id`, plus `error_code` / `message` on errors.

## High Level Design

**Services (logical):** **API service** (REST behind API Gateway or ALB), **Realtime service** (WebSocket, often separate ASG/Lambda + API Gateway WebSocket), **Attachment service** (can share the API service process). Shared **auth** validates JWT at the edge; business logic enforces “is participant.”

**Data:** **Amazon DynamoDB** for `Chat` (PK `chat_id`), `UserChat` (PK `user_id`, SK `chat_id`) for `GET /v1/chats`, and `Message` (PK `chat_id`, SK `created_at#message_id`) for history and retention scans. **Why DynamoDB:** single-digit-ms reads at scale, multi-AZ HA, TTL on `Message` for automatic ~30-day expiry (fits NFR 1/3/5). **Trade-off:** you must model access patterns up front (GSIs for rare queries cost more); no rich ad-hoc SQL.

**Blobs:** **Amazon S3** for bytes; **POST /v1/attachments** (or presigned session) stores metadata in DynamoDB (`Attachment` keyed by `attachment_id`, owner, size, S3 key). **Why S3:** cheap durable object storage; API servers stay stateless. **Trade-off:** extra hops vs uploading straight to DB (not viable for media scale).

**Realtime fan-out:** **`wss://…/v1/ws`** → **Realtime**; **`SEND_MSG`** conditional-writes **Message** in **DynamoDB** (idempotent on `client_message_id`), then publishes **NewMessage**. **Option A — SNS → SQS:** durable if a worker dies (NFR 2); **trade-off:** extra latency vs memory. **Option B — Redis (ElastiCache) Pub/Sub:** fastest fan-out to connection servers (NFR 4); **trade-off:** ephemeral—rely on DynamoDB as truth and optional SQS for backlog. **Kafka / Kinesis:** per-`chat_id` ordering and replay; **trade-off:** more ops than SQS.

**Flows (client → service → storage):**

1. **Group chat (≤100 participants):** `POST /v1/chats` → **API** validates body, writes `Chat` + transactional `UserChat` rows for each member in **DynamoDB**, returns `chat_id`. `PATCH …/participants` updates `Chat` membership and `UserChat` adds/deletes; emits `<- CHAT_UPDATE` via the same bus as below.

2. **Send/receive (online):** `SEND_MSG` on **Realtime** → write **Message** to **DynamoDB** → publish **NewMessage** event → **Realtime** workers consume and push `<- NEW_MSG` only to sockets whose `user_id` is in that chat (membership read from DynamoDB cache or `UserChat`).

3. **Offline / catch-up:** `GET /v1/chats/{id}/messages` → **API** queries **DynamoDB** `Message` by `chat_id` (cursor on SK), returns only rows whose TTL has not expired (retention policy).

4. **Media:** `POST /v1/attachments` (multipart or presigned) → **API** writes object to **S3**, row to **DynamoDB**; `SEND_MSG` references `attachment_id`; `<- NEW_MSG` includes signed **S3** URL or CDN URL.

## Deep Dive

