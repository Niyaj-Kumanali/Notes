# GraphQL

---

## Overview

- **Definition:** GraphQL is a query language and runtime for APIs developed by Meta in 2012 and open-sourced in 2015. Unlike REST where the server defines the response structure, GraphQL allows clients to specify exactly which data they need, eliminating over-fetching and under-fetching.

- **Why It Exists:** REST APIs often force clients to make multiple requests or receive excessive data. GraphQL solves this with a single endpoint, a strong type system, and client-driven queries that request only the needed fields.

- **Core Concepts:**
  - **Schema** — defines types, queries, mutations, and subscriptions
  - **Queries** — read operations (equivalent to GET)
  - **Mutations** — write operations (equivalent to POST/PUT/DELETE)
  - **Subscriptions** — real-time operations (equivalent to WebSockets)
  - **Resolvers** — functions that fetch data for specific fields
  - **Type System** — strongly typed schema defining all possible data shapes

---

## GraphQL Type System

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  age: Int
  posts: [Post!]!
  createdAt: DateTime!
}

type Post {
  id: ID!
  title: String!
  body: String!
  author: User!
  comments: [Comment!]!
}

type Query {
  user(id: ID!): User
  users(page: Int, size: Int): [User!]!
  post(id: ID!): Post
}

type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
  deleteUser(id: ID!): Boolean!
}

type Subscription {
  userCreated: User!
  userUpdated: User!
}

input CreateUserInput { name: String!, email: String!, age: Int }
input UpdateUserInput { name: String, email: String, age: Int }
```

---

## Execution Pipeline

1. Client sends a GraphQL query (POST to `/graphql`)
2. Server parses the query string into an AST (Abstract Syntax Tree)
3. Server validates the query against the schema
4. Server executes each field's resolver, starting from the root
5. Resolvers fetch data from any source (DB, REST, gRPC)
6. Results are collected and serialized into the response shape

---

## Production Code Examples

### Spring Boot Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-graphql</artifactId>
</dependency>
```

### Controller-Based GraphQL

```java
@Controller
public class UserGraphQLController {
    private final UserService userService;
    private final PostService postService;

    public UserGraphQLController(UserService userService, PostService postService) {
        this.userService = userService;
        this.postService = postService;
    }

    @QueryMapping
    public User user(@Argument Long id) { return userService.findById(id); }

    @QueryMapping
    public List<User> users(@Argument int page, @Argument int size) {
        return userService.findAll(PageRequest.of(page, size)).getContent();
    }

    @MutationMapping
    public UserPayload createUser(@Argument CreateUserInput input) {
        return new UserPayload(userService.create(mapToUser(input)), true);
    }

    @MutationMapping
    public DeletePayload deleteUser(@Argument Long id) {
        userService.delete(id);
        return new DeletePayload(id, true);
    }

    @SchemaMapping(typeName = "User", field = "posts")
    public List<Post> getPosts(User user) { return postService.findByUserId(user.getId()); }
}
```

### Batch Loading with DataLoader (N+1 Prevention)

```java
@Controller
public class PostGraphQLController {
    private final PostService postService;
    private final UserService userService;

    public PostGraphQLController(PostService postService, UserService userService,
            BatchLoaderRegistry batchLoaderRegistry) {
        this.postService = postService;
        this.userService = userService;

        batchLoaderRegistry.forType(Long.class, User.class)
            .registerMappedBatchLoader((ids, environment) -> {
                Map<Long, User> users = userService.findAllById(ids).stream()
                    .collect(Collectors.toMap(User::getId, Function.identity()));
                return CompletableFuture.completedFuture(users);
            });
    }

    @QueryMapping
    public List<Post> posts() { return postService.findAll(); }

    @SchemaMapping
    public CompletableFuture<User> author(Post post, DataLoader<Long, User> loader) {
        return loader.load(post.getAuthorId());
    }
}
```

### Subscription Implementation

```java
@Controller
public class UserSubscriptionController {
    private final Sinks.Many<User> userSink;

    public UserSubscriptionController() {
        this.userSink = Sinks.many().multicast().onBackpressureBuffer();
    }

    @SubscriptionMapping
    public Publisher<User> userCreated() { return userSink.asFlux(); }

    public void onUserCreated(User user) { userSink.tryEmitNext(user); }
}
```

### Exception Handling

```java
@ControllerAdvice
public class GraphQLExceptionHandler {
    @ExceptionHandler(ResourceNotFoundException.class)
    public void handleNotFound(ResourceNotFoundException ex) {
        throw new GraphQLErrorException(GraphQLError.newError()
            .message(ex.getMessage()).errorType(ErrorType.NOT_FOUND).build());
    }
}
```

### Query Complexity & Depth Limiting

```java
@Configuration
public class GraphQLSecurityConfig {
    @Bean
    public Instrumentation maxDepthInstrumentation() { return new MaxQueryDepthInstrumentation(8); }

    @Bean
    public Instrumentation maxComplexityInstrumentation() { return new MaxQueryComplexityInstrumentation(500); }
}
```

---

## Common Mistakes

- **Ignoring N+1 problem** — always batch database calls with DataLoader.
  - **Why it looks correct:** the resolver for a single field looks innocent — it just fetches data for one parent — and the quadratic explosion only becomes visible when a client requests a list of 100 items with three nested relationship fields.
- **Over-fetching in resolvers** — only fetch fields that are requested.
  - **Why it looks correct:** the resolver doesn't know which fields the client asked for, so fetching the entire entity seems safe — until a high-traffic field triggers expensive joins that the client never uses.
- **Not using pagination** — GraphQL still needs pagination for lists.
  - **Why it looks correct:** GraphQL already lets clients limit fields, so developers assume that prevents overload — but omitting pagination still allows a request for 100,000 items with all their nested relationships.
- **Deeply nested queries** — limit query depth to prevent abuse.
  - **Why it looks correct:** the schema defines valid relationships, so any query that follows the schema seems legitimate — the exponential blowup of resolver calls at depth 15 is not obvious from the query text.
- **Exposing internal schema** — disable introspection in production.
  - **Why it looks correct:** introspection is a core GraphQL feature used by GraphiQL and developer tools — developers treat it as harmless documentation until an attacker uses it to map the entire API surface for crafting denial-of-service queries.
- **No query complexity limits** — can lead to DoS attacks.
  - **Why it looks correct:** depth limiting seems sufficient, and complexity analysis adds configuration overhead — but a shallow query requesting 50 list fields at depth 3 can still cost more than a deep query with a few fields.
- **Treating GraphQL like REST** — don't create separate endpoints.
  - **Why it looks correct:** REST conventions are deeply familiar, and creating separate endpoints for different operations provides clear separation — but it defeats GraphQL's single-endpoint batching and over-fetching elimination advantages.
- **Not handling errors properly** — use structured error responses.
  - **Why it looks correct:** GraphQL always returns HTTP 200 even on errors, so throwing an exception seems fine — but unstructured errors in the `errors` array force clients to parse message strings instead of handling typed error codes.

---

## Key Design Considerations

- **When to Use GraphQL:**
  - Complex data models with many relationships
  - Clients with varying data requirements
  - Mobile applications (bandwidth constrained)
  - Rapidly evolving frontend requirements

- **When to Use REST Instead:**
  - Simple CRUD operations
  - File upload/download
  - Caching-heavy use cases
  - Public APIs with diverse clients

- **Federation (Microservices):**
  - Gateway composes schemas from multiple services
  - Each service extends the shared types
  - Enables team autonomy with unified API

- **Security Measures:**
  - Disable introspection in production
  - Limit query depth (8-10 levels)
  - Limit query complexity (500-1000)
  - Field-level authorization
  - Rate limit by query cost

---

## Real-World Scenarios

### Scenario 1: N+1 Query Problem with DataLoader
**Context:** A GraphQL query for a list of users and their posts triggers N+1 database queries. When a client queries `users { name posts { title } }`, the resolver fetches all users (1 query), then for each user, fetches their posts separately (N queries). With 100 users, this is 101 database queries. Response time grows linearly with user count.

**Resolution:** Use DataLoader to batch-load related data. DataLoader groups all `post` field resolutions for all users within a single request execution and executes one batch query.

```java
@Controller
public class UserController {
    private final PostService postService;

    public UserController(PostService postService, BatchLoaderRegistry batchLoaderRegistry) {
        this.postService = postService;
        // Register a batch loader for User -> Posts
        batchLoaderRegistry.forType(Long.class, List.class)
            .registerMappedBatchLoader((authorIds, environment) -> {
                Map<Long, List<Post>> postsByAuthor = postService.findByAuthorIdIn(authorIds)
                    .stream().collect(Collectors.groupingBy(Post::getAuthorId));
                return CompletableFuture.completedFuture(postsByAuthor);
            });
    }

    @QueryMapping
    public List<User> users() { return userService.findAll(); }

    @SchemaMapping
    public CompletableFuture<List<Post>> posts(User user, DataLoader<Long, List<Post>> loader) {
        return loader.load(user.getId());  // Batched: loads all posts for all users at once
    }
}
```

The batch loader fires once per request execution, reducing 101 queries to 2 (1 for users, 1 for all posts).

### Scenario 2: Malicious Deeply Nested Query DoS
**Context:** A malicious client sends a query 15 levels deep: `user { posts { author { posts { author { posts { ... } } } } } }`. Each level triggers database queries. The server runs out of memory and crashes. This is a DoS attack exploiting GraphQL's flexible query structure.

**Resolution:** Implement query depth limiting and complexity analysis.

```java
@Configuration
public class GraphQLSecurityConfig {
    @Bean
    public Instrumentation maxDepthInstrumentation() {
        return new MaxQueryDepthInstrumentation(8);  // Max 8 levels deep
    }

    @Bean
    public Instrumentation maxComplexityInstrumentation() {
        return new MaxQueryComplexityInstrumentation(500);  // Reject queries over 500 cost
    }
}
```

Also disable introspection in production: `spring.graphql.schema.introspection.enabled=false`. This prevents attackers from discovering the full schema to craft complex queries.

### Scenario 3: Real-Time Subscription Disconnects
**Context:** Your `orderStatusUpdates` subscription pushes real-time order status changes to clients via WebSocket. Under high load, clients randomly disconnect. The subscription flow: 1,000 clients subscribe → 100 messages/second → WebSocket disconnects after 30 seconds → clients reconnect → cycle repeats.

**Resolution:** Verify the subscription uses proper backpressure and WebSocket configuration.

```java
@Controller
public class OrderSubscriptionController {
    // Use Sinks.many() with backpressure, not Sinks.one()
    private final Sinks.Many<OrderStatus> orderSink = Sinks.many()
        .multicast()
        .onBackpressureBuffer(1024, false);  // Buffer up to 1024, drop oldest on overflow

    @SubscriptionMapping
    public Publisher<OrderStatus> orderStatusUpdates(@Argument String orderId) {
        return orderSink.asFlux()
            .filter(status -> status.orderId().equals(orderId))
            .delayElements(Duration.ofMillis(100));  // Rate-limit to 10 msg/s per client
    }

    public void onOrderStatusChanged(OrderStatus status) {
        orderSink.tryEmitNext(status);  // Non-blocking emission
    }
}
```

WebSocket configuration: enable heartbeats every 15 seconds (Spring's `WebSocketHandler`), set max idle timeout to 60 seconds, and ensure the load balancer/proxy is configured to not drop long-lived connections.

---

## Scenario-Based Questions

1. **Q: Your GraphQL API resolves a list of 500 users, and the `posts` field on each user triggers a separate database query. Response time is 30 seconds. How do you fix this with minimal code changes?**
   - A: Use DataLoader with a `MappedBatchLoader` that loads all posts for all requested users in a single query (`WHERE author_id IN (:ids)`). DataLoader automatically deduplicates and batches requests within a single GraphQL request execution. The resolver returns `CompletableFuture` — DataLoader batches all futures and fires one batch query. This reduces 501 database queries to 2.

2. **Q: A malicious client sends a query that requests `users { posts { comments { user { posts { comments { user { ... } } } } } } }` — 20 levels deep. Each level triggers downstream service calls. The server CPU spikes to 100% and crashes. How do you prevent this?**
   - A: (1) Implement max query depth instrumentation: limit to 8-10 levels. (2) Implement query complexity analysis: assign costs to each field (e.g., `user = 1`, `posts = 5`, `comments = 3`). Reject queries exceeding a total budget (e.g., 500). (3) Rate limit by query cost: charge clients per query cost, not per query count. (4) Disable introspection in production — limits schema discovery. (5) Timeout long-running queries at the server level. Depth and complexity limits are the primary defense against GraphQL DoS attacks.

   - **Follow-up:** You set a complexity limit of 500, but a legitimate admin dashboard query needs cost 600 — how do you handle this without increasing the limit for everyone?

3. **Q: Your mobile team wants to migrate from REST to GraphQL to reduce over-fetching. You have 200 REST endpoints and 50 mobile clients. How do you migrate without downtime?**
   - A: (1) Add a `/graphql` endpoint alongside existing REST endpoints. (2) Build GraphQL resolvers that delegate to the same services as REST controllers — no business logic rewrite needed. (3) Run both systems in parallel for 6-12 months. (4) Use the strangler fig pattern: new features go to GraphQL first, REST receives only bug fixes. (5) Gradually migrate mobile screens REST → GraphQL. (6) When REST traffic drops to zero, deprecate and remove. Add an OpenAPI-to-GraphQL wrapper if clients need time to migrate.

4. **Q: You need to handle file uploads in GraphQL, but GraphQL doesn't natively support multipart uploads. How do you design this?**
   - A: (1) Use the GraphQL multipart request specification: the client sends a multipart request with `operations` (the GraphQL mutation) and `map` (maps multipart parts to mutation variables). (2) Alternatively, use a separate REST upload endpoint and pass the resulting file URL as a GraphQL argument. (3) For Spring GraphQL, implement a custom `MultipartFile` argument resolver. (4) Consider using a pre-signed URL pattern: client requests an upload URL from GraphQL, uploads directly to S3, then submits the file ID in a mutation.

5. **Q: Your GraphQL API has no caching strategy. Each query hits the database directly, and the same data is fetched repeatedly. How do you implement caching at different levels?**
   - A: (1) DataLoader's per-request cache: within a single GraphQL request, repeated loads of the same entity hit the cache, not the database. (2) Resolver-level caching: cache expensive resolver results in Redis with TTL per field type. (3) Persisted queries: store common queries server-side; CDN-cache their results. (4) `@cacheControl` directive: annotate schema fields with `maxAge` and `scope` (PUBLIC/PRIVATE) for CDN caching. (5) Response caching at the HTTP level for GET requests (if using automatic persisted queries).

   - **Follow-up:** If you cache the result of `user(id: 1) { posts { title } }` but another mutation adds a new post, how does the cache know to invalidate that specific query shape?

6. **Q: How do you implement pagination in GraphQL for a feed that updates in real-time (new posts created every second)?**
   - A: Use the Relay Connection spec with cursor-based pagination. The query returns `edges` (each with a cursor) and `pageInfo` (with `hasNextPage` and `endCursor`). The cursor is a timestamp or opaque token. Clients pass `first: 20, after: "cursor"`. New posts don't shift page boundaries because the cursor is fixed. For real-time updates, combine with a subscription that pushes new posts, which the client prepends to the cached list.

7. **Q: You need field-level authorization: regular users can see `name` and `email`, but only admins can see `salary`. How do you implement this without cluttering resolvers with auth checks?**
   - A: (1) Use a custom `DataFetcher` wrapper or Spring AOP that checks authorization before resolving each field. (2) Use GraphQL directives: define a `@auth(role: "ADMIN")` directive on schema fields. (3) Implement a schema transformation that applies auth checks at the field level. (4) For Spring GraphQL, use `@SchemaMapping` with `@PreAuthorize` from Spring Security. (5) The cleanest approach: return `null` for unauthorized fields and include an `errors` entry explaining the restriction. Avoid throwing exceptions for unauthorized fields — use soft denials.

8. **Q: Your GraphQL mutation `createUser` returns only a success boolean. The client needs the new user's ID to navigate to the user profile. What's wrong and how do you fix it?**
   - A: GraphQL mutations should return the affected object(s) so clients can update their cache and proceed with subsequent operations. Always follow the pattern: `mutation { createUser(input: ...) { id name email } }`. Return the created/modified object. If performance is a concern, use a `clientMutationId` pattern or return at minimum the ID. A mutation that returns only a boolean forces the client to refetch, defeating GraphQL's efficiency advantage.

   - **Follow-up:** What security risk does returning the full created object introduce if your mutation accepts a `role` input field that only admins should be allowed to set?

9. **Q: Your GraphQL schema evolves over time — you need to rename `email` to `emailAddress` and change `name` from a single field to `firstName` + `lastName`. How do you handle this without breaking existing queries?**
   - A: GraphQL uses schema evolution, not versioning. (1) Add the new fields alongside old ones: `firstName`, `lastName`, and `emailAddress`. (2) Mark old fields as `@deprecated(reason: "Use firstName and lastName")`. (3) Keep old fields working until you've verified no clients use them. (4) Remove old fields only when their usage drops to zero (can be monitored via query logging). (5) GraphQL's field-based selection means clients that don't request deprecated fields are unaffected — no versioning needed.

10. **Q: Your GraphQL subscription for real-time stock price updates disconnects every few minutes. Users miss price updates. How do you diagnose and fix?**
    - A: (1) Check WebSocket heartbeats: ensure both client and server send pings every 10-15 seconds. Spring GraphQL supports `heartbeatInterval` in WebSocket configuration. (2) Check the subscription publisher: use `Sinks.many().multicast().onBackpressureBuffer()` with backpressure. Without backpressure, fast producers overwhelm slow consumers. (3) Check network proxies/load balancers: some proxies drop idle WebSocket connections after 60 seconds. Configure keep-alive on the proxy. (4) Server-Sent Events (SSE) is an alternative if WebSocket connections are problematic. (5) Implement client-side reconnection with exponential backoff.

---

## Interview Questions

1. **What is GraphQL and how does it differ from REST?**
   - A: GraphQL is a query language where clients specify exactly which fields they need, eliminating over-fetching and under-fetching. Unlike REST's fixed endpoints, GraphQL has a single endpoint with a flexible query structure, strong type system, and client-driven data fetching.

2. **What is the N+1 problem in GraphQL and how do you solve it?**
   - A: When resolving a list of N entities, each triggers an additional database query for related data. Solve with DataLoader: batches all individual loads into a single batch query within a request execution. DataLoader also caches within the request scope.

3. **How do you protect a GraphQL API from malicious queries?**
   - A: Query depth limiting (max 8-10 levels), query complexity analysis (max cost budget per query), rate limiting by query cost, disabling introspection in production, and timeouts. These prevent deeply nested queries from crashing the server.

4. **What are the benefits of using DataLoader?**
   - A: Batching (groups individual loads into batch queries) and caching (deduplicates loads within a request). Prevents N+1 queries without manual batching. Works with any data source (DB, REST, gRPC).

5. **What is a GraphQL subscription and when would you use it?**
   - A: A subscription provides real-time updates from server to client over a persistent connection (WebSocket or SSE). Used for real-time feeds, notifications, chat messages, stock price updates, and any data that changes frequently and needs immediate delivery.

6. **How do you implement pagination in GraphQL?**
   - A: Relay Connection spec with cursor-based pagination. Query returns `edges` (each with a `cursor`) and `pageInfo` (`hasNextPage`, `endCursor`). Clients paginate with `first: 20, after: "cursor"`. This is stable against concurrent inserts.

7. **How does GraphQL handle versioning?**
   - A: Through schema evolution — add new fields alongside old ones, mark old fields as `@deprecated`, and remove them only when usage drops to zero. GraphQL's field-based selection means clients not requesting deprecated fields are unaffected. No explicit version numbers needed.

8. **What is the difference between a query and a mutation in GraphQL?**
   - A: Queries are for reading data (parallel execution). Mutations are for writing data (sequential execution — each mutation runs after the previous completes). Mutations should return the modified object for cache updates.

9. **How do you handle errors in GraphQL?**
   - A: GraphQL always returns HTTP 200 with a JSON body containing `data` and `errors` arrays. Each error has a `message`, `locations`, and `path`. Use structured error types (`NOT_FOUND`, `UNAUTHORIZED`, `VALIDATION_ERROR`). Never expose stack traces.

10. **What is GraphQL federation and when would you use it?**
    - A: Federation composes a single GraphQL schema from multiple microservices' schemas. Each service owns its types and extends shared types. A gateway routes queries to the appropriate service. Use when migrating a monolith GraphQL to microservices or when multiple teams own different data domains.

---

## Developer Recommendations

- **Always use DataLoader to prevent N+1 queries** — The N+1 problem is the most common GraphQL performance issue. DataLoader batches all field resolutions within a single request execution into one batch query. Implement it from the start — retrofitting DataLoader after the schema is built requires rewriting resolvers. Register batch loaders for every relationship (one-to-many, many-to-one, many-to-many).
  - **Production story:** A social media company's GraphQL API crashed during a product launch when a client queried 200 feed posts with `author { avatar comments { user { profile } } }` — the resolver chain triggered 1,400+ database queries per request, taking the database from 5% CPU to 98% in under two minutes.

- **Implement query depth and complexity limits before going to production** — GraphQL's flexible query structure makes it vulnerable to DoS attacks. Depth limits prevent deeply nested queries. Complexity analysis assigns costs to fields and rejects expensive queries. These are GraphQL-specific security measures that REST doesn't need. Without them, a single malicious query can crash your server.
  - **Production story:** An e-commerce site learned this when a competitor sent a query requesting `allProducts { variants { inventory { warehouse { location } } } }` repeated across 15 levels — the server ran out of memory and the site was down for 45 minutes before they added depth limiting.

- **Use cursor-based pagination (Relay Connection spec) for all list fields** — Offset pagination breaks with concurrent inserts and performs poorly on large offsets. The Relay Connection spec provides stable, efficient pagination that works regardless of data changes. Return `hasNextPage` and `endCursor` in every paginated response. This is standard in GraphQL ecosystem — Apollo, Relay, and all major clients support it.

- **Mutations should return the modified object, not just a status** — Clients need the returned data to update their cache. A mutation returning only `{ success: true }` forces clients to refetch, defeating GraphQL's efficiency. Always return the created/updated object. The GraphQL best practice is: `mutation { createUser(input: ...) { id name email } }`.

- **Use schema evolution instead of versioning** — GraphQL's field-based selection makes versioning unnecessary. Add new fields alongside old ones. Mark old fields as `@deprecated`. Remove only when usage drops to zero. This eliminates the versioning tax — no v1/v2 endpoints, no duplicate resolvers, no migration burden. Clients automatically get new capabilities by requesting new fields.

- **Secure subscriptions with authentication and rate limiting** — Subscriptions are persistent connections — a single authenticated client can consume server resources indefinitely. Authenticate at connection time (not just at subscription time). Limit the number of active subscriptions per client. Implement backpressure to prevent fast publishers from overwhelming slow subscribers. Use Server-Sent Events as a simpler fallback when WebSocket connections are problematic (corporate proxies, load balancers).
