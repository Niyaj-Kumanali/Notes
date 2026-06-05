# WebSockets

## 1. Executive Summary

WebSockets provide full-duplex communication channels over a single TCP connection, enabling real-time bidirectional data transfer between clients and servers. Unlike traditional HTTP request-response, WebSockets maintain an open persistent connection that allows both parties to send data at any time. They are essential for real-time applications such as chat systems, live notifications, collaborative editing, online gaming, and financial trading platforms.

## 2. Core Theory

### How WebSockets Work

1. **HTTP Upgrade**: Client sends an HTTP upgrade request.
2. **Handshake**: Server responds with 101 Switching Protocols.
3. **Connection**: Persistent bidirectional TCP connection established.
4. **Framing**: Data is sent in frames (text or binary).
5. **Closing**: Either party sends a close frame.

### WebSocket Lifecycle

```
Client                              Server
  |--- HTTP GET (Upgrade: websocket) -->|
  |<-- 101 Switching Protocols ---------|
  |--- WebSocket Frame (text/binary) -->|
  |<-- WebSocket Frame (text/binary) ---|
  |--- Close Frame --------------------->|
  |<-- Close Frame ---------------------|
```

### WebSocket Frame Structure

```
Frame:
| FIN (1) | RSV (3) | Opcode (4) | MASK (1) | Payload Len (7/16/64) |
| Masking Key (0 or 4 bytes) | Payload Data |
```

### Key Differences from HTTP

| Aspect | HTTP | WebSocket |
|--------|------|-----------|
| Connection | Short-lived | Persistent |
| Direction | Half-duplex | Full-duplex |
| Overhead | High (headers) | Low (minimal framing) |
| Streaming | Not native | Native |
| Protocol | Text-based | Binary framing |
| Latency | Higher | Lower |

## 3. Under-the-Hood Deep Dive

### WebSocket Protocols and Versions

- **RFC 6455**: The standard WebSocket protocol.
- **WSS**: WebSocket over TLS (encrypted).
- **Subprotocols**: Application-level protocols on top of WebSocket (e.g., STOMP, MQTT over WebSocket).
- **Extensions**: Per-message compression, multiplexing.

### Handshake Details

```http
Client Request:
GET /ws/chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Sec-WebSocket-Protocol: chat, superchat
Origin: https://app.example.com

Server Response:
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
```

### STOMP over WebSocket

STOMP (Simple Text Oriented Messaging Protocol) provides a pub-sub model on top of WebSockets:

```
Frame types: CONNECT, SUBSCRIBE, SEND, MESSAGE, DISCONNECT
```

## 4. Production Code Examples

### Spring Boot WebSocket Configuration

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

### WebSocket Configuration with STOMP

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        // Enable in-memory broker for topics and queues
        registry.enableSimpleBroker("/topic", "/queue");

        // Application destination prefix (messages from client)
        registry.setApplicationDestinationPrefixes("/app");

        // User destination prefix for point-to-point
        registry.setUserDestinationPrefix("/user");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        // WebSocket endpoint that clients connect to
        registry.addEndpoint("/ws")
            .setAllowedOrigins("https://app.example.com")
            .withSockJS();  // Fallback for browsers that don't support WebSocket
    }
}
```

### WebSocket Interceptor

```java
@Component
public class WebSocketAuthInterceptor implements ChannelInterceptor {

    private final JwtTokenProvider tokenProvider;

    public WebSocketAuthInterceptor(JwtTokenProvider tokenProvider) {
        this.tokenProvider = tokenProvider;
    }

    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(message);

        switch (accessor.getCommand()) {
            case CONNECT -> {
                // Authenticate on connect
                String token = accessor.getFirstNativeHeader("Authorization");
                if (token != null && token.startsWith("Bearer ")) {
                    token = token.substring(7);
                    if (tokenProvider.validateToken(token)) {
                        Authentication auth = tokenProvider.getAuthentication(token);
                        accessor.setUser(auth);
                    } else {
                        throw new AuthenticationException("Invalid token");
                    }
                } else {
                    // Allow anonymous connections if public
                    accessor.setUser(new AnonymousAuthenticationToken(
                        "anonymous", "anonymousUser",
                        List.of(new SimpleGrantedAuthority("ROLE_ANONYMOUS"))));
                }
            }
            case SUBSCRIBE -> {
                // Authorize subscription
                String destination = accessor.getDestination();
                if (destination != null && destination.startsWith("/topic/admin")
                        && !hasAdminRole(accessor.getUser())) {
                    throw new AuthorizationException("Access denied");
                }
            }
        }

        return message;
    }

    private boolean hasAdminRole(Principal user) {
        if (user instanceof Authentication auth) {
            return auth.getAuthorities().stream()
                .anyMatch(g -> g.getAuthority().equals("ROLE_ADMIN"));
        }
        return false;
    }
}
```

### Chat Controller

```java
@Controller
public class ChatController {

    private final SimpMessagingTemplate messagingTemplate;
    private final ChatService chatService;

    public ChatController(SimpMessagingTemplate messagingTemplate,
                         ChatService chatService) {
        this.messagingTemplate = messagingTemplate;
        this.chatService = chatService;
    }

    // Client sends to /app/chat/{roomId}
    @MessageMapping("/chat/{roomId}")
    @SendTo("/topic/chat/{roomId}")
    public ChatMessage handleChatMessage(
            @DestinationVariable String roomId,
            @Payload ChatMessage message,
            Principal principal) {

        message.setSender(principal.getName());
        message.setTimestamp(Instant.now());
        message.setRoomId(roomId);

        // Persist and return
        chatService.saveMessage(message);
        return message;
    }

    // Private message
    @MessageMapping("/private-message")
    public void handlePrivateMessage(
            @Payload PrivateMessage message,
            Principal principal) {

        message.setSender(principal.getName());
        message.setTimestamp(Instant.now());
        chatService.savePrivateMessage(message);

        // Send to specific user
        messagingTemplate.convertAndSendToUser(
            message.getRecipient(),
            "/queue/private-messages",
            message);
    }

    // Typing indicator
    @MessageMapping("/chat/{roomId}/typing")
    public void handleTyping(
            @DestinationVariable String roomId,
            @Payload TypingIndicator indicator,
            Principal principal) {

        indicator.setUsername(principal.getName());
        messagingTemplate.convertAndSend(
            "/topic/chat/" + roomId + "/typing",
            indicator);
    }
}
```

### REST API + WebSocket Notification

```java
@RestController
@RequestMapping("/api/v1/notifications")
public class NotificationController {

    private final SimpMessagingTemplate messagingTemplate;

    @PostMapping("/broadcast")
    public ResponseEntity<Void> broadcastNotification(
            @Valid @RequestBody BroadcastRequest request) {
        messagingTemplate.convertAndSend("/topic/notifications",
            new Notification("BROADCAST", request.getMessage()));
        return ResponseEntity.ok().build();
    }

    @PostMapping("/user/{userId}")
    public ResponseEntity<Void> sendToUser(
            @PathVariable String userId,
            @Valid @RequestBody NotificationRequest request) {
        messagingTemplate.convertAndSendToUser(
            userId,
            "/queue/notifications",
            new Notification("USER", request.getMessage()));
        return ResponseEntity.ok().build();
    }

    @PostMapping("/order/{orderId}/status")
    public ResponseEntity<Void> notifyOrderStatus(
            @PathVariable String orderId,
            @Valid @RequestBody OrderStatusUpdate status) {
        // Send to order-specific topic
        messagingTemplate.convertAndSend(
            "/topic/order/" + orderId,
            new OrderStatusNotification(orderId, status));

        // Also send to the user who owns the order
        String userId = orderService.getUserId(orderId);
        messagingTemplate.convertAndSendToUser(
            userId,
            "/queue/order-updates",
            new OrderStatusNotification(orderId, status));

        return ResponseEntity.ok().build();
    }
}
```

### Raw WebSocket Handler (Without STOMP)

```java
@Component
public class RawWebSocketHandler extends TextWebSocketHandler {

    private final Set<WebSocketSession> sessions =
        ConcurrentHashMap.newKeySet();

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        sessions.add(session);
        log.info("WebSocket connected: {}", session.getId());
        broadcastMessage("User connected: " + session.getId());
    }

    @Override
    protected void handleTextMessage(WebSocketSession session,
                                    TextMessage message) throws Exception {
        String payload = message.getPayload();
        log.info("Received: {}", payload);

        // Echo back
        session.sendMessage(new TextMessage("Echo: " + payload));
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session,
                                     CloseStatus status) {
        sessions.remove(session);
        log.info("WebSocket disconnected: {} - {}", session.getId(), status);
        broadcastMessage("User disconnected: " + session.getId());
    }

    @Override
    public void handleTransportError(WebSocketSession session,
                                    Throwable exception) {
        log.error("WebSocket error: {}", session.getId(), exception);
        sessions.remove(session);
    }

    private void broadcastMessage(String message) {
        TextMessage textMessage = new TextMessage(message);
        sessions.forEach(session -> {
            if (session.isOpen()) {
                try {
                    session.sendMessage(textMessage);
                } catch (IOException e) {
                    log.error("Failed to send message", e);
                }
            }
        });
    }

    public boolean sendToSession(String sessionId, String message) {
        return sessions.stream()
            .filter(s -> s.getId().equals(sessionId))
            .findFirst()
            .map(session -> {
                try {
                    session.sendMessage(new TextMessage(message));
                    return true;
                } catch (IOException e) {
                    return false;
                }
            })
            .orElse(false);
    }
}
```

### WebSocket Handshake Interceptor

```java
@Component
public class CustomHandshakeInterceptor implements HandshakeInterceptor {

    @Override
    public boolean beforeHandshake(ServerHttpRequest request,
                                  ServerHttpResponse response,
                                  WebSocketHandler wsHandler,
                                  Map<String, Object> attributes) throws Exception {
        // Extract query parameters
        String token = ((ServletServerHttpRequest) request)
            .getServletRequest().getParameter("token");

        if (token != null) {
            // Validate and add to session attributes
            attributes.put("token", token);
            attributes.put("username", extractUsername(token));
            return true;
        }

        // Reject if no token
        response.setStatusCode(HttpStatus.UNAUTHORIZED);
        return false;
    }

    @Override
    public void afterHandshake(ServerHttpRequest request,
                              ServerHttpResponse response,
                              WebSocketHandler wsHandler,
                              Exception exception) {
        log.info("Handshake completed");
    }

    private String extractUsername(String token) {
        // Extract username from JWT or API key
        return jwtTokenProvider.getUsernameFromToken(token);
    }
}
```

### Presence Tracking

```java
@Component
public class PresenceTracker {

    private final Map<String, Set<String>> onlineUsers =
        new ConcurrentHashMap<>();  // roomId -> Set<username>

    public void userConnected(String roomId, String username) {
        onlineUsers.computeIfAbsent(roomId, k -> ConcurrentHashMap.newKeySet())
            .add(username);
    }

    public void userDisconnected(String roomId, String username) {
        Set<String> users = onlineUsers.get(roomId);
        if (users != null) {
            users.remove(username);
            if (users.isEmpty()) {
                onlineUsers.remove(roomId);
            }
        }
    }

    public Set<String> getOnlineUsers(String roomId) {
        return onlineUsers.getOrDefault(roomId, Set.of());
    }

    public int getOnlineCount(String roomId) {
        return getOnlineUsers(roomId).size();
    }
}
```

### Heartbeat and Keepalive

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketHeartbeatConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureWebSocketTransport(WebSocketTransportRegistration registration) {
        registration
            .setSendTimeLimit(15000)      // 15s to send
            .setSendBufferSizeLimit(524288) // 512KB buffer
            .setMessageSizeLimit(65536);    // 64KB per message
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic", "/queue")
            .setHeartbeatValue(new long[]{10000, 10000}) // Server heartbeat
            .setTaskScheduler(heartbeatScheduler());
    }

    @Bean
    public ThreadPoolTaskScheduler heartbeatScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(2);
        scheduler.setThreadNamePrefix("ws-heartbeat-");
        return scheduler;
    }
}
```

### Error Handling

```java
@Controller
public class WebSocketErrorHandler {

    @MessageExceptionHandler
    @SendToUser("/queue/errors")
    public String handleException(Throwable exception) {
        log.error("WebSocket error", exception);
        return "Error: " + exception.getMessage();
    }

    @MessageExceptionHandler(AuthenticationException.class)
    @SendToUser("/queue/errors")
    public String handleAuthException(AuthenticationException exception) {
        return "Authentication failed: " + exception.getMessage();
    }
}
```

### JavaScript Client Example

```javascript
const stompClient = new StompJs.Client({
    brokerURL: 'wss://api.example.com/ws',
    connectHeaders: {
        Authorization: 'Bearer ' + accessToken
    },
    onConnect: () => {
        // Subscribe to public topic
        stompClient.subscribe('/topic/chat/room1', message => {
            const chatMessage = JSON.parse(message.body);
            displayMessage(chatMessage);
        });

        // Subscribe to user's private queue
        stompClient.subscribe('/user/queue/notifications', message => {
            const notification = JSON.parse(message.body);
            showNotification(notification);
        });

        // Send a message
        stompClient.publish({
            destination: '/app/chat/room1',
            body: JSON.stringify({
                content: 'Hello everyone!',
                type: 'TEXT'
            })
        });
    },
    onDisconnect: () => {
        console.log('Disconnected');
    }
});

stompClient.activate();
```

## 5. Real-World Scenarios

### Scenario 1: Real-Time Collaboration

```
User A --(cursor position)--> WebSocket Server --(broadcast)--> User B
User B --(edit operation)--> WebSocket Server --(broadcast)--> User A, C
Server --(document state)--> All users
```

### Scenario 2: Live Trading Dashboard

```java
@Controller
public class TradingController {

    @MessageMapping("/trade/subscribe/{symbol}")
    public void subscribeToSymbol(
            @DestinationVariable String symbol,
            Principal principal) {

        // Subscribe user to real-time price updates
        messagingTemplate.convertAndSendToUser(
            principal.getName(),
            "/queue/trade/subscribed",
            new SubscriptionResponse(symbol, "subscribed"));

        // Start streaming prices
        priceStreamer.addSubscriber(symbol, principal.getName());
    }

    @MessageMapping("/trade/unsubscribe/{symbol}")
    public void unsubscribeFromSymbol(
            @DestinationVariable String symbol,
            Principal principal) {

        priceStreamer.removeSubscriber(symbol, principal.getName());
    }
}
```

### Scenario 3: Live Notifications System

```
Server Events:
  - Order placed -> /topic/orders/new
  - Payment received -> /user/{userId}/queue/payments
  - System alert -> /topic/admin/alerts
  - User status change -> /topic/users/status
```

## 6. Performance

### Performance Considerations

- **Connection Count**: Each WebSocket consumes memory (~20-50KB per connection).
- **Threading**: Use non-blocking I/O for high connection counts.
- **Message Size**: Keep messages small; compress large payloads.
- **Fragmentation**: Large messages should be fragmented into frames.
- **Heartbeat Frequency**: Balance liveness detection with network overhead.
- **Backpressure**: Implement backpressure to prevent overwhelming clients.

### Scaling WebSockets

```java
@Configuration
public class WebSocketScalabilityConfig {

    @Bean
    public SimpleMessageBrokerConfigurer messageBrokerConfigurer() {
        return registry -> {
            // Use external broker for horizontal scaling
            registry.enableStompBrokerRelay("/topic", "/queue")
                .setRelayHost("rabbitmq.example.com")
                .setRelayPort(61613)
                .setClientLogin("guest")
                .setClientPasscode("guest")
                .setSystemLogin("guest")
                .setSystemPasscode("guest")
                .setSystemHeartbeatSendInterval(10000)
                .setSystemHeartbeatReceiveInterval(10000);
        };
    }
}
```

### Connection Limits

```
Tomcat NIO:  ~10K concurrent connections (default)
Netty:        ~100K+ concurrent connections
With tuning:  ~1M concurrent connections (epoll)
```

## 7. Security

### WebSocket Security Concerns

- **Origin Validation**: Check the Origin header to prevent cross-site hijacking.
- **Authentication**: Authenticate during the handshake (token in header/params).
- **Authorization**: Validate subscriptions and messages.
- **Rate Limiting**: Prevent abuse of real-time endpoints.
- **Input Validation**: Sanitize all message payloads.
- **WSS**: Always use wss:// in production (WebSocket over TLS).

### CSRF Protection for WebSockets

```java
@Component
public class WebSocketCsrfInterceptor implements ChannelInterceptor {

    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(message);

        if (accessor.getCommand() == StompCommand.CONNECT) {
            String csrfToken = accessor.getFirstNativeHeader("X-CSRF-TOKEN");
            if (csrfToken == null || !csrfTokenValidator.isValid(csfcToken)) {
                throw new CsrfException("Invalid CSRF token");
            }
        }

        return message;
    }
}
```

## 8. Common Mistakes

- **Not authenticating at handshake**: Always verify identity before establishing connection.
- **Missing authorization**: Validate that users can access the topics they subscribe to.
- **Not using WSS**: Always encrypt WebSocket traffic in production.
- **Memory leaks**: Clean up sessions on disconnect.
- **No backpressure**: Can overwhelm clients with too many messages.
- **Synchronous processing in listeners**: Never block the event loop.
- **Ignoring heartbeat**: Detect dead connections with heartbeats.
- **Not scaling the broker**: Use external broker for multiple instances.
- **Large messages**: Keep messages small and efficient.

## 9. Senior Engineer Perspective

### When to Use WebSockets

**Good for:**
- Real-time chat and messaging.
- Live notifications and alerts.
- Real-time collaboration (documents, whiteboards).
- Live data feeds (sports scores, stock prices).
- Online gaming.
- Real-time dashboards.

**Bad for:**
- Request-response APIs (use REST/gRPC).
- Batch data transfer.
- Simple CRUD operations.
- Stateless operations.

### Architecture Patterns

```
Direct WebSocket: Raw WebSocket protocol (lowest overhead).
STOMP over WebSocket: Pub-sub messaging model (most common).
RSocket over WebSocket: Reactive streams protocol.
WebRTC: Real-time peer-to-peer communication.
```

### Horizontal Scaling

```
Client -> Load Balancer (sticky sessions) -> WebSocket Server
                                          -> Redis Pub/Sub -> Other WebSocket Servers
                                          -> External Broker (RabbitMQ) ->

For horizontal scaling, use:
1. Sticky sessions (least preferred)
2. External broker (Redis, RabbitMQ, ActiveMQ)
3. Message routing at the application layer
```

## 10. Interview Questions (Easy)

1. What is a WebSocket?
2. How does a WebSocket connection start?
3. What is the difference between HTTP and WebSocket?
4. What is the WebSocket handshake?
5. What is WSS?
6. What is a WebSocket frame?
7. What is the STOMP protocol?
8. What is SockJS?
9. What are the main use cases for WebSockets?
10. What HTTP status code is returned for a successful WebSocket upgrade?

## Medium

1. How do you authenticate WebSocket connections?
2. What is the difference between WebSocket and SSE (Server-Sent Events)?
3. How do you scale WebSocket applications horizontally?
4. What is the purpose of heartbeat messages in WebSockets?
5. How does Spring Boot support WebSocket messaging?
6. What is the difference between `@SendTo` and `SimpMessagingTemplate`?
7. How do you handle WebSocket reconnection?
8. What is a STOMP frame structure?
9. How do you implement user-specific messaging in STOMP?
10. What are WebSocket sub-protocols?

## 11. Advanced Interview Questions (Hard)

1. Design a real-time collaborative document editor with WebSockets (like Google Docs).
2. How would you implement presence detection and typing indicators across a WebSocket cluster?
3. Design a WebSocket-based distributed rate limiter.
4. How do you handle WebSocket reconnection with message recovery (no lost messages)?
5. Implement a custom sub-protocol for a real-time multiplayer game.
6. Design a WebSocket-based notification system with delivery guarantees.
7. How would you implement WebSocket clustering using Redis Pub/Sub?
8. Design a backpressure mechanism for WebSocket message streaming.
9. How do you handle large-scale WebSocket deployments (1M+ connections)?
10. Implement a WebSocket health check and auto-recovery system.

## System Design

1. Design a real-time chat system for 10M users using WebSockets.
2. Design a real-time collaboration platform (like Figma or Miro).
3. Design a live streaming analytics dashboard with WebSockets.
4. Design a real-time multiplayer game server using WebSockets.
5. Design a real-time notification system for a social media platform.
6. Design a WebSocket-based live location tracking system.
7. Design a real-time customer support chat platform.
8. Design a WebSocket-based auction bidding system.
9. Design a real-time collaborative code editor.
10. Design a WebSocket-based IoT device monitoring dashboard.

## 12. Expert-Level Interview Questions (Architect-Level)

1. Design a globally distributed WebSocket infrastructure that handles 10M+ concurrent connections with geographic load balancing, automatic failover, and sub-100ms latency.
2. How would you build a WebSocket-based platform that supports millions of concurrent users with exactly-once message delivery guarantees?
3. Design a hybrid real-time system that seamlessly transitions between WebSocket, SSE, and long-polling based on client capabilities and network conditions.
4. How would you implement a WebSocket design system that supports versioned protocols for backward compatibility during rolling deployments?
5. Design a real-time event sourcing system using WebSockets where clients can subscribe to event streams and replay historical events.
6. How would you build a WebSocket-based BFF (Backend for Frontend) that aggregates data from multiple microservices into a single real-time stream?
7. Design a WebSocket system that transparently handles network partitions, reconnection storms, and message deduplication at scale.
8. How would you implement end-to-end encryption for WebSocket messages where the server cannot decrypt the content?
9. Design a WebSocket monitoring and observability platform that tracks per-connection metrics, message latency, and error rates across a distributed cluster.
10. How would you build a multi-tenant WebSocket platform with per-tenant rate limiting, connection limits, and resource isolation?

## 13. Debugging & Troubleshooting

### Common Issues

- **Connection drops**: Check network stability, firewalls, timeouts.
- **Handshake failure**: Check origin, sub-protocols, authentication.
- **WSS errors**: Verify TLS certificate, check cipher suites.
- **Message not received**: Verify destination, subscription, authorization.
- **High memory usage**: Check for session leaks, message buffering.
- **Cross-origin issues**: Verify CORS settings for WebSocket.
- **STOMP frame errors**: Check frame format and headers.

### Debugging with Browser DevTools

```javascript
// Monitor WebSocket frames in browser
// Chrome DevTools -> Network -> WS tab
// Firefox DevTools -> Network -> WebSocket

// Log all STOMP frames
stompClient.onWebSocketClose = (frame) => {
    console.log('WebSocket closed:', frame);
};

stompClient.debug = function(str) {
    console.log('STOMP:', str);
};
```

### WebSocket Session Monitoring

```java
@Component
public class WebSocketMonitor {

    private final SimpUserRegistry userRegistry;

    public WebSocketMonitor(SimpUserRegistry userRegistry) {
        this.userRegistry = userRegistry;
    }

    public WebSocketStats getStats() {
        Set<SimpUser> users = userRegistry.getUsers();
        long sessionCount = users.stream()
            .mapToLong(u -> u.getSessions().size())
            .sum();

        return new WebSocketStats(users.size(), sessionCount);
    }

    public record WebSocketStats(int uniqueUsers, long totalSessions) {}
}
```

## 14. Comparison Section

### WebSocket vs SSE (Server-Sent Events)

| Aspect | WebSocket | SSE |
|--------|-----------|-----|
| Communication | Full-duplex | Server to client only |
| Protocol | ws:// / wss:// | HTTP streaming |
| Binary Data | Yes | Text only (EventSource) |
| Auto-Reconnect | Manual | Built-in |
| Browser Support | Universal | Except IE/Edge legacy |
| Maximum Connections | Unlimited | 6 per domain (HTTP/1.1) |
| Complexity | Higher | Lower |

### WebSocket vs Polling

| Aspect | WebSocket | Polling |
|--------|-----------|---------|
| Latency | Real-time | Polling interval |
| Server Load | Lower | Higher (many requests) |
| Bandwidth | Lower (no headers) | Higher (HTTP headers) |
| Complexity | Higher | Lower |
| Real-time | Yes | No (bounded by interval) |

## 15. Revision Notes

- WebSocket: full-duplex, persistent, bidirectional, low-latency
- Starts with HTTP upgrade (101 Switching Protocols)
- RFC 6455 standard; WSS for encrypted connections
- STOMP provides pub-sub on top of WebSocket
- Spring Boot: `@EnableWebSocketMessageBroker`, `@MessageMapping`, `@SendTo`
- SimpMessagingTemplate for sending messages from server
- Use external broker (RabbitMQ, Redis) for horizontal scaling
- Always authenticate and authorize WebSocket connections
- Handle reconnection, backpressure, and heartbeat

## 16. Cheat Sheet

```
+------------------------------------------------------------------+
| WEBSOCKET CHEAT SHEET                                            |
+------------------------------------------------------------------+
| HANDSHAKE                                                        |
|   Client: GET /ws HTTP/1.1                                       |
|           Upgrade: websocket                                     |
|           Connection: Upgrade                                    |
|           Sec-WebSocket-Key: <base64-encoded-16-bytes>           |
|           Sec-WebSocket-Version: 13                              |
|   Server: 101 Switching Protocols                                |
|           Upgrade: websocket                                     |
|           Connection: Upgrade                                    |
|           Sec-WebSocket-Accept: <sha1-hash>                     |
+------------------------------------------------------------------+
| STOMP FRAMES                                                     |
|   CONNECT     -> Establish connection                            |
|   SUBSCRIBE   -> Subscribe to destination                        |
|   SEND        -> Send message to destination                     |
|   MESSAGE     -> Message received from subscription               |
|   DISCONNECT  -> Close connection                                |
+------------------------------------------------------------------+
| SPRING BOOT ANNOTATIONS                                          |
|   @MessageMapping("/path")          -- Handle STOMP messages     |
|   @SendTo("/topic/...")             -- Send to all subscribers   |
|   @SendToUser("/queue/...")         -- Send to specific user     |
|   @DestinationVariable              -- Path variable             |
|   @Payload                          -- Message body              |
|   @Header                           -- STOMP header              |
+------------------------------------------------------------------+
| DESTINATION PREFIXES                                             |
|   /app   -> Messages from client to server (handled by @MessageMapping) |
|   /topic -> Pub-sub (1 producer, N consumers)                    |
|   /queue -> Point-to-point (1 producer, 1 consumer)              |
|   /user  -> User-specific messages (routed to single user)       |
+------------------------------------------------------------------+
| TYPICAL CONFIG                                                   |
|   @EnableWebSocketMessageBroker                                  |
|   registry.enableSimpleBroker("/topic", "/queue")                |
|   registry.setApplicationDestinationPrefixes("/app")             |
|   registry.addEndpoint("/ws").withSockJS()                       |
+------------------------------------------------------------------+
| SCALING                                                          |
|   Simple:  In-memory broker (single instance)                    |
|   Scaled:  External broker (RabbitMQ STOMP, Redis)               |
|   Global:  Load balancer + external broker per region            |
+------------------------------------------------------------------+
| SECURITY                                                         |
|   Always use WSS in production                                   |
|   Authenticate during CONNECT                                    |
|   Authorize SUBSCRIBE and SEND                                   |
|   Validate Origin header                                         |
|   Sanitize all message payloads                                  |
+------------------------------------------------------------------+
```
