# Design Chat System

## Overview

- **Definition** — A real-time messaging system that enables 1:1 and group communication with presence indicators, delivery guarantees, and media sharing
- **Why It Exists** — Users expect instant, reliable communication across devices with ordering guarantees, read receipts, and offline message delivery
- **Historical Context** — IRC (1988) pioneered chat rooms; XMPP (1999) standardized federated messaging; WhatsApp (2009) and Slack (2013) popularized modern mobile-first chat at global scale
- **Key Concepts** — **WebSocket** for persistent bidirectional communication; **Message ordering** via sequence IDs; **Presence** through heartbeats and last-seen timestamps; **Fan-out** for group message delivery; **Delivery guarantees** (at-least-once, exactly-once)

## Core Concepts

- WebSocket provides persistent bidirectional communication for real-time messaging
  - Full-duplex after initial HTTP upgrade handshake
  - Server pushes messages to clients without polling
- HTTP long polling serves as fallback when WebSocket is unavailable
  - Client sends a request, server holds the connection open until a message arrives or timeout occurs
  - Higher latency and overhead compared to WebSocket
  - Use Server-Sent Events (SSE) as an alternative for one-way server-to-client streaming
- Message storage uses conversation-based partitioning for scalable writes
  - Partition key = conversation ID, sort key = timestamp or sequence ID
  - Cassandra: wide-row per conversation with clustering order
  - PostgreSQL: conversation_id + created_at composite index
- Sequence IDs combine timestamp, shard ID, and monotonic counter for global ordering
  - Example: 64-bit ID = 41-bit millisecond timestamp + 13-bit shard ID + 10-bit sequence
  - Within a conversation, messages are sorted by sequence ID
  - Wall-clock time can skew — use hybrid logical clocks (HLC) or Lamport timestamps for consistency
- Presence tracking uses heartbeats sent every few seconds from the client
  - Server tracks last heartbeat timestamp per user
  - If no heartbeat within N seconds (e.g., 30s), user is marked offline
  - Last-seen timestamp is shown when the user is offline
- Read receipts mark messages as delivered and read with per-message status flags
  - States: sent, delivered, read
  - Updating read receipts for every message in a busy group chat can cause write amplification
  - Optimize by sending read receipts as periodic aggregates ("read up to message X")
- Group chat requires fan-out architecture
  - Sender writes once to the group timeline partition
  - Option A: fan-out on write — replicate message to each recipient's inbox (WhatsApp approach for small groups)
  - Option B: fan-out on read — pull from group timeline on login (Slack approach for large channels)
- Media messages (images, videos, documents) upload to blob storage with CDN delivery
  - Client uploads directly to S3/GCS via presigned URL
  - Only the media URL and thumbnail are sent through the chat service
  - CDN caches media at edge locations for low-latency delivery
- Push notifications use APNS (iOS) and FCM (Android)
  - When the user is offline, the server sends a push notification via platform-specific services
  - Group push notifications should be batched to avoid flooding the push provider

## Common Mistakes

- **Relying on client timestamps for message ordering**
  - Client clocks are unreliable — a user changing time zones or having clock drift can cause messages to appear out of order
  - **Why it looks correct:** The client timestamp is the most natural ordering signal available
  - Use server-assigned sequence IDs. The server assigns a monotonically increasing ID per conversation when it receives the message. Clients display messages sorted by sequence ID, not the client timestamp.
- **Fan-out on write for extremely large groups (100K+)**
  - Writing a copy of each message to every recipient's inbox creates massive write amplification
  - **Why it looks correct:** Fan-out on write gives O(1) read cost, which seems ideal for performance
  - Use hybrid approach: fan-out on write for small groups (<1000 members), fan-out on read for large channels. Maintain a "last read sequence" pointer per user in large channels.
- **Not handling duplicate messages from retries**
  - When the client retries a failed send, the server may process the same message twice
  - **Why it looks correct:** TCP guarantees delivery, so duplicates seem impossible
  - Implement idempotency keys: the client sends a unique message ID; the server deduplicates by ID within a time window (Redis SET NX with TTL)

## Real-World Scenarios

### WhatsApp Architecture — Fan-out on Write

- Each message is written to the recipient's inbox (a per-user message queue)
- Optimized for small groups (most WhatsApp groups are <100 members)
- E2E encrypted — server cannot read message content
- Media uploaded to blob storage with encryption, only thumbnail passed through chat

### Slack Architecture — Fan-out on Read

- Messages stored in a single channel timeline
- Each user tracks their last-read sequence number
- On client reconnect, the client fetches messages after its last-read cursor
- Well-suited for large channels with thousands of members where most members do not read every message

### Discord Architecture — Hybrid Approach

- Small servers (up to ~250 members) use fan-out on write
- Large servers use a pull-based model with caching layers
- Voice and video are handled separately via WebRTC

## Scenario-Based Questions

**Q: Users in a group chat see messages in different orders. Some users claim replies reference messages they cannot see. How do you fix ordering?**

- Ensure the server assigns a monotonically increasing sequence ID per conversation at the moment of reception
- Store messages in a database with a clustered index on (conversation_id, sequence_id)
- Clients sort by sequence ID, never by client timestamp
- **Interview follow-up:** How do you handle the case where two messages arrive at the server in the same millisecond? How do you break the tie?

**Q: Your chat service slows down significantly when a user with 50,000 unread messages reconnects. How do you optimize this?**

- Use pagination: load only the most recent N messages (e.g., 50) on reconnect, offer "load older" as a separate request
- Implement cursor-based pagination using sequence IDs rather than offset-based pagination
- For group channels, use fan-out on read — do not pre-populate inboxes
- **Interview follow-up:** How do you ensure the user sees all missed messages without causing a thundering herd of reads on the database?

**Q: Your chat system uses WebSocket. Some users behind strict corporate firewalls cannot connect. How do you provide a fallback?**

- Implement HTTP long polling as a fallback — the client sends a request, the server holds it open until a message arrives or timeout occurs
- Add Server-Sent Events (SSE) support as an alternative one-way channel for receiving messages, with a separate REST endpoint for sending
- Use a connection library (Socket.IO) that automatically negotiates the best available transport: WebSocket > SSE > long polling
- **Interview follow-up:** How does the fallback mechanism affect message ordering guarantees and delivery latency?

**Q: Your chat system sends push notifications for every message in a busy group chat with 500 members. Users complain about notification spam. How do you reduce it?**

- Implement notification aggregation: batch messages within a 30-second window and send a single notification ("5 new messages in Group Chat")
- Allow users to configure per-chat notification settings: mute, mention-only, or all messages
- For high-traffic groups, use a cooldown per user per chat — send at most one notification per minute regardless of message count
- **Interview follow-up:** How do you balance timely notifications with aggregation delay?

**Q: A user sends a message, but due to a network blip, the client retries and the same message appears twice. How do you deduplicate?**

- The client generates a unique message UUID before sending; the server checks this UUID in a Redis SET NX with TTL before processing
- If the UUID already exists within the dedup window, return the existing message ID without creating a duplicate
- Ensure idempotency handling extends to push notifications and message storage alike
- **Interview follow-up:** How long should the deduplication window be and what happens to message IDs after the window expires?

**Q: Your chat service stores messages in a relational database. As message volume grows, querying conversation history becomes slow. How do you scale?**

- Partition the messages table by conversation_id using a sharding key derived from conversation ID
- Use a time-series approach: create per-month or per-day tables for messages and query only relevant partitions
- Move older messages to a cheaper, slower storage tier (archival) and keep recent messages in fast storage
- **Interview follow-up:** How do you handle cross-shard queries like "search all my conversations for a keyword"?

**Q: In a group chat, user A sends a message but user B claims it arrived 30 seconds late. The server timestamp shows the message was stored immediately. What could go wrong?**

- The server assigned the timestamp at reception, but the message may have been queued before delivery to user B due to fan-out delays
- Check if user B's WebSocket connection was reconnecting during that period — messages may have been buffered on the server
- The message delivery pipeline (server → queue → push notification service → client) may have bottlenecks at the push step
- **Interview follow-up:** How do you measure and monitor end-to-end message delivery latency per user?

**Q: Your chat system stores read receipts as individual records per message per user. In a busy group chat, this generates millions of small writes. How do you optimize?**

- Aggregate read receipts: instead of per-message updates, store a "last read message ID" per user per conversation
- Send read receipts periodically (every 5 seconds) rather than on every read event
- Use a counter-based approach: track that user has read up to message sequence X; infer that earlier messages are also read
- **Interview follow-up:** How does the aggregated approach affect the accuracy of "seen by" indicators?

**Q: A user's device is offline for 2 hours. When they reconnect, the client tries to sync all missed messages and the app freezes. How do you fix this?**

- Implement cursor-based pagination: load the most recent N messages (e.g., 50) on reconnect and load older messages on demand
- For high-volume conversations, provide a "catch up" summary: "150 new messages since your last visit. Tap to view."
- Use a separate sync endpoint that returns only message metadata (sender, sequence ID, timestamp) for quick rendering, with full content loaded on scroll
- **Interview follow-up:** How do you handle the case where a user has millions of unread messages across thousands of conversations?

**Q: Your chat system supports end-to-end encryption. How does this affect server-side search and push notifications?**

- E2E encryption means the server cannot read message content — search must happen client-side by downloading and decrypting message history locally
- Push notifications cannot include message content — send a generic notification ("New message from Alice") without the message text
- For search, consider indexing encrypted message metadata (sender, timestamp) on the server and keeping content search client-side
- **Interview follow-up:** How do you handle key management when a user logs in from a new device — how do they access old encrypted messages?

## Interview Questions

- **How do you implement message ordering in a distributed chat system?**
  - The server assigns a globally unique, monotonically increasing sequence ID per conversation. Sequence IDs can be a combination of timestamp + shard ID + counter. Clients sort messages by this sequence ID. For cross-shard consistency, use hybrid logical clocks or a centralized sequencer per conversation.
- **Explain the tradeoffs between fan-out on write and fan-out on read for group chat.**
  - Fan-out on write: low read latency (pre-computed inbox), high write amplification, suitable for small groups. Fan-out on read: low write cost, higher read latency, suitable for large channels where most users do not read every message. Hybrid approaches switch strategies based on group size.
- **How would you design presence tracking for 100M users?**
  - Use Redis with per-user key (presence:{user_id}) and TTL equal to heartbeat interval * 2. Clients send heartbeats every 15 seconds. A separate presence service subscribes to Redis key expiration events for offline detection. For scaling, shard presence data by user ID hash across multiple Redis clusters.
- **Describe how you would handle media uploads in a chat system.**
  - Clients receive a presigned upload URL from the server, upload directly to blob storage. The server stores only the media URL, thumbnail, and metadata. A CDN caches media at edge locations. Thumbnails are generated server-side using a queue-based image processing pipeline.
- **How do you scale WebSocket connections to 10M concurrent users?**
  - Use a dedicated WebSocket gateway layer that handles connection management separately from application logic. Each gateway node handles 100K–500K connections. Use consistent hashing (by user ID) to route clients to the same gateway. Use Redis pub/sub for cross-gateway message delivery.
- **Explain how message ordering is maintained across shards.**
  - Each conversation is assigned to a single shard based on conversation_id hash. All messages in a conversation go to that shard, which assigns monotonic sequence IDs. This ensures in-order delivery within a conversation. Cross-shard ordering (unified feed) requires a global sequencer or hybrid logical clocks.
- **How do you implement "typing indicators" in a chat system?**
  - The client sends a "typing" event via WebSocket every few seconds while the user is typing. The server broadcasts to other conversation participants with a short TTL (3 seconds). If no new typing event arrives within TTL, indicate "stopped typing." Use Redis pub/sub to fan-out across WebSocket gateway nodes.
- **What is the difference between a "last seen" indicator and presence tracking?**
  - Last seen stores the timestamp of the user's last activity, shown when offline. Presence shows real-time online/offline/busy status, based on heartbeat intervals. Last seen is privacy-friendly (approximate), while presence is real-time but more intrusive. Some systems offer configurable privacy controls for both.
- **How do you handle message deletion for all users?**
  - Store a "deleted" flag or timestamp on the message record. When fetching conversation history, the server filters out deleted messages. For push notifications, check the deletion status before sending. For "delete for everyone," use a server-side timestamp that retroactively marks the message as deleted.
- **Explain the architecture of WhatsApp's end-to-end encryption.**
  - WhatsApp uses the Signal Protocol. Each client generates a public/private key pair. When user A messages user B, the server provides B's public key. A encrypts the message using a per-session symmetric key derived from both parties' keys. The server stores only the encrypted message blob and cannot decrypt it.
- **How do you implement message search across millions of messages?**
  - Use a dedicated search index (Elasticsearch) indexed by conversation_id, sender_id, and message content. The search index is updated asynchronously from the message queue. For E2E encrypted systems, search must be client-side — download and decrypt messages, then search locally.
- **How do you handle message delivery guarantees (at-least-once vs exactly-once)?**
  - At-least-once: the server stores the message and the client acknowledges receipt; if no ack, the server retries. Exactly-once requires idempotency keys plus deduplication on the client and server. Most chat systems use at-least-once for availability and handle duplicates at the UI layer.
- **What is the role of a "message queue" in a chat system?**
  - The message queue (Kafka, RabbitMQ) decouples message ingestion from delivery. The server writes incoming messages to the queue. Fan-out workers consume from the queue and deliver via WebSocket or push notifications. The queue provides buffering during traffic spikes and enables replays for recovery.
- **How do you design a chat system for offline message sync?**
  - Store messages in a per-conversation message log. On client reconnect, the client provides its last-known sequence ID. The server returns all messages after that sequence ID, paginated. The client merges the new messages into its local store, maintaining a local copy for offline access.
- **Explain the concept of "conversation warm-up" and why it matters.**
  - When a user opens a conversation, the system needs to load recent messages, presence status, and typing indicators. If the user switches between conversations, repeatedly loading full message history is wasteful. Pre-load the N most recent conversations' metadata and message snippets, and lazy-load full content on selection.
- **How do you rate limit message sending in a chat system?**
  - Per-user: max N messages per second/minute to prevent spam. Per-conversation: max M messages per minute to prevent flooding. Apply rate limits at the server ingress before message processing. Return 429 with Retry-After for exceeded limits. Consider different limits for 1:1 vs group chats.
- **What is the difference between a "channel" and a "direct message" in chat architecture?**
  - Direct messages typically use a conversation ID computed from the sorted pair of user IDs (min(userA, userB) + max(userA, userB)). Channels (group chats) use a unique group ID. DMs are always two-participant; channels support N participants. Both use the same message storage and fan-out mechanisms internally.
- **How do you handle voice and video calls in a chat system?**
  - Voice/video uses WebRTC for peer-to-peer media after an initial signaling phase through the chat server. The chat server facilitates the ICE/STUN/TURN negotiation by exchanging SDP offers and answers via WebSocket. Media streams are peer-to-peer when possible; TURN relay is used for NAT traversal.
- **How do you implement a "reply to message" feature?**
  - Store a "reply_to_message_id" field on the message record. The client sends the original message ID when composing a reply. The server validates the referenced message exists in the same conversation and includes the quoted snippet when rendering. For deleted replied messages, show "[deleted]" as the quoted content.
- **How do you design a chat system that handles large file transfers (1GB+)?**
  - Use presigned upload URLs for direct client-to-blob-storage uploads, bypassing the chat server. Stream the file in chunks to handle network interruptions. Once uploaded, send only the file metadata and download URL through the chat system. Use a CDN for download delivery to reduce server load.

## Developer Recommendations

- **Use server-assigned sequence IDs for ordering**
  - Never trust client timestamps for message ordering; clock skew, timezone changes, and manual clock adjustments all cause ordering violations
  - Use a per-conversation atomic counter (Redis INCR, Cassandra lightweight transaction) as the sequence ID source
  - **Production story:** A chat app using client timestamps showed out-of-order messages for 12% of users — switching to server-assigned sequence IDs eliminated the issue entirely
- **Implement idempotent message delivery**
  - Client sends a unique message UUID; server deduplicates within a sliding time window
  - Use Redis SET NX with the message UUID as key and TTL of 5 minutes
  - **Production story:** Without idempotency, a network retry caused 5% duplicate messages during a regional outage
- **Choose the right fan-out strategy per group size**
  - Small groups (<1000 members): fan-out on write for instant reads
  - Large channels (>1000 members): fan-out on read with cursor-based pagination
  - Monitor write amplification and adjust thresholds dynamically
- **Use WebSocket with automatic reconnection and fallback**
  - Implement exponential backoff for reconnection (1s, 2s, 4s, max 30s)
  - Fallback to SSE, then long polling if WebSocket fails
  - Include heartbeat (ping/pong) every 15 seconds to detect stale connections
