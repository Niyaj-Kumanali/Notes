# gRPC

## 1. Executive Summary

gRPC is a high-performance, open-source RPC (Remote Procedure Call) framework developed by Google. It uses HTTP/2 for transport, Protocol Buffers for serialization, and generates client/server code from .proto files. gRPC supports four communication patterns: unary, server streaming, client streaming, and bidirectional streaming. It is widely used in microservices architectures, real-time systems, and polyglot environments due to its performance, type safety, and cross-language support.

## 2. Core Theory

### How gRPC Works

1. Define a service in a `.proto` file (service contract).
2. Generate client and server code using `protoc`.
3. Server implements the service interface.
4. Client uses a stub to call server methods.
5. Data is serialized as Protocol Buffers (binary format).
6. Transport is over HTTP/2 with multiplexed streams.

### Protocol Buffers

```protobuf
syntax = "proto3";

package com.example.user;

option java_multiple_files = true;
option java_package = "com.example.user.proto";

message User {
    string id = 1;
    string name = 2;
    string email = 3;
    int32 age = 4;
    repeated string roles = 5;
    Address address = 6;
    google.protobuf.Timestamp created_at = 7;
}

message Address {
    string street = 1;
    string city = 2;
    string country = 3;
    string zip_code = 4;
}

message GetUserRequest {
    string id = 1;
}

message ListUsersRequest {
    int32 page = 1;
    int32 size = 2;
}

message ListUsersResponse {
    repeated User users = 1;
    int32 total_count = 2;
    bool has_more = 3;
}

message CreateUserRequest {
    string name = 1;
    string email = 2;
    int32 age = 3;
}

message DeleteUserRequest {
    string id = 1;
}

message DeleteUserResponse {
    bool success = 1;
}
```

### gRPC Communication Patterns

```protobuf
service UserService {
    // Unary: Client sends one request, server sends one response
    rpc GetUser (GetUserRequest) returns (User);

    // Server streaming: Client sends one request, server streams responses
    rpc ListUsers (ListUsersRequest) returns (stream User);

    // Client streaming: Client streams requests, server sends one response
    rpc CreateUsers (stream CreateUserRequest) returns (CreateUsersResponse);

    // Bidirectional streaming: Both sides stream
    rpc Chat (stream ChatMessage) returns (stream ChatMessage);
}
```

## 3. Under-the-Hood Deep Dive

### HTTP/2 Features Used by gRPC

- **Multiplexing**: Multiple streams over a single TCP connection.
- **Server Push**: Proactive resource delivery.
- **Header Compression**: HPACK compression reduces overhead.
- **Binary Frames**: More efficient than HTTP/1.1 text frames.
- **Stream Prioritization**: Critical streams get priority.
- **Flow Control**: Prevents fast sender from overwhelming slow receiver.

### gRPC Wire Protocol

```
gRPC Frame:
| Length (4 bytes) | Flags (1 byte) | Stream ID (7 bytes) | Data |

Proto3 Encoding:
- Varint: Variable-length integer encoding
- ZigZag: Efficient signed integer encoding
- Packed repeated fields: Length-delimited list
```

### gRPC Lifecycle

1. Client creates a channel (connection to server).
2. Client creates a stub from the channel.
3. Client calls the RPC method on the stub.
4. HTTP/2 stream is created.
5. Protobuf messages are serialized and sent as frames.
6. Server deserializes, processes, and sends response.
7. Stream is closed.

## 4. Production Code Examples

### Protobuf Service Definition

```protobuf
syntax = "proto3";

package com.example.orders;

import "google/protobuf/empty.proto";
import "google/protobuf/timestamp.proto";

option java_package = "com.example.orders.proto";
option java_multiple_files = true;

enum OrderStatus {
    ORDER_STATUS_UNSPECIFIED = 0;
    ORDER_STATUS_PENDING = 1;
    ORDER_STATUS_CONFIRMED = 2;
    ORDER_STATUS_SHIPPED = 3;
    ORDER_STATUS_DELIVERED = 4;
    ORDER_STATUS_CANCELLED = 5;
}

message OrderItem {
    string product_id = 1;
    string product_name = 2;
    int32 quantity = 3;
    double unit_price = 4;
}

message Order {
    string id = 1;
    string user_id = 2;
    repeated OrderItem items = 3;
    double total_amount = 4;
    OrderStatus status = 5;
    string shipping_address = 6;
    google.protobuf.Timestamp created_at = 7;
    google.protobuf.Timestamp updated_at = 8;
}

message CreateOrderRequest {
    string user_id = 1;
    repeated OrderItem items = 2;
    string shipping_address = 3;
}

message GetOrderRequest {
    string id = 1;
}

message ListOrdersRequest {
    string user_id = 1;
    int32 page = 2;
    int32 size = 3;
}

message ListOrdersResponse {
    repeated Order orders = 1;
    int32 total_count = 2;
    bool has_more = 3;
}

message UpdateOrderStatusRequest {
    string id = 1;
    OrderStatus status = 2;
}

message OrderStreamRequest {
    string user_id = 1;
}

service OrderService {
    rpc CreateOrder (CreateOrderRequest) returns (Order);
    rpc GetOrder (GetOrderRequest) returns (Order);
    rpc ListOrders (ListOrdersRequest) returns (ListOrdersResponse);
    rpc UpdateOrderStatus (UpdateOrderStatusRequest) returns (Order);
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

    public OrderGrpcService(OrderService orderService,
                           OrderProtoMapper protoMapper) {
        this.orderService = orderService;
        this.protoMapper = protoMapper;
    }

    @Override
    public void createOrder(CreateOrderRequest request,
                           StreamObserver<Order> responseObserver) {
        try {
            OrderEntity entity = protoMapper.toEntity(request);
            OrderEntity created = orderService.create(entity);
            Order proto = protoMapper.toProto(created);
            responseObserver.onNext(proto);
            responseObserver.onCompleted();
        } catch (Exception e) {
            responseObserver.onError(
                Status.INTERNAL.withDescription(e.getMessage())
                    .asRuntimeException());
        }
    }

    @Override
    public void getOrder(GetOrderRequest request,
                        StreamObserver<Order> responseObserver) {
        try {
            OrderEntity entity = orderService.findById(request.getId());
            if (entity == null) {
                responseObserver.onError(
                    Status.NOT_FOUND.withDescription("Order not found")
                        .asRuntimeException());
                return;
            }
            responseObserver.onNext(protoMapper.toProto(entity));
            responseObserver.onCompleted();
        } catch (Exception e) {
            responseObserver.onError(
                Status.INTERNAL.withDescription(e.getMessage())
                    .asRuntimeException());
        }
    }

    @Override
    public void listOrders(ListOrdersRequest request,
                          StreamObserver<ListOrdersResponse> responseObserver) {
        try {
            Page<OrderEntity> page = orderService.findByUserId(
                request.getUserId(),
                PageRequest.of(request.getPage(), request.getSize()));

            ListOrdersResponse.Builder response = ListOrdersResponse.newBuilder()
                .setTotalCount((int) page.getTotalElements())
                .setHasMore(page.hasNext());

            page.getContent().stream()
                .map(protoMapper::toProto)
                .forEach(response::addOrders);

            responseObserver.onNext(response.build());
            responseObserver.onCompleted();
        } catch (Exception e) {
            responseObserver.onError(
                Status.INTERNAL.withDescription(e.getMessage())
                    .asRuntimeException());
        }
    }

    @Override
    public void streamOrders(OrderStreamRequest request,
                            StreamObserver<Order> responseObserver) {
        try {
            orderService.streamByUserId(request.getUserId())
                .map(protoMapper::toProto)
                .forEach(order -> {
                    responseObserver.onNext(order);
                    // Simulate delay
                    Thread.sleep(100);
                });
            responseObserver.onCompleted();
        } catch (Exception e) {
            responseObserver.onError(
                Status.INTERNAL.withDescription(e.getMessage())
                    .asRuntimeException());
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
            public void onError(Throwable t) {
                responseObserver.onError(t);
            }

            @Override
            public void onCompleted() {
                ListOrdersResponse.Builder response = ListOrdersResponse.newBuilder()
                    .setTotalCount(orders.size());

                orders.stream()
                    .map(protoMapper::toProto)
                    .forEach(response::addOrders);

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

    // Unary call
    public OrderProto.Order createOrder(CreateOrderRequest request) {
        return blockingStub.createOrder(request);
    }

    // Unary call with error handling
    public OrderProto.Order getOrder(String id) {
        try {
            GetOrderRequest request = GetOrderRequest.newBuilder()
                .setId(id).build();
            return blockingStub.getOrder(request);
        } catch (StatusRuntimeException e) {
            if (e.getStatus().getCode() == Status.Code.NOT_FOUND) {
                throw new ResourceNotFoundException("Order not found: " + id);
            }
            throw new GrpcClientException("gRPC call failed", e);
        }
    }

    // Server streaming
    public void streamOrders(String userId, Consumer<OrderProto.Order> consumer) {
        OrderStreamRequest request = OrderStreamRequest.newBuilder()
            .setUserId(userId).build();

        Iterator<OrderProto.Order> orders = blockingStub.streamOrders(request);
        orders.forEachRemaining(consumer);
    }

    // Client streaming
    public List<OrderProto.Order> bulkCreateOrders(List<CreateOrderRequest> requests) {
        CompletableFuture<List<OrderProto.Order>> future = new CompletableFuture<>();
        List<OrderProto.Order> results = new ArrayList<>();

        StreamObserver<OrderProto.ListOrdersResponse> responseObserver =
            new StreamObserver<>() {
                @Override
                public void onNext(ListOrdersResponse response) {
                    results.addAll(response.getOrdersList());
                }

                @Override
                public void onError(Throwable t) {
                    future.completeExceptionally(t);
                }

                @Override
                public void onCompleted() {
                    future.complete(results);
                }
            };

        StreamObserver<CreateOrderRequest> requestObserver =
            asyncStub.bulkCreateOrders(responseObserver);

        requests.forEach(requestObserver::onNext);
        requestObserver.onCompleted();

        return future.join();
    }
}
```

### gRPC Channel Configuration

```java
@Configuration
public class GrpcClientConfig {

    @Bean
    public ManagedChannel orderServiceChannel() {
        return ManagedChannelBuilder.forAddress("localhost", 9090)
            .usePlaintext()  // For development only; use TLS in production
            .keepAliveTime(30, TimeUnit.SECONDS)
            .keepAliveTimeout(10, TimeUnit.SECONDS)
            .keepAliveWithoutCalls(true)
            .maxInboundMessageSize(4 * 1024 * 1024)  // 4MB
            .enableRetry()
            .build();
    }

    @PreDestroy
    public void shutdown() {
        orderServiceChannel().shutdown();
    }
}
```

### Interceptor for Logging

```java
@Component
public class LoggingInterceptor implements ServerInterceptor {

    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {

        String methodName = call.getMethodDescriptor().getFullMethodName();
        log.info("gRPC call started: {}", methodName);

        ServerCall<ReqT, RespT> wrappedCall = new ForwardingServerCall.SimpleForwardingServerCall<>(call) {
            @Override
            public void sendMessage(RespT message) {
                log.debug("Response for {}: {}", methodName, message);
                super.sendMessage(message);
            }

            @Override
            public void close(Status status, Metadata trailers) {
                log.info("gRPC call {} completed with status: {}",
                    methodName, status.getCode());
                super.close(status, trailers);
            }
        };

        return new ForwardingServerCallListener.SimpleForwardingServerCallListener<>(
            next.startCall(wrappedCall, headers)) {

            @Override
            public void onMessage(ReqT message) {
                log.debug("Request for {}: {}", methodName, message);
                super.onMessage(message);
            }

            @Override
            public void onCancel() {
                log.warn("gRPC call {} cancelled", methodName);
                super.onCancel();
            }
        };
    }
}
```

### Error Handling Interceptor

```java
@Component
public class ErrorHandlingInterceptor implements ServerInterceptor {

    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {

        ServerCall<ReqT, RespT> wrappedCall = new ForwardingServerCall.SimpleForwardingServerCall<>(call) {
            @Override
            public void close(Status status, Metadata trailers) {
                if (status.isOk()) {
                    super.close(status, trailers);
                } else {
                    // Log the error with stack trace
                    log.error("gRPC error: {} - {}", status.getCode(),
                        status.getDescription());

                    // Return a sanitized error to client
                    Metadata sanitizedTrailers = new Metadata();
                    sanitizedTrailers.put(
                        Metadata.Key.of("error-code",
                            Metadata.ASCII_STRING_MARSHALLER),
                        status.getCode().toString());

                    super.close(status, sanitizedTrailers);
                }
            }
        };

        return next.startCall(wrappedCall, headers);
    }
}
```

### Protobuf to Entity Mapping

```java
@Component
public class OrderProtoMapper {

    public OrderEntity toEntity(CreateOrderRequest proto) {
        OrderEntity entity = new OrderEntity();
        entity.setUserId(proto.getUserId());
        entity.setShippingAddress(proto.getShippingAddress());
        entity.setItems(proto.getItemsList().stream()
            .map(this::toEntity)
            .collect(Collectors.toList()));
        return entity;
    }

    public OrderItemEntity toEntity(OrderItem proto) {
        OrderItemEntity entity = new OrderItemEntity();
        entity.setProductId(proto.getProductId());
        entity.setProductName(proto.getProductName());
        entity.setQuantity(proto.getQuantity());
        entity.setUnitPrice(proto.getUnitPrice());
        return entity;
    }

    public Order toProto(OrderEntity entity) {
        Order.Builder builder = Order.newBuilder()
            .setId(entity.getId())
            .setUserId(entity.getUserId())
            .setTotalAmount(entity.getTotalAmount())
            .setStatus(OrderStatus.valueOf(entity.getStatus().name()))
            .setShippingAddress(entity.getShippingAddress())
            .setCreatedAt(Timestamps.fromMillis(
                entity.getCreatedAt().toEpochMilli()))
            .setUpdatedAt(Timestamps.fromMillis(
                entity.getUpdatedAt().toEpochMilli()));

        entity.getItems().stream()
            .map(this::toProto)
            .forEach(builder::addItems);

        return builder.build();
    }

    public OrderItem toProto(OrderItemEntity entity) {
        return OrderItem.newBuilder()
            .setProductId(entity.getProductId())
            .setProductName(entity.getProductName())
            .setQuantity(entity.getQuantity())
            .setUnitPrice(entity.getUnitPrice())
            .build();
    }
}
```

### TLS Configuration

```java
@Configuration
public class GrpcTlsConfig {

    @Bean
    public ManagedChannel secureChannel() {
        try {
            SslContext sslContext = GrpcSslContexts.forClient()
                .trustManager(new File("path/to/ca.crt"))
                .build();

            return Grpc.newChannelBuilderForAddress(
                    "api.example.com", 443,
                    TlsChannelCredentials.newBuilder()
                        .sslContext(sslContext)
                        .build())
                .build();
        } catch (Exception e) {
            throw new RuntimeException("Failed to configure TLS", e);
        }
    }
}
```

## 5. Real-World Scenarios

### Scenario 1: Real-Time Order Tracking

```protobuf
service OrderTrackingService {
    rpc TrackOrder (TrackOrderRequest) returns (stream OrderLocation);
    rpc SubscribeToUpdates (SubscriptionRequest) returns (stream OrderStatusUpdate);
}

message OrderLocation {
    string order_id = 1;
    double latitude = 2;
    double longitude = 3;
    string location_name = 4;
    google.protobuf.Timestamp timestamp = 5;
}
```

### Scenario 2: Multi-Service Communication

```
API Gateway (REST/GraphQL)
    |
    |--- gRPC ---> User Service
    |--- gRPC ---> Order Service
    |--- gRPC ---> Payment Service
    |--- gRPC ---> Notification Service
    |--- gRPC ---> Inventory Service
```

### Scenario 3: File Upload Service

```protobuf
service FileService {
    rpc UploadFile (stream FileChunk) returns (UploadResponse);
    rpc DownloadFile (DownloadRequest) returns (stream FileChunk);
    rpc ListFiles (ListFilesRequest) returns (ListFilesResponse);
}

message FileChunk {
    bytes data = 1;
    string filename = 2;
    int32 chunk_number = 3;
    int32 total_chunks = 4;
}

message UploadResponse {
    string file_id = 1;
    int64 size_bytes = 2;
}

message DownloadRequest {
    string file_id = 1;
}
```

## 6. Performance

### Performance Benchmarks

gRPC is significantly faster than REST/JSON:
- **2-5x faster** than REST for simple requests.
- **10x faster** for streaming use cases.
- **70-80% smaller payloads** due to Protobuf binary encoding.

### Optimization Techniques

- **Connection Pooling**: Reuse gRPC channels across requests.
- **Keepalive Pings**: Maintain persistent connections.
- **Message Size Limits**: Balance throughput and memory.
- **Streaming**: Use server streaming for large datasets.
- **Compression**: Enable gzip compression for large messages.
- **Flow Control**: Configure HTTP/2 flow control windows.

### Performance Configuration

```java
@Configuration
public class GrpcPerformanceConfig {

    @Bean
    public ManagedChannel performanceChannel() {
        return ManagedChannelBuilder.forAddress("localhost", 9090)
            .usePlaintext()
            .maxInboundMessageSize(16 * 1024 * 1024)  // 16MB
            .flowControlWindow(128 * 1024)  // 128KB flow control window
            .keepAliveTime(30, TimeUnit.SECONDS)
            .keepAliveTimeout(5, TimeUnit.SECONDS)
            .idleTimeout(60, TimeUnit.MINUTES)
            .enableRetry()
            .retryBufferSize(16 * 1024 * 1024)  // 16MB retry buffer
            .build();
    }
}
```

## 7. Security

### gRPC Security Considerations

- **TLS**: Always use TLS in production (never usePlaintext).
- **mTLS**: Mutual TLS for service-to-service authentication.
- **Authentication**: Use interceptors for JWT/OAuth2 validation.
- **Authorization**: Implement interceptors for role-based access.
- **Input Validation**: Validate all protobuf messages on server side.
- **Rate Limiting**: Limit requests per client.

### JWT Authentication Interceptor

```java
@Component
public class JwtAuthInterceptor implements ClientInterceptor {

    private final JwtTokenProvider tokenProvider;

    public JwtAuthInterceptor(JwtTokenProvider tokenProvider) {
        this.tokenProvider = tokenProvider;
    }

    @Override
    public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(
            MethodDescriptor<ReqT, RespT> method,
            CallOptions callOptions,
            Channel next) {

        return new ForwardingClientCall.SimpleForwardingClientCall<ReqT, RespT>(
                next.newCall(method, callOptions)) {

            @Override
            public void start(Listener<RespT> responseListener, Metadata headers) {
                String token = tokenProvider.getAccessToken();
                headers.put(
                    Metadata.Key.of("authorization",
                        Metadata.ASCII_STRING_MARSHALLER),
                    "Bearer " + token);
                super.start(responseListener, headers);
            }
        };
    }
}
```

### Server-Side Auth Interceptor

```java
@Component
public class ServerAuthInterceptor implements ServerInterceptor {

    private final JwtTokenProvider tokenProvider;

    public ServerAuthInterceptor(JwtTokenProvider tokenProvider) {
        this.tokenProvider = tokenProvider;
    }

    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {

        String authHeader = headers.get(
            Metadata.Key.of("authorization",
                Metadata.ASCII_STRING_MARSHALLER));

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            call.close(Status.UNAUTHENTICATED
                .withDescription("Missing or invalid token"), headers);
            return new ServerCall.Listener<>() {};
        }

        String token = authHeader.substring(7);
        if (!tokenProvider.validateToken(token)) {
            call.close(Status.UNAUTHENTICATED
                .withDescription("Invalid or expired token"), headers);
            return new ServerCall.Listener<>() {};
        }

        return next.startCall(call, headers);
    }
}
```

## 8. Common Mistakes

- **Using plaintext in production**: Always use TLS in production.
- **Ignoring connection management**: Failing to reuse channels and shut down properly.
- **Large protobuf messages**: Keep messages under 4MB; use streaming for large data.
- **Breaking proto field numbering**: Never reuse field numbers when removing fields.
- **No error handling on client**: Always handle `StatusRuntimeException`.
- **Blocking on streaming**: Use async stubs for streaming calls.
- **Not setting deadlines**: Always set timeouts on gRPC calls.
- **Over-fetching data**: Request only the fields needed.
- **Missing Proto backward compatibility**: Follow proto evolution best practices.

## 9. Senior Engineer Perspective

### When to Use gRPC

**Ideal for:**
- Internal microservices communication.
- Real-time streaming systems.
- Polyglot environments (multiple languages).
- Low-latency, high-throughput systems.
- Mobile applications (reduced payload size).

**Less ideal for:**
- Public REST APIs (browser clients).
- Simple CRUD applications.
- Serverless functions.

### Proto Evolution Best Practices

1. Never change field numbers.
2. Add new fields with new numbers only.
3. Use `reserved` for removed fields.
4. Set sensible defaults for new fields.
5. Use `optional` for fields that may be absent.
6. Avoid `oneof` in shared messages.
7. Version your packages in proto files.

### gRPC in Microservices Architecture

- **Service Mesh**: gRPC works well with Istio/Linkerd.
- **API Gateway**: gRPC-web for browser exposure.
- **Load Balancing**: Client-side load balancing or proxy-based.
- **Observability**: OpenTelemetry integration for tracing.

## 10. Interview Questions (Easy)

1. What is gRPC and who developed it?
2. What serialization format does gRPC use?
3. What transport protocol does gRPC use?
4. What is a .proto file?
5. Name the four gRPC communication patterns.
6. What is a protobuf message?
7. What is a gRPC stub?
8. What is the difference between gRPC and REST?
9. What is HTTP/2 multiplexing?
10. What is the default port for gRPC?

## Medium

1. How does gRPC handle streaming differently from REST?
2. What is Protocol Buffer field numbering and why does it matter?
3. How do you handle errors in gRPC?
4. What are gRPC interceptors and how are they used?
5. How do you implement authentication in gRPC?
6. What is the difference between blocking and async stubs?
7. How do you configure deadlines/timeouts in gRPC?
8. How does gRPC load balancing work?
9. What are the Proto3 scalar types?
10. How do you enable TLS in gRPC?

## 11. Advanced Interview Questions (Hard)

1. How would you implement bidirectional streaming for a real-time chat application?
2. Design a gRPC API for a distributed database query engine.
3. How do you handle protobuf schema evolution across many microservices?
4. Implement a gRPC retry mechanism with exponential backoff.
5. How would you migrate a REST API to gRPC incrementally?
6. Design a gRPC gateway that also serves REST clients.
7. How do you implement transactions across multiple gRPC services?
8. Design a gRPC-based event sourcing system.
9. How do you handle large file transfers in gRPC?
10. Implement custom protobuf serialization for performance-critical paths.

## System Design

1. Design a real-time stock trading system using gRPC streaming.
2. Design a gRPC-based microservices architecture for an e-commerce platform.
3. Design a distributed logging system using gRPC bidirectional streaming.
4. Design a gRPC API for a real-time multiplayer game server.
5. Design a gRPC-based notification delivery system.
6. Design a gRPC API gateway with protocol translation.
7. Design a distributed file system using gRPC streaming.
8. Design a gRPC-based CI/CD pipeline coordinator.
9. Design a real-time data replication system using gRPC.
10. Design a multi-language SDK generation system from protobuf definitions.

## 12. Expert-Level Interview Questions (Architect-Level)

1. Design a global-scale microservices mesh where gRPC is the primary communication protocol, with automatic failover, circuit breaking, and distributed tracing across 200+ services.
2. How would you design a gRPC-based event-driven architecture where services communicate asynchronously while maintaining exactly-once delivery semantics?
3. Design a schema registry for protobuf schemas across multiple teams, with compatibility checking, automatic code generation, and version management.
4. How would you implement a zero-downtime protobuf schema migration across hundreds of services without coordinated deployments?
5. Design a gRPC load balancing strategy for a multi-region deployment with traffic prioritization, locality-based routing, and failover.
6. How would you build a gRPC-based BFF (Backend for Frontend) that supports web, mobile, and IoT clients with different communication patterns?
7. Design a system that allows gRPC services to be consumed by REST clients through automatic protocol translation without performance degradation.
8. How would you implement flow control and backpressure across a chain of gRPC services to prevent cascading failures?
9. Design a gRPC API versioning strategy that supports both backward-compatible and breaking changes across a federation of services.
10. How would you build a chaos engineering system specifically for gRPC microservices that can inject latency, errors, and network partitions?

## 13. Debugging & Troubleshooting

### Common Issues

- **Deadline exceeded**: Check network latency, server load, timeout configuration.
- **Unavailable**: Service may be down or not registered in service discovery.
- **Internal errors**: Check server logs for exceptions.
- **Message too large**: Increase max inbound message size or use streaming.
- **TLS handshake failure**: Verify certificate chains and mutual TLS configuration.
- **Connection reset**: Check keepalive settings and proxy configurations.

### gRPC Health Check

```protobuf
service Health {
    rpc Check (HealthCheckRequest) returns (HealthCheckResponse);
    rpc Watch (HealthCheckRequest) returns (stream HealthCheckResponse);
}

message HealthCheckRequest {
    string service = 1;
}

message HealthCheckResponse {
    enum ServingStatus {
        UNKNOWN = 0;
        SERVING = 1;
        NOT_SERVING = 2;
    }
    ServingStatus status = 1;
}
```

### Debugging with gRPC Reflection

```java
// Enable reflection in server
import io.grpc.protobuf.services.ProtoReflectionService;

@GrpcService
public class ReflectionService extends ProtoReflectionService {
    // Automatically enables grpcurl debugging
}
```

```bash
# Use grpcurl to debug
grpcurl -plaintext localhost:9090 list
grpcurl -plaintext localhost:9090 describe com.example.orders.Order
grpcurl -plaintext -d '{"id": "123"}' localhost:9090 com.example.orders.OrderService/GetOrder
```

## 14. Comparison Section

### gRPC vs REST

| Aspect | gRPC | REST |
|--------|------|------|
| Protocol | HTTP/2 | HTTP/1.1 (HTTP/2 optional) |
| Serialization | Protobuf (binary) | JSON/XML (text) |
| Performance | 2-10x faster | Slower |
| Payload Size | ~80% smaller | Larger |
| Schema | Required (.proto) | Optional (OpenAPI) |
| Streaming | Native support | WebSocket add-on |
| Code Generation | Built-in | OpenAPI generators |
| Human Readability | Low (binary) | High (JSON) |
| Browser Support | Via gRPC-web | Native |
| Caching | Not built-in | Native HTTP caching |

### gRPC vs Message Queues

| Aspect | gRPC | Message Queues |
|--------|------|----------------|
| Communication | Synchronous/Async streaming | Asynchronous |
| Coupling | Tight (RPC) | Loose (events) |
| Ordering | In-stream ordering | Queue ordering |
| Persistence | No built-in | Durable messages |
| Use Case | Service-to-service calls | Event-driven, decoupling |

## 15. Revision Notes

- gRPC: Google RPC, uses HTTP/2 + Protobuf
- 4 patterns: Unary, Server Streaming, Client Streaming, Bidirectional
- .proto file defines service contract
- Protobuf: binary, efficient, strongly typed, backward-compatible
- Spring Boot: `@GrpcService`, generated stubs
- Always use TLS in production
- Interceptors for cross-cutting concerns (auth, logging, metrics)
- Deadlines prevent resource leaks
- Proto evolution: never change field numbers, use reserved

## 16. Cheat Sheet

```
+------------------------------------------------------------------+
| GRPC CHEAT SHEET                                                 |
+------------------------------------------------------------------+
| COMMUNICATION PATTERNS                                           |
|   Unary:              client ---- req ----> server                |
|                       client <--- resp ---- server               |
|   Server Streaming:   client ---- req ----> server                |
|                       client <--- stream -- server               |
|   Client Streaming:   client ---- stream -> server                |
|                       client <--- resp ---- server               |
|   Bidirectional:      client <=== stream ===> server              |
+------------------------------------------------------------------+
| PROTOBUF FIELD RULES                                             |
|   Field numbers: 1-15 (1 byte)  16-2047 (2 bytes)               |
|   Use 1-15 for frequently occurring fields                       |
|   Never reuse field numbers when deleting fields                 |
|   Use "reserved" for removed fields                              |
+------------------------------------------------------------------+
| GRPC STATUS CODES                                                |
|   OK(0)  CANCELLED(1)  UNKNOWN(2)  INVALID_ARGUMENT(3)          |
|   DEADLINE_EXCEEDED(4)  NOT_FOUND(5)  ALREADY_EXISTS(6)         |
|   PERMISSION_DENIED(7)  UNAUTHENTICATED(16)                     |
|   UNIMPLEMENTED(12)  INTERNAL(13)  UNAVAILABLE(14)              |
|   RESOURCE_EXHAUSTED(8)  FAILED_PRECONDITION(9)                 |
+------------------------------------------------------------------+
| SPRING BOOT / GRPC                                               |
|   @GrpcService              -> gRPC service implementation        |
|   @GrpcClient               -> Injected gRPC client stub          |
|   ServerInterceptor         -> Server-side interceptor            |
|   ClientInterceptor         -> Client-side interceptor            |
+------------------------------------------------------------------+
| PROTO FILE STRUCTURE                                             |
|   syntax = "proto3";                                             |
|   package com.example;                                           |
|   option java_package = "...";                                    |
|   option java_multiple_files = true;                             |
|                                                                  |
|   message Request {                                              |
|       string id = 1;                                             |
|   }                                                              |
|                                                                  |
|   service MyService {                                            |
|       rpc DoSomething (Request) returns (Response);              |
|   }                                                              |
+------------------------------------------------------------------+
| PERFORMANCE COMPARISON (vs REST/JSON)                            |
|   Latency:     2-5x faster                                       |
|   Throughput:  5-10x higher                                      |
|   Payload Size: ~70-80% smaller                                  |
|   CPU Usage:   Lower (binary parsing vs JSON)                    |
+------------------------------------------------------------------+
```
