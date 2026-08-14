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
   - **Answer:** REST is an architectural style for designing networked applications using stateless HTTP operations on resources identified by URLs. In our cold-chain project, we exposed endpoints like `GET /api/sensors/{id}/readings` to fetch time-series data for a specific sensor.
   - **If asked more:** I would explain the six REST constraints and how statelessness simplified our EC2 deployment - any instance could handle any request without session affinity.
2. What are REST constraints?
   - **Answer:** The six constraints are: client-server separation, statelessness, cacheability, uniform interface (URI-based resource identification), layered system, and optionally code on demand. Our APIs were stateless and used Redis for cacheability.
   - **If asked more:** I would discuss how statelessness impacts scalability - no server-side session needed, any EC2 instance handles any request - and how we used ETags for caching.
3. Difference between REST and SOAP.
   - **Answer:** REST uses JSON/HTTP with resource-based URLs and standard methods, while SOAP is an XML-based protocol with strict schemas and WSDL. I have not used SOAP in my projects; our APIs were always RESTful JSON over HTTP, simpler and faster for web dashboards.
   - **If asked more:** I would explain that SOAP has built-in error handling and WS-Security for enterprise use, while REST is preferred for web/mobile APIs due to simplicity and browser compatibility.
4. Difference between REST and GraphQL.
   - **Answer:** REST has fixed response structures per endpoint, while GraphQL lets clients request exactly the fields they need. Our inventory API used REST because partners needed consistent, predefined reports rather than flexible querying.
   - **If asked more:** I would discuss REST's caching advantage (each URL is cacheable) vs GraphQL's flexibility, and how for real-time IoT data, REST with pagination was sufficient.
5. What are HTTP methods?
   - **Answer:** The main methods are GET (read), POST (create), PUT (full update/replace), PATCH (partial update), DELETE (remove). In our inventory API, we used POST to submit serial records, GET to fetch inventory status, and PUT to update baseline data.
   - **If asked more:** I would explain safe (GET/HEAD/OPTIONS) vs idempotent (GET/PUT/DELETE) methods, and how this affects retry behavior.
6. Difference between PUT and PATCH.
   - **Answer:** PUT replaces the entire resource; PATCH applies a partial update. If updating only a partner's status, PATCH is efficient - send just the status field. PUT requires the full object. In our inventory system, PATCH was useful for updating specific validation flags.
   - **If asked more:** I would discuss that PUT is inherently idempotent, while PATCH requires careful handling to remain idempotent using JSON Patch format.
7. Difference between POST and PUT.
   - **Answer:** POST creates a resource at a server-generated URL (non-idempotent), while PUT creates or replaces at a client-specified URL (idempotent). In CDMS, POST submitted new partner data, PUT updated existing records with known IDs.
   - **If asked more:** I would explain: POST for submissions where the server assigns an ID, PUT for updates where the client knows the resource's exact URL.
8. What is idempotency?
   - **Answer:** Idempotency means making the same request multiple times produces the same result as once. In our inventory submission, batch reference IDs made the endpoint idempotent - retried batches returned existing results instead of processing duplicate records.
   - **If asked more:** I would discuss idempotency keys in headers and how Redis stores key-response mappings with TTL for retries within a time window.
9. Which HTTP methods are idempotent?
   - **Answer:** GET, PUT, DELETE, HEAD, and OPTIONS are idempotent. POST and PATCH are not inherently idempotent. For our critical POST endpoints like inventory submission, we added idempotency-key support separately.
   - **If asked more:** I would explain that DELETE idempotency means the second call returns 404 instead of 200, but the server state remains the same - the resource is still deleted.
10. What are common HTTP status codes?
    - **Answer:** Common codes: 200 OK, 201 Created, 204 No Content, 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable Entity, 500 Internal Server Error, 503 Service Unavailable. We consistently used these across our APIs.
    - **If asked more:** I would group codes by category - 2xx success, 3xx redirection, 4xx client error, 5xx server error - and explain how consistent status codes make API consumers more predictable.
11. Difference between 400 and 422.
    - **Answer:** 400 means the request is malformed (invalid JSON, missing required fields), while 422 means the format is valid but content violates business rules. In our inventory API, we returned 400 for JSON parse errors and 422 for serial number validation failures.
    - **If asked more:** I would explain the practical distinction: 400 is for syntax-level issues, 422 for semantic-level issues. Proper distinction helps clients show better error messages.
12. Difference between 401 and 403.
    - **Answer:** 401 means the client is not authenticated - needs valid credentials. 403 means the client is authenticated but lacks permission. In our Spring Security setup, 401 was returned for missing/invalid JWT, 403 for insufficient roles.
    - **If asked more:** I would explain that 401 triggers browser login prompts, while 403 does not. Confusing these is a common API design mistake.
13. Difference between 404 and 410.
    - **Answer:** 404 means the resource does not exist currently; 410 means it intentionally existed but was permanently removed. In CDMS, deactivated partners' reports could return 410 to indicate intentional unavailability rather than a missing resource.
    - **If asked more:** I would discuss how many APIs return 404 instead of 403 for hidden resources to avoid revealing existence to unauthorized users.
14. Difference between 500 and 503.
    - **Answer:** 500 is a generic server error; 503 means the server is temporarily unavailable (overloaded or under maintenance). In our cold-chain project, if InfluxDB was down, the API returned 503 to signal temporary unavailability and prompt client retries.
    - **If asked more:** I would explain that 503 should include a Retry-After header, and monitoring should alert on sudden 5xx rate increases.
15. What is request validation?
    - **Answer:** Request validation ensures incoming data meets format, type, and business rules before processing. In our inventory API, we used Spring Boot's `@Valid` with Bean Validation annotations and custom validators for serial number rules - rejecting invalid requests with 400/422 before they reached the service layer.
    - **If asked more:** I would discuss multi-layer validation: schema (field types), format (regex patterns), and business (duplicate checks, status transitions).
16. What is DTO?
    - **Answer:** DTO (Data Transfer Object) decouples the API contract from internal entities. In our projects, DTOs exposed only necessary fields to API consumers, preventing internal entity changes from breaking the API contract and avoiding oversharing sensitive data.
    - **If asked more:** I would explain the security aspect - DTOs prevent exposing password hashes or internal IDs and reduce payload size by including only needed fields.
17. Why not expose entity directly?
    - **Answer:** Exposing JPA entities directly couples the database schema to the API contract - any schema change breaks the API. It also risks exposing sensitive fields and causes lazy loading or infinite serialization issues with bidirectional relationships.
    - **If asked more:** I would add that direct entity exposure can cause performance overhead from loading unnecessary fields and make API evolution harder.
18. What is global exception handling?
    - **Answer:** Global exception handling centralizes error handling across all controllers, returning consistent error responses. In our Spring Boot projects, `@ControllerAdvice` caught validation errors, not-found exceptions, and server errors - all returned in a uniform `{error, message, timestamp}` JSON format.
    - **If asked more:** I would explain our error response structure: HTTP status, machine-readable error code, human-readable message, and timestamp. Consistency helped the React frontend handle errors generically.
19. How do you design error response?
    - **Answer:** I design error responses with a consistent structure: `{"status": 400, "error": "VALIDATION_ERROR", "message": "Serial number must be alphanumeric", "timestamp": "2024-01-15T10:30:00Z"}`. For 422 errors, we included a `details` array with field-level validation information.
    - **If asked more:** I would discuss including a trace ID in error responses for debugging, and the importance of not exposing sensitive information in error messages.
20. What is API versioning?
    - **Answer:** API versioning allows multiple API versions to coexist, giving consumers migration time. I prefer URI versioning (`/api/v1/inventory`) because it is explicit and easy to route. In our internal projects, versioning was not needed due to controlled consumers.
    - **If asked more:** I would compare URI vs header vs query parameter versioning, and recommend URI for simplicity and cacheability.
21. URL versioning vs header versioning.
    - **Answer:** URL versioning places the version in the path (`/api/v1/`), making it visible and easy to route. Header versioning uses a custom Accept header, keeping URLs clean. I prefer URL versioning for simplicity - anyone testing the API can see the version immediately in the URL.
    - **If asked more:** I would discuss that URL versioning can lead to code duplication across versions, while header versioning keeps URL structure clean but requires client library support.
22. What is pagination?
    - **Answer:** Pagination splits large result sets into smaller pages, reducing response size and server load. In the inventory dashboard, partner lists were paginated with page/pageSize parameters, returning metadata like total pages for the UI to render pagination controls.
    - **If asked more:** I would discuss cursor vs offset-based pagination: offset was fine for slow-changing inventory data, but cursor is better for real-time sensor data where new records are constantly added.
23. What is sorting?
    - **Answer:** Sorting allows clients to order results by specified fields. In CDMS reports, the API accepted `sort=partnerName,asc&sort=revenue,desc` for multi-field ordering. We used Spring Data's Sort object with a whitelist to prevent SQL injection via sort fields.
    - **If asked more:** I would discuss sort field whitelisting - never passing user input to ORDER BY without validation, as it risks injection and performance issues from unindexed columns.
24. What is filtering?
    - **Answer:** Filtering narrows results based on criteria like date range, status, or category. In CDMS, partners filtered reports by date range, region, and product line. We used Spring Data JPA Specifications for dynamic WHERE clauses with proper indexes on filtered columns.
    - **If asked more:** I would discuss filter syntax: query parameters (`?status=active`) for simple filters and JSON filter objects for complex nested filters.
25. What is HATEOAS?
    - **Answer:** HATEOAS includes links in API responses to guide clients to related actions. For example, an inventory record response might include links to "update", "delete", and "view-history". We did not use HATEOAS in our projects, but I understand it makes APIs self-documenting.
    - **If asked more:** I would discuss the tradeoff: HATEOAS increases payload size and complexity but enables loose coupling where clients discover actions dynamically.
26. What is content negotiation?
    - **Answer:** Content negotiation lets clients request different response formats using the Accept header. Our APIs primarily returned JSON via `application/json`. Spring Boot handles this automatically based on the Accept header and registered message converters.
    - **If asked more:** I would explain producer vs consumer negotiation and how we used `produces = "application/json"` in controller mappings.
27. What is CORS?
    - **Answer:** CORS is a browser security mechanism controlling which domains can access a web API. In our cold-chain project, the React dashboard ran on a different port than the Spring Boot API, so we configured CORS in Spring Security to allow requests from the dashboard's origin.
    - **If asked more:** I would explain preflight requests (OPTIONS), CORS headers like `Access-Control-Allow-Origin`, and that CORS is only enforced by browsers, not by server-to-server calls or Postman.
28. How do you secure REST APIs?
    - **Answer:** I secure APIs with HTTPS, JWT authentication, role-based access control, input validation, rate limiting, and CORS configuration. We used Spring Security with JWT - users authenticated, received a token, and included it in the Authorization header for subsequent requests.
    - **If asked more:** I would discuss OAuth2 for third-party access and the principle of least privilege - each endpoint validates the authenticated user has permission for the specific action.
29. How do you document REST APIs?
    - **Answer:** I document APIs using OpenAPI/Swagger via springdoc-openapi, which auto-generates interactive documentation from code annotations. This allowed our frontend developers to test endpoints from the browser without needing curl.
    - **If asked more:** I would discuss the importance of keeping docs in sync with code (annotation-based ensures this), adding meaningful descriptions, and including example request/response bodies.
30. What is OpenAPI?
    - **Answer:** OpenAPI is a standard specification for describing REST APIs using JSON/YAML, defining endpoints, parameters, schemas, and authentication. We used springdoc-openapi to auto-generate OpenAPI docs from annotations, producing a Swagger UI for team consumption.
    - **If asked more:** I would explain how OpenAPI enables client SDK generation, automated testing, and API gateway configuration, making it a valuable contract-first approach.
31. How do you test REST APIs?
    - **Answer:** I test at multiple levels: unit tests for service logic, `@WebMvcTest` for controllers with mocked services, `@SpringBootTest` with Testcontainers for integration tests, and Postman collections for end-to-end validation. We used Postman in the cold-chain project before frontend integration.
    - **If asked more:** I would discuss contract testing with Pact, automated smoke tests in the Jenkins pipeline, and performance testing with JMeter to validate response times under load.
32. How do you handle long-running API requests?
    - **Answer:** For operations taking more than a few seconds, I return 202 Accepted with a tracking ID and process asynchronously. The client polls a status endpoint using the tracking ID. In CDMS, large report generation used this async pattern - submit request, get tracking ID, poll for completion.
    - **If asked more:** I would add that webhooks or Server-Sent Events can push completion notifications instead of polling, reducing load on the status endpoint.
33. How do you handle file upload?
    - **Answer:** I use multipart form data with Spring Boot's MultipartFile, validate file type and size, store to filesystem or AWS S3, and return the file URL. In our inventory system, partners uploaded serial record files which were parsed and validated against the baseline.
    - **If asked more:** I would discuss streaming large files to avoid OutOfMemoryError, virus scanning, and using presigned S3 URLs for direct client uploads.
34. How do you handle large API response?
    - **Answer:** I handle large responses with pagination, field selection, gzip compression, and streaming. In the cold-chain project, sensor reading responses were paginated by default, and we enabled gzip compression at the load balancer level for slow network connections.
    - **If asked more:** I would discuss JSON streaming for very large datasets, allowing clients to start processing before the full response is received.
35. How do you handle backward compatibility?
    - **Answer:** I follow the Robustness Principle - be conservative in what you send, liberal in what you accept. I never remove or rename fields, add new fields as optional, and deprecate endpoints gradually with a migration guide in Swagger documentation.
    - **If asked more:** I would discuss what constitutes breaking vs non-breaking changes: adding fields is safe, changing types is breaking, removing endpoints requires versioning.

