# Content Delivery Networks (CDN)

---

## Overview

- **Definition:** A CDN is a geographically distributed network of proxy servers that delivers internet content to users from edge locations closest to them.
- **Why It Exists:** Reduces latency, offloads origin servers, provides DDoS protection, and improves availability for static assets, dynamic content, video streams, and APIs.
- **Key Concepts:** **Edge Server / PoP** (physical cache location), **Origin Server** (authoritative source), **Cache Hit/Miss**, **TTL** (time-to-live), **Purge/Invalidation** (forced removal), **Origin Shield** (aggregation layer preventing stampede), **Anycast Routing** (same IP advertised from multiple PoPs).
- **CDN Tiers and Pricing Models** — CDNs charge by data transfer (egress), request count, and optional features (WAF, edge compute, image optimization). Pay-as-you-go for variable traffic; committed contracts for predictable volumes. Multi-CDN strategies use a primary CDN for most traffic with a secondary for failover or overflow during peaks.
- **Edge Computing vs Traditional CDN** — Traditional CDNs only cache and serve static content. Edge computing (CloudFront Functions, Lambda@Edge, Cloudflare Workers) allows executing code at the edge for request transformation, A/B testing, authentication, and dynamic assembly. Edge compute reduces origin round-trips for semi-dynamic content.

---

## Core Concepts

- **Request Routing:** DNS-based routing resolves user to nearest edge based on resolver location. Anycast routing advertises the same IP from multiple PoPs; BGP naturally routes to the closest. Anycast provides optimal routing and fast failover.
- **Caching Hierarchy:** Edge Node → Regional Hub → Origin Shield → Origin Server. Each layer reduces load on the next.
- **Cache Key:** Default is full URL. Customize by ignoring query params (`?utm_*`), including headers (`Accept-Language`), or cookies. Poorly configured keys cause low hit ratio.
- **Content Purging:** Exact URI, tag-based (surrogate keys), directory, or wildcard. Tag-based is most efficient for selective invalidation.
- **Cache Key Design** — Default cache key is the full URL. Optimize by: ignoring irrelevant query parameters (`?utm_*`, `?_t`), including relevant headers (`Accept-Language`, `Accept-Encoding`), and adding custom cache key prefixes for versioning. Poor cache key design creates thousands of unique entries, cratering the hit ratio.
- **Pre-warming** — Proactively populate CDN cache before expected traffic spikes (product launches, event ticketing, flash sales). Push content to all edge locations via CDN APIs. Monitor edge fill rate to verify pre-warming completed before the event starts. Pre-warm during off-peak hours to avoid origin load spikes.

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
  - **Why it looks correct:** The CDN caches the response and subsequent requests load faster, so aggressive caching seems like a performance win — the data leak only surfaces when User A sees User B's dashboard because the CDN cached an authenticated response.
- **Not Versioning Static Assets** — Content hash in filenames (`style.a1b2c3.css`) enables infinite TTL without stale issues.
  - **Why it looks correct:** `style.css` with a 1-year TTL caches perfectly and users load the site fast — the stale CSS only becomes visible when a deployment changes the layout but the CDN still serves `style.css` from the old version to users who previously visited the site.
- **Origin Behind CDN Without IP Restrictions** — Anyone discovering origin IP bypasses CDN protection.
  - **Why it looks correct:** The CDN is the public-facing endpoint, so the origin feels hidden behind it — the origin IP only gets discovered when a DNS leak or a public code repository accidentally exposes it, and then the DDoS hits the origin directly.
- **Wrong Cache Key Configuration** — Cookies, query params, and headers can create thousands of variations, cratering hit ratio.
  - **Why it looks correct:** Including `Accept-Language` or `?page=2` seems like the correct way to serve the right content per user — the hit ratio crater only becomes visible when 50,000 unique `?sessionId=*` values create 50,000 cache entries for the same page, each serving only one user.
- **Caching User-Specific Content at CDN Level** — Authenticated responses with user-specific data must use `Cache-Control: private` to prevent CDN caching. A misconfigured cache could serve User A's dashboard to User B, causing a data exposure incident.
  - **Why it looks correct:** The CDN caches all content to improve performance, so caching an API response containing user preferences seems like standard optimization — the data exposure only occurs when a user logs out and the next user sees the previous user's dashboard because the CDN served its cached copy.
- **Not Using Origin Shield** — Without origin shield, every edge node that misses simultaneously requests the origin, causing a stampede. Origin shield aggregates misses and reduces origin load by 10-100x.
  - **Why it looks correct:** Edge nodes request the origin individually, which seems like normal CDN behavior — the stampede only becomes visible when 200 edge nodes simultaneously miss on the same video file and the origin server gets 200 concurrent requests for a 2GB file.
- **No Compression at Edge** — Serving uncompressed content increases bandwidth costs and latency. Enable Brotli or Gzip compression at the CDN level. For images, enable automatic format conversion (WebP/AVIF) and resizing.
  - **Why it looks correct:** The origin server sends compressed responses, so compression at the CDN seems redundant — the bandwidth waste only becomes visible when the monthly CDN bill shows $50K in egress that could have been $15K with Brotli compression re-enabled after the last CDN configuration migration.

---

## Key Design Considerations

- **Cache Hit Ratio Optimization** — Long TTL for hashed static assets (1 year), stale-while-revalidate for dynamic content, origin shield to aggregate misses.
- **Multi-CDN Strategy** — Use multiple providers to avoid single-provider failure. DNS-based failover or client-side latency measurement for routing.
- **Edge Computing** — Lambda@Edge or CloudFront Functions for request transformation, device detection, A/B testing, and lightweight personalization without origin round-trip.
- **Security** — DDoS absorption at edge, WAF, signed URLs for private content, origin IP allowlisting only CDN egress ranges.
- **CDN Cost Optimization** — Enable compression, auto-format images (WebP), origin shield, pre-warm during off-peak, optimize cache key cardinality.
- **Multi-CDN Strategy** — Use multiple CDN providers to avoid single-provider failure and improve global coverage. DNS-based failover for backup; latency-based routing for active-active. Monitor each CDN's performance (TTFB, availability) separately. Trade-off: higher complexity and management overhead.
- **CDN for Dynamic API Acceleration** — CDNs can accelerate dynamic APIs through TCP optimizations (TLS 1.3, connection reuse, HTTP/2 multiplexing), edge caching with short TTLs and stale-while-revalidate, and origin offload via request collapsing. API responses with `Cache-Control: s-maxage=60` benefit from CDN-level caching even for dynamic content.

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

  - **Interview follow-up:** Your CDN serves product images with a 1-year TTL via content-hashed filenames. A product image needs to be updated (e.g., price tag overlay changes) but the filename stays the same because the image content hash hasn't changed — how do you force the CDN to serve the new image without breaking the cache for other unchanged images?

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

  - **Interview follow-up:** Tenant A uploads a new logo, and the CDN cache key includes `X-Tenant-Id`. The logo URL is `/static/logo.png` with content hashing — but another tenant uses the same image hash and gets Tenant A's logo served from the CDN cache. How do you prevent cross-tenant cache collisions for identical content?

7. **Q: Your marketing team needs to invalidate the CDN cache for specific product pages after a price change. Invalidation takes 5-15 minutes to propagate globally. During that time, users see wrong prices. How do you reduce the invalidation window?**
   - A: Use short TTLs (10-30s) for price-sensitive content instead of relying on invalidation. This ensures fresh prices within 30s max. For instant consistency, serve prices from a client-side API call that bypasses CDN cache. For the product page HTML, use ESI to include a price fragment that's either not cached or has a very short TTL. Accept that CDN invalidation is eventually consistent — design your pricing model accordingly.

8. **Q: Your CDN bill is increasing 50% month-over-month. The cache hit ratio dropped from 95% to 60%. How do you diagnose and fix this?**
   - A: Check cache key configuration first. Common cause: a developer added a query parameter (e.g., `?sessionId=abc`) or header to the cache key, creating millions of unique cache entries. Remove high-cardinality parameters from the cache key. Check TTL configuration — if TTLs were accidentally reduced, restore them. Add origin shield to aggregate misses. Enable compression to reduce bandwidth costs.

  - **Interview follow-up:** You identify that `?_t=timestamp` was added to the cache key, creating 1M unique entries for the same page. After removing it from the cache key, the hit ratio recovers — but now users with cached URLs containing `?_t=123` and `?_t=456` both hit the same cache entry. How do you ensure these existing browser-cached URLs all map to the same CDN cache entry?

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
  - **Production story:** A news website used 5-second TTLs for breaking news headlines to keep them fresh — the origin server handled 500 requests/second just for headline refreshes. Switching to `max-age=60, stale-while-revalidate=3600` dropped origin headline requests to 10/second, and users never saw stale headlines because the CDN refreshed asynchronously.

- **Implement tag-based cache invalidation for selective purging** — Setting `Surrogate-Key: product-123 category-electronics` on cached responses allows targeted invalidation. When a product price changes, purge by `product-123` instead of the entire cache. This is essential for e-commerce and content platforms with frequent updates.

- **Monitor cache hit ratio and origin bandwidth as critical metrics** — Cache hit ratio below 90% indicates misconfiguration. Use CDN provider analytics dashboards. Set up alerts for sudden drops in hit ratio. Track origin bandwidth savings — if the CDN isn't reducing origin load by 80%+, investigate cache key configuration or TTL settings.
  - **Production story:** A SaaS platform's CDN hit ratio dropped from 92% to 45% after a front-end deployment added `?v={buildTimestamp}` to every asset URL. The build timestamp changed with every deployment, so all cached assets became unique on each deploy. The fix was switching to content-hashed filenames instead of build timestamps, restoring the hit ratio within hours after the old cache entries expired.
- **Use surrogate keys for targeted cache invalidation** — Tag cached responses with `Surrogate-Key: product-{id} category-{catId}`. When content changes, purge by specific tags rather than entire cache. This enables surgical invalidation — changing one product price doesn't require purging all product pages. Essential for e-commerce and content platforms with frequent, targeted updates.
- **Implement CDN failover for high availability** — Configure health checks on the origin. If the origin is unhealthy, the CDN serves stale content from cache (stale-while-revalidate) or routes to a backup origin. For multi-CDN setups, use DNS failover (route to secondary CDN) or client-side failover (JavaScript measures latency and switches). Test failover scenarios regularly to ensure they work.
