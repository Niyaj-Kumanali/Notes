# GraphQL

## 1. Executive Summary

GraphQL is a query language and runtime for APIs developed by Meta in 2012 and open-sourced in 2015. Unlike REST, where the server defines the response structure, GraphQL allows clients to specify exactly which data they need. This eliminates over-fetching and under-fetching, enabling more efficient data loading. GraphQL operates through a single endpoint, uses a type system for schema definition, and supports queries, mutations, and subscriptions.

## 2. Core Theory

### GraphQL Fundamentals

- **Schema**: Defines the types, queries, mutations, and subscriptions available.
- **Queries**: Read operations (equivalent to GET).
- **Mutations**: Write operations (equivalent to POST/PUT/DELETE).
- **Subscriptions**: Real-time operations (equivalent to WebSockets).
- **Resolver**: Function that fetches data for a specific field.
- **Type System**: Strongly typed schema defining all possible data shapes.
- **Query Language**: Client specifies the exact fields needed.

### Core Concepts

```
Query (read)       -> { user(id: "1") { name email } }
Mutation (write)  -> mutation { createUser(name: "John") { id } }
Subscription      -> subscription { userUpdated { name } }
```

### GraphQL Type System

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

type Comment {
  id: ID!
  text: String!
  author: User!
}

type Query {
  user(id: ID!): User
  users(page: Int, size: Int): [User!]!
  post(id: ID!): Post
  posts: [Post!]!
}

type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
  deleteUser(id: ID!): Boolean!
}

input CreateUserInput {
  name: String!
  email: String!
  age: Int
}

input UpdateUserInput {
  name: String
  email: String
  age: Int
}
```

## 3. Under-the-Hood Deep Dive

### Query Execution Pipeline

1. Client sends a GraphQL query (POST to `/graphql`).
2. Server parses the query string into an AST (Abstract Syntax Tree).
3. Server validates the query against the schema.
4. Server executes each field's resolver, starting from the root.
5. Resolvers can fetch data from any source (DB, REST, gRPC).
6. Results are collected and serialized into the response shape.

### N+1 Problem in GraphQL

GraphQL is susceptible to the N+1 problem when resolving lists:

```java
// Inefficient resolver - N+1 queries
public List<Post> posts(User user) {
    return postRepository.findByUserId(user.getId()); // N queries for N users
}
```

### Solving N+1 with DataLoader

```java
@Component
public class UserBatchLoader {

    private final UserRepository userRepository;

    public DataLoader<Long, User> createUserLoader() {
        return DataLoader.newMappedDataLoader(new MappedBatchLoader<>() {
            @Override
            public CompletionStage<Map<Long, User>> load(Set<Long> ids) {
                return CompletableFuture.supplyAsync(() ->
                    userRepository.findAllById(ids).stream()
                        .collect(Collectors.toMap(User::getId, Function.identity()))
                );
            }
        });
    }
}
```

## 4. Production Code Examples

### Spring Boot GraphQL Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-graphql</artifactId>
</dependency>
```

### Schema Definition (graphql/schema.graphqls)

```graphql
type Query {
    user(id: ID!): User
    users(page: Int = 0, size: Int = 20): UserConnection!
    searchUsers(name: String): [User!]!
}

type Mutation {
    createUser(input: CreateUserInput!): UserPayload!
    updateUser(id: ID!, input: UpdateUserInput!): UserPayload!
    deleteUser(id: ID!): DeletePayload!
}

type Subscription {
    userCreated: User!
    userUpdated: User!
    userDeleted: ID!
}

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
    createdAt: DateTime!
}

type UserConnection {
    edges: [UserEdge!]!
    pageInfo: PageInfo!
}

type UserEdge {
    node: User!
    cursor: String!
}

type PageInfo {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: String
    endCursor: String
}

type UserPayload {
    user: User!
    success: Boolean!
}

type DeletePayload {
    id: ID!
    success: Boolean!
}

input CreateUserInput {
    name: String!
    email: String!
    age: Int
}

input UpdateUserInput {
    name: String
    email: String
    age: Int
}
```

### Controller-Based GraphQL (Spring for GraphQL)

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
    public User user(@Argument Long id) {
        return userService.findById(id);
    }

    @QueryMapping
    public List<User> users(@Argument int page, @Argument int size) {
        return userService.findAll(PageRequest.of(page, size)).getContent();
    }

    @QueryMapping
    public List<User> searchUsers(@Argument String name) {
        return userService.searchByName(name);
    }

    @MutationMapping
    public UserPayload createUser(@Argument CreateUserInput input) {
        User user = userService.create(mapToUser(input));
        return new UserPayload(user, true);
    }

    @MutationMapping
    public UserPayload updateUser(@Argument Long id, @Argument UpdateUserInput input) {
        User user = userService.update(id, mapToUser(input));
        return new UserPayload(user, true);
    }

    @MutationMapping
    public DeletePayload deleteUser(@Argument Long id) {
        userService.delete(id);
        return new DeletePayload(id, true);
    }

    @SchemaMapping(typeName = "User", field = "posts")
    public List<Post> getPosts(User user) {
        return postService.findByUserId(user.getId());
    }
}
```

### Resolver Pattern

```java
@Component
public class UserResolver implements GraphQlResolver<User> {

    private final PostService postService;
    private final CommentService commentService;

    public UserResolver(PostService postService, CommentService commentService) {
        this.postService = postService;
        this.commentService = commentService;
    }

    public List<Post> posts(User user) {
        return postService.findByUserId(user.getId());
    }

    public int postCount(User user) {
        return postService.countByUserId(user.getId());
    }

    public String displayName(User user) {
        return user.getName() + " (" + user.getEmail() + ")";
    }
}
```

### Batch Loading with DataLoader

```java
@Controller
public class PostGraphQLController {

    private final PostService postService;
    private final DataLoader<Long, User> userLoader;

    public PostGraphQLController(PostService postService,
                                BatchLoaderRegistry batchLoaderRegistry) {
        this.postService = postService;

        // Register a DataLoader for User
        batchLoaderRegistry.forType(Long.class, User.class)
            .registerMappedBatchLoader((ids, environment) -> {
                Map<Long, User> users = userService.findAllById(ids).stream()
                    .collect(Collectors.toMap(User::getId, Function.identity()));
                return CompletableFuture.completedFuture(users);
            });
    }

    @QueryMapping
    public List<Post> posts() {
        return postService.findAll();
    }

    @SchemaMapping
    public CompletableFuture<User> author(Post post,
                                          DataLoader<Long, User> loader) {
        return loader.load(post.getAuthorId());
    }
}
```

### Exception Handling

```java
@ControllerAdvice
public class GraphQLExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public void handleNotFound(ResourceNotFoundException ex) {
        throw new GraphQLErrorException(
            GraphQLError.newError()
                .message(ex.getMessage())
                .errorType(ErrorType.NOT_FOUND)
                .build()
        );
    }

    @ExceptionHandler(ConstraintViolationException.class)
    public void handleValidation(ConstraintViolationException ex) {
        List<GraphQLError> errors = ex.getConstraintViolations().stream()
            .map(violation -> GraphQLError.newError()
                .message(violation.getMessage())
                .errorType(ErrorType.BAD_REQUEST)
                .location(violation.getPropertyPath().toString())
                .build())
            .collect(Collectors.toList());

        throw new GraphQLErrorException(errors);
    }
}
```

### Custom Scalar Type

```java
@Component
public class DateTimeScalarConfiguration implements RuntimeWiringConfigurer {

    @Override
    public void configure(RuntimeWiring.Builder builder) {
        builder.scalar(new ExtendedScalars.Date());
        builder.scalar(new ExtendedScalars.DateTime());
        builder.scalar(new ExtendedScalars.LocalTime());
    }
}
```

### Subscription Implementation

```java
@Controller
public class UserSubscriptionController {

    private final UserService userService;
    private final Sinks.Many<User> userSink;

    public UserSubscriptionController(UserService userService) {
        this.userService = userService;
        this.userSink = Sinks.many().multicast().onBackpressureBuffer();
    }

    @SubscriptionMapping
    public Publisher<User> userCreated() {
        return userSink.asFlux().filter(user -> true);
    }

    @SubscriptionMapping
    public Publisher<User> userUpdated() {
        return userSink.asFlux();
    }

    @SubscriptionMapping
    public Publisher<Long> userDeleted() {
        return userSink.asFlux().map(User::getId);
    }

    // Called by service when user is created
    public void onUserCreated(User user) {
        userSink.tryEmitNext(user);
    }
}
```

### GraphQL Client with WebClient

```java
@Service
public class GraphQLClient {

    private final WebClient webClient;

    public GraphQLClient(WebClient.Builder builder) {
        this.webClient = builder
            .baseUrl("https://api.example.com/graphql")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .build();
    }

    public <T> T query(String query, Map<String, Object> variables,
                      Class<T> responseType) {
        Map<String, Object> body = new HashMap<>();
        body.put("query", query);
        body.put("variables", variables);

        return webClient.post()
            .bodyValue(body)
            .retrieve()
            .bodyToMono(responseType)
            .block();
    }

    public UserResponse getUserById(Long id) {
        String query = """
            query getUser($id: ID!) {
                user(id: $id) {
                    id
                    name
                    email
                    posts {
                        id
                        title
                    }
                }
            }
            """;

        Map<String, Object> variables = Map.of("id", id);
        return query(query, variables, UserResponse.class);
    }
}
```

### Request Validation with Interceptor

```java
@Component
public class GraphQLInterceptor implements ExecutionInputCustomizer {

    @Override
    public CompletableFuture<ExecutionInput> customize(
            ExecutionInput executionInput,
            ExecutionGraphQlService schema,
            DataFetchingEnvironment environment) {

        // Add request context
        Object context = new HashMap<>(Map.of(
            "requestId", UUID.randomUUID().toString(),
            "timestamp", Instant.now()
        ));

        return CompletableFuture.completedFuture(
            ExecutionInput.newExecutionInput()
                .query(executionInput.getDocument().toString())
                .variables(executionInput.getVariables())
                .graphQLContext(context)
                .build()
        );
    }
}
```

## 5. Real-World Scenarios

### Scenario 1: Social Media Feed

```graphql
query FeedQuery {
  feed(first: 20) {
    edges {
      node {
        id
        text
        images
        author {
          id
          name
          avatar
        }
        comments(first: 3) {
          edges {
            node {
              text
              author { name }
            }
          }
        }
        likes {
          count
          likedByMe
        }
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

### Scenario 2: E-Commerce Product Catalog

```java
@Controller
public class ProductController {

    @QueryMapping
    public Product product(@Argument Long id) {
        return productService.findById(id);
    }

    @QueryMapping
    public ProductConnection products(
            @Argument Long categoryId,
            @Argument String search,
            @Argument Double minPrice,
            @Argument Double maxPrice,
            @Argument int first,
            @Argument String after) {

        return productConnectionService.getProducts(
            categoryId, search, minPrice, maxPrice, first, after);
    }

    @SchemaMapping
    public List<ProductVariant> variants(Product product) {
        return variantService.findByProductId(product.getId());
    }

    @SchemaMapping
    public Inventory inventory(Product product) {
        return inventoryService.getByProductId(product.getId());
    }
}
```

### Scenario 3: Real-Time Dashboard

```graphql
subscription DashboardSubscription {
  metricsUpdated {
    cpu
    memory
    requestsPerSecond
    errorRate
    activeUsers
  }
}
```

## 6. Performance

### Performance Optimization Techniques

- **DataLoader**: Batch and cache database requests per query.
- **Query Complexity Analysis**: Limit expensive queries.
- **Persisted Queries**: Pre-register queries to avoid parsing overhead.
- **Response Caching**: Cache full query responses at CDN level.
- **Field-Level Metrics**: Monitor resolver performance.
- **Max Query Depth**: Prevent deeply nested queries.

### Query Complexity Analysis

```java
@Component
public class ComplexityAnalysisFilter implements Coercing<Object, Object> {

    private final int maxComplexity = 1000;

    @Override
    public Object serialize(Object dataFetcherResult) {
        return dataFetcherResult;
    }

    @Override
    public Object parseValue(Object input) {
        return input;
    }

    @Override
    public Object parseLiteral(Object input) {
        return input;
    }
}

@Configuration
public class GraphQLConfig {

    @Bean
    public GraphQlSourceBuilderCustomizer complexityCustomizer() {
        return builder -> builder
            .configureGraphQl(graphQl -> graphQl
                .instrumentation(new MaxQueryComplexityInstrumentation(1000))
                .instrumentation(new MaxQueryDepthInstrumentation(10))
            );
    }
}
```

### Persisted Queries

```java
@Component
public class PersistedQueryRegistry {

    private final Map<String, String> queries = new HashMap<>();

    public PersistedQueryRegistry() {
        queries.put("get-user-profile", """
            query getUserProfile($id: ID!) {
                user(id: $id) {
                    name
                    email
                    posts { title }
                }
            }
            """);
        queries.put("get-post-feed", """
            query getPostFeed($first: Int!) {
                feed(first: $first) {
                    edges { node { id text author { name } } }
                }
            }
            """);
    }

    public String getQuery(String id) {
        return queries.get(id);
    }

    public boolean isRegistered(String id) {
        return queries.containsKey(id);
    }
}
```

## 7. Security

### GraphQL Security Concerns

- **Over-fetching abuse**: Malicious clients can request deeply nested data.
- **Query batching attacks**: Attackers can batch many expensive queries.
- **Introspection leaks**: Production should disable introspection.
- **Rate limiting**: Implement per-query cost rate limiting.
- **Authorization at field level**: Ensure users can only access authorized fields.

### Depth and Complexity Limiting

```java
@Component
public class GraphQLSecurityConfig {

    @Bean
    public Instrumentation maxDepthInstrumentation() {
        return new MaxQueryDepthInstrumentation(8);
    }

    @Bean
    public Instrumentation maxComplexityInstrumentation() {
        return new MaxQueryComplexityInstrumentation(500);
    }
}
```

### Field-Level Authorization

```java
@Component
public class AuthorizationDataFetcher implements DataFetcher<Object> {

    private final DataFetcher<Object> delegate;
    private final String requiredRole;

    public AuthorizationDataFetcher(DataFetcher<Object> delegate,
                                   String requiredRole) {
        this.delegate = delegate;
        this.requiredRole = requiredRole;
    }

    @Override
    public Object get(DataFetchingEnvironment environment) throws Exception {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();

        if (auth == null || !auth.getAuthorities().stream()
                .anyMatch(g -> g.getAuthority().equals(requiredRole))) {
            throw new AuthorizationException("Access denied");
        }

        return delegate.get(environment);
    }
}
```

## 8. Common Mistakes

- **Ignoring N+1 problem**: Always batch database calls with DataLoader.
- **Over-fetching in resolvers**: Only fetch fields that are requested.
- **Not using pagination**: GraphQL still needs pagination for lists.
- **Deeply nested queries**: Limit query depth to prevent abuse.
- **Exposing internal schema**: Disable introspection in production.
- **No query complexity limits**: Can lead to DoS attacks.
- **Treating GraphQL like REST**: Don't create separate endpoints.
- **Not handling errors properly**: Use structured error responses.

## 9. Senior Engineer Perspective

### When to Use GraphQL vs REST

**GraphQL is better for:**
- Complex data models with many relationships.
- Clients with varying data requirements.
- Mobile applications (bandwidth constrained).
- Rapidly evolving frontend requirements.

**REST is better for:**
- Simple CRUD operations.
- File upload/download.
- Caching-heavy use cases.
- Public APIs with diverse clients.

### GraphQL Federation (Microservices)

```
Gateway
  -> User Service (extends Query, User type)
  -> Post Service (extends Query, Post type)
  -> Comment Service (extends Query, Comment type)
  -> Notification Service (extends Subscription)
```

## 10. Interview Questions (Easy)

1. What is GraphQL and who developed it?
2. What are the three operation types in GraphQL?
3. What is the difference between a query and a mutation?
4. What is a resolver in GraphQL?
5. What is the GraphQL schema?
6. What is the difference between GraphQL and REST?
7. What is a GraphQL subscription?
8. What is the `!` symbol in GraphQL type definitions?
9. What is introspection in GraphQL?
10. How do you pass arguments to a GraphQL query?

## Medium

1. What is the N+1 problem in GraphQL and how do you solve it?
2. What is DataLoader and how does it work?
3. How do you implement pagination in GraphQL?
4. What is GraphQL federation?
5. How do you handle file uploads in GraphQL?
6. What is query complexity analysis?
7. How do you implement error handling in GraphQL?
8. What are GraphQL directives?
9. How do you test GraphQL APIs?
10. What is persisted queries in GraphQL?

## 11. Advanced Interview Questions (Hard)

1. Design a GraphQL schema for a multi-tenant SaaS platform.
2. How would you implement a real-time chat system using GraphQL subscriptions?
3. Implement a custom GraphQL scalar type for handling encrypted data.
4. How do you implement rate limiting based on query complexity in GraphQL?
5. Design a caching strategy for a GraphQL API with DataLoader.
6. How would you implement GraphQL federation across 10+ microservices?
7. Design an authorization system that works at the field level in GraphQL.
8. How do you handle versioning in GraphQL without breaking existing queries?
9. Implement a query cost analysis system that rejects expensive queries.
10. How would you migrate a REST API to GraphQL incrementally?

## System Design

1. Design a GraphQL API for a social media platform.
2. Design a GraphQL gateway for a microservices architecture.
3. Design a real-time analytics dashboard using GraphQL subscriptions.
4. Design a GraphQL-based content management system.
5. Design a GraphQL API for an e-commerce platform.
6. Design a GraphQL API with offline support for mobile applications.
7. Design a GraphQL monitoring and observability system.
8. Design a GraphQL API for a collaborative document editor.
9. Design a GraphQL-based BFF (Backend for Frontend) layer.
10. Design a GraphQL API with automatic schema stitching.

## 12. Expert-Level Interview Questions (Architect-Level)

1. Design a globally distributed GraphQL federation where each service can be independently deployed and scaled, with automatic schema composition and conflict resolution.
2. How would you implement a GraphQL API that transparently queries across both SQL databases and search indexes (Elasticsearch) with optimal performance?
3. Design a GraphQL schema evolution strategy that supports zero-downtime schema changes across a federation of services.
4. How would you implement a real-time collaborative system using GraphQL subscriptions with conflict-free replicated data types (CRDTs)?
5. Design a GraphQL caching layer that invalidates intelligently based on data dependencies rather than time-based expiry.
6. How would you build a GraphQL API that can serve both GraphQL and REST clients from the same backend logic?
7. Design a system that automatically generates GraphQL schemas from database schemas while allowing manual customization for business logic.
8. How would you implement rate limiting and cost management in a GraphQL API where different fields have vastly different computational costs?
9. Design a GraphQL API for a financial system that requires strict consistency, audit trails, and complex transactional mutations.
10. How would you implement a self-documenting GraphQL API that generates human-readable documentation from the schema and resolver annotations?

## 13. Debugging & Troubleshooting

### Common Issues

- **Slow queries**: Check for N+1 problems, missing DataLoader batching.
- **Timeout errors**: Reduce query depth, limit complexity.
- **Introspection disabled**: Enable introspection in development only.
- **Schema errors**: Validate schema changes before deployment.
- **Subscription disconnects**: Check WebSocket configuration and heartbeat.

### GraphQL Tracing

```java
@Component
public class GraphQLTracingInstrumentation extends SimpleInstrumentation {

    @Override
    public DataFetcher<?> instrumentDataFetcher(
            DataFetcher<?> dataFetcher,
            InstrumentationState fieldsState) {
        return environment -> {
            long start = System.currentTimeMillis();
            try {
                return dataFetcher.get(environment);
            } finally {
                long duration = System.currentTimeMillis() - start;
                log.info("Field {} resolved in {}ms",
                    environment.getField().getName(), duration);
            }
        };
    }
}
```

## 14. Comparison Section

### GraphQL vs REST

| Aspect | GraphQL | REST |
|--------|---------|------|
| Endpoints | Single endpoint | Multiple endpoints |
| Data Fetching | Client specifies exact needs | Server defines structure |
| Over-fetching | None (by design) | Common |
| Under-fetching | None (by design) | Common (multiple requests) |
| Caching | Complex (custom) | Simple (HTTP caching) |
| Versioning | Schema evolution | URI/header versioning |
| File Upload | Complex | Simple (multipart) |
| Tooling | Growing | Mature |
| Learning Curve | Moderate | Low |
| Type System | Built-in | OpenAPI/Swagger |

### GraphQL vs gRPC

| Aspect | GraphQL | gRPC |
|--------|---------|------|
| Query Flexibility | High (client defines query) | Low (fixed RPC methods) |
| Protocol | HTTP/1.1 or HTTP/2 | HTTP/2 |
| Data Format | JSON (text) | Protobuf (binary) |
| Streaming | Subscriptions | Native bidirectional |
| Code Generation | Schema-first | Contract-first (proto) |
| Use Case | Client-driven UIs | Service-to-service |

## 15. Revision Notes

- GraphQL: query language by Meta, single endpoint, client-specified fields
- 3 operations: Query (read), Mutation (write), Subscription (real-time)
- Resolvers fetch data per field; DataLoader batches to prevent N+1
- Schema defines types, inputs, and operations
- Spring for GraphQL: `@QueryMapping`, `@MutationMapping`, `@SubscriptionMapping`
- Security: limit depth, complexity, disable introspection in prod
- Federation: compose multiple GraphQL services into one endpoint
- Caching: harder than REST, use DataLoader and response caching

## 16. Cheat Sheet

```
+------------------------------------------------------------------+
| GRAPHQL CHEAT SHEET                                              |
+------------------------------------------------------------------+
| OPERATION TYPES                                                  |
|   query      { user(id: 1) { name } }    -- Read                 |
|   mutation   { createUser(...) { id } }  -- Write                |
|   subscription { userCreated { ... } }   -- Real-time            |
+------------------------------------------------------------------+
| TYPE SYSTEM                                                      |
|   Scalar: Int, Float, String, Boolean, ID                        |
|   Object: type User { id: ID! name: String! }                   |
|   Input:  input CreateUserInput { name: String! }                |
|   Enum:   enum Role { ADMIN USER }                               |
|   List:   [Post!]!  (non-null list of non-null items)            |
|   Union:  union SearchResult = User | Post                       |
|   Interface: interface Node { id: ID! }                          |
+------------------------------------------------------------------+
| SPRING BOOT ANNOTATIONS                                          |
|   @Controller            @QueryMapping                           |
|   @MutationMapping       @SubscriptionMapping                    |
|   @SchemaMapping         @Argument                               |
|   @BatchMapping          @GraphQlException                        |
+------------------------------------------------------------------+
| DATALOADER PATTERN                                               |
|   1. Collect all IDs from a batch                                |
|   2. Load all at once (SELECT * FROM users WHERE id IN (...))    |
|   3. Return Map<ID, Entity>                                      |
|   4. DataLoader caches per-request                                |
+------------------------------------------------------------------+
| PAGINATION (Relay Connection Spec)                               |
|   type UserConnection {                                           |
|     edges: [UserEdge!]!                                          |
|     pageInfo: PageInfo!                                          |
|   }                                                              |
|   type UserEdge { node: User! cursor: String! }                 |
|   type PageInfo { hasNextPage: Boolean! endCursor: String }      |
+------------------------------------------------------------------+
| SECURITY                                                         |
|   Disable introspection in production                            |
|   Max query depth (8-10)                                         |
|   Max query complexity (500-1000)                                |
|   Field-level authorization                                      |
|   Rate limit by query cost                                       |
|   Validate all inputs                                            |
+------------------------------------------------------------------+
```
