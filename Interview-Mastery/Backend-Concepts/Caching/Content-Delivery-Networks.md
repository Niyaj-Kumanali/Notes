# Content Delivery Networks (CDN)

## 1. Executive Summary

A Content Delivery Network (CDN) is a geographically distributed network of proxy servers and data centers that delivers internet content to users with high availability and performance. CDNs reduce latency by serving content from edge locations closest to the user, offload traffic from origin servers, and provide DDoS protection. Modern CDNs handle static assets (images, CSS, JS), dynamic content, API responses, video streaming, and even serverless compute at the edge.

## 2. Core Theory

CDNs operate on the principle of caching content at edge nodes distributed across the globe. When a user requests content, the CDN routes the request to the nearest edge server, which serves the cached copy if available, or fetches it from the origin server.

**Key Concepts:**
- **Edge Server / Point of Presence (PoP):** Physical server at a CDN datacenter that caches and serves content.
- **Origin Server:** The authoritative source of the content (your web server or cloud storage).
- **Cache Hit:** Content served from edge cache (fast).
- **Cache Miss:** Content fetched from origin (slower, uses origin bandwidth).
- **Cache Ratio:** Percentage of requests served from cache (target >90%).
- **TTL (Time-To-Live):** How long the CDN caches content before revalidating.
- **Purge / Invalidation:** Forcibly removing cached content across all edges.
- **Pre-warming / Pre-fetching:** Proactively pushing content to edge caches before it is requested.

**CDN Topology:**
```
[User in Tokyo] <-low latency-> [CDN Edge: Tokyo]
                                   |
[User in London] <-low latency-> [CDN Edge: London]
                                   |
                              [Origin (us-east-1)]
```

## 3. Under-the-Hood Deep Dive

### Request Routing

CDNs use DNS-based or Anycast-based routing to direct users to the optimal edge:

**DNS-based routing:** User's DNS resolver gets different IPs based on geolocation. CDN resolves to the edge closest to the resolver.
- Pros: Simple, no protocol changes.
- Cons: DNS caching can cause suboptimal routing, resolver location may differ from user location.

**Anycast routing:** Same IP advertised from multiple PoPs. BGP routing naturally sends user to closest PoP.
- Pros: Optimal routing, fast failover, simpler DNS.
- Cons: BGP convergence time on failure, complex network engineering.

### Caching Hierarchy

```
[Edge Node] -> [Regional Hub] -> [Origin Shield] -> [Origin Server]
```

- **Edge Node:** First-level cache, serves local users. Small capacity, high churn.
- **Regional Hub:** Aggregation layer. Caches less popular content. Reduces origin load.
- **Origin Shield:** Dedicated layer that absorbs cache misses from all edges, preventing "cache stampede" on origin.
- **Origin:** Your application server.

### Content Purging Mechanisms

- **Exact URI Purge:** Remove specific URL pattern.
- **Tag-based Purge:** Invalidate all content tagged with a specific tag (e.g., "product-123").
- **Directory Purge:** Invalidate entire directory.
- **Wildcard Purge:** `/*.jpg` or `/products/*`.
- **Cache Key Manipulation:** Changing the cache key to force new content.

### CDN Cache Key

A cache key is how the CDN uniquely identifies a cached object. Default is the full URL. Customization options:
- Ignore query parameters: `?utm_source=facebook` should not create different cache entries.
- Include headers: `Accept-Language` for language-specific content.
- Include cookies: Session-specific limited caching.

## 4. Production Code Examples

### Spring Boot Integration with CloudFront via Custom Headers

```java
@Configuration
public class CdnConfig {

    @Value("${cdn.base-url}")
    private String cdnBaseUrl;

    @Value("${cdn.secret-header}")
    private String secretHeader;

    @Value("${cdn.secret-value}")
    private String secretValue;

    @Bean
    public RestTemplate cdnRestTemplate() {
        return new RestTemplateBuilder()
            .defaultHeader(secretHeader, secretValue)
            .build();
    }

    public String getCdnUrl(String path) {
        return cdnBaseUrl + "/" + path;
    }
}
```

### CDN Cache Invalidation via AWS CloudFront API

```java
@Service
public class CloudFrontCacheService {

    @Autowired
    private CloudFrontClient cloudFrontClient;

    @Value("${cloudfront.distribution-id}")
    private String distributionId;

    public void invalidatePaths(List<String> paths) {
        CloudFrontWaiter waiter = cloudFrontClient.waiter();

        CreateInvalidationRequest request = CreateInvalidationRequest.builder()
            .distributionId(distributionId)
            .invalidationBatch(InvalidationBatch.builder()
                .paths(Paths.builder()
                    .items(paths)
                    .quantity(paths.size())
                    .build())
                .callerReference(String.valueOf(System.currentTimeMillis()))
                .build())
            .build();

        cloudFrontClient.createInvalidation(request);
    }

    public void invalidateByTag(String tag) {
        // With Lambda@Edge or CloudFront Functions, associate tags to response headers.
        // Then use tag-based invalidation if supported.
        // Alternatively, maintain a tag->URL mapping in Redis and invalidate by URL.
        throw new UnsupportedOperationException("Tag-based invalidation requires CDN support");
    }
}
```

### Akamai Cache Invalidation via REST API

```java
@Service
public class AkamaiCacheService {

    @Autowired
    private RestTemplate akamaiRestTemplate;

    @Value("${akamai.base-url}")
    private String akamaiBaseUrl;

    public void purgeUrls(List<String> urls) {
        PurgeRequest request = PurgeRequest.builder()
            .action("remove")
            .type("arl")
            .domain("production")
            .objects(urls)
            .build();

        akamaiRestTemplate.postForEntity(
            akamaiBaseUrl + "/ccu/v3/invalidate/url/production",
            request,
            PurgeResponse.class
        );
    }

    public void purgeByCpCode(String cpCode) {
        PurgeRequest request = PurgeRequest.builder()
            .action("remove")
            .type("cpcode")
            .domain("production")
            .objects(List.of(cpCode))
            .build();

        akamaiRestTemplate.postForEntity(
            akamaiBaseUrl + "/ccu/v3/invalidate/cpcode/production",
            request,
            PurgeResponse.class
        );
    }
}
```

### Serving Different Content Based on User-Agent (Device Detection)

```java
@Controller
public class ContentController {

    public static final String CDN_URL = "https://d2s8k2f4g9.example.com";

    @GetMapping("/images/{imageName}")
    public String serveImage(@PathVariable String imageName,
                             @RequestHeader("User-Agent") String userAgent,
                             RedirectAttributes redirectAttributes) {
        String deviceType = detectDevice(userAgent);
        // CDN caches per device type by varying cache key
        String cdnPath = String.format("/%s/images/%s", deviceType, imageName);
        return "redirect:" + CDN_URL + cdnPath;
    }

    private String detectDevice(String userAgent) {
        if (userAgent.toLowerCase().contains("mobile")) {
            return "mobile";
        } else if (userAgent.toLowerCase().contains("tablet")) {
            return "tablet";
        }
        return "desktop";
    }
}
```

### CDN Signed URLs for Private Content

```java
@Service
public class SignedUrlService {

    @Value("${cdn.private-key-path}")
    private String privateKeyPath;

    @Value("${cdn.key-pair-id}")
    private String keyPairId;

    public String generateSignedUrl(String resourcePath, Duration expiration) {
        Date expirationDate = new Date(System.currentTimeMillis() + expiration.toMillis());

        try {
            PrivateKey privateKey = loadPrivateKey(privateKeyPath);

            String signedUrl = CloudFrontUrlSigner.getSignedURLWithCannedPolicy(
                CDN_URL + resourcePath,
                keyPairId,
                privateKey,
                expirationDate
            );
            return signedUrl;
        } catch (Exception e) {
            throw new RuntimeException("Failed to sign CDN URL", e);
        }
    }

    private PrivateKey loadPrivateKey(String path) throws Exception {
        try (InputStream is = new FileInputStream(path)) {
            return KeyFactory.getInstance("RSA")
                .generatePrivate(new PKCS8EncodedKeySpec(is.readAllBytes()));
        }
    }
}
```

### Cache Header Configuration in Spring Boot

```java
@Configuration
public class CacheHeaderConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/**")
            .addResourceLocations("classpath:/static/")
            .setCacheControl(CacheControl.maxAge(365, TimeUnit.DAYS)
                .cachePublic()
                .staleWhileRevalidate(30, TimeUnit.DAYS));
    }

    @Bean
    public FilterRegistrationBean<CacheHeaderFilter> cacheHeaderFilter() {
        FilterRegistrationBean<CacheHeaderFilter> bean = new FilterRegistrationBean<>();
        bean.setFilter(new CacheHeaderFilter());
        bean.addUrlPatterns("/api/public/*");
        bean.setOrder(1);
        return bean;
    }
}

class CacheHeaderFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletResponse httpResponse = (HttpServletResponse) response;

        chain.doFilter(request, response);

        if (httpResponse.getStatus() == 200) {
            httpResponse.setHeader("Cache-Control", "public, max-age=300, s-maxage=600");
            httpResponse.setHeader("CDN-Cache-Control", "max-age=600");
        }
    }
}
```

## 5. Real-World Scenarios

### E-Commerce Image Optimization
- Product images served via CDN with automatic compression and resizing.
- Responsive images: CDN resizes based on device width (Imgix, Cloudinary).
- WebP/AVIF format negotiation via Accept header.
- Cache TTL: 1 year for product images, 1 hour for banners.
- Pre-warming: New product images pushed to CDN before launch.

### Video Streaming (Netflix-style)
- Adaptive bitrate streaming (HLS/DASH) via CDN.
- Video segments cached at edge for popular content.
- Private CDN appliances (Open Connect) at ISP data centers.
- Predictive pre-positioning: popular content pushed during off-peak hours.

### API Acceleration
- Dynamic content cached at edge with short TTL (10-60 seconds).
- API responses compressed and minified.
- Custom cache key based on authentication status.
- Edge compute (CloudFront Functions, Lambda@Edge) for request transformation.

### Software Package Registry
- npm/Maven packages cached globally.
- Immutable artifacts: infinite TTL.
- Purging only for yanked or updated packages.
- Reduced latency for developers worldwide.

### Live Streaming
- Real-time video chunk caching at edge.
- Origin shielding to reduce transcoder load.
- WebSocket-like delivery via chunked transfer encoding.
- Multi-CDN for reliability: fallback between providers.

## 6. Performance

### CDN Performance Benchmarks

| Metric | Without CDN | With CDN | Improvement |
|--------|-------------|----------|-------------|
| Time to First Byte (TTFB) | 500ms (cross-region) | 30ms (edge) | 15x |
| Page Load Time | 4.2s | 1.1s | 3.8x |
| Origin Load | 100% of requests | 5-10% of requests | 10-20x reduction |
| Bandwidth Cost | $1000/TB | $80/TB (CDN egress) | 12x cheaper |

### Cache Hit Ratio Optimization

- **Longer TTL**: Static assets: 1 year, use content hashing in filenames (e.g., `style.a1b2c3.css`).
- **Stale-while-revalidate**: Serve stale cache while refreshing in background.
- **Prefetching**: Pre-load next likely pages (guess next product, next article).
- **Shield layer**: Origin shield aggregates cache misses from multiple edges.
- **Avoid cookie-based cache variance**: Hash cookies into cache key to limit cardinality.

### Cache Stampede Prevention
Multiple edge nodes simultaneously request the same content from origin when it expires.

Solutions:
- Origin shield (single aggregation point).
- Request collapsing: CDN merges concurrent requests for the same object.
- Staggered TTLs: Add jitter to TTL.
- Proactive revalidation: Refresh content before TTL expires (CloudFront + Lambda@Edge).

## 7. Security

### DDoS Protection
- CDN absorbs massive traffic at edge (Tbps scale).
- Web Application Firewall (WAF) at edge filters malicious requests.
- Rate limiting per IP at edge.
- Geo-blocking: deny traffic from specific regions.
- Challenge-based protection: CAPTCHA, JavaScript challenges.

### Origin Protection
- Origin accessible only from CDN IP ranges.
- Shared secret header between CDN and origin.
- Signed requests: CDN signs requests with HMAC, origin validates.
- IP allowlisting: only CDN egress IPs can reach origin.

### Private Content
- Signed URLs: Time-limited access to specific files.
- Signed Cookies: Session-based access to protected content.
- Token authentication: CDN validates a token before serving.
- Geo-restriction: IP-based access control at edge.

### CDN Configuration for Secure Headers

```java
// Spring Boot: Security headers for content behind CDN
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.headers(headers -> headers
            .contentSecurityPolicy(csp -> csp
                .policyDirectives("default-src 'self'; img-src 'self' https://*.cdn.com"))
            .referrerPolicy(referrer -> referrer
                .policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN))
            .permissionsPolicy(permissions -> permissions
                .policy("camera=(), microphone=(), geolocation=()"))
        );
        return http.build();
    }
}
```

## 8. Common Mistakes

### Mistake 1: Caching Dynamic Content Too Aggressively
```java
// WRONG - API key changes with environmental impact
@GetMapping("/api/user/{id}")
public User getUser(@PathVariable Long id) {
    // This API should probably not be cached at CDN
    return userService.getUser(id);
}

// RIGHT - Exclude authenticated/sensitive data from CDN cache
@GetMapping("/api/user/{id}")
public User getUser(@PathVariable Long id) {
    return userService.getUser(id);
    // Add Cache-Control: private or Cache-Control: no-store header
    // or vary by Authorization header
}
```

### Mistake 2: Not Versioning Static Assets
```html
<!-- WRONG - browser/CDN caches old version -->
<link rel="stylesheet" href="/css/style.css" />

<!-- RIGHT - content hash in filename -->
<link rel="stylesheet" href="/css/style.a1b2c3d4.css" />
```

### Mistake 3: Origin Behind CDN Without IP Restrictions
Anyone who discovers origin IP can bypass CDN (and its DDoS protection).

### Mistake 4: Wrong Cache Invalidation Strategy
Purging by URL pattern when the cache key includes query parameters that weren't purged.

### Mistake 5: Not Configuring Proper Cache Key
Cookies, query parameters, and headers can create thousands of cache variations, reducing hit ratio.

## 9. Senior Engineer Perspective

### Multi-CDN Strategy
Use multiple CDN providers to avoid single-provider failure and improve global coverage.

```java
@Component
public class MultiCdnRouter {

    private final List<CdnProvider> providers;
    private final HealthChecker healthChecker;

    @Value("${cdn.primary:cloudfront}")
    private String primaryCdn;

    public String getUrl(String path) {
        // Primary CDN
        for (CdnProvider provider : getHealthyProviders()) {
            if (provider.isHealthy()) {
                return provider.getUrl(path);
            }
        }
        // Fallback to origin (degraded)
        return originUrl + path;
    }

    private List<CdnProvider> getHealthyProviders() {
        return providers.stream()
            .filter(CdnProvider::isHealthy)
            .collect(Collectors.toList());
    }
}
```

### Edge Compute for Request Transformation
```java
// Lambda@Edge (Node.js example for reference)
// In Spring Boot, handle edge-side transforms via:
@Bean
public Function<Map<String, Object>, Map<String, Object>> edgeRedirect() {
    return request -> {
        String uri = (String) request.get("uri");
        String userAgent = (String) request.get("headers.user-agent");
        if (userAgent != null && userAgent.contains("Mobile")) {
            return Map.of("status", "302", "location", "/mobile" + uri);
        }
        return Map.of("status", "200", "body", "OK");
    };
}
```

### CDN Cost Optimization
- Compression: Enable gzip/brotli at edge.
- Image optimization: Auto-format WebP, resize, strip metadata.
- Tiered caching: Origin shield reduces origin requests.
- Pre-warming: Push content during off-peak for lower transfer costs.
- Cache key optimization: Fewer cache variations = higher hit ratio.
- Traffic shaping: Route high-value traffic to premium CDN, standard to budget.

## 10. Interview Questions (20: 10 easy + 10 medium)

### Easy

1. **Q:** What is a CDN?
   **A:** A Content Delivery Network is a distributed network of servers that delivers web content to users based on their geographic location, reducing latency and improving performance.

2. **Q:** How does a CDN reduce latency?
   **A:** By serving content from edge servers closest to the user, reducing network round-trip time.

3. **Q:** What types of content can a CDN deliver?
   **A:** Static files (images, CSS, JS), dynamic content (API responses), video streams, software downloads, and live streaming.

4. **Q:** What is the difference between cache hit and cache miss in a CDN?
   **A:** Cache hit serves content from CDN edge cache. Cache miss requires fetching from origin server.

5. **Q:** What is TTL in CDN caching?
   **A:** Time-To-Live determines how long content is cached at the edge before revalidating with the origin.

6. **Q:** Name three popular CDN providers.
   **A:** AWS CloudFront, Cloudflare, Akamai, Fastly, Google Cloud CDN.

7. **Q:** What is a CDN Point of Presence (PoP)?
   **A:** A physical data center location where CDN edge servers are deployed to cache and serve content.

8. **Q:** What is CDN cache purging?
   **A:** The process of forcibly invalidating cached content across all edge nodes.

9. **Q:** What is origin shielding?
   **A:** An intermediate cache layer between edge nodes and origin that aggregates cache misses, reducing origin load.

10. **Q:** How does CDN help with DDoS attacks?
    **A:** CDN distributes traffic across global edge network, absorbs volumetric attacks, and provides WAF/filtering at edge.

### Medium

11. **Q:** Explain the difference between DNS-based and Anycast-based CDN routing.
    **A:** DNS routing directs users via geographic DNS resolution; can be suboptimal due to DNS caching. Anycast advertises same IP from multiple PoPs; BGP routes to closest, providing automatic failover and optimal routing.

12. **Q:** How do you handle dynamic content caching in a CDN?
    **A:** Use short TTLs (10-60s), stale-while-revalidate, custom cache keys based on relevant parameters, and edge-side includes (ESI) for personalization fragments.

13. **Q:** What is cache key normalization and why is it important?
    **A:** Normalizing cache keys ensures multiple URLs for the same content resolve to one cache entry. Example: ignoring utm_* parameters, sorting query parameters, normalizing case.

14. **Q:** How do you implement CDN signed URLs?
    **A:** Generate a time-limited URL signed with a private key. CDN verifies signature before serving. Used for paid content, private downloads, or time-limited access.

15. **Q:** What is the difference between cache invalidation and cache TTL expiry?
    **A:** TTL expiry is automatic based on time. Invalidation is explicit: immediately removes content from cache regardless of TTL. Invalidation is slower (propagation delay) while TTL is predictable.

16. **Q:** How do you configure a CDN for an SPA (single page application)?
    **A:** Cache HTML with short TTL (or no-cache), cache JS/CSS with long TTL + content hash in filename. Serve index.html for all routes (error page fallback to SPA). Disable caching for /api/* routes.

17. **Q:** What is HTTP caching and how does it relate to CDNs?
    **A:** HTTP caching uses Cache-Control, ETag, and Last-Modified headers. CDNs respect these headers to decide caching behavior. The CDN acts as a shared HTTP cache.

18. **Q:** How do you handle CDN cache stampede?
    **A:** Use origin shield to consolidate miss requests, enable request collapsing (merge concurrent requests), add jitter to TTLs, and use stale-while-revalidate.

19. **Q:** What is the role of CDN in API gateway architecture?
    **A:** CDN caches API responses (GET), provides TLS termination, WAF protection, rate limiting, DDoS mitigation, and edge authentication (Lambda@Edge).

20. **Q:** How do you measure CDN performance?
    **A:** Cache hit ratio, TTFB (Time to First Byte), origin offload percentage, error rate, latency percentiles (P50, P95, P99), data transfer volume, and purge propagation time.

## 11. Advanced Interview Questions (20: 10 hard + 10 system design)

### Hard

1. **Q:** Design a CDN cache invalidation strategy for a large e-commerce platform with 10M+ products and frequent price changes.
    **A:** Use tag-based invalidation with surrogate keys. Each product page cached with Surrogate-Key: `product-{id}`, `category-{catId}`, `brand-{brandId}`. On price change, purge by `product-123` tag. On category price update, purge entire category. Use fast-purge API with async propagation. Batch purge requests. Maintain tag->URL mapping in Redis for custom CDNs. Use version-based keys for atomic switch.

2. **Q:** How do you implement a CDN-based geo-blocking strategy that respects privacy regulations (GDPR)?
    **A:** Use CDN geo-IP detection to determine user location. For GDPR: redirect EU users to EU-only origin, cache content separately per region (EU vs rest-of-world). Use Vary: CloudFront-Viewer-Country header in cache key. Apply different caching and WAF rules per region. Log only necessary data. Use edge compute to strip personal data from cached responses.

3. **Q:** Explain how you would implement origin failover across multiple cloud providers behind a CDN.
    **A:** Primary origin on AWS, secondary on GCP. CDN configured with health-check-based origin groups. If primary returns 5xx or timeout, CDN automatically switches to secondary. Use active-active (both origins serve) or active-passive. DNS-level failover as last resort. Database replication between providers for consistent content.

4. **Q:** Design a CDN caching strategy for a server-rendered React application.
    **A:** Cache rendered HTML at CDN with TTL 60s, stale-while-revalidate 600s. Cache key includes language, device type, AB-test variant. Invalidate on content publish via tag-based purge. Critical pages (homepage, top products) pre-warmed. For authenticated users, bypass CDN cache or use ESI (Edge Side Includes) for personalized fragments.

5. **Q:** How do you handle CDN cache poisoning attacks?
    **A:** Never parse or modify cache keys based on untrusted input. Use strict validation of origin response headers at CDN level. Implement response signing: origin signs responses, CDN verifies signature before caching. Validate Content-Type matches expected format. Use separate cache behaviors for user-uploaded vs application content.

6. **Q:** What is request collapsing and how does it affect origin load?
    **A:** When multiple edge nodes request the same content simultaneously after cache expiry, request collapsing merges them at the shield/origin so a single fetch occurs. This prevents cache stampede and reduces origin load. Trade-off: slightly higher latency for the first collapsed request.

7. **Q:** Design a CDN-based WebSocket termination strategy.
    **A:** Most CDNs don't support persistent WebSocket connections natively. Solutions: 1) Route WebSocket directly to origin (bypass CDN). 2) Use CDN with WebSocket support (Cloudflare, Fastly). 3) Terminate WebSocket at edge and proxy to origin via persistent connections. 4) Use HTTP/2 server push as WebSocket alternative behind CDN.

8. **Q:** How do you implement a "stale-while-revalidate" pattern for dynamic content via CDN?
    **A:** Set Cache-Control: stale-while-revalidate=86400. CDN serves stale content for up to 24h while refreshing in background. Origin receives the revalidation request and returns fresh content. If origin is down, CDN continues serving stale content (graceful degradation). For critical dynamic data, use short stale window.

9. **Q:** What are the challenges of serving personalized content through a CDN?
    **A:** Cache key explosion (each user creates unique cache entry), reduced hit ratio, privacy concerns (cached PII). Solutions: fragment caching with ESI, client-side personalization (API calls for user data), edge compute for lightweight personalization, cache key segmentation (cookie hash buckets).

10. **Q:** How do you perform A/B testing behind a CDN without invalidating the cache for all users?
    **A:** Use CloudFront Functions/Lambda@Edge to assign A/B test variant via cookies. Cache key includes experiment variant (e.g., `page.html;exp=control` vs `page.html;exp=variant`). This segments cache by experiment group. Use staged rollout: gradually increase variant percentage. Monitor cache hit ratio per variant group.

### System Design

11. **Q:** Design a global video streaming CDN architecture.
    **A:** Multi-tier: Edge PoP (last mile), Regional hub (transcoding, packaging), Origin (source content, DRM). Video segments cached at edge for popular content (LRU). Adaptive bitrate manifests generated at origin, served via CDN. Private CDN appliances (like Netflix Open Connect) at major ISPs. Pre-position popular content during off-peak. Multi-CDN for resilience.

12. **Q:** Design a CDN caching layer for a real-time sports scores API.
    **A:** Short TTL (5 seconds) for live scores. Use Server-Sent Events (SSE) or WebSocket for score updates pushed through CDN. Edge compute transforms and caches score feeds. Custom cache key: `score:{league}:{matchId}`. Write-through: on score change, CDN purge specific match cache. Aggressive connection keep-alive at edge.

13. **Q:** Design a CDN architecture for a global SaaS platform with multi-tenant isolation.
    **A:** Per-tenant CDN configurations: custom domain per tenant (tenant.example.com). Tenant-specific cache behaviors, WAF rules, SSL certificates. Origin groups based on tenant tier (free/pro/enterprise). Tenant isolation enforced at CDN via custom headers (X-Tenant-ID). Cache key includes tenant ID. Separate purge endpoints per tenant.

14. **Q:** Design a system for serving user-generated content via CDN with instant content moderation.
    **A:** Upload goes to origin processing pipeline (scan, moderate, transform). After processing, push to CDN storage (S3 + CloudFront). For moderated content: CDN URL points to placeholder image until moderation passes. Use feature flags to toggle visibility. Tag-based purge on moderation status change. Only cache after moderation approval to prevent caching of harmful content.

15. **Q:** Design a CDN-based software update distribution system (like apt/yum mirrors).
    **A:** Origin stores packages in cloud storage (S3/GCS). CDN distributes globally with infinite TTL for immutable packages. Metadata (package lists) have shorter TTL (5 min). Signed metadata for integrity. CDN handles download resumption (Range requests). Origin shield reduces load. Pre-seed new package versions to major PoPs during release.

16. **Q:** Design a CDN strategy for a live auction platform.
    **A:** Real-time bids: WebSocket direct to origin (bypass CDN). Static content (item images, descriptions): CDN with long TTL. Bid count/current price: CDN with 1s TTL (polling) or SSE via CDN. Auction countdown: edge-side timers using CloudFront Functions. Cache vary by auction status (open/closed). Post-auction results: CDN with long TTL after closure.

17. **Q:** Design a CDN-based image processing pipeline.
    **A:** Origin stores master images in S3. CDN with image optimization: on-demand resizing via edge compute or cloud functions (e.g., CloudFront + Lambda@Edge, Cloudflare Image Resizing). Request pattern: `cdn.example.com/images/product123.jpg?w=200&h=200`. CDN checks cache for exact dimensions, missing = fetch master, resize, cache result. Supported formats: WebP/AVIF based on Accept header.

18. **Q:** Design a multi-CDN failover system.
    **A:** DNS-based failover: Route53 with health checks. Primary CDN CNAME points to CloudFront, secondary to Cloudflare. Health check monitors CDN endpoint performance (latency, error rate). On failure: update DNS to point to secondary. Client-side: dynamic CDN selection via JavaScript (measure RTT to each CDN, pick fastest). Useperformance.now() for measurement.

19. **Q:** Design a CDN caching strategy for an API that returns different content based on user roles.
    **A:** Segment cache by role tier: anonymous, authenticated, admin. Cache key includes role hash. Use edge compute to read JWT and set cache key. For authenticated users: cache at CDN with short TTL (10s) for read APIs. For admin: bypass CDN entirely (admin always needs fresh data). Use surrogate keys for role-based invalidation.

20. **Q:** Design a global CDN deployment for a live streaming platform (like Twitch).
    **A:** Ingest: streamer pushes to nearest PoP. Transcoding: PoP transcodes to multiple bitrates. Distribution: edge PoPs serve viewers. Caching: short segments (2-4 seconds) cached at edge for replay. For truly live: low-latency HLS/CMAF with chunked encoding. Multi-CDN for viewers. Predictive pre-positioning for popular streamers. Origin shield reduces transcoder load.

## 12. Expert-Level Interview Questions (10: architect-level)

1. **Q:** Design a global CDN architecture with edge computing that supports dynamic content assembly (personalized pages) with 99.99% availability.
    **A:** Edge compute (CloudFront Functions + Lambda@Edge) handles per-request assembly. Base page template cached at CDN (TTL 300s). Personalization fragments fetched from regional Redis via edge compute. User profile data stored in edge-adjacent Redis (write-behind to central DB). Fragment assembly occurs at edge: fetch base template from cache, embed personalized fragments, return to user. This reduces origin load by 90% while serving personalized content. Availability via multi-region active-active deployment.

2. **Q:** How do you design a CDN-based real-time communication system with sub-100ms latency globally?
    **A:** WebRTC with CDN TURN servers at every PoP. Signaling via WebSocket through CDN with persistent connection optimization. Real-time messaging: CDN with SSE/WebSocket support (Cloudflare Durable Objects, Fastly Fanout). Media routing: mesh of PoP-to-PoP connections with intelligent routing (latency-based). Fallback: HTTP long-polling via CDN edge. Multi-provider: use lowest-latency CDN per user.

3. **Q:** Architect a CDN caching system for a global financial data platform that requires both low latency (<10ms) and strong consistency for price data.
    **A:** Three-tier: Hot data (price ticks) - no CDN cache, direct WebSocket to origin via optimized routing. Warm data (current day candles) - CDN with 1-second TTL, stale-while-revalidate 5s. Cold data (historical) - CDN with 24h TTL. Use CDN with strong consistency guarantees (CloudFront with origin shield). Origin publishes invalidation on every price change. Cache key includes timestamp for version differentiation. Circuit breaker: if price data staleness > 100ms, bypass CDN.

4. **Q:** Propose a CDN strategy for a platform that serves content from third-party origins (user-generated websites).
    **A:** Reverse proxy CDN: each user's custom domain points to CDN. CDN fetches content from user's origin. Multi-tenant isolation: per-customer WAF rules, rate limits, cache configuration. Edge compute transforms headers (security, compression). DDoS protection at CDN shields user origins. Auto-scaling origin group based on traffic. Cache optimizer: automated TTL suggestions based on content type and update frequency.

5. **Q:** Design a zero-trust CDN architecture for enterprise API delivery.
    **A:** CDN as zero-trust gateway: mutual TLS (mTLS) between CDN and client + CDN and origin. All traffic through CDN (no direct origin access). CDN validates JWT at edge before forwarding. WAF at edge with positive security model (allowlist). CDN rewrites URLs to hide internal paths. API key rotation automated via CDN configuration API. Audit logs: every request logged at edge for SIEM ingestion.

6. **Q:** How do you implement CDN-based origin offload for serverless applications (AWS Lambda)?
    **A:** API Gateway + CloudFront: Lambda invoked via API Gateway, responses cached at CloudFront. Reduce Lambda cold starts by using CloudFront Functions for lightweight request handling. Cache compute: use Lambda@Edge for cacheable compute (image resize, header rewrite). Warm Lambda via keep-alive and provisioned concurrency. Shield layer: CloudFront Origin Shield caches Lambda responses, reducing invocation count by 80%+.

7. **Q:** Design a CDN that supports HTTP/3, WebTransport, and server push for a next-generation web application.
    **A:** HTTP/3 (QUIC) at all CDN layers for reduced connection establishment time. WebTransport for low-latency bidirectional streaming through CDN. Server push: CDN pushes critical assets (CSS, JS, fonts) alongside HTML response. Priority-based push scheduling: push highest priority assets first. Connection coalescing: reuse QUIC connection for multiple domains. Fallback: HTTP/2 with multiplexing for browsers without HTTP/3.

8. **Q:** How would you design a CDN-based global load balancer with automatic failover and latency-based routing?
    **A:** Use CDN's global traffic management (GTM). Health checks from every PoP to every origin. Latency measurements: RTT from each PoP to each origin, stored in time-series DB. Dynamic steering: CDN routes user to origin with lowest latency based on PoP location. Failover: if origin fails health check from >50% of PoPs, removed from rotation. Load: spread traffic across healthy origins based on capacity. Use weighted random selection.

9. **Q:** Architect a CDN caching strategy for a GraphQL API.
    **A:** GET-based GraphQL queries cached at CDN (persisted queries). Cache key: query hash + variables + authentication state. POST queries go directly to origin (bypass cache for mutations). Automated persisted queries (APQ): clients send hash, CDN/edge compute resolves full query. Normalize cache keys: sort arguments, ignore introspection. Fragment caching: use @cacheControl directive to mark cacheable fields, edge compute caches fragments independently and assembles response.

10. **Q:** Design a CDN architecture that supports dynamic content caching with automatic invalidation based on database change data capture (CDC).
    **A:** CDC pipeline: Debezium captures DB changes, publishes to Kafka. Change events processed: extract entity ID, determine affected CDN cache tags. Send purge request to CDN API with affected tags. Use idempotent purge: identical purge requests within 5 min window are deduplicated. For high-frequency changes, use versioned cache keys: `page:123:v{version}`. Increment version in Redis on DB change. CDN serves `v1` until `v2` is populated.

## 13. Debugging & Troubleshooting

### Common CDN Issues

**Issue: Stale content served after update**
- Check purge propagation (some CDNs take minutes for global purge).
- Verify purge request was for correct URL/cache key.
- Check if content has different cache key (query params, cookies, headers).
- Check HTTP response headers: `X-Cache: Hit from cloudfront` (CloudFront).

**Issue: Mixed content warnings (HTTP vs HTTPS)**
- Ensure all URLs use HTTPS (protocol-relative URLs: `//cdn.example.com/image.jpg`).
- Configure CDN for HTTPS-only.
- Check for hardcoded HTTP URLs in JavaScript/CSS.

**Issue: CORS errors for CDN resources**
```java
@Configuration
public class CdnCorsConfig {

    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/static/**")
                    .allowedOrigins("https://app.example.com")
                    .allowedMethods("GET")
                    .allowCredentials(false)
                    .maxAge(86400); // Preflight cache
            }
        };
    }
}
```

**Issue: CDN returning 502/503 errors**
- Check origin health and accessibility.
- Check origin response timeout (CDN has short timeout, often 30s).
- Verify origin IP allowlisting includes CDN egress IPs.
- Check SSL certificate validity on origin (if using HTTPS).

**Issue: Low cache hit ratio**
- Check if cache key varies too much (session IDs, timestamps in URLs).
- Check TTL settings: too short for static content.
- Check for unexpected Cache-Control: private or no-store headers.
- Check for cookies in cache key that differ per user.

### Debugging Tools
```bash
# Check CDN response headers
curl -I https://cdn.example.com/style.css -H "User-Agent: Mozilla/5.0"
# Look for: x-cache, age, cf-cache-status, x-amz-cf-pop

# Trace CDN routing
curl -v https://cdn.example.com/image.jpg 2>&1 | grep -i "server\|x-cache\|cf-ray"

# Check DNS resolution
nslookup cdn.example.com
# Check for Anycast (same IP from different locations)
dig cdn.example.com

# Measure latency from different PoPs
curl -o /dev/null -s -w "TCP handshake: %{time_connect}s, TTFB: %{time_starttransfer}s, Total: %{time_total}s\n" https://cdn.example.com/image.jpg
```

## 14. Comparison Section

### CDN Providers

| Feature | CloudFront | Cloudflare | Akamai | Fastly |
|---------|------------|------------|--------|--------|
| PoPs | 600+ | 330+ | 4,100+ | 150+ |
| Pricing | Pay-as-you-go | Freemium | Enterprise-tier | Pay-as-you-go |
| Edge Compute | Lambda@Edge, CF Functions | Workers | EdgeWorkers | Compute@Edge |
| WAF | AWS WAF | Built-in | Kona Site Defender | Web Application Accelerator |
| Cache Invalidation | Free (1000/mo), $0.005/path after | Instant (unlimited free) | By request (price varies) | Instant (included) |
| Origin Shield | Yes | Tiered Cache | SureRoute | Request collapsing |
| HTTP/3 | Yes | Yes | Yes | Yes |
| Image Optimization | Lambda@Edge | Polish, Image Resizing | Image & Video Manager | Image Optimizer |

### CDN Cache Types

| Type | TTL | Invalidation | Use Case |
|------|-----|-------------|----------|
| Static (immutable) | 1 year | Never (new URL on change) | JS, CSS, fonts, images |
| Semi-static (mutable) | 1-24 hours | On content update | News articles, blog posts |
| Dynamic (API) | 10-60 seconds | Time-based | API responses, stock prices |
| Personalized | No cache | N/A | User dashboards, settings |
| Streaming | Segment duration | On new content | Video/audio chunks |

### CDN vs Edge Computing vs Origin

| Aspect | CDN Cache | Edge Compute | Origin Server |
|--------|-----------|--------------|---------------|
| Latency | 1-10ms | 1-10ms | 50-500ms |
| Compute | None | Limited (CPU/memory caps) | Full |
| State | In-memory cache only | Stateful (KV store, Durable Objects) | Full database |
| Capacity | Static file size limits | Code size + execution time | Scalable |
| Cost | Low per GB served | Per invocation | Higher per request |

## 15. Revision Notes

### Quick Recap
- **CDN distributes content to edge locations** nearest to users.
- **Cache hit ratio** is the primary performance metric (target >90%).
- **Cache key** determines uniqueness: URL + headers + cookies + query params.
- **TTL** should be long for static assets (1 year with content hashing), short for dynamic.
- **Invalidation** removes cached content: tag-based is more efficient than URL-based.
- **Origin shield** prevents cache stampede on origin.
- **Edge compute** enables request transformation, device detection, A/B testing.
- **Security**: DDoS protection, WAF, signed URLs, origin IP protection.

### Key Headers
```
Cache-Control: public, max-age=3600, s-maxage=86400, stale-while-revalidate=86400
Cache-Control: private (don't cache at CDN)
Cache-Control: no-store (never cache anywhere)
CDN-Cache-Control: max-age=600 (CDN-specific override)
Surrogate-Key: product-123 category-electronics brand-sony (Fastly)
X-Cache: Hit from cloudfront (verification header)
Age: 1234 (seconds since cached)
```

### Anti-Patterns to Avoid
- Caching authenticated/sensitive data without isolation.
- Not versioning static assets (cache forever issue).
- Single CDN provider without fallback.
- CDN as the only security layer (defense in depth).
- Over-invalidation (purging frequently = low hit ratio = high origin load).

## 16. Cheat Sheet

```
+-------------------------------------------------------------------+
|                    CDN (CONTENT DELIVERY NETWORK) CHEAT SHEET      |
+-------------------------------------------------------------------+
| ARCHITECTURE                                                       |
+-------------------------------------------------------------------+
| [User] -> DNS (Anycast) -> Edge PoP (closest)                      |
|   -> Cache HIT? -> Serve from edge                                 |
|   -> Cache MISS? -> Shield Layer -> Origin Server                  |
|                        |                                           |
|                   [Regional Hub] -> [Origin Shield] -> [Origin]    |
+-------------------------------------------------------------------+
| CACHE HEADERS                | USE CASE                            |
+------------------------------+--------------------------------------|
| Cache-Control: public,       | Static assets (JS/CSS/images)       |
|   max-age=31536000           | Versioned filenames                 |
| Cache-Control: public,       | Dynamic content, fresh every 5min   |
|   max-age=300, s-maxage=600  | Stale for 1 day                     |
| Cache-Control: stale-while-  | Content that can be slightly stale  |
|   revalidate=86400           | While revalidating in background    |
| Cache-Control: private       | User-specific content (no CDN)      |
| Cache-Control: no-store      | Sensitive data, banking, etc.       |
+------------------------------+--------------------------------------|
| CACHE STRATEGIES                                                   |
+-------------------------------------------------------------------|
| IMMUTABLE: hash in filename, TTL=1y, never invalidate             |
| MUTABLE: TTL=1h, tag-based purge on update                        |
| DYNAMIC: TTL=10-60s, stale-while-revalidate                       |
| PERSONALIZED: no CDN cache OR ESI fragments                       |
+-------------------------------------------------------------------+
| CACHE KEY COMPONENTS                                               |
+-------------------------------------------------------------------|
| URL (path + query params)            | Required                    |
| Host header                          | For multi-tenant            |
| Accept-Encoding                      | For compression variants    |
| Accept-Language                      | For i18n                    |
| User-Agent (device class)            | Responsive content          |
| CloudFront-Viewer-Country            | Geo-specific content        |
| Custom headers/cookies               | Auth, A/B segments          |
+-------------------------------------------------------------------+
| SECURITY                                                           |
+-------------------------------------------------------------------+
| Signed URLs: time-limited access to private content                |
| Signed Cookies: session-based private content access               |
| WAF Rules: SQL injection, XSS, rate limiting at edge              |
| Origin IP restriction: only allow CDN egress IPs                  |
| mTLS: mutual TLS between CDN and origin                           |
| Geo-blocking: restrict content by country                         |
| Token authentication: validate token before serving               |
+-------------------------------------------------------------------+
| PERFORMANCE METRICS                                                |
+-------------------------------------------------------------------+
| Cache Hit Ratio     | hits / (hits + misses)          target >90%  |
| Origin Offload      | % of traffic served from cache  target >95%  |
| TTFB                | Time to First Byte              target <50ms |
| P50/P95/P99 Latency | Median/Tail latency             varies       |
| Purge Propagation   | Time for global invalidation    target <10s  |
| Error Rate          | 5xx from CDN/origin             target <0.1% |
+-------------------------------------------------------------------+
| CDN PROVIDERS (COMMON)                                             |
+-------------------------------------------------------------------+
| AWS CloudFront | Deep AWS integration, Lambda@Edge                 |
| Cloudflare     | Integrated security, Workers, free tier           |
| Akamai         | Largest network, enterprise features              |
| Fastly         | Instant purge, Compute@Edge, VCL                  |
| GCP CDN        | GCP integration, low cost                         |
| Azure CDN      | Azure + Verizon/Akamai backend                    |
+-------------------------------------------------------------------+
```
