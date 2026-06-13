# gRPC

---

## Overview

- **Definition:** gRPC is a high-performance, open-source RPC (Remote Procedure Call) framework developed by Google. It uses HTTP/2 for transport, Protocol Buffers for serialization, and generates client/server code from `.proto` files. It supports four communication patterns: unary, server streaming, client streaming, and bidirectional streaming.

- **Why It Exists:** Traditional REST/JSON APIs are text-based, slow to serialize, and lack native streaming. gRPC provides binary serialization (Protobuf), HTTP/2 multiplexing, built-in code generation, and native streaming — making it ideal for microservices communication, real-time systems, and polyglot environments.

- **Communication Patterns:**
  - **Unary** — client sends one request, server sends one response
  - **Server Streaming** — client sends one request, server streams responses
  - **Client Streaming** — client streams requests, server sends one response
  - **Bidirectional Streaming** — both sides stream simultaneously

---

## Core Concepts

- Define a service in a `.proto` file (service contract)
- Generate client and server code using `protoc`
- Server implements the generated service interface
- Client uses a stub to call server methods
- Data is serialized as Protocol Buffers (binary format)
- Transport is over HTTP/2 with multiplexed streams

- **HTTP/2 Features Used by gRPC:**
  - **Multiplexing** — multiple streams over a single TCP connection
  - **Header Compression** — HPACK compression reduces overhead
  - **Binary Frames** — more efficient than HTTP/1.1 text frames
  - **Flow Control** — prevents fast senders from overwhelming slow receivers
  - **Stream Prioritization** — critical streams get priority

### Protobuf Service Definition

```protobuf
syntax = "proto3";
package com.example.orders;
option java_package = "com.example.orders.proto";
option java_multiple_files = true;

message Order {
    string id = 1;
    string user_id = 2;
    repeated OrderItem items = 3;
    double total_amount = 4;
    OrderStatus status = 5;
}

enum OrderStatus {
    ORDER_STATUS_UNSPECIFIED = 0;
    ORDER_STATUS_PENDING = 1;
    ORDER_STATUS_CONFIRMED = 2;
    ORDER_STATUS_SHIPPED = 3;
    ORDER_STATUS_DELIVERED = 4;
    ORDER_STATUS_CANCELLED = 5;
}

message CreateOrderRequest { string user_id = 1; repeated OrderItem items = 2; string shipping_address = 3; }
message GetOrderRequest { string id = 1; }
message ListOrdersRequest { string user_id = 1; int32 page = 2; int32 size = 3; }
message ListOrdersResponse { repeated Order orders = 1; int32 total_count = 2; bool has_more = 3; }

service OrderService {
    rpc CreateOrder (CreateOrderRequest) returns (Order);
    rpc GetOrder (GetOrderRequest) returns (Order);
    rpc ListOrders (ListOrdersRequest) returns (ListOrdersResponse);
    rpc StreamOrders (OrderStreamRequest) returns (stream Order);
    rpc BulkCreateOrders (stream CreateOrderRequest) returns (ListOrdersResponse);
    rpc OrderChat (stream OrderNote) returns (stream OrderNote);
}
```

### Server Implementation (Spring Boot)

```java
@GrpcService
public class OrderGrpcService extends OrderServiceGrpc.OrderServiceImplBase {
    private final OrderService orderService;
    private final OrderProtoMapper protoMapper;

    public OrderGrpcService(OrderService orderService, OrderProtoMapper protoMapper) {
        this.orderService = orderService;
        this.protoMapper = protoMapper;
    }

    @Override
    public void createOrder(CreateOrderRequest request, StreamObserver<Order> responseObserver) {
        try {
            OrderEntity created = orderService.create(protoMapper.toEntity(request));
            responseObserver.onNext(protoMapper.toProto(created));
            responseObserver.onCompleted();
        } catch (Exception e) {
            responseObserver.onError(Status.INTERNAL.withDescription(e.getMessage()).asRuntimeException());
        }
    }

    @Override
    public void getOrder(GetOrderRequest request, StreamObserver<Order> responseObserver) {
        try {
            OrderEntity entity = orderService.findById(request.getId());
            if (entity == null) {
                responseObserver.onError(Status.NOT_FOUND.withDescription("Order not found").asRuntimeException());
                return;
            }
            responseObserver.onNext(protoMapper.toProto(entity));
            responseObserver.onCompleted();
        } catch (Exception e) {
            responseObserver.onError(Status.INTERNAL.withDescription(e.getMessage()).asRuntimeException());
        }
    }

    @Override
    public StreamObserver<CreateOrderRequest> bulkCreateOrders(
            StreamObserver<ListOrdersResponse> responseObserver) {
        return new StreamObserver<>() {
            private final List<OrderEntity> orders = new ArrayList<>();

            @Override
            public void onNext(CreateOrderRequest request) {
                orders.add(orderService.create(protoMapper.toEntity(request)));
            }

            @Override
            public void onError(Throwable t) { responseObserver.onError(t); }

            @Override
            public void onCompleted() {
                ListOrdersResponse.Builder response = ListOrdersResponse.newBuilder()
                    .setTotalCount(orders.size());
                orders.stream().map(protoMapper::toProto).forEach(response::addOrders);
                responseObserver.onNext(response.build());
                responseObserver.onCompleted();
            }
        };
    }
}
```

### Client Implementation

```java
@Service
public class OrderGrpcClient {
    private final OrderServiceGrpc.OrderServiceBlockingStub blockingStub;
    private final OrderServiceGrpc.OrderServiceStub asyncStub;

    public OrderGrpcClient(ManagedChannel channel) {
        this.blockingStub = OrderServiceGrpc.newBlockingStub(channel);
        this.asyncStub = OrderServiceGrpc.newStub(channel);
    }

    public Order createOrder(CreateOrderRequest request) { return blockingStub.createOrder(request); }

    public Order getOrder(String id) {
        try {
            return blockingStub.getOrder(GetOrderRequest.newBuilder().setId(id).build());
        } catch (StatusRuntimeException e) {
            if (e.getStatus().getCode() == Status.Code.NOT_FOUND)
                throw new ResourceNotFoundException("Order not found: " + id);
            throw new GrpcClientException("gRPC call failed", e);
        }
    }

    public void streamOrders(String userId, Consumer<Order> consumer) {
        Iterator<Order> orders = blockingStub.streamOrders(
            OrderStreamRequest.newBuilder().setUserId(userId).build());
        orders.forEachRemaining(consumer);
    }
}
```

### Channel Configuration

```java
@Configuration
public class GrpcClientConfig {
    @Bean
    public ManagedChannel orderServiceChannel() {
        return ManagedChannelBuilder.forAddress("localhost", 9090)
            .usePlaintext()  // For dev only; use TLS in production
            .keepAliveTime(30, TimeUnit.SECONDS)
            .keepAliveTimeout(10, TimeUnit.SECONDS)
            .maxInboundMessageSize(4 * 1024 * 1024)  // 4MB
            .enableRetry()
            .build();
    }
}
```

### JWT Authentication Interceptor

```java
@Component
public class JwtAuthInterceptor implements ClientInterceptor {
    private final JwtTokenProvider tokenProvider;

    public JwtAuthInterceptor(JwtTokenProvider tokenProvider) { this.tokenProvider = tokenProvider; }

    @Override
    public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(
            MethodDescriptor<ReqT, RespT> method, CallOptions callOptions, Channel next) {
        return new ForwardingClientCall.SimpleForwardingClientCall<ReqT, RespT>(next.newCall(method, callOptions)) {
            @Override
            public void start(Listener<RespT> responseListener, Metadata headers) {
                headers.put(Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER),
                    "Bearer " + tokenProvider.getAccessToken());
                super.start(responseListener, headers);
            }
        };
    }
}
```

- **When to Use gRPC:**
  - Internal microservices communication
  - Real-time streaming systems
  - Polyglot environments (multiple languages)
  - Low-latency, high-throughput systems
  - Mobile applications (reduced payload size)

- **When to Avoid:**
  - Public REST APIs (browser clients)
  - Simple CRUD applications
  - Serverless functions

- **Proto Evolution Best Practices:**
  - Never change field numbers
  - Add new fields with new numbers only
  - Use `reserved` for removed fields
  - Set sensible defaults for new fields
  - Use `optional` for fields that may be absent
  - Version your packages in proto files

- **Performance:**
  - 2-5x faster than REST for simple requests
  - 10x faster for streaming use cases
  - 70-80% smaller payloads (Protobuf binary)
  - Connection pooling and keepalive pings

---

## Common Mistakes

- **Using plaintext in production** — always use TLS.
  - **Why it looks correct:** in development and internal networks, plaintext works fine and avoids certificate management overhead, so teams assume their private network provides sufficient protection.
- **Ignoring connection management** — failing to reuse channels and shut down properly.
  - **Why it looks correct:** creating a new channel per request still works functionally and is the simplest pattern to write, hiding the resource leak until connection counts exhaust the OS file descriptor limit.
- **Large protobuf messages** — keep under 4MB; use streaming for large data.
  - **Why it looks correct:** Protobuf is binary and fast, so developers assume any reasonable payload size is fine — the 4MB limit and its effect on memory allocation are hidden until a service runs out of heap.
- **Breaking proto field numbering** — never reuse field numbers when removing fields.
  - **Why it looks correct:** field numbers look like arbitrary labels, and reusing a freed number seems efficient — the silent data corruption only surfaces when old and new binaries exchange messages with the same number pointing to different fields.
- **No error handling on client** — always handle `StatusRuntimeException`.
  - **Why it looks correct:** in demos and tests, gRPC calls always succeed, so exception handling seems like boilerplate — until a network blip or server restart causes an unexplained crash in production.
- **Blocking on streaming** — use async stubs for streaming calls.
  - **Why it looks correct:** blocking stubs are simpler to write with a familiar synchronous programming model, and the thread-per-stream cost is invisible until hundreds of concurrent streams exhaust the thread pool.
- **Not setting deadlines** — always set timeouts on gRPC calls.
  - **Why it looks correct:** the service usually responds within milliseconds, so a timeout seems unnecessary — until a downstream service hangs and the caller waits forever, accumulating threads and connections until the entire system freezes.
- **Missing Proto backward compatibility** — follow proto evolution best practices.
  - **Why it looks correct:** adding a new field seems harmless — old binaries ignore unknown fields — but removing or renaming a field without using `reserved` creates silent data corruption that may not be noticed until financial reports are wrong.

---

## Real-World Scenarios

### Scenario 1: gRPC Streaming for Real-Time Order Tracking
- **Context:** A food delivery app needs to show real-time driver location on a map. The mobile client polls `GET /api/v1/orders/{id}/location` every 2 seconds. This creates 30 requests per minute per user. With 10,000 active users, the REST endpoint handles 300,000 polling requests per minute — most returning the same data. Server CPU is at 90% from request overhead alone.
- **Resolution:** Switch to gRPC server streaming. The mobile client establishes a gRPC connection and calls a server-streaming RPC. The server pushes location updates only when the driver's position changes (every 3-5 seconds instead of every 2 seconds of polling). This eliminates polling overhead, reduces server load by 80%, and provides near-real-time updates.

```protobuf
service OrderTrackingService {
    rpc TrackOrder(TrackOrderRequest) returns (stream OrderLocationUpdate);
}

message OrderLocationUpdate {
    double latitude = 1;
    double longitude = 2;
    int64 timestamp = 3;
    DeliveryStatus status = 4;
}
```

- The client receives push updates over a single persistent HTTP/2 connection — no polling, no repeated TLS handshakes, no HTTP request/response overhead.

### Scenario 2: gRPC Deadline Exceeded Cascading Failures
- **Context:** Service A calls Service B with a 5-second deadline. Service B calls Service C with a 4-second deadline. Service C has a temporary slowdown (3-second response time instead of the usual 100ms). Service B's deadline expires before Service C responds. Service A retries, sending more traffic to B. B accumulates requests, its deadline expires, and the failure cascades. All downstream services are overwhelmed by retries.
- **Resolution:** (1) Increase the deadline at the outermost service (Service A: 10s, Service B: 8s, Service C: 5s) — cascading deadlines must be decreasing. (2) Implement retry with exponential backoff and jitter — don't retry immediately. (3) Configure per-method deadlines: fast operations (100ms) get 1s deadlines; slow operations (2s) get 5s deadlines. (4) Use a circuit breaker: if Service C's error rate exceeds 50%, Service B fails fast without calling C. (5) Track deadline propagation: gRPC propagates deadlines across services — ensure each hop has enough remaining time.

### Scenario 3: Protobuf Backward Compatibility Failure
- **Context:** A team adds a `discount` field to the `Order` protobuf message with field number 10. A month later, another team removes the `discount` field (no longer needed) and adds a `notes` field using field number 10 (reusing the freed number). Services that still have the old proto definition receive an `Order` message where field number 10 contains `notes` — they interpret it as `discount`. Users see order notes displayed as discount values. Financial reports are corrupted.
- **Resolution:** Never reuse field numbers. Use `reserved` for removed fields.

```protobuf
message Order {
    // Removed fields
    reserved 10;                    // Was discount, removed in v2.1
    reserved "discount";            // Also reserve the name to prevent reuse

    string id = 1;
    string user_id = 2;
    // ...
    string notes = 11;              // New field with new number
}
```

- The `reserved` keyword prevents field numbers and names from being reused, catching the error at compile time.

---

## Scenario-Based Questions

- **Q: You need real-time driver location updates for a food delivery app with 50,000 concurrent users. REST polling is too expensive. Which gRPC pattern do you use and how do you handle 50,000 concurrent connections?**
  - A: Use server streaming — the client calls `TrackOrder(TrackOrderRequest)` and the server streams `OrderLocationUpdate` messages. Each connection stays open and receives push updates. For 50K concurrent connections: (1) gRPC over HTTP/2 multiplexes streams over fewer TCP connections. (2) Use an async server with a non-blocking I/O model (Netty, not Tomcat). (3) Set `keepAliveTime` (30s) and `keepAliveTimeout` (10s) to detect dead connections. (4) Use a connection pool on the server side. (5) Consider using a dedicated push service or a service mesh that handles connection management.
  - **Follow-up:** When 50,000 mobile clients hold persistent HTTP/2 connections, how do you roll out a server update without disconnecting every single client simultaneously?

- **Q: Your gRPC service returns `DEADLINE_EXCEEDED` errors during peak traffic. The service's P50 latency is 50ms, but deadlined requests have a 500ms deadline. What's causing this, and how do you fix it?**
  - A: The server is likely experiencing head-of-line blocking or connection pool exhaustion. Causes: (1) Server thread pool is saturated — requests are queued before processing. Fix: increase worker threads or switch to async processing. (2) gRPC channel is shared — a slow streaming call blocks other requests on the same channel. Fix: use separate channels for streaming and unary calls. (3) Database connection pool exhaustion — requests wait for DB connections. Fix: increase pool size or optimize queries. (4) The deadline is too tight: with P50 at 50ms, a 500ms deadline should be fine at low utilization, but at high utilization, queueing adds latency. Use Little's Law to calculate appropriate deadlines: `deadline = p99_latency × 2`.

- **Q: You need to add a `shippingAddress` field to an `Order` protobuf message consumed by 50 services. How do you ensure zero downtime during deployment?**
  - A: (1) Add the new field with a new field number: `string shipping_address = 11;`. (2) Compile and deploy all 50 services with the updated proto definition. (3) The old services ignore unknown fields — `shippingAddress` won't be present in their messages. (4) New services can write the new field; old services preserve it when re-serializing (protobuf's unknown field preservation). (5) First deploy services that write the new field, then services that read it (or all at once). (6) Never make a new field required — always keep it optional with a sensible default.
  - **Follow-up:** What happens when a service compiled with the old proto definition receives a message where `shipping_address` is set — does it silently drop the data or preserve it through re-serialization?

- **Q: Your system uses gRPC for internal microservices communication but also needs to expose some services to external REST clients. How do you support both protocols without maintaining two implementations?**
  - A: Use gRPC Gateway (grpc-gateway): annotate your proto file with `google.api.http` options that define REST endpoints and JSON mapping. The gateway generates a reverse proxy that translates REST/JSON to gRPC/Protobuf. Run the gateway alongside your gRPC server. External clients use REST; internal services use gRPC. The business logic exists only in the gRPC server. Added benefit: the gateway also generates OpenAPI specs for REST clients.

- **Q: Your gRPC bidirectional streaming chat application loses messages under load. Users report that messages sent are not received by the other party. What's the root cause and fix?**
  - A: Likely flow control or backpressure issues. Causes: (1) The sender's `onNext` calls are failing silently — gRPC's flow control blocks the sender if the receiver is slow, and `onNext` throws `StatusRuntimeException` if the stream is cancelled. Fix: wrap `onNext` in try-catch and implement backpressure handling. (2) The server's `StreamObserver` is not thread-safe — concurrent calls from multiple threads cause data races and lost messages. Fix: synchronize `onNext` calls or use a dedicated executor. (3) Message acknowledgment: add a simple ACK pattern where the receiver sends an acknowledgment message for each received message. If the sender doesn't receive an ACK within a timeout, retry.
  - **Follow-up:** If you add ACKs to a bidirectional stream, how do you prevent the ACK messages themselves from competing with data messages for flow control credits, potentially causing a deadlock?

- **Q: How do you implement authentication in gRPC between microservices in a service mesh?**
  - A: (1) mTLS for service-to-service authentication: each service has a client certificate signed by the mesh CA. The server validates the client certificate on every connection. Istio/Linkerd can enforce this at the proxy level. (2) For user-level auth: pass JWT/OAuth2 tokens in gRPC metadata headers (`authorization: Bearer <token>`). Implement client and server interceptors that extract/validate tokens. (3) In the interceptor: extract token from metadata → validate signature and claims → set the authenticated principal in the gRPC Context. (4) Use SPIRE for workload identity: each service gets a unique SPIFFE ID that can be used for authentication.

- **Q: A protobuf field numbered 5 was removed 6 months ago. A new developer adds a field with number 5. Old services still running from 6 months ago decode field 5 as the old field, interpreting the new data incorrectly. How do you prevent this?**
  - A: Use `reserved 5;` in the proto file. The `reserved` keyword prevents any future use of field number 5 (or field name). The protobuf compiler will reject any attempt to reuse it. Best practice: whenever removing a field, immediately add it to `reserved`. Don't rely on team memory or documentation — enforce it in the proto definition. Also use `reserved` for removed field names to prevent accidental reuse of the same name with a different number.

- **Q: You need to transfer a 500MB video file from a mobile client to a server via gRPC. The default max message size is 4MB. How do you design this?**
  - A: Use client streaming to chunk the file. Define a `FileChunk` message with `bytes data`, `chunk_index`, and `total_chunks`. The client streams chunks; the server reassembles. Set `maxInboundMessageSize` to 8MB (large enough for efficient chunks, small enough to avoid memory pressure). Use flow control to prevent the client from overwhelming the server. Consider compression (gzip) at the gRPC level. For very large files, generate a pre-signed S3 URL instead of transferring through gRPC.

- **Q: Your gRPC client calls a service that occasionally returns `UNAVAILABLE` (transient network error). How do you implement retry logic?**
  - A: (1) Enable `enableRetry()` on the ManagedChannel. (2) Configure retry policy in the service config JSON: `{ "methodConfig": [{ "name": [{}], "retryPolicy": { "maxAttempts": 3, "initialBackoff": "0.1s", "maxBackoff": "1s", "backoffMultiplier": 2, "retryableStatusCodes": ["UNAVAILABLE"] } }] }`. (3) For custom logic, implement a client interceptor that catches `StatusRuntimeException` and retries with exponential backoff with jitter. (4) Only retry for idempotent operations (read methods). For mutations, use idempotency keys. (5) Set a maximum retry limit (3-5 attempts) to prevent retry storms during outages.

- **Q: Your gRPC services are experiencing cascading failures — one slow service causes all upstream services to exhaust their resources. How do you implement circuit breaking?**
  - A: (1) Use a client interceptor that tracks the success/failure ratio per method over a sliding window (e.g., last 100 requests). (2) If the failure rate exceeds a threshold (e.g., 50%), open the circuit — fail fast with `Status.UNAVAILABLE` without calling the downstream service. (3) Periodically transition to half-open: allow a single probe request to check if the service recovered. (4) If the probe succeeds, close the circuit. If it fails, stay open. (5) Integrate with a service mesh (Istio's `DestinationRule` with circuit breaker) for proxy-level enforcement. (6) Use Resilience4j or Hystrix (legacy) for application-level circuit breaking.

---

## Interview Questions

- **Q: What is gRPC and what are its key advantages over REST?**
  - A: gRPC is a high-performance RPC framework using HTTP/2, Protocol Buffers, and code generation. Advantages: binary serialization (smaller, faster), HTTP/2 multiplexing (single connection), native streaming (server, client, bidirectional), strong typing via proto files, and multi-language code generation.

- **Q: What are the four gRPC communication patterns?**
  - A: Unary (one request, one response), Server streaming (one request, stream of responses), Client streaming (stream of requests, one response), Bidirectional streaming (both sides stream independently).

- **Q: How does Protobuf ensure backward compatibility?**
  - A: Fields are identified by number, not name. New fields can be added with new numbers. Old clients ignore unknown fields. Removed fields must use `reserved` to prevent number reuse. Never change field types or numbers.

- **Q: What is the N+1 problem in gRPC?**
  - A: When a gRPC response contains a list of entities, and the client makes individual RPCs for each entity's related data. Mitigation: design composite RPCs that return nested data, use batch endpoints, or implement a GraphQL layer on top.

- **Q: How do you handle errors in gRPC?**
  - A: Use gRPC status codes (OK, NOT_FOUND, INVALID_ARGUMENT, UNAVAILABLE, DEADLINE_EXCEEDED, etc.) with descriptive error messages and optional details. Never use HTTP status codes — gRPC has its own error model.

- **Q: What is gRPC deadline propagation?**
  - A: A deadline set on the client RPC is propagated to downstream services automatically. Each service checks the remaining time before processing. If the deadline expires, the request is cancelled. This prevents resource waste from already-timed-out requests.

- **Q: How does gRPC handle authentication?**
  - A: mTLS for service-to-service (certificate-based), JWT/OAuth2 tokens in metadata for user-level auth, interceptors for centralized token validation. gRPC supports SSL/TLS natively.

- **Q: What is the difference between blocking and async stubs?**
  - A: Blocking stubs block the calling thread until the response arrives (simple, but uses threads). Async stubs return immediately and call a callback when the response arrives (non-blocking, better for streaming and high concurrency).

- **Q: How do you implement retry logic in gRPC?**
  - A: Configure retry policy in the service config JSON with exponential backoff, max attempts (3-5), and retryable status codes (UNAVAILABLE, RESOURCE_EXHAUSTED). Only retry idempotent operations. Enable `enableRetry()` on the channel.

- **Q: When should you use gRPC vs REST?**
  - A: gRPC for internal microservices (high throughput, low latency, streaming, polyglot environments). REST for public APIs (browser clients, simple CRUD, caching-heavy use cases, diverse client ecosystem). Use gRPC Gateway to expose gRPC services as REST.

---

## Developer Recommendations

- **Always set deadlines on gRPC calls** — Without deadlines, a gRPC client will wait forever for a response. Server crashes, network partitions, and slow deployments will make the client hang indefinitely, accumulating resources. Every RPC call must have a deadline: `stub.withDeadlineAfter(5, TimeUnit.SECONDS).call()`. Deadlines also propagate to downstream services, providing end-to-end timeout. Without deadlines, a single stuck service can cascade to all callers.
  - **Production story:** A production incident at a streaming platform: a deployment script restarted the downstream service, but no gRPC deadline was set — all upstream services hung on connections to the old instance, exhausting their thread pools within 90 seconds and taking down three unrelated services before anyone identified the root cause.

- **Use the `reserved` keyword for removed protobuf fields** — Protobuf identifies fields by number, not name. Reusing a field number after removing a field causes silent data corruption — old binaries decode the new field as the old type. Always add `reserved <number>;` and `reserved "<name>";` when removing a field. The compiler will prevent reuse. This is a one-line change that prevents production data corruption. There is no good reason to skip it.
  - **Production story:** A fintech company learned this the hard way: a team removed a `discount_percent` field and reused its number for `notes` — old services reading `notes` as `discount_percent` applied phantom discounts for three days before the discrepancy in revenue reports was traced back to the proto change.

- **Prefer server streaming over client polling for real-time data** — Polling (HTTP or gRPC unary) wastes server resources: each poll requires TLS handshake, request parsing, and response serialization — most returning the same data. gRPC server streaming maintains a single persistent connection and pushes updates only when data changes. This reduces server load by 60-90% for real-time features and provides lower latency updates. The trade-off: managing persistent connections (keep-alive, reconnection, flow control). Acceptable for the efficiency gain.

- **Use async stubs for streaming and high-concurrency scenarios** — Blocking stubs tie up a thread per in-flight request. With 1000 concurrent requests, you need 1000 threads — leading to thread pool exhaustion and context switching overhead. Async stubs are non-blocking: one thread handles many concurrent requests via callbacks. For unary calls with low concurrency (<100 req/s), blocking stubs are simpler. For streaming or high concurrency, async stubs are essential.

- **Implement circuit breakers for gRPC service-to-service calls** — A single slow downstream service can exhaust the thread pool and connection pool of all upstream services, causing cascading failures. Circuit breakers detect failure patterns and fail fast without calling the degraded service. Use a client interceptor that tracks failure rates over a sliding window. Open the circuit when errors exceed 50% in the last 100 requests. Periodically probe for recovery. This prevents cascading failures and allows services to recover under reduced load.

- **Use gRPC Gateway for REST clients** — gRPC's binary protocol is not suitable for public REST APIs (browser clients, third-party integrations). gRPC Gateway generates a REST/JSON reverse proxy from proto annotations. This lets you maintain one service implementation that supports both gRPC (internal, efficient) and REST (external, universal). The gateway handles all protocol translation — no need to maintain parallel REST implementations.
