# Content Delivery Networks (CDN)

---

## Overview

- **Definition:** A CDN is a geographically distributed network of proxy servers that delivers internet content to users from edge locations closest to them.
- **Why It Exists:** Reduces latency, offloads origin servers, provides DDoS protection, and improves availability for static assets, dynamic content, video streams, and APIs.
- **Key Concepts:** **Edge Server / PoP** (physical cache location), **Origin Server** (authoritative source), **Cache Hit/Miss**, **TTL** (time-to-live), **Purge/Invalidation** (forced removal), **Origin Shield** (aggregation layer preventing stampede), **Anycast Routing** (same IP advertised from multiple PoPs).

---

## Core Concepts

- **Request Routing:** DNS-based routing resolves user to nearest edge based on resolver location. Anycast routing advertises the same IP from multiple PoPs; BGP naturally routes to the closest. Anycast provides optimal routing and fast failover.
- **Caching Hierarchy:** Edge Node → Regional Hub → Origin Shield → Origin Server. Each layer reduces load on the next.
- **Cache Key:** Default is full URL. Customize by ignoring query params (`?utm_*`), including headers (`Accept-Language`), or cookies. Poorly configured keys cause low hit ratio.
- **Content Purging:** Exact URI, tag-based (surrogate keys), directory, or wildcard. Tag-based is most efficient for selective invalidation.

```java
// CDN cache invalidation via CloudFront API
public void invalidatePaths(List<String> paths) {
    CreateInvalidationRequest request = CreateInvalidationRequest.builder()
        .distributionId(distributionId)
        .invalidationBatch(InvalidationBatch.builder()
            .paths(Paths.builder().items(paths).quantity(paths.size()).build())
            .callerReference(String.valueOf(System.currentTimeMillis()))
            .build())
        .build();
    cloudFrontClient.createInvalidation(request);
}
```

---

## Common Mistakes

- **Caching Dynamic Content Too Aggressively** — Authenticated/sensitive data should use `Cache-Control: private` or `no-store`.
- **Not Versioning Static Assets** — Content hash in filenames (`style.a1b2c3.css`) enables infinite TTL without stale issues.
- **Origin Behind CDN Without IP Restrictions** — Anyone discovering origin IP bypasses CDN protection.
- **Wrong Cache Key Configuration** — Cookies, query params, and headers can create thousands of variations, cratering hit ratio.

---

## Key Design Considerations

- **Cache Hit Ratio Optimization** — Long TTL for hashed static assets (1 year), stale-while-revalidate for dynamic content, origin shield to aggregate misses.
- **Multi-CDN Strategy** — Use multiple providers to avoid single-provider failure. DNS-based failover or client-side latency measurement for routing.
- **Edge Computing** — Lambda@Edge or CloudFront Functions for request transformation, device detection, A/B testing, and lightweight personalization without origin round-trip.
- **Security** — DDoS absorption at edge, WAF, signed URLs for private content, origin IP allowlisting only CDN egress ranges.
- **CDN Cost Optimization** — Enable compression, auto-format images (WebP), origin shield, pre-warm during off-peak, optimize cache key cardinality.

---

## Real-World Scenarios

### Scenario 1: Global E-Commerce with CDN
**Context:** A global e-commerce platform serves product images, CSS, JavaScript, and API responses to users in North America, Europe, and Asia. Without a CDN, a user in Tokyo loads product images from the US-East origin server, taking 800ms.

**Resolution:** Deploy a CDN (CloudFront/Akamai) with edge locations in Tokyo, Singapore, Frankfurt, and London. Static assets (images, CSS, JS) are cached with 1-year TTL using content-hashed filenames (`style.a1b2c3.css`). Product API responses are cached with 60s TTL using stale-while-revalidate. The Tokyo user now loads images from the Tokyo edge in 20ms. Origin bandwidth drops 80%.

```java
// Cache-Control headers for different content types
@Configuration
public class CacheHeaderConfig implements WebMvcConfigurer {
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/**")
            .addResourceLocations("classpath:/static/")
            .setCacheControl(CacheControl.maxAge(365, TimeUnit.DAYS)
                .cachePublic());
    }
}

// API response with short TTL and stale-while-revalidate
@GetMapping("/api/products/{sku}")
public ResponseEntity<Product> getProduct(@PathVariable String sku) {
    Product product = productService.findBySku(sku);
    CacheControl cc = CacheControl.maxAge(60, TimeUnit.SECONDS)
        .staleWhileRevalidate(300, TimeUnit.SECONDS)
        .cachePublic();
    return ResponseEntity.ok().cacheControl(cc).body(product);
}
```

### Scenario 2: Video Streaming with Origin Shield
**Context:** A video streaming platform has 10M daily active users. When a new episode of a popular show is released, millions of users request the same video simultaneously. The origin server gets overwhelmed by cache misses.

**Resolution:** Use origin shielding — an intermediate cache layer between edge nodes and the origin server. When the Tokyo edge misses, it requests from the Asia origin shield. The shield aggregates multiple misses for the same content into a single origin request (request collapsing). Without the shield, 100 edge nodes would each fetch the video from the origin, creating a stampede.

### Scenario 3: API Caching with Signed URLs
**Context:** A SaaS platform provides premium content (PDF reports, data exports) to paying customers. Content must be accessible only to authorized users for a limited time.

**Resolution:** Use CDN with signed URLs. When a user requests a report, the application generates a time-limited signed URL (valid for 1 hour) and redirects the user. The CDN validates the signature and serves the content from the edge if cached, or fetches from origin once. The origin server is protected because only authenticated, authorized users can generate valid signed URLs.

---

## Scenario-Based Questions

1. **Q: You're launching a global e-commerce site serving users in the US, Europe, and Asia. Product images take 2+ seconds to load for Asian users. How do you optimize?**
   - A: Deploy a CDN with edge locations near your user base. Configure origin shield per region (US origin, EU origin shield, Asia origin shield). Cache static assets with content hashing for infinite TTL. Cache product images aggressively (1-year TTL). Use Anycast routing so users automatically reach the nearest edge. Target: <100ms image load time for all regions.

2. **Q: Your e-commerce site changes product prices frequently. The CDN cache serves stale prices for up to 1 hour. Customers see old prices and are charged wrong amounts. How do you handle dynamic content caching?**
   - A: Use a short TTL (10-30s) for price-related API responses with stale-while-revalidate. Implement tag-based invalidation: set `Surrogate-Key: product-{id}` header on cached responses. On price change, the application sends a fast-purge request for that specific product tag. The CDN instantly invalidates only the affected product pages. For critical price accuracy, bypass CDN cache entirely for price endpoints with `Cache-Control: no-store`.

3. **Q: Your video platform releases a new episode, and millions of users watch simultaneously. The origin server gets slammed by cache misses. How do you prevent this?**
   - A: Request collapsing + origin shielding. Configure an origin shield layer that aggregates multiple misses for the same content. Pre-warm the CDN: before the episode release, push the content to all edge locations via CDN pre-fetch APIs. Use chunk-based caching — cache video in 10-second chunks, so the first chunk is fetched from origin, but subsequent chunks are served from the CDN as the user watches.

4. **Q: A competitor discovers your origin server IP and starts DDoSing it directly, bypassing the CDN. How do you protect the origin?**
   - A: Restrict origin server access to only CDN egress IP ranges. Use a security group/firewall rule that allows traffic only from the CDN provider's published IP ranges. Additionally: hide the origin behind a reverse proxy, use a different domain for the origin (not publicly resolvable), and enable DDoS protection (AWS Shield, Cloudflare) on the origin.

5. **Q: Your API responses include user-specific content (name, recommendations). If you cache these responses, users see each other's data. How do you handle personalization with a CDN?**
   - A: Never cache personalized responses at the CDN level. Use `Cache-Control: private` — this tells the CDN not to cache the response; it's cached only in the user's browser. For semi-personalized content (e.g., recommended products based on browsing history), use edge-side includes (ESI) — the CDN caches the common fragments and assembles personalized fragments at the edge.

6. **Q: Your multi-tenant SaaS application serves white-labeled content. Tenant A and Tenant B should never see each other's data. How do you configure CDN cache keys?**
   - A: Include the tenant ID in the cache key. Configure the CDN to use a custom cache key that includes a custom header (`X-Tenant-Id`) or the hostname (if each tenant has a subdomain). This ensures tenant A's content never serves to tenant B. Use tag-based invalidation with tenant prefix (`Surrogate-Key: tenant-A product-123`) for selective purging.

7. **Q: Your marketing team needs to invalidate the CDN cache for specific product pages after a price change. Invalidation takes 5-15 minutes to propagate globally. During that time, users see wrong prices. How do you reduce the invalidation window?**
   - A: Use short TTLs (10-30s) for price-sensitive content instead of relying on invalidation. This ensures fresh prices within 30s max. For instant consistency, serve prices from a client-side API call that bypasses CDN cache. For the product page HTML, use ESI to include a price fragment that's either not cached or has a very short TTL. Accept that CDN invalidation is eventually consistent — design your pricing model accordingly.

8. **Q: Your CDN bill is increasing 50% month-over-month. The cache hit ratio dropped from 95% to 60%. How do you diagnose and fix this?**
   - A: Check cache key configuration first. Common cause: a developer added a query parameter (e.g., `?sessionId=abc`) or header to the cache key, creating millions of unique cache entries. Remove high-cardinality parameters from the cache key. Check TTL configuration — if TTLs were accidentally reduced, restore them. Add origin shield to aggregate misses. Enable compression to reduce bandwidth costs.

9. **Q: Your application supports 20 languages. The same page in different languages has completely different cached versions at the CDN. How do you configure this efficiently?**
   - A: Include the `Accept-Language` header in the cache key. This creates one cache entry per language. For the 20 most common language variants, the cache hit ratio remains high because users in the same region typically request the same language. Use ESI for multi-language pages — cache the common layout and only the translated text fragments vary per language, reducing the total number of cache entries.

10. **Q: You're deploying a canary release where 5% of users see the new UI and 95% see the old UI. How do you prevent CDN caching from mixing old and new UI for different users?**
    - A: Include the experiment variant (A/B test ID) in the cache key. This creates separate cache entries for the old and new UI. For the 5% canary group, the cache hit ratio will be lower initially, but it will warm up as more canary users request the same variant. Use a sticky cookie for consistent user assignment — the cookie is part of the cache key.

---

## Interview Questions

1. **What is a CDN and how does it reduce latency?**
   - A: A Content Delivery Network is a geographically distributed network of proxy servers (edge locations/PoPs). It reduces latency by serving content from the edge closest to the user, minimizing network round-trip time from potentially hundreds of milliseconds to single-digit milliseconds.

2. **Explain the difference between DNS-based and Anycast-based CDN routing.**
   - A: DNS routing resolves different IPs per geographic region via geo-DNS — can be suboptimal due to DNS caching and resolution inaccuracies. Anycast advertises the same IP from multiple PoPs; BGP routing naturally directs traffic to the closest available PoP with automatic failover.

3. **What is origin shielding?**
   - A: An intermediate cache layer between edge nodes and the origin server. It aggregates cache misses from multiple edges — when 100 edges all miss, the shield makes a single request to the origin (request collapsing). Prevents cache stampede on the origin.

4. **How do you handle dynamic content caching in a CDN?**
   - A: Short TTLs (10-60s) for dynamic content, stale-while-revalidate for freshness, custom cache keys excluding irrelevant parameters, ESI for fragment caching of personalized sections. For truly dynamic content, use `Cache-Control: private` or `no-store`.

5. **What is a signed URL and when would you use it?**
   - A: A time-limited URL signed with a private key that grants temporary access to protected content. Used for: paid downloads, user-specific documents, video streaming access, temporary file sharing. The CDN validates the signature before serving the content.

6. **How do you implement cache invalidation for frequent price changes?**
   - A: Tag-based invalidation using surrogate keys (`Surrogate-Key: product-{id}`, `category-{catId}`). On price change, call the CDN's fast-purge API with the specific tags. For near-instant consistency, use very short TTLs (10-30s) and stale-while-revalidate instead of relying on purge.

7. **How does a CDN help with DDoS attacks?**
   - A: The CDN's distributed infrastructure absorbs volumetric attacks at Tbps scale across global edge nodes. WAF at the edge filters malicious requests (SQL injection, XSS). Rate limiting per IP throttles abusive traffic. The origin remains protected behind the CDN.

8. **What is request collapsing?**
   - A: When multiple edge nodes request the same content simultaneously after expiry, request collapsing at the origin shield merges them into a single origin fetch. Prevents cache stampede and reduces origin load. Essential for content with many concurrent viewers.

9. **How do you serve personalized content through a CDN?**
   - A: Fragment caching with ESI (Edge-Side Includes) — cache common layout, personalize fragments. Client-side personalization via API calls from the browser. Edge compute (Lambda@Edge, CloudFront Functions) for lightweight per-user transformations. Segment cache key by variant for A/B testing.

10. **What headers control CDN caching behavior?**
    - A: `Cache-Control: public/private/no-store`, `max-age` (browser TTL), `s-maxage` (CDN TTL overrides max-age), `stale-while-revalidate` (serve stale while refreshing), `CDN-Cache-Control` (provider-specific), `Surrogate-Key` (tags for invalidation). The `Age` header indicates seconds since cached.

---

## Developer Recommendations

- **Use content hashing in filenames for infinite TTL on static assets** — `style.a1b2c3.css` enables a 1-year `Cache-Control: max-age=31536000` because the filename changes when content changes. Without hashing, you either use short TTLs (hurting performance) or rely on cache invalidation (slow and unreliable). Modern build tools (Webpack, Vite) handle this automatically.

- **Restrict origin server access to CDN IP ranges only** — Anyone who discovers your origin IP can bypass the CDN and attack your server directly. Configure security groups/firewalls to allow traffic only from your CDN provider's published IP ranges. Use a separate origin domain that is not publicly resolvable. This is the most important security measure when using a CDN.

- **Remove high-cardinality query parameters from cache keys** — Query parameters like `?utm_source=facebook`, `?session=abc`, or `?_t=timestamp` create millions of unique cache entries, cratering your hit ratio. Configure the CDN to ignore these parameters in the cache key. Use a whitelist approach: only include parameters you explicitly need (like `?page=2` or `?lang=en`).

- **Use stale-while-revalidate for dynamic content** — Instead of choosing between stale content (long TTL) and origin load (short TTL), use `Cache-Control: max-age=60, stale-while-revalidate=3600`. Users get instant responses (stale content) while the CDN asynchronously refreshes the cache. The user never waits for the origin.

- **Implement tag-based cache invalidation for selective purging** — Setting `Surrogate-Key: product-123 category-electronics` on cached responses allows targeted invalidation. When a product price changes, purge by `product-123` instead of the entire cache. This is essential for e-commerce and content platforms with frequent updates.

- **Monitor cache hit ratio and origin bandwidth as critical metrics** — Cache hit ratio below 90% indicates misconfiguration. Use CDN provider analytics dashboards. Set up alerts for sudden drops in hit ratio. Track origin bandwidth savings — if the CDN isn't reducing origin load by 80%+, investigate cache key configuration or TTL settings.
