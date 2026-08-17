# REST API Questions

## Questions

1. What is REST?
2. What are REST constraints?
3. Difference between REST and SOAP.
4. Difference between REST and GraphQL.
5. What are HTTP methods?
6. Difference between PUT and PATCH.
7. Difference between POST and PUT.
8. What is idempotency?
9. Which HTTP methods are idempotent?
10. What are common HTTP status codes?
11. Difference between 400 and 422.
12. Difference between 401 and 403.
13. Difference between 404 and 410.
14. Difference between 500 and 503.
15. What is request validation?
16. What is DTO?
17. Why not expose entity directly?
18. What is global exception handling?
19. How do you design error response?
20. What is API versioning?
21. URL versioning vs header versioning.
22. What is pagination?
23. What is sorting?
24. What is filtering?
25. What is HATEOAS?
26. What is content negotiation?
27. What is CORS?
28. How do you secure REST APIs?
29. How do you document REST APIs?
30. What is OpenAPI?
31. How do you test REST APIs?
32. How do you handle long-running API requests?
33. How do you handle file upload?
34. How do you handle large API response?
35. How do you handle backward compatibility?

---

## Answers

1. What is REST?
   - **Answer:**
      - An architectural style for designing networked applications using stateless HTTP operations on resources identified by URLs
      - In our cold-chain project, we exposed endpoints like `GET /api/sensors/{id}/readings` to fetch time-series data for a specific sensor
      - The six REST constraints: client-server, statelessness, cacheability, uniform interface, layered system, and code on demand; statelessness simplified our EC2 deployment since any instance could handle any request without session affinity

2. What are REST constraints?
   - **Answer:**
      - The six constraints are: client-server separation, statelessness, cacheability, uniform interface (URI-based resource identification), layered system, and optionally code on demand
      - Our APIs were stateless and used Redis for cacheability
      - Statelessness impacts scalability: no server-side session needed, any EC2 instance handles any request; we used ETags for caching to avoid redundant data transfers

3. Difference between REST and SOAP.
   - **Answer:**
      - REST uses JSON/HTTP with resource-based URLs and standard methods, while SOAP is an XML-based protocol with strict schemas and WSDL
      - I have not used SOAP in my projects; our APIs were always RESTful JSON over HTTP, simpler and faster for web dashboards
      - SOAP has built-in error handling via SOAP Faults and WS-Security for enterprise use (digital signatures, encryption), while REST is preferred for web/mobile APIs due to simplicity, lower overhead, and browser compatibility

4. Difference between REST and GraphQL.
   - **Answer:**
      - REST has fixed response structures per endpoint, while GraphQL lets clients request exactly the fields they need
      - Our inventory API used REST because partners needed consistent, predefined reports rather than flexible querying
      - REST's caching advantage: each URL is independently cacheable by CDNs and browsers, while GraphQL uses a single endpoint making HTTP-level caching harder; for our real-time IoT data, REST with pagination was sufficient

5. What are HTTP methods?
   - **Answer:**
      - The main methods are GET (read), POST (create), PUT (full update/replace), PATCH (partial update), DELETE (remove)
      - In our inventory API, we used POST to submit serial records, GET to fetch inventory status, and PUT to update baseline data
      - Safe methods (GET, HEAD, OPTIONS) do not modify server state; idempotent methods (GET, PUT, DELETE) produce the same result when called multiple times, making them safe to retry on network failures

6. Difference between PUT and PATCH.
   - **Answer:**
      - PUT replaces the entire resource; PATCH applies a partial update
      - If updating only a partner's status, PATCH is efficient - send just the status field
      - PUT requires the full object
      - In our inventory system, PATCH was useful for updating specific validation flags
      - PUT is inherently idempotent (sending the same full resource twice yields the same state), while PATCH requires careful handling to remain idempotent - using JSON Patch format (RFC 6902) with explicit operations ensures repeatable partial updates

7. Difference between POST and PUT.
   - **Answer:**
      - POST creates a resource at a server-generated URL (non-idempotent), while PUT creates or replaces at a client-specified URL (idempotent)
      - In CDMS, POST submitted new partner data, PUT updated existing records with known IDs
      - POST is for submissions where the server assigns the ID and returns 201 with Location header; PUT is for updates where the client knows the resource's exact URL and sends the complete representation

8. What is idempotency?
   - **Answer:**
      - Idempotency means making the same request multiple times produces the same result as once
      - In our inventory submission, batch reference IDs made the endpoint idempotent - retried batches returned existing results instead of processing duplicate records
      - Idempotency keys are sent in request headers (e.g., `Idempotency-Key: abc123`); Redis stores key-to-response mappings with a TTL so that repeated requests within the time window return the cached response without reprocessing

9. Which HTTP methods are idempotent?
   - **Answer:**
      - GET, PUT, DELETE, HEAD, and OPTIONS are idempotent
      - POST and PATCH are not inherently idempotent
      - For our critical POST endpoints like inventory submission, we added idempotency-key support separately
      - DELETE idempotency means the first call returns 200 (deleted), but the second call returns 404 (not found) - the server state remains the same (resource stays deleted), so the operation is idempotent despite different status codes

10. What are common HTTP status codes?
    - **Answer:**
       - Common codes: 200 OK, 201 Created, 204 No Content, 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable Entity, 500 Internal Server Error, 503 Service Unavailable
       - We consistently used these across our APIs
       - Status codes grouped by category: 2xx (success: 200 OK, 201 Created, 204 No Content), 3xx (redirection: 301 Moved, 304 Not Modified), 4xx (client error: 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found), 5xx (server error: 500 Internal, 503 Unavailable); consistent codes make API consumers more predictable

11. Difference between 400 and 422.
    - **Answer:**
       - 400 means the request is malformed (invalid JSON, missing required fields), while 422 means the format is valid but content violates business rules
       - In our inventory API, we returned 400 for JSON parse errors and 422 for serial number validation failures
       - 400 is for syntax-level issues (malformed JSON, invalid URL format), 422 is for semantic-level issues (valid JSON but business rule violations); proper distinction helps clients show better error messages - "fix your request format" vs "your data failed validation"

12. Difference between 401 and 403.
    - **Answer:**
       - 401 means the client is not authenticated - needs valid credentials
       - 403 means the client is authenticated but lacks permission
       - In our Spring Security setup, 401 was returned for missing/invalid JWT, 403 for insufficient roles
       - 401 triggers browser login prompts because it includes a WWW-Authenticate header indicating authentication is required, while 403 does not; confusing these is a common API design mistake - 401 means "who are you?" and 403 means "I know who you are, but you can't do this"

13. Difference between 404 and 410.
    - **Answer:**
       - 404 means the resource does not exist currently; 410 means it intentionally existed but was permanently removed
       - In CDMS, deactivated partners' reports could return 410 to indicate intentional unavailability rather than a missing resource
       - Many APIs intentionally return 404 instead of 403 for hidden resources to avoid revealing existence to unauthorized users - returning 403 confirms a resource exists, while 404 maintains plausible deniability

14. Difference between 500 and 503.
    - **Answer:**
       - 500 is a generic server error; 503 means the server is temporarily unavailable (overloaded or under maintenance)
       - In our cold-chain project, if InfluxDB was down, the API returned 503 to signal temporary unavailability and prompt client retries
       - 503 should include a Retry-After header telling clients when to retry (e.g., `Retry-After: 30`), and monitoring should alert on sudden 5xx rate increases to catch outages early

15. What is request validation?
    - **Answer:**
       - Request validation ensures incoming data meets format, type, and business rules before processing
       - In our inventory API, we used Spring Boot's `@Valid` with Bean Validation annotations and custom validators for serial number rules - rejecting invalid requests with 400/422 before they reached the service layer
       - Multi-layer validation: schema validation (field types and required fields), format validation (regex patterns like alphanumeric serial numbers), and business validation (duplicate checks, status transitions like can't deactivate an already inactive partner)

16. What is DTO?
    - **Answer:**
       - DTO (Data Transfer Object) decouples the API contract from internal entities
       - In our projects, DTOs exposed only necessary fields to API consumers, preventing internal entity changes from breaking the API contract and avoiding oversharing sensitive data
       - DTOs prevent exposing sensitive fields like password hashes, internal database IDs, and audit metadata; they reduce payload size by including only the fields the client actually needs

17. Why not expose entity directly?
    - **Answer:**
       - Exposing JPA entities directly couples the database schema to the API contract - any schema change breaks the API
       - It also risks exposing sensitive fields and causes lazy loading or infinite serialization issues with bidirectional relationships
       - Direct entity exposure causes performance overhead from loading unnecessary fields (e.g., large text blobs, relationship collections), and makes API evolution harder because any field addition or removal becomes a breaking API change

18. What is global exception handling?
    - **Answer:**
       - Global exception handling centralizes error handling across all controllers, returning consistent error responses
       - In our Spring Boot projects, `@ControllerAdvice` caught validation errors, not-found exceptions, and server errors - all returned in a uniform `{error, message, timestamp}` JSON format
       - Our error response structure included HTTP status, machine-readable error code (e.g., VALIDATION_ERROR), human-readable message, and timestamp; this consistency helped the React frontend handle errors generically with a single error interceptor

19. How do you design error response?
    - **Answer:**
       - I design error responses with a consistent structure: `{"status": 400, "error": "VALIDATION_ERROR", "message": "Serial number must be alphanumeric", "timestamp": "2024-01-15T10:30:00Z"}`
       - For 422 errors, we included a `details` array with field-level validation information
       - Error responses should include a trace ID (correlation ID) for debugging and log correlation, and must not expose sensitive information like stack traces, database queries, or internal file paths to the client

20. What is API versioning?
    - **Answer:**
       - API versioning allows multiple API versions to coexist, giving consumers migration time
       - I prefer URI versioning (`/api/v1/inventory`) because it is explicit and easy to route
       - In our internal projects, versioning was not needed due to controlled consumers
       - URI versioning (`/api/v1/`) is explicit and cacheable but duplicates code across versions; header versioning keeps URLs clean but requires client library support; query parameter versioning (`?version=1`) is simple but clutters URLs and complicates caching; I recommend URI versioning for its simplicity and cacheability

21. URL versioning vs header versioning.
    - **Answer:**
       - URL versioning places the version in the path (`/api/v1/`), making it visible and easy to route
       - Header versioning uses a custom Accept header, keeping URLs clean
       - I prefer URL versioning for simplicity - anyone testing the API can see the version immediately in the URL
       - URL versioning can lead to code duplication across versions (maintaining v1 and v2 controllers), while header versioning keeps URL structure clean but requires client library support and makes direct browser testing harder

22. What is pagination?
    - **Answer:**
       - Pagination splits large result sets into smaller pages, reducing response size and server load
       - In the inventory dashboard, partner lists were paginated with page/pageSize parameters, returning metadata like total pages for the UI to render pagination controls
       - Offset-based pagination (`?page=2&size=20`) works fine for slow-changing data like inventory records, but cursor-based pagination (`?cursor=abc123&limit=20`) is better for real-time sensor data where new records are constantly inserted - offset pagination can skip or duplicate records when data shifts between requests

23. What is sorting?
    - **Answer:**
       - Sorting allows clients to order results by specified fields
       - In CDMS reports, the API accepted `sort=partnerName,asc&sort=revenue,desc` for multi-field ordering
       - We used Spring Data's Sort object with a whitelist to prevent SQL injection via sort fields
       - Sort field whitelisting is critical - never pass user input directly to ORDER BY without validation, as it risks SQL injection and performance issues from sorting on unindexed columns; we maintained a whitelist of allowed sort fields mapped to actual column names

24. What is filtering?
    - **Answer:**
       - Filtering narrows results based on criteria like date range, status, or category
       - In CDMS, partners filtered reports by date range, region, and product line
       - We used Spring Data JPA Specifications for dynamic WHERE clauses with proper indexes on filtered columns
       - Simple filters use query parameters (`?status=active&region=north`) which are easy to read and bookmark; complex nested filters use JSON filter objects in the request body (`{"and": [{"field": "status", "op": "eq", "value": "active"}]}`) for advanced conditional logic

25. What is HATEOAS?
    - **Answer:**
       - HATEOAS includes links in API responses to guide clients to related actions
       - For example, an inventory record response might include links to "update", "delete", and "view-history"
       - We did not use HATEOAS in our projects, but I understand it makes APIs self-documenting
       - HATEOAS increases payload size and complexity but enables loose coupling where clients discover available actions dynamically from response links rather than hardcoding URLs; clients can navigate the API without prior knowledge of endpoint structures

26. What is content negotiation?
    - **Answer:**
       - Content negotiation lets clients request different response formats using the Accept header
       - Our APIs primarily returned JSON via `application/json`
       - Spring Boot handles this automatically based on the Accept header and registered message converters
       - Producer negotiation: the server declares what it can produce via `produces = "application/json"` in controller mappings; consumer negotiation: the client declares what it wants via the Accept header; Spring Boot matches both and returns 406 Not Acceptable if no match is found

27. What is CORS?
    - **Answer:**
       - CORS is a browser security mechanism controlling which domains can access a web API
       - In our cold-chain project, the React dashboard ran on a different port than the Spring Boot API, so we configured CORS in Spring Security to allow requests from the dashboard's origin
       - Preflight requests: browsers send an OPTIONS request before the actual request to check if the server allows it; CORS headers like `Access-Control-Allow-Origin` and `Access-Control-Allow-Methods` control access; CORS is only enforced by browsers - server-to-server calls and tools like Postman/curl bypass it entirely

28. How do you secure REST APIs?
    - **Answer:**
       - I secure APIs with HTTPS, JWT authentication, role-based access control, input validation, rate limiting, and CORS configuration
       - We used Spring Security with JWT - users authenticated, received a token, and included it in the Authorization header for subsequent requests
       - OAuth2 enables third-party access via authorization codes and access tokens without sharing credentials; the principle of least privilege means each endpoint validates the authenticated user has permission for the specific action - an admin endpoint rejects regular users even if they have a valid JWT

29. How do you document REST APIs?
    - **Answer:**
       - I document APIs using OpenAPI/Swagger via springdoc-openapi, which auto-generates interactive documentation from code annotations
       - This allowed our frontend developers to test endpoints from the browser without needing curl
       - Keeping docs in sync with code is critical - annotation-based documentation (springdoc-openapi) ensures docs update automatically when code changes; meaningful descriptions and example request/response bodies help API consumers understand expected formats without reading source code

30. What is OpenAPI?
    - **Answer:**
       - OpenAPI is a standard specification for describing REST APIs using JSON/YAML, defining endpoints, parameters, schemas, and authentication
       - We used springdoc-openapi to auto-generate OpenAPI docs from annotations, producing a Swagger UI for team consumption
       - OpenAPI enables automatic client SDK generation (TypeScript, Python, Java clients), automated testing against the API contract, and API gateway configuration - making it a valuable contract-first approach where the API spec drives development

31. How do you test REST APIs?
    - **Answer:**
       - I test at multiple levels: unit tests for service logic, `@WebMvcTest` for controllers with mocked services, `@SpringBootTest` with Testcontainers for integration tests, and Postman collections for end-to-end validation
       - We used Postman in the cold-chain project before frontend integration
       - Contract testing with Pact ensures API providers and consumers agree on the contract; automated smoke tests in the Jenkins pipeline catch regressions on every build; performance testing with JMeter validates response times under load to prevent latency regressions

32. How do you handle long-running API requests?
    - **Answer:**
       - For operations taking more than a few seconds, I return 202 Accepted with a tracking ID and process asynchronously
       - The client polls a status endpoint using the tracking ID
       - In CDMS, large report generation used this async pattern - submit request, get tracking ID, poll for completion
       - Webhooks push a callback notification to the client's URL when processing completes; Server-Sent Events (SSE) allow the client to subscribe to a stream of status updates - both approaches reduce load on the status endpoint compared to repeated polling

33. How do you handle file upload?
    - **Answer:**
       - I use multipart form data with Spring Boot's MultipartFile, validate file type and size, store to filesystem or AWS S3, and return the file URL
       - In our inventory system, partners uploaded serial record files which were parsed and validated against the baseline
       - Streaming large files avoids OutOfMemoryError by processing the upload in chunks rather than loading the entire file into memory; virus scanning runs before storage; presigned S3 URLs let clients upload directly to S3 without proxying through the application server, reducing server load

34. How do you handle large API response?
    - **Answer:**
       - I handle large responses with pagination, field selection, gzip compression, and streaming
       - In the cold-chain project, sensor reading responses were paginated by default, and we enabled gzip compression at the load balancer level for slow network connections
       - JSON streaming (e.g., using Jackson's streaming API or NDJSON) sends data chunk by chunk, allowing clients to start processing records before the full response is received, reducing time-to-first-byte and memory usage on both server and client

35. How do you handle backward compatibility?
    - **Answer:**
       - I follow the Robustness Principle - be conservative in what you send, liberal in what you accept
       - I never remove or rename fields, add new fields as optional, and deprecate endpoints gradually with a migration guide in Swagger documentation
       - Breaking vs non-breaking changes: adding new optional fields is safe (clients ignore unknown fields), changing field types is breaking (e.g., string to integer), removing or renaming fields is breaking, changing URL paths requires versioning; always deprecate first with documentation before removing
