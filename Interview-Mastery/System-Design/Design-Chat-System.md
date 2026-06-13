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

## Interview Questions

- **How do you implement message ordering in a distributed chat system?**
  - The server assigns a globally unique, monotonically increasing sequence ID per conversation. Sequence IDs can be a combination of timestamp + shard ID + counter. Clients sort messages by this sequence ID. For cross-shard consistency, use hybrid logical clocks or a centralized sequencer per conversation.
- **Explain the tradeoffs between fan-out on write and fan-out on read for group chat.**
  - Fan-out on write: low read latency (pre-computed inbox), high write amplification, suitable for small groups. Fan-out on read: low write cost, higher read latency, suitable for large channels where most users do not read every message. Hybrid approaches switch strategies based on group size.
- **How would you design presence tracking for 100M users?**
  - Use Redis with per-user key (presence:{user_id}) and TTL equal to heartbeat interval * 2. Clients send heartbeats every 15 seconds. A separate presence service subscribes to Redis key expiration events for offline detection. For scaling, shard presence data by user ID hash across multiple Redis clusters.
- **Describe how you would handle media uploads in a chat system.**
  - Clients receive a presigned upload URL from the server, upload directly to blob storage. The server stores only the media URL, thumbnail, and metadata. A CDN caches media at edge locations. Thumbnails are generated server-side using a queue-based image processing pipeline.

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
