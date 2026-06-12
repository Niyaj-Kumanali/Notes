# WebSockets

---

## Overview

- **Definition:** WebSockets provide full-duplex communication channels over a single TCP connection, enabling real-time bidirectional data transfer between clients and servers.
- **Why It Exists:** HTTP is half-duplex and stateless — clients must poll for updates. WebSockets maintain a persistent connection, eliminating polling overhead and enabling sub-100ms real-time communication for chat, live notifications, collaborative editing, and trading platforms.
- **Key Concepts:** **Upgrade** (HTTP to WebSocket handshake), **Frame** (data unit with opcode, payload, masking), **Full-Duplex** (both sides send anytime), **Persistent Connection** (stays open until explicitly closed), **Subprotocol** (application protocol on top — e.g., STOMP), **WSS** (WebSocket over TLS).

---

## Core Concepts

### Connection Lifecycle

```
Client                              Server
  |--- HTTP GET (Upgrade: websocket) -->|
  |<-- 101 Switching Protocols ---------|
  |--- WebSocket Frame (text/binary) -->|
  |<-- WebSocket Frame (text/binary) ---|
  |--- Close Frame --------------------->|
  |<-- Close Frame ---------------------|
```

### Frame Structure

```
| FIN (1) | RSV (3) | Opcode (4) | MASK (1) | Payload Len (7/16/64) |
| Masking Key (0 or 4 bytes) | Payload Data |
```

- **Opcode:** 1 = text, 2 = binary, 8 = close, 9 = ping, 10 = pong.

### Key Differences from HTTP

| Aspect | HTTP | WebSocket |
|--------|------|-----------|
| Connection | Short-lived | Persistent |
| Direction | Half-duplex | Full-duplex |
| Overhead | High (headers) | Low (minimal framing) |
| Streaming | Not native | Native |
| Protocol | Text-based | Binary framing |
| Latency | Higher | Lower |

### Handshake Detail

```http
GET /ws/chat HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

### Spring Boot STOMP Configuration

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic", "/queue");
        registry.setApplicationDestinationPrefixes("/app");
        registry.setUserDestinationPrefix("/user");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
            .setAllowedOrigins("https://app.example.com")
            .withSockJS();
    }
}
```

### Chat Controller

```java
@Controller
public class ChatController {
    private final SimpMessagingTemplate messagingTemplate;

    @MessageMapping("/chat/{roomId}")
    @SendTo("/topic/chat/{roomId}")
    public ChatMessage handleChatMessage(
            @DestinationVariable String roomId,
            @Payload ChatMessage message,
            Principal principal) {
        message.setSender(principal.getName());
        message.setTimestamp(Instant.now());
        chatService.saveMessage(message);
        return message;
    }

    @MessageMapping("/private-message")
    public void handlePrivateMessage(@Payload PrivateMessage message, Principal principal) {
        message.setSender(principal.getName());
        messagingTemplate.convertAndSendToUser(
            message.getRecipient(), "/queue/private-messages", message);
    }
}
```

### REST API Pushing WebSocket Notifications

```java
@RestController
@RequestMapping("/api/v1/notifications")
public class NotificationController {
    private final SimpMessagingTemplate messagingTemplate;

    @PostMapping("/broadcast")
    public ResponseEntity<Void> broadcast(@RequestBody BroadcastRequest request) {
        messagingTemplate.convertAndSend("/topic/notifications",
            new Notification("BROADCAST", request.getMessage()));
        return ResponseEntity.ok().build();
    }

    @PostMapping("/user/{userId}")
    public ResponseEntity<Void> sendToUser(@PathVariable String userId,
                                           @RequestBody NotificationRequest request) {
        messagingTemplate.convertAndSendToUser(userId, "/queue/notifications",
            new Notification("USER", request.getMessage()));
        return ResponseEntity.ok().build();
    }
}
```

### WebSocket Auth Interceptor

```java
@Component
public class WebSocketAuthInterceptor implements ChannelInterceptor {
    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(message);
        switch (accessor.getCommand()) {
            case CONNECT -> {
                String token = accessor.getFirstNativeHeader("Authorization");
                if (token != null && token.startsWith("Bearer ")) {
                    Authentication auth = tokenProvider.getAuthentication(token.substring(7));
                    accessor.setUser(auth);
                }
            }
            case SUBSCRIBE -> {
                String dest = accessor.getDestination();
                if (dest != null && dest.startsWith("/topic/admin")
                        && !hasAdminRole(accessor.getUser())) {
                    throw new AuthorizationException("Access denied");
                }
            }
        }
        return message;
    }
}
```

### Raw WebSocket Handler (Without STOMP)

```java
@Component
public class RawWebSocketHandler extends TextWebSocketHandler {
    private final Set<WebSocketSession> sessions = ConcurrentHashMap.newKeySet();

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        sessions.add(session);
        broadcastMessage("User connected: " + session.getId());
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) {
        session.sendMessage(new TextMessage("Echo: " + message.getPayload()));
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        sessions.remove(session);
        broadcastMessage("User disconnected: " + session.getId());
    }

    private void broadcastMessage(String message) {
        TextMessage textMessage = new TextMessage(message);
        sessions.forEach(s -> {
            if (s.isOpen()) try { s.sendMessage(textMessage); } catch (IOException ignored) {}
        });
    }
}
```

### Horizontal Scaling with External Broker

```java
@Configuration
public class WebSocketScalabilityConfig {
    @Bean
    public SimpleMessageBrokerConfigurer messageBrokerConfigurer() {
        return registry -> registry.enableStompBrokerRelay("/topic", "/queue")
            .setRelayHost("rabbitmq.example.com")
            .setRelayPort(61613)
            .setSystemHeartbeatSendInterval(10000)
            .setSystemHeartbeatReceiveInterval(10000);
    }
}
```

### JavaScript Client

```javascript
const stompClient = new StompJs.Client({
    brokerURL: 'wss://api.example.com/ws',
    connectHeaders: { Authorization: 'Bearer ' + accessToken },
    onConnect: () => {
        stompClient.subscribe('/topic/chat/room1', msg => displayMessage(JSON.parse(msg.body)));
        stompClient.subscribe('/user/queue/notifications', msg => showNotification(JSON.parse(msg.body)));
        stompClient.publish({ destination: '/app/chat/room1', body: JSON.stringify({ content: 'Hello!' }) });
    }
});
stompClient.activate();
```

---

## Common Mistakes

- **Not authenticating at handshake** — verify identity before establishing the connection. This *looks correct* because the WebSocket connects successfully — the vulnerability only matters when an attacker probes topics they shouldn't access.
- **Missing authorization** — validate which topics/queues each user can subscribe to. This *looks correct* because the user is authenticated at connect time — assuming the same identity implies blanket access to every topic is a subtle leap.
- **Not using WSS** — always encrypt WebSocket traffic in production. This *looks correct* because the WebSocket protocol works over plain TCP just fine — the data is readable in transit but the developer never sees the traffic they aren't intercepting.
- **Memory leaks** — clean up sessions and subscriptions on disconnect. This *looks correct* because the server doesn't crash immediately after a disconnect — the leaked session object sits in memory, accumulating across thousands of disconnects until the OOM killer acts.
- **No backpressure** — can overwhelm clients with too many messages. This *looks correct* because sending a message succeeds instantly — the buffer fills silently on the slow client, eventually causing the connection to drop without explanation.
- **Synchronous processing in listeners** — never block the event loop. This *looks correct* because a single blocking operation completes quickly — only under concurrent connections does the event loop stall become visible as dropped heartbeats and timeouts.
- **Ignoring heartbeat** — without heartbeats, dead connections go undetected. This *looks correct* because the server has no way to distinguish a silent but alive connection from a dead one — stale sessions accumulate with no symptom until memory pressure mounts.
- **Not scaling the broker** — in-memory broker won't work across multiple instances. This *looks correct* because the application works fine with one server — the problem only manifests when a second instance is added and messages published on one server never reach users on the other.
- **Large messages** — keep messages small; compress or paginate large payloads. This *looks correct* because a single 10MB message sends fine — the cumulative cost in serialization time, network throughput, and client memory only appears at scale.

---

## Key Design Considerations

- **When to Use:** Real-time chat, live notifications, collaborative editing, live data feeds (sports, stocks), online gaming, real-time dashboards
- **When NOT to Use:** Request-response APIs (use REST/gRPC), batch data transfer, simple CRUD, stateless operations
- **Connection Limits:** Tomcat NIO ~10K concurrent, Netty ~100K+, tuned epoll ~1M
- **Memory:** Each connection consumes ~20–50KB
- **Scaling:** Use external STOMP broker (RabbitMQ, ActiveMQ) or Redis Pub/Sub for multi-instance fan-out
- **Heartbeat:** Balance liveness detection with network overhead — typical: 10s interval
- **Security:** Validate Origin header, authenticate on handshake, authorize subscriptions, sanitize payloads, enforce WSS
- **Architecture Patterns:** Direct WebSocket (raw), STOMP (pub-sub, most common), RSocket (reactive), WebRTC (P2P)

```yaml
# Heartbeat configuration
registry.enableSimpleBroker("/topic", "/queue")
    .setHeartbeatValue(new long[]{10000, 10000});
```

---

## Real-World Scenarios

### Scenario 1: WebSocket Authentication Bypass
**Context:** A stock trading app uses STOMP over WebSocket for real-time price updates. The WebSocket handshake accepts any connection without authentication. A malicious user connects to the WebSocket endpoint and subscribes to `/topic/admin/price-alerts` (intended for administrators only). They receive real-time price alerts meant for the trading desk, gaining an unfair market advantage.

**Resolution:** Authenticate at the WebSocket handshake and authorize every subscription.

```java
@Component
public class WebSocketAuthInterceptor implements ChannelInterceptor {
    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(message);
        switch (accessor.getCommand()) {
            case CONNECT -> {
                // Authenticate during handshake
                String token = accessor.getFirstNativeHeader("Authorization");
                if (token == null || !token.startsWith("Bearer ")) {
                    throw new AuthenticationException("Missing or invalid token");
                }
                Authentication auth = tokenProvider.getAuthentication(token.substring(7));
                accessor.setUser(auth);
            }
            case SUBSCRIBE -> {
                // Authorize every subscription
                String destination = accessor.getDestination();
                Authentication user = accessor.getUser();
                if (destination != null && destination.startsWith("/topic/admin")
                        && !hasAdminRole(user)) {
                    throw new AuthorizationException("Access denied to " + destination);
                }
            }
        }
        return message;
    }
}
```

### Scenario 2: WebSocket Connection Exhaustion from Memory Leak
**Context:** A real-time dashboard application uses WebSockets to push updates to 5,000 concurrent users. After 24 hours of operation, server memory grows from 512MB to 4GB. The server crashes with `OutOfMemoryError`. Investigation reveals that each WebSocket session maintains a reference to a user-specific data object that is never cleaned up on disconnect — the sessions list grows monotonically.

**Resolution:** Track WebSocket sessions properly and clean up on disconnect. Use a concurrent map with weak references or explicit cleanup in `afterConnectionClosed`.

```java
@Component
public class DashboardWebSocketHandler extends TextWebSocketHandler {
    private final ConcurrentMap<String, UserSession> activeSessions = new ConcurrentHashMap<>();

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        UserSession userSession = new UserSession(session.getId(), extractUser(session));
        activeSessions.put(session.getId(), userSession);
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        UserSession removed = activeSessions.remove(session.getId());
        if (removed != null) {
            removed.cleanup();  // Release any resources held by the session
        }
    }
}
```

### Scenario 3: Horizontal Scaling with Message Stomping
**Context:** A chat application runs on 3 server instances behind a load balancer. User A (connected to Server 1) sends a message to Room 1. User B (connected to Server 2) is in the same room. Because the in-memory STOMP broker is local to Server 1, the message is never delivered to Server 2's users. User B sees messages from User A only when both are on the same server.

**Resolution:** Switch to an external STOMP broker (RabbitMQ or ActiveMQ) that all server instances connect to. Messages published to any server are fanned out through the broker to all connected servers.

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        // Use RabbitMQ as the external STOMP broker instead of in-memory broker
        registry.enableStompBrokerRelay("/topic", "/queue")
            .setRelayHost("rabbitmq.example.com")
            .setRelayPort(61613)
            .setSystemHeartbeatSendInterval(10000)
            .setSystemHeartbeatReceiveInterval(10000);
    }
}
```

---

## Scenario-Based Questions

1. **Q: Design a real-time chat system for 10M users. Requirements: group chat (1M users in a room), private messaging, message history, and horizontal scalability. How do you architect the WebSocket layer?**
    - A: (1) Use STOMP over WebSocket with an external broker (RabbitMQ) for horizontal scaling. (2) Load balancer with sticky sessions (hash by user ID). (3) Group chat: each room has a topic `/topic/chat/{roomId}` — 1M users in one room means 1M subscribers. Use fanout per server (the broker sends one message per server, not per user). (4) Private messaging: use `/user/{userId}/queue/messages` — RabbitMQ routes to the specific server where the user is connected. (5) Message history: persist in Cassandra/DynamoDB. On connect, query last N messages and send via WebSocket. (6) Presence tracking: Redis with user → server mapping, heartbeats, and TTL.
    > **Interview follow-up:** With 1M subscribers to a single room, fanout per server means one message is published once per connected server — but what happens when the fanout message must be replicated across 100 servers? Does the STOMP relay itself become a bottleneck?

2. **Q: Your WebSocket server runs on a single instance with 10,000 connections. During a deployment, all connections drop. Users must manually refresh the page. How do you implement zero-downtime WebSocket deploys?**
   - A: (1) Graceful shutdown: register a JVM shutdown hook that stops accepting new connections, then gracefully closes existing connections with a "server restarting" message and a suggested reconnect delay. (2) Client reconnection: the client receives the close frame with the delay, waits for the specified duration, then reconnects. (3) Rolling update with a load balancer: remove one server from the pool, drain its connections gracefully, update, add back. (4) Session persistence: store session state in Redis — on reconnect, the new server picks up the user's subscriptions and state. (5) Use Kubernetes preStop hook: `sleep 30` before SIGTERM — gives connections time to drain.

3. **Q: Your WebSocket connection drops frequently on mobile networks. Users lose messages sent during the disconnection. How do you implement reliable messaging with offline support?**
   - A: (1) Client-side message ID: each sent message has a unique client-generated ID. (2) Server ACK: the server responds with `{"ack": "<messageId>"}` after persistence. (3) Message buffer: unacknowledged messages are stored in the client's local storage and replayed on reconnect. (4) Server-side offline buffer: use Redis to store the last N messages per user. On reconnect, the server replays missed messages. (5) Client reconnect with exponential backoff: 1s → 2s → 4s → 8s → max 30s. On reconnect, the client sends the last received message ID. (6) Detect connection state with `navigator.onLine` events and WebSocket pings.

4. **Q: Your collaborative document editor uses WebSockets with Operational Transformation. When 100 users edit the same document simultaneously, the server CPU spikes to 100% and some operations are lost. How do you scale the server-side processing?**
    - A: (1) Use CRDTs (Conflict-Free Replicated Data Types) instead of OT — CRDTs are commutative and don't require a central ordering server, reducing server CPU. (2) Batch operations: don't process each keystroke individually — buffer edits for 50ms and apply as a batch. (3) Shard by document ID: route all operations for document A to Server 1, document B to Server 2. (4) Use a dedicated OT/CRDT processing cluster separate from the WebSocket server. (5) Rate limit operations per user (e.g., 10 ops/second) — human typing speed is <10 chars/second; higher rates indicate automated edits.
    > **Interview follow-up:** CRDTs guarantee convergence but not ordering — two users editing the same sentence concurrently can produce a grammatically incorrect result that neither intended. How do you handle the UX of semantically-conflicting edits that CRDTs resolve by arbitrary merge rules?

5. **Q: How do you handle 1M+ concurrent WebSocket connections on a single server?**
   - A: (1) Use Netty (NIO event-loop model) — handles millions of connections with a few threads. (2) OS tuning: increase `fs.file-max` and `ulimit -n` to 2M+, enable `SO_REUSEPORT` for multi-threaded accept, tune TCP keepalive. (3) Memory per connection: ~50KB. 1M connections = 50GB RAM. Use off-heap memory and efficient session storage. (4) Use epoll (Linux) for O(1) event notification. (5) In practice: scale horizontally. Each server handles 100K-200K connections. Use a load balancer (HAProxy, Nginx) with `least-connections` algorithm and proxy protocol for client IP preservation.

6. **Q: Your chat application needs to show online/offline status for 1M users across 10 server instances. How do you implement accurate presence detection without overloading the system?**
   - A: (1) On connect: add `user:{userId}` to Redis set `online:users` with TTL = heartbeat_interval × 3. (2) Heartbeat: each WebSocket client sends a ping every 15 seconds. The server updates the TTL of the user's Redis key. (3) On disconnect: remove from `online:users`. (4) Broadcast presence changes to `/topic/presence` — each server subscribes and updates its local user list. (5) For 1M users, batch presence updates: send "user went online" only after 30 seconds of confirmed uptime (debounce). (6) Use a separate Redis instance or cluster for presence data to avoid impacting business data.

7. **Q: A malicious WebSocket client sends 10,000 messages per second to your server. Each message triggers a database write. How do you implement rate limiting for WebSocket connections?**
   - A: (1) Per-connection token bucket: each connection has a `RateLimiter` with capacity = 10 messages/second, refill rate = 10/second. (2) If exceeded, send a rate-limit error frame and close the connection after 3 warnings. (3) For cross-cluster rate limiting: use Redis with a sliding window per user. (4) Message size limits: reject messages larger than 64KB at the frame level. (5) Backpressure: if the consumer is slow, stop reading from the socket (Netty auto-read control). (6) Different rate limits per operation type: typing indicator = 5/s, chat message = 1/s, file upload = 1/minute.

8. **Q: Your WebSocket server sometimes sends messages faster than clients can process them. Messages queue up in the client's receive buffer, memory grows, and the connection becomes unresponsive. How do you implement backpressure?**
    - A: (1) Monitor the client's send buffer: `session.getTextMessageSizeLimit()` or Netty's `Channel.isWritable()`. If the buffer exceeds a threshold (e.g., 64KB), stop sending to that client. (2) Sliding window protocol: the server maintains a window of N in-flight messages per client. Each message requires a client ACK. When the window is full, stop sending. (3) Prioritize messages: drop non-critical messages (typing indicators, presence updates) under backpressure. Always deliver critical messages (chat, notifications). (4) Implement adaptive rate limiting: if a client's ACK rate drops below a threshold, reduce send rate.
    > **Interview follow-up:** If you drop non-critical messages like typing indicators under backpressure, how do you prevent the dropped updates from creating a permanently incorrect UI state — for example, a user who appears to still be typing because the "stopped typing" event was also dropped?

9. **Q: You need to implement a real-time multiplayer game server. Requirements: <50ms latency, 60 updates/second, 100 players per game session. Why would you choose raw WebSocket over STOMP?**
   - A: (1) STOMP adds framing overhead: each STOMP frame has a command header, content-type, and destination header. For 60 updates/second, this overhead is significant. (2) Raw WebSocket has minimal framing: just opcode + payload. (3) Raw WebSocket supports binary frames — send compressed game state as Protocol Buffers or FlatBuffers instead of JSON. (4) Custom protocol: define your own message types (1 byte message ID + payload) — far more efficient than STOMP's text-based protocol. (5) STOMP's pub-sub model adds routing overhead. In a game, you typically broadcast to all players in a session — raw WebSocket with a session collection is simpler and faster.

10. **Q: Your WebSocket application sends sensitive user data. A security audit requires end-to-end encryption (E2EE) where the server cannot decrypt messages. How do you design this?**
    - A: (1) Key exchange during handshake: use the WebSocket subprotocol negotiation to perform a Diffie-Hellman key exchange (or use the `Sec-WebSocket-Protocol` header to agree on an E2EE subprotocol). (2) After handshake, both client and server have a shared symmetric key without the server knowing the key (server relays key material between clients without decrypting). (3) Group chats: use a group key that's distributed to all members via their individual encrypted channels. The server relays the encrypted group messages without decrypting them. (4) Key rotation: periodically rotate keys and re-distribute. (5) Trade-off: the server cannot perform content-based features (search, moderation, spam detection). For compliance requirements, use client-side scanning or anonymous statistical analysis instead.

---

## Interview Questions

1. **What is a WebSocket and how does it differ from HTTP?**
   - A: WebSocket provides full-duplex communication over a single persistent TCP connection. Unlike HTTP's half-duplex request-response model, both sides can send messages anytime with minimal overhead (low framing, no headers per message).

2. **How does the WebSocket handshake work?**
   - A: Client sends an HTTP GET with `Upgrade: websocket`, `Connection: Upgrade`, and `Sec-WebSocket-Key`. Server responds with `101 Switching Protocols` and `Sec-WebSocket-Accept`. The connection upgrades from HTTP to WebSocket.

3. **What is STOMP and why use it over raw WebSockets?**
   - A: STOMP is a text-based messaging protocol that runs on top of WebSocket. It provides pub-sub semantics (topics, queues), destination routing, and message headers. Use STOMP for chat, notifications, and dashboards. Use raw WebSocket for games and low-latency applications.

4. **How do you scale WebSocket connections across multiple servers?**
   - A: Use an external STOMP broker (RabbitMQ, ActiveMQ) or Redis Pub/Sub. All servers connect to the broker. Messages published to any server are fanned out through the broker. Load balancer with sticky sessions routes clients to their connected server.

5. **How do you authenticate WebSocket connections?**
   - A: Validate a JWT/OAuth2 token during the handshake (in the `Authorization` header or as a query parameter). Store the authenticated principal in the session. Authorize subscription destinations — reject subscriptions to topics the user shouldn't access.

6. **How do you detect and handle WebSocket disconnections?**
   - A: Server-side: `afterConnectionClosed` callback, heartbeat/ping frames (10-30s interval). Client-side: `onclose` event, reconnection with exponential backoff, message buffering during disconnection.

7. **How do you implement WebSocket reconnection with message recovery?**
   - A: Client sends messages with unique IDs. Server ACKs on receipt. Unacknowledged messages are stored client-side and replayed on reconnect. Server buffers last N messages per user in Redis and replays on reconnect based on last received message ID.

8. **How do you implement backpressure in WebSockets?**
   - A: Monitor the send buffer — stop sending if the buffer exceeds a threshold. Use a sliding window protocol with in-flight message tracking and client ACKs. Drop non-critical messages under pressure. Prioritize critical messages.

9. **What are the memory considerations for WebSocket connections?**
   - A: Each connection consumes ~20-50KB of server memory. 100K connections = 2-5GB. Use Netty for efficient connection handling (NIO event-loop). Set max connections per server. Offload session state to Redis.

10. **When would you choose Server-Sent Events (SSE) over WebSockets?**
    - A: SSE when you need only server-to-client push (notifications, feeds) and HTTP/2 is available. SSE is simpler (runs over HTTP, auto-reconnects, standard EventSource API). WebSocket when you need bidirectional communication (chat, games, collaborative editing).

---

## Developer Recommendations

- **Always authenticate at the WebSocket handshake, not just in subscriptions** — Authenticating only in subscription handlers allows an attacker to open a WebSocket connection and keep it alive without identity. Authenticate the `CONNECT` frame (STOMP) or the handshake request itself. Reject connections with invalid or missing tokens immediately. After authentication, authorize every subscription against the user's permissions — don't assume that a connected user is authorized for all topics. One team learned this the hard way when a security audit found that an intern's demo script — left running overnight — had an open WebSocket that was still receiving admin alerts because subscriptions were never re-validated.

- **Use an external STOMP broker for multi-instance deployments** — The in-memory STOMP broker works only on a single server instance. As soon as you have 2+ servers, messages published on Server 1 never reach users connected to Server 2. Use RabbitMQ or ActiveMQ as a STOMP relay: all servers connect to the broker, and messages fan out through it. The configuration change is minimal (swap `enableSimpleBroker` for `enableStompBrokerRelay`). Don't wait until you need it — set it up from day one.

- **Implement heartbeats to detect dead connections** — WebSocket connections can die silently: network cable unplugged, laptop sleeps, mobile loses signal. Without heartbeats, the server holds stale sessions indefinitely, wasting memory and causing incorrect presence status. Send server-to-client heartbeats every 10-30 seconds. If the server detects a missing heartbeat, close the connection and clean up resources. The client also uses heartbeats to trigger reconnection.

- **Handle backpressure to prevent overwhelming slow clients** — A fast publisher can fill a slow consumer's TCP buffer, causing memory growth and eventual connection timeout. Monitor the send buffer size. If it exceeds a threshold (e.g., 64KB), stop sending and either buffer (with limits), drop non-critical messages, or apply rate limiting. For critical messages, use a sliding window protocol with client ACKs. Without backpressure, a single slow client can consume disproportionate server resources.

- **Store session state in Redis for graceful failover and deploys** — If a server crashes or is taken down for deployment, all WebSocket connections on that server are lost. Without session state recovery, users must re-subscribe to topics and re-establish their state. Store each user's subscriptions and last message IDs in Redis. On reconnect, restore subscriptions from Redis. Combined with sticky sessions and graceful shutdown, this enables zero-downtime deployments.

- **Use Protocol Buffers or FlatBuffers instead of JSON for high-throughput WebSocket apps** — JSON parsing is CPU-intensive for high-frequency updates (60 updates/second per user × 10,000 users). Binary protocols like Protocol Buffers are 3-10x faster to serialize/deserialize and produce smaller payloads. For real-time games, stock tickers, and collaborative editing, the difference between JSON and Protobuf is the difference between 50% CPU and 10% CPU. Raw WebSocket with binary frames enables this — STOMP is text-based and doesn't support binary payloads natively.
